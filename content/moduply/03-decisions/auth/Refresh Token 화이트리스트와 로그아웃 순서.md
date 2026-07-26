# Refresh Token 화이트리스트와 로그아웃 순서

## 결정
refresh token 재발급은 Redis whitelist를 기준으로 검증하고, logout은 클라이언트 정리를 먼저 보장한 뒤 Redis 정리는 best-effort로 처리한다.

추가로, refresh token 갱신은 자바에서 `getData()` 후 `setDataExpire()`로 나누지 않고 Redis Lua 스크립트로 `조회 + 비교 + 갱신`을 원자적으로 처리한다.

## 이유
refresh token은 단순한 인증 토큰이 아니라 재발급의 기준이 되는 서버 상태와 연결되어 있다.
따라서 `REFRESH_TOKEN:{email}` 키에 저장된 값과 요청 토큰을 비교해 일치하지 않으면 재사용 또는 탈취 가능성으로 보고 즉시 차단해야 한다.

자바 레벨에서 `GET` 후 `SET`으로 나누면 동시 refresh 요청이 같은 현재값을 동시에 통과할 수 있다. 이때 마지막 write만 살아남고 먼저 발급된 새 refresh token이 즉시 무효화될 수 있다.
Lua 스크립트는 이 구간을 Redis 내부에서 한 번에 처리해 레이스 컨디션을 막는다.

logout은 반대로 사용자 경험과 세션 종료가 먼저다.
Redis 삭제가 먼저이고 그 과정에서 예외가 나면 쿠키 만료와 SecurityContext 정리까지 도달하지 못해 로그아웃 자체가 깨질 수 있다.

## 구현 의도
- refresh token 재발급
  - `jwtTokenProvider.validateRefreshToken()`으로 형식 검증
  - `jwtTokenProvider.getEmail(refreshToken)`으로 Redis key 계산
  - Redis Lua 스크립트로 현재값 일치 확인 + 새 토큰 저장 + 불일치 시 삭제를 원자적으로 처리
  - 결과가 `OK`가 아니면 재사용/오염으로 보고 예외 발생
  - 일치 시 새 AT/RT 발급 후 응답

- logout
  - refresh cookie 만료
  - SecurityContext 정리
  - Redis key 삭제는 후처리로 시도
  - Redis 삭제 실패는 사용자 응답을 막지 않음

## 테스트 기준
- refresh token 재발급 성공 시 Redis whitelist가 새 토큰으로 갱신된다.
- 저장된 refresh token이 다르면 Lua 스크립트가 `MISMATCH`를 반환하고 재사용 예외가 발생한다.
- 동일 refresh token의 동시 갱신 요청은 한 번만 성공해야 한다.
- logout 시 refresh cookie가 먼저 만료되고 SecurityContext가 비워진다.
- Redis 삭제 호출이 실패해도 클라이언트 세션 종료는 유지된다.

## 남은 고민
Redis 삭제 실패를 단순 로그로 둘지, 재시도 큐나 스케줄링 복구 대상으로 둘지 운영 정책이 필요하다.
보안적으로는 fail-closed가 맞지만, logout 같은 정리 작업은 사용자 응답을 깨지 않도록 분리해야 한다.
refresh token rotation 결과가 `MISSING`일 때도 별도 보안 이벤트로 볼지, 단순 만료로 볼지 후속 정리가 필요하다.

## 날짜
2026-06-30

## 관련
- [[03-decisions/auth/계정 잠금 상태와 토큰 블랙리스트 정책]]
- [[03-decisions/auth/보안 Redis 작업 동기 처리 결정]]
