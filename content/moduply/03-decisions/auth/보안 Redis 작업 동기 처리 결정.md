# 보안 Redis 작업 동기 처리 결정

## 결정
계정 잠금/잠금 해제, 비밀번호 변경 후 임시 비밀번호 파기처럼 보안 상태와 직접 연결되는 Redis 작업은 비동기 이벤트로 처리하지 않고 서비스 로직에서 동기로 처리한다.

## 이전 구조
기존 구조는 `UserService`가 이벤트를 발행하고, `UserSecurityEventListener`가 `@Async + @TransactionalEventListener(AFTER_COMMIT)`로 Redis 작업을 수행했다.

- 비밀번호 변경: `PasswordChangeEvent` 발행 후 임시 비밀번호 key 삭제
- 계정 잠금: `UserLockedEvent` 발행 후 `blacklist:locked:{email}` 저장
- 계정 잠금 해제: `UserUnlockedEvent` 발행 후 `blacklist:locked:{email}` 삭제

## 문제
이 구조는 보안 상태 불일치를 만들 수 있다.

`AFTER_COMMIT` 리스너는 DB 커밋 이후 실행된다. 따라서 Redis 작업이 실패해도 DB 변경은 이미 반영된 상태다.

또한 리스너가 예외를 잡고 로그만 남기면 호출자는 성공 응답을 받는다. 이 경우 다음 문제가 생긴다.

- 비밀번호는 변경됐지만 임시 비밀번호가 Redis에 남을 수 있다.
- DB는 계정 잠금 상태지만 Redis blacklist가 없어 기존 access token이 계속 통과할 수 있다.
- DB는 잠금 해제 상태지만 Redis blacklist가 남아 사용자가 계속 차단될 수 있다.

## 이유
이 작업들은 단순 캐시 갱신이 아니라 인증/인가 판단에 직접 영향을 준다.

따라서 Redis 실패를 숨기기보다 요청 실패로 전파하는 것이 맞다. 서비스 메서드가 트랜잭션 안에서 Redis 작업을 수행하면 Redis 예외가 발생했을 때 DB 변경도 롤백 대상이 된다.

## 구현 결과
`UserService`가 Redis를 직접 호출하도록 변경했다.

- `updateUserPassword()`
  - 새 비밀번호 인코딩 및 반영
  - `RedisKeyPolicy.PASSWORD_RESET.generateKey(email)` 삭제
- `updateUserLocked(locked=true)`
  - DB 잠금 상태 반영
  - `RedisKeyPolicy.BLACKLIST_LOCKED.generateKey(email)` 저장
- `updateUserLocked(locked=false)`
  - DB 잠금 해제 상태 반영
  - `RedisKeyPolicy.BLACKLIST_LOCKED.generateKey(email)` 삭제

삭제한 것:

- `UserSecurityEventListener`
- `PasswordChangeEvent`
- `UserLockedEvent`
- `UserUnlockedEvent`
- 사용자 보안 이벤트 전용 async executor

`RedisUtil.setDataExpire()`에는 recover 메서드를 추가해 재시도 실패 시 예외를 다시 던지도록 명시했다.

## 테스트 결과
테스트도 이벤트 발행 검증에서 Redis 동기 처리 검증으로 변경했다.

- 비밀번호 변경 성공 시 임시 비밀번호 key 삭제 검증
- 계정 잠금 성공 시 blacklist key 저장 검증
- 계정 잠금 해제 성공 시 blacklist key 삭제 검증
- Redis 삭제/저장 실패 시 서비스 메서드가 예외를 전파하는지 검증

실행한 테스트:

```bash
./gradlew test --tests 'com.team6.moduply.user.UserServiceTest' --tests 'com.team6.moduply.user.UserControllerMethodSecurityTest' --tests 'com.team6.moduply.common.config.AsyncConfigTest'
```

결과: 통과

## 한계
이 결정은 DB와 Redis를 완전한 분산 트랜잭션으로 묶는 것은 아니다.

서비스 트랜잭션 안에서 Redis 실패를 예외로 올려 DB 롤백 가능성을 확보한 것이며, Redis 작업 성공 후 DB 커밋 실패 같은 극단 상황까지 원자적으로 해결하지는 못한다.

더 강한 보장이 필요하면 다음 전략을 검토한다.

- outbox 패턴
- 보상 트랜잭션
- 주기적 reconciliation job
- Redis 장애 시 DB 상태 기반 fallback

## 추가 고민
`RedisUtil.getData()`는 조회 재시도 실패 시 `null`을 반환한다.

일반 캐시 조회에서는 허용 가능하지만, `JwtAuthenticationFilter`의 계정 잠금 blacklist 조회에서는 위험하다. Redis 장애로 `null`이 반환되면 잠긴 사용자가 아닌 것처럼 처리될 수 있기 때문이다.

따라서 보안 판단용 조회는 별도 strict 메서드로 분리해 Redis 조회 실패 시 인증을 중단하는 fail-closed 전략을 검토한다.

## 날짜
2026-06-29

## 관련
- [[03-decisions/auth/계정 잠금 상태와 토큰 블랙리스트 정책]]
- [[02-knowledge/auth/JWT 이메일 claim과 계정 잠금 블랙리스트]]
- [[03-decisions/infra/Redis 공통 정책과 컴포넌트]]
