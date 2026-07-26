# JWT 이메일 claim과 계정 잠금 블랙리스트

## 문제
계정 잠금 상태를 변경한 뒤 이미 발급된 access token도 차단해야 한다.

처음에는 `JwtAuthenticationFilter`에서 Redis blacklist key를 확인하려고 했지만, 현재 token payload에는 subject로 user id만 들어 있었다. Redis key 정책은 email 기반으로 잡혀 있었기 때문에 필터에서 email을 얻을 방법이 필요했다.

## 원인
기존 JWT payload는 다음 정보만 포함했다.

- `sub`: 사용자 id
- `type`: access 또는 refresh
- `iat`: 발급 시각
- `exp`: 만료 시각
- refresh token일 때만 `jti`

email 기반 key를 만들려면 매 요청마다 DB에서 user id로 사용자를 조회하거나, token에 email claim을 포함해야 했다.

## 해결
JWT 발급 시 access/refresh token 공통 payload에 `email` claim을 추가했다.

`JwtTokenProvider`에는 `getEmail(String token)`을 추가해 필터나 refresh token 처리 로직에서 email을 추출할 수 있게 했다.

`JwtAuthenticationFilter`는 access token 검증 후 다음 순서로 처리한다.

1. token에서 user id를 꺼낸다.
2. token에서 email을 꺼낸다.
3. `RedisKeyPolicy.BLACKLIST_LOCKED.generateKey(email)`로 blacklist key를 만든다.
4. Redis에 해당 key가 있으면 `LockedException`으로 인증을 차단한다.
5. blacklist가 없으면 `AuthService.getAuthentication(userId)`로 인증 객체를 만든다.

## 주의
JWT payload는 서명되어 위변조는 막지만 암호화된 것은 아니다.

따라서 email claim은 클라이언트가 볼 수 있다고 봐야 한다. email보다 민감한 정보, 권한 변경에 따라 자주 바뀌는 상태, 계정 잠금 여부 같은 값은 token에 넣지 않는 편이 낫다.

이번에는 blacklist key 생성을 위해 email을 넣었지만, 장기적으로는 다음 대안도 검토할 수 있다.

- Redis key를 user id 기반으로 변경
- access token에 `jti`를 넣고 jti 단위 blacklist 관리
- refresh token은 `jti` 기반 저장/rotation으로 관리

## 테스트로 확인한 것
- access token에 email claim이 들어간다.
- refresh token에도 email claim이 들어간다.
- `JwtTokenProvider.getEmail(token)`이 email을 반환한다.
- 잘못된 token에서 email을 꺼내면 `INVALID_TOKEN_EXCEPTION`이 발생한다.
- `JwtAuthenticationFilter` 생성자 의존성에 `RedisUtil`이 추가되어 필터 테스트도 mock 의존성을 맞췄다.

## 관련
- [[03-decisions/auth/계정 잠금 상태와 토큰 블랙리스트 정책]]
- [[02-knowledge/auth/인증인가와 JWT 정리]]
- [[02-knowledge/auth/JWT 액세스-리프레시 토큰 타입 검증]]
