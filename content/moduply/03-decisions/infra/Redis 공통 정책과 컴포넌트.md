# Redis 공통 정책과 컴포넌트

## 날짜
- 2026-06-27

## 배경
- 비밀번호 재발급 기능에서 Redis를 사용하게 되었다.
- 이후 refresh token, 세션성 데이터, 인증 보조 데이터에서도 Redis를 사용할 가능성이 높다.
- 도메인마다 Redis key와 TTL을 직접 문자열로 관리하면 중복과 정책 불일치가 생길 수 있다.
- Redis 접근 실패 시 어떤 방식으로 재시도하고 실패를 전파할지도 공통 기준이 필요했다.

## 결정
- Redis key prefix와 TTL은 `RedisKeyPolicy`에서 관리한다.
- Redis 접근은 `RedisUtil`로 모은다.
- 문자열 기반 값 저장은 `StringRedisTemplate` 사용을 기본 방향으로 둔다.
- 기존 `RedisTemplate<String, String>` 빈은 이미 주입해서 쓰는 코드가 있을 수 있어 당장은 유지한다.
- 대체 가능 여부를 확인한 뒤 `StringRedisTemplate`으로 통일할 수 있으면 기존 `RedisTemplate<String, String>` 빈 제거를 검토한다.

## RedisKeyPolicy
- Redis key prefix와 TTL을 한 곳에서 관리한다.
- 비밀번호 재발급 임시 비밀번호는 `PASSWORD_RESET` 정책을 사용한다.
- 임시 비밀번호 TTL은 3분이다.
- 실제 Redis key는 정책의 prefix와 식별자를 조합해 생성한다.

## RedisUtil
- Redis 읽기, 쓰기, 삭제 기능을 공통 유틸로 제공한다.
- 현재 주요 메서드는 다음과 같다.
  - `setDateExpire`: TTL이 있는 문자열 값 저장
  - `getData`: 문자열 값 조회
  - `deleteData`: key 삭제
- `getDate()`는 의미가 맞지 않아 `getData()`로 이름을 변경했다.

## Retry / Recover 정책
- `setDateExpire`, `getData`, `deleteData`에는 retry를 적용한다.
- Redis 일시 장애에는 짧게 재시도한다.
- `getData`는 retry 후에도 실패하면 fallback으로 다른 인증 수단을 사용하지 않는다.
- 특히 임시 비밀번호 인증에서는 Redis 장애 시 DB 비밀번호로 우회 로그인되면 보안 정책이 깨질 수 있다.
- 따라서 Redis 조회가 최종 실패하면 예외를 다시 던져 로그인 실패로 이어지게 한다.

## 현재 보류한 부분
- Redis 장애를 어느 수준까지 인증 실패로 볼지 팀 정책 확정이 필요하다.
- refresh token 저장소가 들어오면 `RedisKeyPolicy`에 별도 정책을 추가해야 한다.
- 기존 `RedisTemplate<String, String>` 빈을 실제로 사용하는 코드가 있는지 확인한 뒤 제거 여부를 결정한다.

## 관련 구현
- `RedisKeyPolicy`
- `RedisUtil`
- `RedisConfig`
- `ModuPlyUserDetailsService.loadUserByUsername`
- `AuthService.resetPassword`
