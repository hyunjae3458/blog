---
tags: [auth, csrf, spring-security, spa]
date: 2026-07-01
status: 해결됨
---

# SPA CSRF 토큰 발급과 XOR 핸들러

## 한 줄 요약
> Spring Security 6의 기본 CSRF 요청 핸들러는 토큰을 지연 생성하고 XOR 마스킹 흐름을 사용하기 때문에, SPA가 쿠키의 `XSRF-TOKEN`을 읽어 `X-XSRF-TOKEN` 헤더로 보내는 방식과 발급 타이밍이 어긋날 수 있다. SPA 전용 핸들러에서 `deferredCsrfToken.get()`을 호출해 토큰을 강제로 resolve하자 로그인 이후 쿠키 발급이 안정화됐다.

## 배경 / 어떤 상황이었는가
프론트는 SPA이고, CSRF는 쿠키 기반 방식으로 처리하려고 했다.

기대 흐름은 다음과 같았다.

```text
서버가 XSRF-TOKEN 쿠키 발급
프론트가 document.cookie에서 XSRF-TOKEN 읽음
프론트가 변경 요청 때 X-XSRF-TOKEN 헤더로 전송
Spring Security가 헤더 값을 검증
```

백엔드 설정은 `CookieCsrfTokenRepository.withHttpOnlyFalse()`를 사용했다.

```java
@Bean
public CsrfTokenRepository csrfTokenRepository() {
  return CookieCsrfTokenRepository.withHttpOnlyFalse();
}
```

로그인 성공 핸들러에서도 CSRF 토큰을 직접 생성하고 저장했다.

```java
CsrfToken csrfToken = csrfTokenRepository.generateToken(request);
csrfTokenRepository.saveToken(csrfToken, request, response);
```

그런데 로그인 응답 헤더에는 `Set-Cookie: XSRF-TOKEN=...`가 있는데, 브라우저 쿠키 저장소에서는 로그인 직후 토큰이 기대처럼 보이지 않는 현상이 있었다. 이후 SPA 전용 `CsrfTokenRequestHandler`를 추가하자 로그인 이후 CSRF 재발급이 정상 동작했다.

## 문제
- 응답 헤더에는 `XSRF-TOKEN` 쿠키가 포함되어 있었다.
- 하지만 로그인 이후 프론트에서 CSRF 토큰이 안정적으로 읽히지 않았다.
- Spring Security 기본 CSRF 처리 방식이 SPA의 쿠키-헤더 방식과 정확히 어떻게 맞물리는지 이해가 필요했다.

## 원인 분석
Spring Security 6에서는 CSRF 토큰을 매 요청마다 즉시 만들지 않고 지연 생성한다.

핵심은 `DeferredCsrfToken`이다.

```text
요청 진입
→ CSRF 토큰이 바로 만들어지는 것이 아님
→ 실제로 csrfToken.getToken() 또는 deferredCsrfToken.get()처럼 접근될 때 생성/저장됨
```

또 기본 요청 핸들러는 `XorCsrfTokenRequestAttributeHandler` 계열이다.

이 핸들러는 request attribute에 노출하는 토큰을 XOR 방식으로 마스킹한다. 서버 렌더링 폼에서는 유용하지만, SPA에서는 보통 쿠키에 담긴 평문 토큰을 읽어 헤더로 다시 보내는 방식이므로 흐름이 다르다.

즉 문제의 핵심은 두 가지였다.

```text
1. 토큰 생성이 지연됨
2. 기본 핸들러는 XOR 마스킹 토큰 흐름을 우선함
```

## 실제 구현

```java
public class SpaCsrfTokenRequestHandler implements CsrfTokenRequestHandler {
  private final CsrfTokenRequestHandler plain = new CsrfTokenRequestAttributeHandler();
  private final CsrfTokenRequestHandler delegate = new XorCsrfTokenRequestAttributeHandler();

  @Override
  public void handle(
      HttpServletRequest request,
      HttpServletResponse response,
      Supplier<CsrfToken> deferredCsrfToken
  ) {
    this.delegate.handle(request, response, deferredCsrfToken);
    deferredCsrfToken.get();
  }

  @Override
  public String resolveCsrfTokenValue(HttpServletRequest request, CsrfToken csrfToken) {
    if (StringUtils.hasText(request.getHeader(csrfToken.getHeaderName()))) {
      return plain.resolveCsrfTokenValue(request, csrfToken);
    }
    return this.delegate.resolveCsrfTokenValue(request, csrfToken);
  }
}
```

Security 설정에는 다음처럼 연결했다.

```java
csrf(csrf -> csrf
    .csrfTokenRepository(csrfTokenRepository)
    .csrfTokenRequestHandler(new SpaCsrfTokenRequestHandler())
)
```

## 코드가 의미하는 것

### 1. 기본 처리는 XOR 핸들러에게 맡김

```java
this.delegate.handle(request, response, deferredCsrfToken);
```

기본 Spring Security 흐름과의 호환성을 유지한다. request attribute에 노출되는 토큰은 XOR 핸들러가 처리한다.

### 2. 지연 토큰을 강제로 resolve

```java
deferredCsrfToken.get();
```

이 호출이 중요하다. 지연 상태였던 CSRF 토큰을 실제로 생성하게 만든다. `CookieCsrfTokenRepository`는 이 시점에 토큰을 쿠키로 저장할 수 있다.

이 호출을 넣은 뒤 로그인 이후 토큰 재발급이 잘 보이기 시작한 이유가 여기에 있다.

### 3. SPA 헤더 요청은 평문 토큰으로 검증

```java
if (StringUtils.hasText(request.getHeader(csrfToken.getHeaderName()))) {
  return plain.resolveCsrfTokenValue(request, csrfToken);
}
```

SPA는 쿠키에서 읽은 값을 그대로 `X-XSRF-TOKEN` 헤더에 넣는다. 이 값은 XOR 마스킹된 request attribute 토큰이 아니라 쿠키에 있던 평문 토큰이다. 그래서 헤더가 있으면 `CsrfTokenRequestAttributeHandler`를 통해 평문 토큰으로 해석하게 했다.

## 결과
- 로그인 이후 `XSRF-TOKEN` 쿠키가 안정적으로 발급/갱신되는 흐름을 확인했다.
- SPA가 쿠키의 `XSRF-TOKEN`을 읽어 `X-XSRF-TOKEN` 헤더로 보내는 방식과 Spring Security CSRF 검증 흐름을 맞췄다.

## 주의할 점
- `CookieCsrfTokenRepository.withHttpOnlyFalse()`는 프론트 JS가 쿠키를 읽기 위해 필요하다.
- cross-origin 환경에서는 프론트 요청에 `withCredentials: true` 또는 `credentials: "include"`가 필요하다.
- 서버 응답에는 `Access-Control-Allow-Credentials: true`와 정확한 Origin이 필요하다.
- 로그인, 로그아웃, refresh를 CSRF ignore로 둘지 여부는 보안 정책상 다시 검토해야 한다.

## 다음에 또 마주치면
- `Set-Cookie`가 응답에 있는데 브라우저 쿠키 저장소에 안 보이면 먼저 CORS, credentials, SameSite, Secure 속성을 확인한다.
- 응답 헤더 자체에 `XSRF-TOKEN`이 없다면 `deferredCsrfToken.get()`처럼 토큰을 실제로 resolve하는 코드가 있는지 확인한다.
- SPA에서는 기본 `XorCsrfTokenRequestAttributeHandler`만으로 충분하지 않을 수 있다는 점을 기억한다.
