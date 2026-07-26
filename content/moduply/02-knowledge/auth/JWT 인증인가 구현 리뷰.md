# JWT 인증/인가 구현 리뷰

## 현재 구현 흐름
현재 인증/인가 구현은 다음 흐름을 가진다.

```text
로그인 요청
→ Spring Security formLogin
→ UserDetailsService 사용자 조회
→ 로그인 성공 시 JWT 발급
→ 이후 요청에서 JwtAuthenticationFilter가 Authorization 헤더 검사
→ access token 검증
→ Authentication 생성
→ SecurityContextHolder 저장
→ 컨트롤러/서비스 요청 처리
```

로그아웃은 다음 흐름으로 정리했다.

```text
POST /api/auth/sign-out
→ LogoutHandler에서 refresh token 쿠키 만료
→ SecurityContextHolder.clearContext()
→ HttpStatusReturningLogoutSuccessHandler가 204 No Content 반환
```

## Handler 역할 구분
### AuthenticationEntryPoint
인증이 실패했거나, 인증이 필요한 리소스에 인증 없이 접근했을 때 사용한다.

JWT 필터에서 토큰 검증 실패를 직접 catch하는 경우에도 같은 응답 포맷을 쓰기 위해 `AuthenticationEntryPoint.commence(...)`를 직접 호출할 수 있다.

### AccessDeniedHandler
인증은 되었지만 권한이 부족한 경우 사용한다.

예:
- `USER`가 `ADMIN` API 호출
- 인증 객체는 있지만 필요한 authority가 없음

### AuthenticationFailureHandler
로그인 실패 전용 핸들러다.

예:
- 이메일 없음
- 비밀번호 불일치
- 계정 잠김

로그인 실패 응답은 `AuthException`으로 변환한 뒤 `ErrorResponse.from(exception)`을 사용하도록 통일했다.

### LogoutHandler
로그아웃 처리 자체를 담당한다.

현재 단계의 책임:
- refresh token 쿠키 만료
- `SecurityContextHolder.clearContext()`

나중에 추가될 수 있는 책임:
- Redis refresh token 삭제
- access token blacklist 등록
- 사용자별 `logoutAt` 기록

### LogoutSuccessHandler
로그아웃 성공 응답을 담당한다.

현재 API 명세는 성공 시 응답 body가 필요 없으므로 커스텀 핸들러 대신 다음 기본 핸들러를 사용한다.

```java
new HttpStatusReturningLogoutSuccessHandler(HttpStatus.NO_CONTENT)
```

## Access Token을 서버에서 삭제할 수 없는 이유
JWT access token은 클라이언트가 들고 있는 문자열이다.

서버가 stateless하게 서명과 만료 시간만 검증하는 구조라면, 로그아웃 핸들러에서 access token 자체를 삭제할 수 없다.

진짜 무효화가 필요하면 서버 측 상태가 필요하다.

예:
- Redis blacklist
- 사용자별 `tokenVersion`
- 사용자별 `logoutAt`
- 권한 변경 시 `roleChangedAt`
- 계정 잠금 시 `lockedAt`

## Access Token payload 최소화 결정
처음에는 access token payload에 다음 정보를 넣었다.

```text
sub
type
iat
exp
email
role
locked
```

하지만 `email`, `role`, `locked`는 변경될 수 있다.

특히 `role`, `locked`를 token claim에 넣고 이후 요청에서 DB 조회 없이 신뢰하면 문제가 생긴다.

예:
- 사용자가 잠겨도 기존 access token 만료 전까지 계속 요청 가능
- 권한이 ADMIN에서 USER로 내려가도 기존 access token에는 ADMIN으로 남아 있을 수 있음

따라서 지금 단계에서는 access token payload를 최소화하기로 했다.

```text
sub
type
iat
exp
```

`sub`에는 사용자 ID만 넣고, 권한/잠금 상태는 인증 객체 생성 시 DB 또는 Redis에서 최신 상태를 조회하는 방향이 더 안전하다.

## 권한 변경/계정 잠금 즉시 반영 전략
선택지는 두 가지다.

### 1. Access Token을 서버에서 관리
Redis blacklist, token version, logoutAt 등을 사용한다.

장점:
- stateless JWT의 단점을 보완할 수 있음
- 로그아웃/권한 변경/계정 잠금 즉시 반영 가능

단점:
- Redis 의존성 증가
- 인증 요청마다 Redis 조회 필요
- 설계 범위가 커짐

### 2. Token에는 사용자 ID만 넣고 DB에서 최신 상태 조회
JWT는 사용자 식별과 위변조 검증에만 사용하고, 권한/잠금 여부는 DB 기준으로 판단한다.

