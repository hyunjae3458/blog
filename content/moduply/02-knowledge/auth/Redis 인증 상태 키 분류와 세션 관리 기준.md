---
tags: [auth, redis, jwt, session, refresh-token, access-token, security]
date: 2026-07-13
status: 정리됨
---

# Redis 인증 상태 키 분류와 세션 관리 기준

## 한 줄 요약
> Redis에는 "현재 이 토큰/세션이 유효한가"처럼 빠르게 확인하고 만료되면 사라져도 되는
> 인증 상태를 저장하고, RDBMS에는 사용자 계정과 감사 로그처럼 영구 보존해야 하는
> 도메인 데이터를 저장한다.

## 배경 / 어떤 상황이었는가
현재 인증 구조에서는 JWT를 사용하지만, 완전한 stateless 구조만으로는 로그아웃,
권한 변경, 계정 잠금, Refresh Token 재발급, 중복 세션 방지 같은 요구사항을 안정적으로
처리하기 어렵다.

그래서 Redis에 다음 인증 보조 상태들을 저장하고 있다.

```text
- 잠금 이메일 블랙리스트
- 로그아웃된 Access Token 블랙리스트
- Refresh Token 화이트리스트
- Access Token 보조 상태
- Token Version
- 임시 비밀번호
```

임시 비밀번호는 인증 흐름에 Redis를 사용하긴 하지만, 세션/토큰 유효성 관리와 성격이
다르므로 이 노트에서는 별도 영역으로 분리한다.

## 왜 RDBMS가 아니라 Redis인가

### 1. TTL이 자연스럽다
인증 상태 데이터는 대부분 만료 시간이 있다.

```text
Access Token blacklist
→ Access Token의 남은 만료 시간까지만 필요

Refresh Token whitelist
→ Refresh Token 만료 시간까지만 필요

Session registry
→ 로그인 세션 만료 시간까지만 필요

Browser session mapping
→ 브라우저 식별 쿠키 또는 Refresh Token 만료 시간까지만 필요
```

Redis는 key 단위로 TTL을 걸 수 있다.

```text
SET auth:session:{sessionId} ... EX 604800
```

RDBMS로 관리하면 `expires_at` 컬럼, 만료 row 정리 batch, 인덱스 관리가 추가로 필요하다.
인증 상태처럼 만료되면 의미가 없어지는 데이터는 Redis TTL과 잘 맞는다.

### 2. 매 요청마다 조회하는 데이터다
JWT 인증 필터나 WebSocket interceptor에서는 거의 모든 요청마다 Redis를 확인할 수 있다.

```text
1. Access Token blacklist 확인
2. tokenVersion 확인
3. sessionId ACTIVE 확인
```

이 조회를 모두 RDBMS로 보내면 인증 트래픽이 DB 부하로 직결된다. Redis는 메모리 기반
key-value 조회에 최적화되어 있으므로 이런 인증 상태 확인에 적합하다.

### 3. 관계형 모델보다 key-value 모델이 자연스럽다
인증 상태는 복잡한 join보다 key로 바로 확인하는 경우가 많다.

```text
이 Access Token이 blacklist인가?
이 sessionId가 active인가?
이 userId + browserId의 현재 sessionId는 무엇인가?
이 userId의 현재 tokenVersion은 몇인가?
```

이런 데이터는 RDBMS 테이블보다 Redis key 설계가 단순하다.

### 4. 영구 보존 가치가 낮다
Redis에 둘 데이터는 현재 상태 확인용이다.

```text
현재 유효한 refresh token
현재 차단된 access token
현재 살아 있는 session
현재 tokenVersion
```

반대로 아래 데이터는 RDBMS가 맞다.

```text
사용자 계정
권한 변경 이력
로그인 이력
로그아웃 이력
관리자 조치 감사 로그
보안 이벤트 기록
```

정리하면 다음 기준이다.

```text
Redis = 지금 이 토큰/세션이 유효한가?
RDBMS = 누가 언제 무엇을 했는가?
```

