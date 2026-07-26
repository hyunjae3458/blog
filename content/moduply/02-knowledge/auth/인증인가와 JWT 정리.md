# 인증/인가와 JWT 정리

## 개요
인증, 인가, JWT 구현을 공부하면서 정리한 노트 묶음이다.

현재 프로젝트의 흐름은 다음 순서로 진행 중이다.

```text
회원가입
→ 로그인
→ JWT 발급
→ JWT 검증
→ SecurityContext 인증 정보 저장
→ 인증/인가가 필요한 요청 처리
→ 로그아웃 및 토큰 무효화
```

## 흐름 정리
- [[02-knowledge/auth/회원가입 로그인 JWT 요청 흐름]]
  - 회원가입, 로그인, JWT 발급, 인증 요청 처리 흐름
  - `SecurityContextHolder`, `UserDetailsService`, `JwtAuthenticationFilter` 역할 정리

## JWT 생성
- [[02-knowledge/auth/JWT 생성 공통화와 Builder 패턴]]
  - Access Token / Refresh Token 생성 공통화
  - `JWTClaimsSet.Builder` 사용 방식
  - token type별 claim 분리

## JWT 검증
- [[02-knowledge/auth/JWT 액세스-리프레시 토큰 타입 검증]]
  - access/refresh token type 검증 필요성
  - `validateToken(token, expectedType)` 구조
  - refresh token이 access token처럼 사용되는 문제 방지

## 구현 리뷰
- [[02-knowledge/auth/JWT 인증인가 구현 리뷰]]
  - `JwtAuthenticationFilter`, `AuthenticationEntryPoint`, `AccessDeniedHandler`, 로그인 실패 핸들러, 로그아웃 핸들러 역할 구분
  - access token payload 최소화 결정
  - `JwtTokenProvider.getAuthentication` 분리와 `JwtAuthenticationService` 위임 구조
  - 권한 변경/계정 잠금 즉시 반영 전략
  - 구현 리뷰에서 발견한 보완 포인트
- [[02-knowledge/test/spring-security/Spring Security 테스트 slice와 WebSocket]]
  - Spring Security 도입 후 `@WebMvcTest` 실패 원인
  - `JwtAuthenticationFilter`를 컨트롤러 slice 테스트에서 제외하는 이유
  - SockJS `/ws/info`와 WebSocket 인가 설정 문제
  - WebSocket STOMP CONNECT JWT 검증 필요성
- [[02-knowledge/test/spring-security/Spring Security 메서드 시큐리티 테스트]]
  - `@PreAuthorize`가 `@InjectMocks` 단위 테스트에서 검증되지 않는 이유
  - 컨트롤러 slice 테스트에 실제 `UserService` Bean을 import한 이유
  - 본인/ADMIN 허용 케이스, 권한 거부 케이스, CSRF 누락 케이스를 함께 테스트하는 방식

## 앞으로 추가할 내용
- Redis 기반 refresh token 저장/삭제 전략
- 계정 잠금/권한 변경 시 토큰 무효화 전략
- JSON 로그인 요청을 받을 경우의 커스텀 인증 필터 구조
- JWT 필터/핸들러 테스트 케이스 정리

## 관련
