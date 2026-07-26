---
tags: [auth, oauth2, cookie, refresh-token, deployment]
date: 2026-07-09
status: 임시 해결
---

# HTTP 배포에서 OAuth Refresh Token 쿠키가 누락된 원인

## 한 줄 요약
> 로컬에서는 정상 동작하던 OAuth 로그인이 HTTP 운영 도메인에서는 `Secure` Refresh Token
> 쿠키가 전송되지 않아 실패했고, 인증서 적용 전까지 쿠키의 `Secure` 속성을 임시로
> 제거했다.

## 배경 / 어떤 상황이었는가
`dev` 브랜치를 ECS에 배포한 뒤 `http://moduply.co.kr`에서 OAuth 로그인을 확인했다.
OAuth 성공 후 프론트는 `/api/auth/refresh`를 호출해 Access Token과 사용자 정보를
복원하도록 구현되어 있다.

배포 환경에서는 다음 로그가 반복되면서 로그인 상태가 만들어지지 않았다.

```text
POST /api/auth/refresh
[AuthException] 인증 토큰이 없습니다. details: {tokenType=refresh}
```

같은 코드는 `http://localhost`에서 정상 동작했기 때문에 OAuth Client ID나 Redis 저장
문제로 먼저 의심했지만, 실제 원인은 브라우저의 쿠키 전송 정책 차이였다.

## 문제 (구체적으로)
OAuth 성공 핸들러는 Access Token을 URL에 노출하지 않는다. 대신 다음 순서로 로그인을
완료한다.

```text
OAuth provider 인증 성공
→ OAuth2SuccessHandler가 Refresh Token을 Redis에 저장
→ REFRESH_TOKEN HttpOnly 쿠키 발급
→ /#/contents로 이동
→ 프론트가 POST /api/auth/refresh 호출
→ Refresh Token 검증 후 Access Token 발급
```

기존 쿠키에는 `Secure=true`가 설정되어 있었다.

```java
ResponseCookie.from("REFRESH_TOKEN", refreshToken)
    .httpOnly(true)
    .secure(true)
    .path("/")
    .sameSite("Lax")
    .build();
```

`Secure` 쿠키는 HTTPS 요청에서만 전송된다. 배포 주소는
`http://moduply.co.kr`이므로 브라우저가 `/api/auth/refresh` 요청에
`REFRESH_TOKEN`을 포함하지 않았고, 서버는 인증 근거가 없어 요청을 거절했다.

로컬에서 문제가 드러나지 않은 이유는 브라우저와 OAuth 제공자가 `localhost`를 개발
환경으로 특별 취급하기 때문이다. 따라서 `localhost`에서 성공했다는 사실만으로 공개
HTTP 도메인에서도 같은 쿠키 정책이 동작한다고 판단하면 안 된다.

## Google과 Kakao의 차이
- Google은 `localhost`를 제외한 공개 도메인의 OAuth Redirect URI에 HTTPS를 요구한다.
  따라서 인증서 적용 전에는 배포 환경에서 Google OAuth를 정상 운영할 수 없다.
- Kakao는 HTTP Redirect URI도 등록할 수 있다. 다만 애플리케이션의 Refresh Token
  쿠키가 HTTP 요청에 포함되어야 최종 로그인까지 완료된다.

OAuth 시작 URL과 callback URL도 구분해야 한다.

```text
OAuth 시작: /oauth2/authorization/{provider}
OAuth callback: /login/oauth2/code/{provider}
로그인 완료 화면: /#/contents
```

`/api/auth/refresh`의 토큰 누락 로그는 OAuth 로그인 화면으로 이동하는 단계의 CORS
문제가 아니라, OAuth 성공 후 애플리케이션 세션을 복원하는 단계에서 발생한 문제였다.

## 고민한 대안들

| 대안 | 장점 | 단점 | 판단 |
|---|---|---|---|
| 즉시 HTTPS와 인증서 적용 | Refresh Token을 암호화된 구간에서 안전하게 전달, Google OAuth 사용 가능 | 현재 배포 일정에서 바로 적용하기 어려움 | 최종 해결책 |
| Refresh Token 없이 `/refresh` 허용 | 별도 쿠키 없이 Access Token 발급 가능 | 인증 근거 없이 토큰을 발급하는 치명적인 인증 우회 | 기각 |
| OAuth 성공 시 Access Token을 URL에 전달 | HTTP 쿠키 문제를 우회 | 브라우저 기록, 로그, Referer 등에 토큰 노출 가능 | 기각 |
| 인증서 적용 전 `Secure` 임시 제거 | Kakao OAuth와 일반 로그인을 HTTP에서 검증 가능 | Refresh Token이 평문 HTTP 구간에 노출 | 임시 채택 |