## Redis 인증 키 분류

### 1. Access Token blacklist
로그아웃된 Access Token이 아직 만료되지 않았을 때 재사용을 막기 위한 키다.

```text
key: auth:blacklist:access:{accessTokenHash}
value: logout 또는 revoked
ttl: Access Token 남은 만료 시간
```

역할:

```text
- 특정 Access Token 하나를 차단한다.
- 로그아웃 직후, 기존 Access Token이 만료되기 전까지 재사용되는 것을 막는다.
- WebSocket CONNECT뿐 아니라 SEND/SUBSCRIBE 같은 후속 프레임에서도 확인해야 한다.
```

주의:

```text
Access Token 원문을 Redis key에 그대로 쓰지 말고 hash 값을 쓰는 것이 안전하다.
```

### 2. Refresh Token whitelist
Refresh Token 재발급 가능 여부를 검증하는 키다.

```text
key: auth:refresh:{userId 또는 email}
value: refreshToken 또는 refreshTokenHash
ttl: Refresh Token 만료 시간
```

역할:

```text
- 클라이언트가 보낸 Refresh Token이 서버가 인정한 최신 토큰인지 확인한다.
- 로그아웃 시 삭제한다.
- Refresh Token rotation을 사용하면 재발급 시 새 Refresh Token으로 교체한다.
```

주의:

```text
현재 value를 조회한 뒤 비교하고 새 값으로 갱신하는 흐름은 레이스 컨디션이 생길 수 있다.
Redis Lua script로 "조회 + 비교 + 갱신"을 원자적으로 처리하는 것이 안전하다.
```

관련 결정:

```text
[[03-decisions/auth/Refresh Token 화이트리스트와 로그아웃 순서]]
```

### 3. Token Version
권한 변경, 계정 잠금, 비밀번호 변경처럼 사용자 보안 상태가 바뀌었을 때 기존 토큰을
전체 무효화하기 위한 사용자 단위 kill switch다.

```text
key: auth:token-version:{userId}
value: number
ttl: 없음 또는 사용자 생명주기 기준
```

Access Token과 Refresh Token에는 발급 당시의 `tokenVersion` claim을 넣는다.

```text
JWT claim tokenVersion = 3
Redis auth:token-version:{userId} = 4
→ 불일치
→ 기존 토큰 차단
```

역할:

```text
- 권한 변경 시 기존 토큰 전체 무효화
- 계정 잠금 시 기존 토큰 전체 무효화
- 비밀번호 변경 시 기존 토큰 전체 무효화
```

세션 registry가 도입되면 tokenVersion이 완전히 불필요해 보일 수 있다. 하지만 둘은
무효화 단위가 다르다.

```text
sessionId
→ 특정 로그인 세션 단위 제어

tokenVersion
→ 사용자 전체 보안 상태 변경에 따른 전체 토큰 제어
```

권한 변경 시 모든 sessionId를 찾아 폐기해도 되지만, 그러려면 userId → sessionId 목록이
항상 정확해야 한다. tokenVersion은 세션 목록 누락에 대한 방어막으로도 쓸 수 있다.

### 4. 잠금 이메일 블랙리스트
계정 잠금 상태 사용자의 인증을 빠르게 차단하기 위한 키다.

```text
key: auth:blacklist:locked-email:{email}
value: locked
ttl: 정책에 따라 없음 또는 잠금 해제 전까지
```

역할:

```text
- 잠금 계정의 로그인 또는 인증 흐름을 빠르게 차단한다.
- JWT에 email claim이 필요했던 이유가 이 키 조회와 연결된다.
```

주의:

```text
이메일은 변경 가능성이 있는 식별자라 장기적으로는 userId 기반 key가 더 안정적이다.
다만 현재 구현이 email 기반이라면 Jwt claim에 email을 포함하거나 Redis key 정책을
userId 기반으로 변경해야 한다.
```

관련 노트:

```text
[[02-knowledge/auth/JWT 이메일 claim과 계정 잠금 블랙리스트]]
```

