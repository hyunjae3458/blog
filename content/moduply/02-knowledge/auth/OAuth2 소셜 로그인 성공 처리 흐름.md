---
tags: [auth, oauth2, spring-security, refresh-token]
date: 2026-07-02
status: 진행중
---

# OAuth2 소셜 로그인 성공 처리 흐름

## 한 줄 요약
> Google OAuth2 로그인 성공 후 자체 JWT를 발급하고, Refresh Token은 HttpOnly 쿠키와 Redis 화이트리스트에 저장한 뒤 프론트 해시 라우터 주소로 리다이렉트한다.

## 배경 / 어떤 상황이었는가
폼 로그인은 `JwtLoginSuccessHandler`에서 Access Token을 응답 바디/헤더로 주고, Refresh Token은 쿠키와 Redis에 저장하는 구조였다.
소셜 로그인은 Spring Security OAuth2 Login 흐름을 타기 때문에 별도 `OAuth2SuccessHandler`가 필요했다.

처음 확인한 문제는 소셜 로그인/회원가입 자체는 성공하지만, 마지막 리다이렉트 이후 화면 이동이나 로그인 상태 복원이 불안정하다는 점이었다.

## 전체 흐름
1. 사용자가 `/oauth2/authorization/google`로 진입한다.
2. Spring Security가 Google 인증 페이지로 302 리다이렉트한다.
3. Google 인증 성공 후 `/login/oauth2/code/google`로 authorization code가 돌아온다.
4. `ModuplyOAuth2UserService`가 provider 사용자 정보를 조회한다.
5. 이메일 기준으로 기존 회원을 찾거나 신규 회원과 `SocialAccount`를 생성한다.
6. `OAuth2SuccessHandler`가 자체 Access Token, Refresh Token을 발급한다.
7. Refresh Token을 Redis 화이트리스트에 저장하고 HttpOnly 쿠키로 내려준다.
8. 프론트 라우터 주소인 `http://localhost:8080/#/contents`로 리다이렉트한다.

## 실제 구현

### SecurityConfig에서 OAuth2 Login 연결

```java
.oauth2Login(oauth2 -> oauth2
    .userInfoEndpoint(userInfo -> userInfo
        .userService(moduplyOAuth2UserService))
    .successHandler(oAuth2SuccessHandler))
```

`userService`는 provider 사용자 정보를 우리 도메인 사용자로 연결하고, `successHandler`는 로그인 성공 후 자체 JWT 발급과 리다이렉트를 담당한다.

### OAuth2 사용자 로드 및 회원 생성

```java
OAuth2User oAuth2User = super.loadUser(userRequest);
Map<String, Object> attributes = oAuth2User.getAttributes();

String provider = userRequest.getClientRegistration().getRegistrationId();
Provider currentProvider = Provider.fromString(provider);

OAuth2UserInfo oAuth2UserInfo =
    OAuth2UserInfoFactory.getOAuth2UserInfo(provider, attributes);

String providerId = oAuth2UserInfo.getProviderId();
String email = oAuth2UserInfo.getEmail();
String name = oAuth2UserInfo.getNickname();

User user = userRepository.findByEmail(email).orElse(null);

if (user == null) {
  user = new User(email, "", name, Role.USER);
  SocialAccount socialAccount = new SocialAccount(currentProvider, providerId, user);

  userRepository.save(user);
  socialAccountRepository.save(socialAccount);
}
```

신규 사용자는 `users`와 `social_accounts`를 함께 생성한다. 기존 사용자는 `SocialAccount`가 없으면 연결하고, 이미 다른 provider가 연결되어 있으면 중복 계정 예외를 던진다.

### Provider별 attribute 추상화

```java
public class OAuth2UserInfoFactory {
  public static OAuth2UserInfo getOAuth2UserInfo(
      String provider,
      Map<String, Object> attributes
  ) {
    return switch (provider.toLowerCase()) {
      case "google" -> new GoogleUserInfo(attributes);
      case "kakao" -> new KakaoUserInfo(attributes);
      default -> throw new IllegalArgumentException(
          "지원하지 않는 소셜 로그인 공급자입니다: " + provider);
    };
  }
}
```

Google, Kakao는 사용자 정보 응답 구조가 다르기 때문에 `OAuth2UserInfo` 인터페이스로 provider별 파싱을 분리했다.

### OAuth2 성공 후 JWT 발급

```java
String accessToken = jwtTokenProvider.generateAccessToken(authentication);
String refreshToken = jwtTokenProvider.generateRefreshToken(authentication);

String redisKey = RedisKeyPolicy.REFRESH_TOKEN.generateKey(userDto.getEmail());
redisUtil.setDataExpire(redisKey, refreshToken, RedisKeyPolicy.REFRESH_TOKEN.getTtl());

ResponseCookie responseCookie = ResponseCookie.from("REFRESH_TOKEN", refreshToken)
    .httpOnly(true)
    .path("/")
    .secure(true)
    .maxAge(refreshTokenExpirationMinutes * 60)
    .build();

response.addHeader("Set-Cookie", responseCookie.toString());
getRedirectStrategy().sendRedirect(request, response, redirectUrl);
```

Access Token은 URL에 싣지 않고, Refresh Token만 HttpOnly 쿠키로 전달하는 방향으로 정리했다.
프론트는 OAuth 리다이렉트 후 `/api/auth/refresh`를 호출해서 Access Token과 `UserDto`를 복원한다.

## 리다이렉트 주소에서 주의할 점
현재 프론트는 해시 라우터를 사용하므로 리다이렉트 주소는 아래처럼 `#`를 포함해야 한다.

```yaml
jwt:
  frontend-url: http://localhost:8080/#/contents
```

`http://localhost:8080/contents`로 보내면 브라우저가 서버에 `/contents`를 직접 요청한다.
백엔드가 SPA fallback을 별도로 처리하지 않으면 404 또는 인증 오류가 날 수 있다.

## 확인한 내용
- `/oauth2/authorization/google`은 Google 인증 페이지로 302 리다이렉트한다.
- `/login/oauth2/code/google`은 provider가 code를 들고 돌아오는 callback URL이다.
- `/login/oauth2/authorization/google`은 OAuth 시작 URL이 아니므로 401이 발생한다.
- OAuth 성공 후 Refresh Token 쿠키 기반으로 `/api/auth/refresh`를 호출하는 방식은 체감 지연 없이 정상 동작하는 것을 확인했다.

## 남은 검토
- 로컬 HTTP 환경에서는 `secure(true)` 쿠키가 브라우저에 저장되지 않을 수 있으므로 local/prod 쿠키 설정 분리가 필요하다.
- OAuth 성공 응답에서 CSRF 쿠키까지 같이 내려줄지, 프론트가 `/api/auth/csrf-token`을 호출하게 할지 결정해야 한다.
- `ModuplyOAuth2UserService`가 `UserService.getUser()`를 호출해 presigned URL 생성까지 끌고 오는 구조는 추후 전용 mapper/factory로 분리할 여지가 있다.

## 관련 노트
- [[03-decisions/auth/OAuth2 성공 후 토큰 전달 방식]]
- [[02-knowledge/auth/SPA CSRF 토큰 발급과 XOR 핸들러]]
- [[03-decisions/auth/Refresh Token 화이트리스트와 로그아웃 순서]]