## 임시 구현
쿠키 정책이 발급 경로마다 달라지면 첫 refresh 이후 다시 실패할 수 있다. 예를 들어 OAuth
성공 쿠키만 `Secure=false`여도 `/api/auth/refresh`가 회전된 토큰을 `Secure=true`로
내려주면 브라우저가 새 쿠키를 사용할 수 없다.

그래서 다음 네 경로에서 동일하게 `Secure` 설정을 제거했다.

1. 일반 로그인 Refresh Token 발급
2. OAuth 성공 Refresh Token 발급
3. `/api/auth/refresh`의 Refresh Token 회전
4. 로그아웃 Refresh Token 삭제 쿠키

현재 임시 형태는 다음과 같다.

```java
ResponseCookie newRefreshTokenCookie = ResponseCookie.from(
        "REFRESH_TOKEN",
        tokenRefreshDto.getRefreshToken()
    )
    .httpOnly(true)
    .path("/")
    .maxAge(refreshTokenExpirationMinutes * 60)
    .sameSite("Lax")
    .build();
```

`Secure`를 지정하지 않으면 기본값은 `false`이므로 HTTP 요청에도 쿠키가 포함된다.
`HttpOnly`는 유지해 JavaScript가 Refresh Token을 직접 읽지 못하도록 했다.

## 결과 / 검증 방법
- OAuth, 일반 로그인, refresh, 로그아웃 쿠키 테스트에서 `Secure=false`를 검증했다.
- 관련 인증 테스트를 통과했다.
- 전체 `./gradlew test` 실행 결과 `BUILD SUCCESSFUL`을 확인했다.
- 배포 환경에서는 브라우저 개발자 도구에서 다음 항목을 확인해야 한다.
  - OAuth callback 응답의 `Set-Cookie`
  - Application 탭의 `REFRESH_TOKEN` 저장 여부
  - `/api/auth/refresh` 요청의 `Cookie` 헤더 포함 여부

## 보안 영향과 후속 조치
현재 조치는 최종 운영 정책이 아니다. `Secure`가 없는 Refresh Token은 HTTP 네트워크
구간에서 탈취될 수 있다.

HTTPS 인증서 적용 후 반드시 다음 작업을 수행해야 한다.

- HTTP 요청을 HTTPS로 리다이렉트
- OAuth Redirect URI와 성공·실패 URL을 HTTPS 기준으로 변경
- Refresh Token 발급·회전·삭제 경로 모두에 `Secure=true` 복구
- 쿠키 테스트도 `Secure=true` 검증으로 복구
- Google/Kakao 개발자 콘솔에 HTTPS callback URI 등록

다음 수정에서는 쿠키 보안 설정을 환경변수로 분리하는 것이 안전하다.

```yaml
auth:
  cookie:
    secure: ${AUTH_COOKIE_SECURE:true}
```

운영 기본값은 반드시 `true`로 두고, 인증서가 없는 제한된 HTTP 테스트 환경에서만
명시적으로 `false`를 사용해야 한다.

## 배운 점 / 다음에 또 마주치면
- OAuth provider 인증 성공과 애플리케이션 로그인 완료는 같은 단계가 아니다.
- Refresh Token 쿠키 기반 구조에서는 callback 성공뿐 아니라 이후 `/refresh` 요청의
  쿠키 전송 여부까지 확인해야 한다.
- 쿠키 속성 변경은 최초 발급뿐 아니라 회전과 삭제 경로까지 일관되게 적용해야 한다.
- 로컬 환경의 보안 예외 때문에 운영 HTTP 환경의 문제가 가려질 수 있다.

## 관련 노트
- [[02-knowledge/auth/OAuth2 소셜 로그인 성공 처리 흐름]]
- [[03-decisions/auth/OAuth2 성공 후 토큰 전달 방식]]
- [[03-decisions/auth/Refresh Token 화이트리스트와 로그아웃 순서]]

## 참고 자료
- [Google OAuth 2.0 Web Server Applications](https://developers.google.com/identity/protocols/oauth2/web-server#uri-validation)
- [Kakao Redirect URI 설정](https://developers.kakao.com/docs/en/app-setting/app#redirect-uri)