### 5. Session registry
중복 세션 방지와 로그인 세션 단위 제어를 위해 도입한 Redis 키다.

```text
key: auth:session:{sessionId}
value: ACTIVE 또는 REVOKED
ttl: 세션 만료 시간 또는 Refresh Token 만료 시간
```

역할:

```text
- Access Token의 sessionId claim이 현재 살아 있는 세션인지 확인한다.
- 로그아웃 시 해당 sessionId를 폐기한다.
- 같은 브라우저 재로그인 시 기존 sessionId를 폐기하고 새 sessionId만 active로 만든다.
- 특정 브라우저/특정 로그인 세션만 종료할 수 있다.
```

요청 검증 흐름:

```text
1. JWT 서명 검증
2. JWT 만료 검증
3. Access Token blacklist 확인
4. tokenVersion 확인
5. sessionId claim 추출
6. auth:session:{sessionId} 조회
7. status가 ACTIVE가 아니면 401
```

현재 구현에서는 세션 메타데이터 전체를 JSON 객체로 저장하지 않고 `ACTIVE` / `REVOKED`
상태만 저장한다. 요청 검증에 필요한 값은 JWT claim의 `sessionId`이고, 세션 소유자 정보는
Access Token의 `userId`, `email` claim에서 확인할 수 있기 때문이다. 초기 구현에서 저장
구조를 단순하게 가져가면 Redis 조회와 테스트 범위가 명확해진다.

핵심 구현:

```java
redisUtil.setDataExpire(
    sessionKey(sessionId),
    ACTIVE,
    RedisKeyPolicy.AUTH_SESSION.getTtl()
);
```

### 6. User-Browser active session index
"다른 브라우저 로그인은 허용하되, 같은 브라우저에서 다시 로그인하면 기존 세션을
무효화"하기 위한 인덱스다.

```text
key: auth:user-browser-session:{userId}:{browserId}
value: sessionId
ttl: browserId 쿠키 유지 기간 또는 Refresh Token 만료 시간
```

역할:

```text
- 같은 사용자라도 브라우저별로 현재 active session을 구분한다.
- 같은 브라우저에서 재로그인하면 기존 sessionId를 찾아 폐기한다.
- 다른 브라우저의 sessionId는 유지한다.
```

예시:

```text
Chrome browserId=A
auth:user-browser-session:1:A = session-111

Safari browserId=B
auth:user-browser-session:1:B = session-222
```

Chrome에서 다시 로그인하면:

```text
기존 auth:user-browser-session:1:A = session-111 조회
session-111 REVOKED
새 session-333 생성
auth:user-browser-session:1:A = session-333
```

Safari의 `session-222`는 유지된다.

### 7. User sessions index - 선택 사항
사용자 전체 세션을 한 번에 무효화하려면 userId 기준 sessionId 목록이 필요하다.

```text
key: auth:user-sessions:{userId}
value: Set<sessionId>
ttl: 각 session과 비슷한 기간 또는 cleanup 정책 필요
```

역할:

```text
- 관리자 권한 변경 시 전체 세션 폐기
- 계정 잠금 시 전체 세션 폐기
- 내 기기 목록 조회 기능 구현
```

단점:

```text
- 세션 생성/삭제 시 index도 함께 관리해야 한다.
- TTL과 cleanup 정책을 잘못 잡으면 죽은 sessionId가 Set에 남을 수 있다.
- tokenVersion이 있으면 전체 토큰 무효화에는 반드시 필요하지 않다.
```

따라서 초기 구현에서는 tokenVersion을 유지하고, user sessions index는 "내 로그인 기기
목록" 같은 기능이 필요할 때 추가하는 것이 현실적이다.

### 8. Browser ID 쿠키
브라우저 단위 세션 방지를 위해 클라이언트를 구분하는 식별자다.

```text
cookie: BROWSER_ID
value: UUID
HttpOnly: true
Secure: true
SameSite: Lax
Max-Age: 긴 기간
```

역할:

```text
- 같은 브라우저인지 구분하는 기준
- userId + browserId 조합으로 현재 sessionId를 찾는다.
```

주의:

```text
쿠키 삭제, 시크릿 모드, 다른 브라우저 사용 시 새로운 browserId가 된다.
브라우저를 완벽히 식별하는 것은 현실적으로 불가능하므로 쿠키 기반 식별을 정책으로
받아들여야 한다.
```

## 중복 세션 방지에서 Access Token을 세션 기준으로 쓰지 않는 이유
처음에는 "같은 브라우저에서는 같은 Access Token이 헤더에 담겨 오니까 Access Token별
세션을 기록하면 되지 않을까"라고 생각할 수 있다.

하지만 Access Token은 세션 식별자로 적합하지 않다.

```text
1. Access Token은 재발급될 때마다 값이 바뀐다.
2. 같은 로그인 세션이어도 Access Token은 여러 번 교체될 수 있다.
3. 여러 탭은 같은 Refresh Token/브라우저 쿠키를 공유한다.
4. Access Token을 세션 기준으로 삼으면 refresh 이후 기존 세션과 새 토큰의 연결이 꼬인다.
```

따라서 별도의 `sessionId`를 발급하고 JWT claim에 넣는 방식이 낫다.

```text
Access Token claims:
- userId
- email
- role
- tokenVersion
- sessionId
```

Refresh Token에도 같은 `sessionId`를 넣어야 refresh 재발급 시 현재 세션이 ACTIVE인지
검증할 수 있다.

## 실제 구현 후 트러블슈팅: 세션은 REVOKED 됐는데 화면이 로그아웃되지 않던 문제

### 증상
같은 브라우저에서 같은 계정으로 다시 로그인하면 Redis 상태는 기대한 대로 바뀌었다.

```text
1차 로그인
auth:user-browser-session:{userId}:{browserId} = session-111
auth:session:session-111 = ACTIVE

2차 로그인
auth:session:session-111 = REVOKED
auth:user-browser-session:{userId}:{browserId} = session-222
auth:session:session-222 = ACTIVE
```

그런데 1차 로그인 탭에서 다음 요청을 보내도 화면이 즉시 로그아웃되지 않았다.
백엔드에서는 인증 실패가 발생했지만, 프론트 화면은 로그인 상태처럼 유지되는 현상이 있었다.

### 처음 의심한 지점
처음에는 `JwtAuthenticationFilter`가 Access Token 안의 옛 `sessionId`가 아니라 새
`sessionId`를 보고 있는 것처럼 보였다. Access Token 값도 서로 달랐고, `browserId`는
같았기 때문에 세션 매핑이 꼬였다고 의심했다.

확인 결과 백엔드는 정상적으로 옛 Access Token의 `sessionId`를 읽고 있었다.

```java
String sessionId = jwtTokenProvider.getSessionId(token);

if (!authService.isSessionActive(sessionId)) {
  throw new BadCredentialsException(
      "로그인 세션이 유효하지 않습니다. 다시 로그인 해주세요.",
      new AuthException(AuthErrorCode.SESSION_REVOKED_EXCEPTION, Map.of(
          "userId", userId,
          "email", email,
          "sessionId", sessionId
      ))
  );
}
```

즉, 문제는 세션 검증 자체가 아니라 "401 응답 이후 프론트가 어떤 후속 행동을 하느냐"였다.

### 실제 원인
프론트의 공통 API interceptor는 일반적인 401 응답을 받으면 자동으로
`POST /api/auth/refresh`를 호출해 Access Token을 재발급하도록 되어 있었다.

이 구조는 만료된 Access Token에는 맞다.

```text
Access Token 만료
→ API 401
→ refresh token으로 새 Access Token 발급
→ 원래 요청 재시도
```

하지만 세션 폐기나 권한 변경 같은 보안성 401에는 맞지 않았다.

