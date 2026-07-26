---
tags: [auth, oauth2, transaction, spring-security]
date: 2026-07-03
status: 해결됨
---

# OAuth2 외부 API와 트랜잭션 분리

## 한 줄 요약
> OAuth2 provider user-info API 호출은 트랜잭션 밖에서 실행하고, 사용자/소셜 계정 DB 동기화만 짧은 트랜잭션으로 분리했다.

## 배경 / 어떤 상황이었는가
소셜 로그인에서는 `DefaultOAuth2UserService.super.loadUser(userRequest)`가 provider user-info endpoint를 호출한다.
즉, 카카오/구글 리소스 서버에 Access Token을 보내 사용자 정보를 받아오는 외부 API 호출이다.

기존 구조는 `ModuplyOAuth2UserService.loadUser()` 전체에 `@Transactional`이 붙어 있었다.

```java
@Override
@Transactional
public OAuth2User loadUser(OAuth2UserRequest userRequest) {
  OAuth2User oAuth2User = super.loadUser(userRequest);
  Map<String, Object> attributes = oAuth2User.getAttributes();

  User user = userRepository.findByEmail(email).orElse(null);
  ...

  return new ModuPlyUserDetails(userService.getUser(user.getId()), attributes);
}
```

## 문제
외부 OAuth2 provider 응답이 지연되면, 트랜잭션 경계가 불필요하게 넓어진다.

특히 트랜잭션 안에서 외부 API 호출과 DB 조회/저장을 같이 처리하면 다음 위험이 있다.

- 외부 API 지연 시간이 DB 트랜잭션 시간에 섞인다.
- 트랜잭션 안에서 커넥션을 획득한 뒤 외부 작업이 길어지면 커넥션 점유 시간이 늘어날 수 있다.
- 로그인 요청이 몰리면 커넥션 풀 고갈로 이어질 수 있다.
- OAuth2 provider 장애가 내부 DB 트랜잭션 처리 안정성까지 흔들 수 있다.

## 고민한 대안들

| 대안 | 장점 | 단점 | 판단 |
|---|---|---|---|
| `loadUser()` 전체 트랜잭션 유지 | 구현이 단순 | 외부 API 호출과 DB 트랜잭션이 한 경계에 묶임 | 기각 |
| sync 서비스에서 `UserDto` 반환 | presigned URL 포함 DTO를 바로 만들 수 있음 | DTO 생성/URL 생성까지 트랜잭션 안에 들어갈 수 있음 | 보류 |
| sync 서비스는 `userId`만 반환하고, 밖에서 `userService.getUser(userId)` 호출 | 외부 API, DB 동기화, DTO 조회 경계가 분리됨 | 조회가 한 번 더 발생할 수 있음 | 채택 |

## 최종 흐름

```text
ModuplyOAuth2UserService.loadUser()
1. super.loadUser()로 카카오/구글 user-info 조회       // 트랜잭션 밖
2. OAuth2UserSyncService.sync()로 사용자/소셜 계정 동기화 // 짧은 트랜잭션
3. userService.getUser(userId)로 UserDto 조회            // sync 트랜잭션 밖
4. ModuPlyUserDetails 생성
```

## 실제 구현

### OAuth2UserService는 외부 API 호출과 인증 객체 조립만 담당

```java
@Override
public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
  OAuth2User oAuth2User = super.loadUser(userRequest);
  Map<String, Object> attributes = oAuth2User.getAttributes();

  String provider = userRequest.getClientRegistration().getRegistrationId();
  OAuth2UserInfo oAuth2UserInfo = OAuth2UserInfoFactory.getOAuth2UserInfo(provider, attributes);

  UUID userId = oAuth2UserSyncService.sync(provider, oAuth2UserInfo);
  UserDto userDto = userService.getUser(userId);

  return new ModuPlyUserDetails(userDto, attributes);
}
```

`loadUser()`에는 더 이상 `@Transactional`을 붙이지 않는다.
외부 provider API 호출은 트랜잭션 밖에서 끝나고, 그 뒤 DB 동기화 서비스만 호출한다.

### DB 동기화는 별도 서비스의 짧은 트랜잭션으로 제한