장점:
- Redis 없이도 즉시 반영 가능
- token payload가 단순해짐
- 변경 가능한 사용자 정보를 token에 오래 들고 있지 않음

단점:
- 인증 요청마다 DB 조회 발생
- 트래픽이 커지면 캐시 전략 필요

현재 프로젝트 단계에서는 2번이 더 단순하고 안전하다고 판단했다.

## 앞으로의 구조 방향
`JwtTokenProvider`가 DB 조회까지 맡으면 책임이 커진다.

따라서 장기적으로는 역할을 나누는 것이 좋다.

```text
JwtTokenProvider
→ token 생성
→ token 검증
→ userId 추출

JwtAuthenticationService
→ userId로 사용자 조회
→ locked 검증
→ 권한 생성
→ Authentication 생성

JwtAuthenticationFilter
→ Authorization 헤더 추출
→ token 검증
→ userId 추출
→ JwtAuthenticationService 호출
→ SecurityContextHolder 저장
```

## getAuthentication 분리 결정
기존 구조는 `JwtTokenProvider.getAuthentication(token)`이 token claim을 읽고 바로 `Authentication`을 만들었다.

```text
JWT claim
→ email / role / locked 추출
→ UserDto 생성
→ ModuPlyUserDetails 생성
→ Authentication 생성
```

이 방식은 구현은 단순하지만, token 발급 시점의 값을 계속 사용한다는 문제가 있다.

예:
- token 발급 후 계정이 잠겨도 기존 token의 `locked=false`를 계속 믿을 수 있음
- token 발급 후 권한이 변경되어도 기존 token의 `role`을 계속 믿을 수 있음

따라서 `JwtTokenProvider`는 인증 객체를 만들지 않고, token에서 `userId`만 추출하도록 책임을 줄인다.

```java
UUID userId = jwtTokenProvider.getUserId(token);
Authentication authentication = jwtAuthenticationService.getAuthentication(userId);
SecurityContextHolder.getContext().setAuthentication(authentication);
```

이 구조의 의미는 다음과 같다.

```text
JWT
→ 사용자가 누구인지 식별하는 최소 정보만 제공

DB
→ 현재 요청 시점의 최신 사용자 상태 제공

SecurityContextHolder
→ 최신 상태로 만든 Authentication 저장
```

결과적으로 access token payload는 최소화하면서도, 각 요청의 `SecurityContextHolder`에는 최신 role/locked 상태가 들어간다.

## SecurityContextHolder 사용 기준
`SecurityContextHolder`에 담긴 principal은 현재 요청의 인증 주체 정보다.

사용해도 되는 정보:
- 현재 로그인한 사용자 ID
- email
- role
- locked 여부
- 인증/인가 판단에 필요한 최소 사용자 정보

다시 DB를 조회해야 하는 경우:
- 사용자 이름이나 프로필 이미지처럼 변경될 수 있고 응답 정확도가 중요한 정보
- 연관관계가 필요한 경우
- 도메인 규칙 판단에 사용자 엔티티 자체가 필요한 경우
- 트랜잭션 안에서 사용자 정보를 변경해야 하는 경우

즉 `SecurityContextHolder`는 인증 주체 정보를 빠르게 참조하기 위한 곳이지, 모든 사용자 조회를 대체하는 캐시가 아니다.

## 리뷰로 발견한 보완 포인트
### Claim 파싱 예외 처리
`getAuthentication()`에서 `ParseException`만 잡으면 부족하다.

다음 상황은 별도로 처리해야 한다.
- `sub`가 UUID 형식이 아님
- `role` claim이 없음
- `role` 값이 enum에 없음
- `locked` claim이 없음

payload 최소화 이후에는 `sub` 파싱 실패를 `AuthException(AuthErrorCode.INVALID_TOKEN_EXCEPTION)`으로 변환하면 된다.

### UserDetailsService 예외 타입
`loadUserByUsername()`에서 사용자 없음은 Spring Security 흐름상 `UsernameNotFoundException`이 자연스럽다.

도메인 서비스에서는 `UserException`이 적절하지만, 인증 Provider가 호출하는 `UserDetailsService`에서는 Spring Security 계약에 맞추는 편이 낫다.

### 로그인 요청 Content-Type
현재 `formLogin`은 기본적으로 form 요청에 맞다.

JSON 로그인 API를 원한다면 다음 중 하나를 고려해야 한다.
- 커스텀 `UsernamePasswordAuthenticationFilter`
- `AuthenticationConverter`
- 별도 로그인 컨트롤러와 `AuthenticationManager`