```text
같은 브라우저에서 2차 로그인
→ 1차 로그인 sessionId = REVOKED
→ 1차 탭에서 API 요청
→ 백엔드가 auth16 세션 폐기 401 반환
→ 프론트가 모든 401을 Access Token 만료처럼 취급
→ /api/auth/refresh 호출
→ 같은 브라우저의 Refresh Token 쿠키는 이미 최신 세션 기준으로 교체되어 있음
→ refresh 성공
→ 새 Access Token 저장
→ 1차 탭이 다시 로그인 상태처럼 살아남음
```

결국 "세션 폐기 401"과 "Access Token 만료 401"을 프론트에서 구분하지 못한 것이 핵심
원인이었다. 백엔드는 정상적으로 거절했지만, 프론트가 refresh로 다시 살려버린 것이다.

### 왜 계정 잠금은 로그아웃되는데 세션 폐기는 안 됐는가
계정 잠금은 프론트에서 별도 코드로 잡아 로그아웃 처리하고 있었다. 그래서 잠금 케이스는
다음 요청에서 로그아웃되는 것처럼 보였다.

반면 권한 변경, tokenVersion mismatch, 세션 폐기, 로그아웃 토큰 재사용은 모두 401이지만
프론트가 "refresh 가능한 401"로 처리하고 있었다.

### 수정 방향
백엔드 인증 실패를 더 구체적인 에러 코드로 분리했다.

```text
auth15 = TOKEN_VERSION_MISMATCH_EXCEPTION
auth16 = SESSION_REVOKED_EXCEPTION
auth17 = BLACKLISTED_TOKEN_EXCEPTION
```

그리고 프론트 interceptor에서 이 코드들은 refresh를 시도하지 않고 즉시 signOut 처리하도록
수정했다.

```javascript
if (["auth03", "auth15", "auth16", "auth17"].includes(errorBody?.code)
    && !config?.url?.includes("/auth/")) {
  await authStore.getState().signOut();
  return Promise.reject(error);
}

if (response?.status === 401 && !config?.url?.includes("/auth/") && !config._retry) {
  // 일반 Access Token 만료만 refresh 시도
}
```

주의할 점은 현재 수정 위치가 `src/main/resources/static/assets/index-BT-MLJSc.js`라는
빌드된 정적 번들이라는 점이다. 프론트 소스가 따로 관리된다면 같은 로직을 원본 API
interceptor에 반영해야 한다. 그렇지 않으면 프론트 재빌드 시 이 수정은 덮어써질 수 있다.

### 최종 흐름

```text
1. 같은 브라우저에서 재로그인
2. AuthSessionService가 기존 sessionId를 REVOKED 처리
3. 새 sessionId를 ACTIVE로 저장
4. 이전 탭이 옛 Access Token으로 API 요청
5. JwtAuthenticationFilter가 token 안의 sessionId를 Redis에서 조회
6. auth:session:{oldSessionId} = REVOKED
7. 백엔드가 auth16 401 반환
8. 프론트 interceptor가 auth16을 refresh 금지 에러로 판단
9. signOut 실행
10. 이전 탭 로그아웃 처리
```

### 검증
로컬 QA에서 Redis를 비운 뒤 다음 순서로 확인했다.

```text
1. 브라우저 A에서 계정 로그인
2. 같은 브라우저 A의 다른 탭/화면에서 같은 계정 재로그인
3. Redis에서 기존 auth:session:{oldSessionId} = REVOKED 확인
4. 첫 번째 탭에서 API 요청
5. 백엔드가 auth16을 반환하고 프론트가 refresh 없이 signOut 처리하는지 확인
```

테스트 코드도 인증 실패 코드에 맞춰 정리했다.

```text
invalid access token -> auth06
revoked session -> auth16
blacklisted access token -> auth17
```

전체 테스트는 `./gradlew test`로 통과를 확인했다.

### 배운 점
- 모든 401이 같은 의미는 아니다. Access Token 만료 401은 refresh 대상이지만, 세션 폐기,
  권한 변경, 계정 잠금, 로그아웃 토큰 재사용은 refresh하면 안 된다.