```java
@Service
@RequiredArgsConstructor
public class OAuth2UserSyncService {

  private final UserRepository userRepository;
  private final SocialAccountRepository socialAccountRepository;

  @Transactional
  public UUID sync(String provider, OAuth2UserInfo oAuth2UserInfo) {
    Provider currentProvider = Provider.fromString(provider);
    String providerId = oAuth2UserInfo.getProviderId();
    String email = oAuth2UserInfo.getEmail();
    String name = oAuth2UserInfo.getNickname();

    User user = userRepository.findByEmail(email).orElse(null);

    // 가입된 사용자가 없으면 소셜 로그인 정보로 새 사용자와 소셜 계정을 함께 생성한다.
    if (user == null) {
      user = new User(email, "", name, Role.USER);
      SocialAccount socialAccount = new SocialAccount(currentProvider, providerId, user);

      userRepository.save(user);
      socialAccountRepository.save(socialAccount);
      return user.getId();
    }

    // 가입된 사용자가 잠금 상태이면 OAuth2 로그인 실패 흐름으로 넘긴다.
    if (user.isBlocked()) {
      oauth2Exception(AuthErrorCode.ACCOUNT_LOCKED_EXCEPTION);
    }

    Optional<SocialAccount> existingAccount = socialAccountRepository.findByUserId(user.getId());

    // 가입된 사용자는 있지만 소셜 계정 연동이 없으면 현재 provider로 연동한다.
    if (existingAccount.isEmpty()) {
      SocialAccount socialAccount = new SocialAccount(currentProvider, providerId, user);
      socialAccountRepository.save(socialAccount);
      return user.getId();
    }

    SocialAccount socialAccount = existingAccount.get();

    // 이미 다른 provider로 연동된 계정이면 OAuth2 로그인 실패 흐름으로 넘긴다.
    if (socialAccount.getProvider() != currentProvider) {
      oauth2Exception(AuthErrorCode.DUPLICATED_ACCOUNT_EXCEPTION);
    }

    // 같은 provider로 이미 연동된 계정이면 그대로 로그인한다.
    return user.getId();
  }
}
```

### OAuth2 예외 생성은 한 곳으로 모음

```java
private void oauth2Exception(AuthErrorCode errorCode) {
  throw new OAuth2AuthenticationException(new OAuth2Error(
      errorCode.getCode(),
      errorCode.getMessage(),
      null
  ));
}
```

호출부에서는 아래처럼 상황만 드러난다.

```java
if (user.isBlocked()) {
  oauth2Exception(AuthErrorCode.ACCOUNT_LOCKED_EXCEPTION);
}
```

## 왜 `UserDto`가 아니라 `UUID`를 반환했는가
OAuth 로그인 후 `ModuPlyUserDetails`에는 presigned URL이 포함된 `UserDto`가 필요하다.
단순 mapper로 DTO를 만들면 profile image URL이 채워지지 않는다.

처음에는 sync 서비스에서 `UserDto`를 반환하는 방식도 고려했다.
하지만 `userService.getUser()` 내부에서 presigned URL 생성 로직이 섞일 수 있으므로, 이 호출까지 sync 트랜잭션 안에 넣는 것은 다시 외부성 작업을 트랜잭션 안으로 끌어들이는 문제가 있다.

그래서 sync 서비스는 DB 상태를 맞춘 뒤 `userId`만 반환하고, `loadUser()`에서 트랜잭션 밖으로 돌아온 뒤 기존 `userService.getUser(userId)`를 호출한다.

```java
UUID userId = oAuth2UserSyncService.sync(provider, oAuth2UserInfo);
UserDto userDto = userService.getUser(userId);
```

이렇게 하면 presigned URL 포함 DTO 요구사항을 만족하면서도 OAuth2 provider API 호출과 DB sync 트랜잭션을 분리할 수 있다.

## 결과 / 검증 방법
OAuth2 관련 핸들러/팩토리 테스트와 애플리케이션 컨텍스트 로딩 테스트를 실행했다.

```bash
./gradlew test \
  --tests com.team6.moduply.auth.oauth2.OAuth2FailureHandlerTest \
  --tests com.team6.moduply.auth.oauth2.OAuth2SuccessHandlerTest \
  --tests com.team6.moduply.auth.oauth2.OAuth2UserInfoFactoryTest \
  --tests com.team6.moduply.Sb10ModuplyTeam6ApplicationTests.contextLoads
```

결과는 `BUILD SUCCESSFUL`.

## 배운 점 / 다음에 또 마주치면
서비스 메서드 하나에 `@Transactional`을 크게 거는 것은 편하지만, 그 안에 외부 API 호출이 섞이면 장애 전파 범위가 넓어진다.

OAuth2 로그인처럼 외부 provider와 내부 DB를 함께 다루는 흐름에서는 아래처럼 경계를 나누는 편이 낫다.

- 외부 API 호출: 트랜잭션 밖
- 내부 DB 상태 변경: 짧은 트랜잭션
- DTO 조회/외부 URL 생성: 필요하면 별도 read-only 흐름

## 관련 노트
- [[02-knowledge/auth/OAuth2 소셜 로그인 성공 처리 흐름]]
- [[02-knowledge/auth/OAuth2 잠금 계정 실패 처리]]
- [[03-decisions/auth/OAuth2 성공 후 토큰 전달 방식]]