### Refresh Token 쿠키 보안
refresh token 쿠키는 최소한 다음 옵션을 고려해야 한다.

```text
HttpOnly
Secure
SameSite
Path
Max-Age
```

개발 환경에서는 `Secure`가 불편할 수 있지만, 운영 환경에서는 HTTPS 전제하에 활성화하는 것이 맞다.

## CSRF 토큰 발급 흐름
`CookieCsrfTokenRepository`를 사용해도 모든 응답에 자동으로 `XSRF-TOKEN` 쿠키가 내려가는 것은 아니다.

CSRF 토큰은 보통 다음 상황에서 생성되고 저장된다.

- CSRF 필터가 요청을 처리하면서 token이 필요해짐
- 컨트롤러에서 `CsrfToken`을 주입받고 `getToken()`을 호출함
- 직접 `CsrfTokenRepository.generateToken(...)` / `saveToken(...)`을 호출함

현재 로그인 엔드포인트는 CSRF ignore 대상이다.

```text
POST /api/auth/sign-in
→ CSRF 검사 제외
→ JwtLoginSuccessHandler 실행
→ REFRESH_TOKEN 쿠키 발급
```

이 흐름에서는 CSRF 필터가 token을 만들 기회가 없으므로 로그인 성공 시 직접 `XSRF-TOKEN` 쿠키를 발급하거나, 프론트가 별도 조회 엔드포인트를 호출해야 한다.

현재 보강한 흐름은 다음과 같다.

```text
GET /api/auth/csrf-token
→ CsrfToken 생성 보장
→ XSRF-TOKEN 쿠키 발급

POST /api/auth/sign-in
→ 로그인 성공
→ REFRESH_TOKEN 쿠키 발급
→ XSRF-TOKEN 쿠키도 함께 발급
```

`CsrfTokenRepository`는 `SecurityConfig` 내부 Bean으로 두면 `JwtLoginSuccessHandler`와 순환참조가 발생할 수 있다.

따라서 별도 `CsrfConfig`에서 Bean을 만들고, `SecurityConfig`와 `JwtLoginSuccessHandler`가 각각 같은 Bean을 주입받도록 분리했다.

## Refresh Token 재발급 임시 구현
새로고침 시 프론트가 `POST /api/auth/refresh`를 호출한다.

초기에는 이 엔드포인트가 없어서 다음 예외가 발생했다.

```text
NoResourceFoundException: No static resource api/auth/refresh
```

기본 로그인/새로고침 동작 확인을 위해 최소 refresh 엔드포인트를 추가했다.

현재 흐름:

```text
POST /api/auth/refresh
→ REFRESH_TOKEN 쿠키 읽기
→ refresh token JWT 검증
→ token에서 userId 추출
→ DB에서 최신 사용자 상태 조회
→ access token 재발급
→ JwtDto body 반환
```

현재는 refresh token 저장소가 없으므로 JWT 자체 검증만 수행한다.

정식 구현 시 추가할 것:
- Redis에 refresh token 저장
- refresh 요청 시 Redis에 저장된 token과 쿠키 token 비교
- refresh token rotation
- 로그아웃 시 Redis refresh token 삭제
- `/api/auth/refresh` CSRF ignore 제거
- 프론트가 `X-XSRF-TOKEN` 헤더를 포함해 refresh 요청하도록 변경

주의:
refresh token은 HttpOnly 쿠키로 자동 전송되므로 `POST /api/auth/refresh`는 CSRF 공격 표면이 될 수 있다. 현재 CSRF ignore는 동작 확인용 임시 설정이다.

## 테스트로 확인해야 할 것
- access token 생성 성공
- refresh token 생성 성공
- access token과 refresh token type 검증
- 잘못된 type token 거부
- 만료 token 거부
- 잘못된 `sub` claim 거부
- JWT 필터에서 유효 token이면 SecurityContext 저장
- JWT 필터에서 잘못된 token이면 401 응답
- 로그인 실패 핸들러가 예외별 에러 코드를 올바르게 반환
- 로그아웃 시 refresh token 삭제 쿠키가 내려가는지
- 로그아웃 시 SecurityContext가 정리되는지

## 관련
- [[02-knowledge/auth/인증인가와 JWT 정리]]
- [[02-knowledge/auth/회원가입 로그인 JWT 요청 흐름]]
- [[02-knowledge/auth/JWT 생성 공통화와 Builder 패턴]]
- [[02-knowledge/auth/JWT 액세스-리프레시 토큰 타입 검증]]