- JWT 기반 구조라도 서버 상태를 보는 순간 백엔드와 프론트의 에러 계약이 중요해진다.
- 백엔드에서 세션을 정확히 REVOKE해도 프론트가 refresh 정책을 잘못 잡으면 보안 정책이
  UX 레이어에서 우회될 수 있다.
- `sessionId` 기반 중복 세션 방지는 백엔드 검증만으로 끝나지 않고, 클라이언트의 401 처리
  정책까지 함께 설계해야 한다.

## Redis 키 역할 분리 요약

| 분류 | 무효화 단위 | 주요 목적 | TTL |
|---|---|---|---|
| Access Token blacklist | 특정 Access Token | 로그아웃 토큰 재사용 차단 | Access Token 남은 TTL |
| Refresh Token whitelist | Refresh Token / 사용자 | 재발급 가능 여부 검증 | Refresh Token TTL |
| Token Version | 사용자 전체 | 권한/잠금/비밀번호 변경 시 전체 토큰 차단 | 없음 또는 장기 |
| Locked Email blacklist | 계정/email | 잠금 계정 인증 차단 | 잠금 정책 기준 |
| Session registry | 로그인 세션 | 특정 세션 ACTIVE 여부 확인 | 세션/Refresh Token TTL |
| User-Browser active session | 사용자 + 브라우저 | 같은 브라우저 중복 로그인 방지 | browserId/Refresh Token TTL |
| User sessions index | 사용자 전체 | 전체 세션 폐기/기기 목록 | 별도 cleanup 필요 |

## 구현 시 권장 검증 흐름

### HTTP API 요청

```text
1. Authorization Bearer Access Token 추출
2. JWT 서명 검증
3. JWT 만료 검증
4. Access Token blacklist 확인
5. tokenVersion 확인
6. sessionId ACTIVE 확인
7. SecurityContext 설정
```

### WebSocket CONNECT

```text
1. STOMP CONNECT Authorization 헤더 추출
2. JWT 서명/만료 검증
3. Access Token blacklist 확인
4. tokenVersion 확인
5. sessionId ACTIVE 확인
6. session attributes에 userId, sessionId, accessToken 저장
```

### WebSocket 후속 프레임

```text
1. session attributes에서 accessToken, sessionId 조회
2. Access Token blacklist 재확인
3. tokenVersion 재확인
4. sessionId ACTIVE 재확인
5. SEND/SUBSCRIBE/UNSUBSCRIBE 처리
```

이렇게 해야 로그아웃된 Access Token으로 이미 연결된 WebSocket 세션이 계속 동작하는
문제를 막을 수 있다.

## 현재 추천 방향
지금 프로젝트에서는 tokenVersion을 제거하지 않고 session registry를 추가하는 방향이
안전하다.

```text
tokenVersion
→ 권한 변경, 계정 잠금, 비밀번호 변경 같은 사용자 전체 보안 상태 변경 대응

sessionId
→ 로그인 세션 단위 관리, 같은 브라우저 중복 로그인 방지

Access Token blacklist
→ 로그아웃된 특정 Access Token의 남은 TTL 동안 재사용 차단

Refresh Token whitelist
→ 재발급 가능한 Refresh Token 검증
```

나중에 session registry가 충분히 안정화되고 user sessions index까지 정확하게 관리할 수
있다면 tokenVersion을 session invalidation으로 흡수할지 재검토할 수 있다. 지금은 기존
보안 정책과 테스트 범위를 유지하면서 sessionId를 추가하는 것이 변경 리스크가 낮다.

## 관련 노트
- [[02-knowledge/auth/JWT 토큰 버전으로 권한 변경 시 전체 로그아웃]]
- [[02-knowledge/auth/JWT 이메일 claim과 계정 잠금 블랙리스트]]
- [[03-decisions/auth/Refresh Token 화이트리스트와 로그아웃 순서]]
- [[03-decisions/auth/계정 잠금 상태와 토큰 블랙리스트 정책]]
- [[03-decisions/infra/Redis 공통 정책과 컴포넌트]]
