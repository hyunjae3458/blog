---
tags: [auth, oauth2, jwt, decision]
date: 2026-07-02
status: 채택됨
---

# OAuth2 성공 후 토큰 전달 방식

## 결정
OAuth2 로그인 성공 후 Access Token과 Refresh Token을 리다이렉트 URL에 싣지 않는다.

대신 백엔드는 Refresh Token만 HttpOnly 쿠키로 내려주고, 프론트는 리다이렉트 후 `/api/auth/refresh`를 호출해 Access Token과 사용자 정보를 복원한다.

## 배경 / 왜 이 결정이 필요했는가
소셜 로그인 성공 후 아래처럼 토큰을 URL에 실으면 프론트에서 바로 로그인 상태를 만들 수 있다.

```text
http://localhost:8080/#/contents?accessToken=...&refreshToken=...
```

실제로 이 방식은 동작했다. 하지만 토큰이 URL에 남으면 브라우저 히스토리, 개발자 도구, referer, 프록시/서버 로그 등으로 노출될 수 있다.
특히 Refresh Token은 Access Token보다 수명이 길고 재발급 권한을 가지므로 URL에 싣는 것은 위험하다.

## 고려한 대안

| 대안 | 장점 | 단점 | 판단 |
|---|---|---|---|
| Access Token과 Refresh Token을 URL query/hash에 포함 | 구현이 단순하고 프론트가 즉시 저장 가능 | 토큰이 URL에 노출됨. Refresh Token 노출 시 피해가 큼 | 기각 |
| Access Token만 URL에 포함, Refresh Token은 쿠키 | Refresh Token 노출은 막음 | Access Token도 히스토리/로그에 남을 수 있음 | 기각 |
| Refresh Token HttpOnly 쿠키 + `/api/auth/refresh` 호출 | 토큰 노출 최소화, 기존 refresh 구조 재사용 가능 | OAuth 직후 네트워크 요청 1회 추가 | 채택 |
| 일회용 OAuth 교환 code 발급 후 프론트가 교환 | 가장 안전한 편, URL에는 일회용 code만 노출 | Redis code 저장/교환 API 추가 필요 | 추후 고도화 후보 |

## 채택한 흐름

```text
OAuth2 로그인 성공
-> OAuth2SuccessHandler
-> 자체 Access Token / Refresh Token 발급
-> Refresh Token Redis 화이트리스트 저장
-> REFRESH_TOKEN HttpOnly 쿠키 내려줌
-> http://localhost:8080/#/contents 리다이렉트
-> 프론트가 /api/auth/refresh 호출
-> JwtDto(accessToken, userDto) 저장
```

## 핵심 코드

```java
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

Access Token을 생성하더라도 리다이렉트 URL에는 포함하지 않는다.
프론트는 refresh 엔드포인트 응답으로 새 Access Token을 받는다.

## 트레이드오프 / 감수한 것
- OAuth 성공 직후 `/api/auth/refresh` 요청이 한 번 더 발생한다.
- 다만 직접 실행해본 결과 화면 이동에서 체감 지연은 거의 없었다.
- 로컬 HTTP 환경에서는 `secure(true)` 쿠키가 저장되지 않을 수 있으므로 환경별 쿠키 설정 분리는 필요하다.

## 재검토 시점
- 프론트에서 OAuth callback 전용 페이지를 만들 때
- 운영 HTTPS / 도메인 / SameSite 정책을 최종 확정할 때
- Refresh Token 재사용 탐지나 일회용 OAuth code 교환 방식이 필요해질 때

## 관련 노트
- [[02-knowledge/auth/OAuth2 소셜 로그인 성공 처리 흐름]]
- [[03-decisions/auth/Refresh Token 화이트리스트와 로그아웃 순서]]
