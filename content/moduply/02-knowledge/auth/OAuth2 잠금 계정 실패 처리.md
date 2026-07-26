---
tags: [auth, oauth2, spring-security, account-lock]
date: 2026-07-03
status: 해결됨
---

# OAuth2 잠금 계정 실패 처리

## 한 줄 요약
> OAuth2 로그인 중 잠금 계정을 만나면 `loadUser()`에서 `OAuth2AuthenticationException`을 던지고, `OAuth2FailureHandler`가 프론트 로그인 화면으로 에러 코드를 포함해 리다이렉트한다.

## 배경 / 어떤 상황이었는가
폼 로그인은 `UserDetailsService` 또는 인증 과정에서 잠금 계정을 감지하면 로그인 실패 응답이 프론트로 전달된다.
OAuth2 로그인은 흐름이 다르다. provider 인증이 끝난 뒤 `/login/oauth2/code/{provider}` 콜백으로 돌아오고, 그 다음 `OAuth2UserService.loadUser()`에서 우리 서비스 사용자 정보를 확인한다.

문제는 잠금 계정으로 카카오 로그인을 시도했을 때 로그인 실패 토글이 정상적으로 뜨지 않고, 브라우저가 OAuth2 콜백 URL 근처에 머무는 것처럼 보였다는 점이다.

## 문제
기존 구현은 이메일로 사용자를 찾은 뒤 곧바로 `user.isBlocked()`를 호출했다.

```java
User user = userRepository.findByEmail(email).orElse(null);

if (user.isBlocked()) {
  throw new OAuth2AuthenticationException("잠긴 계정입니다.");
}

if (user == null) {
  user = new User(email, "", name, Role.USER);
  ...
}
```

이 코드는 두 가지 문제가 있었다.

- 신규 OAuth2 사용자라면 `user == null`인데 먼저 `user.isBlocked()`를 호출해 `NullPointerException`이 발생할 수 있다.
- 잠금 계정 실패 사유를 문자열 예외로 던져서 `OAuth2FailureHandler`에서 우리 도메인 에러 코드(`auth03`)를 안정적으로 구분하기 어렵다.

## 해결
사용자가 존재할 때만 잠금 여부를 확인하도록 분기를 정리했다.
그리고 잠금 계정이면 `OAuth2Error`에 `AuthErrorCode.ACCOUNT_LOCKED_EXCEPTION` 정보를 담아 `OAuth2AuthenticationException`을 던지게 했다.

```java
User user = userRepository.findByEmail(email).orElse(null);

if (user == null) {
  user = new User(email, "", name, Role.USER);
  SocialAccount socialAccount = new SocialAccount(currentProvider, providerId, user);

  userRepository.save(user);
  socialAccountRepository.save(socialAccount);
} else {
  if (user.isBlocked()) {
    throw new OAuth2AuthenticationException(new OAuth2Error(
        AuthErrorCode.ACCOUNT_LOCKED_EXCEPTION.getCode(),
        AuthErrorCode.ACCOUNT_LOCKED_EXCEPTION.getMessage(),
        null
    ));
  }

  Optional<SocialAccount> existingAccount = socialAccountRepository.findByUserId(user.getId());
  ...
}
```

이렇게 하면 Spring Security OAuth2 로그인 흐름에서 인증 실패로 처리되고, 등록한 failure handler로 넘어간다.

## FailureHandler 처리
`OAuth2FailureHandler`는 `OAuth2AuthenticationException`이면 내부의 `OAuth2Error`에서 에러 코드를 꺼내고, 아니면 일반 로그인 실패 코드(`auth02`)로 처리한다.

```java
String errorCode = exception instanceof OAuth2AuthenticationException oauthException
    ? oauthException.getError().getErrorCode()
    : AuthErrorCode.BAD_CREDENTIALS_EXCEPTION.getCode();

log.warn("OAuth2 로그인 실패: uri={}, code={}, message={}",
    request.getRequestURI(), errorCode, exception.getMessage());

String separator = redirectUrl.contains("?") ? "&" : "?";
String targetUrl = redirectUrl + separator + "error="
    + URLEncoder.encode(errorCode, StandardCharsets.UTF_8);

getRedirectStrategy().sendRedirect(request, response, targetUrl);
```

현재 설정값은 해시 라우터 주소다.

```yaml
jwt:
  oauth-failure-url: http://localhost:8080/#/sign-in
```

그래서 잠금 계정이면 최종적으로 아래 주소로 이동한다.

```text
http://localhost:8080/#/sign-in?error=auth03
```

프론트가 당장 같은 “로그인 실패” 토글만 띄우더라도, QA나 개발 중에는 URL의 `auth03`과 서버 로그로 원인을 확인할 수 있다.

## 왜 `separator`를 계산했는가
설정된 redirect URL에 이미 query string이 있을 수도 있기 때문이다.

```java
String separator = redirectUrl.contains("?") ? "&" : "?";
```

예를 들어 설정값이 아래라면 `?`를 붙인다.

```text
http://localhost:8080/#/sign-in
-> http://localhost:8080/#/sign-in?error=auth03
```

이미 query가 있다면 `&`를 붙여야 URL이 깨지지 않는다.

```text
http://localhost:8080/#/sign-in?from=oauth
-> http://localhost:8080/#/sign-in?from=oauth&error=auth03
```

## 검증 방법
OAuth2 failure handler 단위 테스트를 추가했다.

```java
OAuth2Error oauth2Error = new OAuth2Error(
    AuthErrorCode.ACCOUNT_LOCKED_EXCEPTION.getCode(),
    AuthErrorCode.ACCOUNT_LOCKED_EXCEPTION.getMessage(),
    null
);
OAuth2AuthenticationException exception = new OAuth2AuthenticationException(oauth2Error);

handler.onAuthenticationFailure(request, response, exception);

assertThat(response.getRedirectedUrl())
    .isEqualTo("http://localhost:8080/#/sign-in?error=auth03");
```

실행한 테스트:

```bash
./gradlew test --tests com.team6.moduply.auth.oauth2.OAuth2FailureHandlerTest
```

결과는 `BUILD SUCCESSFUL`.

## 배운 점 / 다음에 또 마주치면
OAuth2 로그인은 provider 인증 성공과 우리 서비스 로그인 성공이 다르다.
카카오나 구글 인증이 성공해도, 우리 서비스의 계정 잠금/탈퇴/연동 충돌 정책에서 실패할 수 있다.

따라서 OAuth2에서는 아래 지점들을 분리해서 봐야 한다.

- `/oauth2/authorization/{provider}`: OAuth2 시작 URL
- `/login/oauth2/code/{provider}`: provider가 authorization code를 들고 돌아오는 callback URL
- `OAuth2UserService.loadUser()`: provider 사용자 정보를 우리 서비스 사용자와 연결하는 지점
- `OAuth2SuccessHandler`: 자체 JWT 발급과 성공 리다이렉트
- `OAuth2FailureHandler`: 우리 서비스 정책 실패까지 포함한 실패 리다이렉트

## 관련 노트
- [[02-knowledge/auth/OAuth2 소셜 로그인 성공 처리 흐름]]
- [[03-decisions/auth/OAuth2 성공 후 토큰 전달 방식]]
- [[03-decisions/auth/계정 잠금 상태와 토큰 블랙리스트 정책]]
