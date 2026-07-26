---
tags: [auth, jwt, redis, spring-security, websocket]
date: 2026-07-09
status: 해결됨
---

# JWT 토큰 버전으로 권한 변경 시 전체 로그아웃

## 한 줄 요약
> 사용자별 `tokenVersion`을 Redis와 Access Token에 함께 기록하고 매 인증 시 비교하여, 권한 변경이나 계정 잠금 시 기존 기기의 토큰을 한 번에 무효화했다.

## 배경 / 어떤 상황이었는가
기존 로그아웃은 Refresh Token 화이트리스트를 삭제했지만 이미 발급된 Access Token은 만료 전까지 사용할 수 있었다.
일반 로그아웃은 현재 요청의 Access Token을 블랙리스트에 넣어 해결했지만, 권한 변경에서는 여러 기기에 발급된 모든 Access Token을 알 수 없기 때문에 개별 블랙리스트만으로 처리하기 어려웠다.

프로젝트 보안 규칙상 권한 변경과 계정 잠금은 기존 인증 상태에 즉시 반영되어야 했다.

## 문제 (구체적으로)
- JWT Access Token은 서버가 발급 후 내용을 직접 변경하거나 회수할 수 없다.
- 사용자가 여러 기기에서 로그인했다면 서버가 모든 Access Token 원문을 보관하지 않는 이상 개별 블랙리스트 등록이 어렵다.
- 권한 변경 후 Refresh Token을 남겨두면 기존 인증 상태로 토큰 재발급을 시도할 수 있다.
- HTTP만 차단하면 이미 연결된 WebSocket에서 기존 인증 정보가 계속 사용될 수 있다.

## 고민한 대안들
1. 모든 Access Token을 Redis에 저장
   - 장점: 토큰별로 직접 폐기할 수 있다.
   - 단점: 저장량과 관리 대상이 많아지고 JWT를 사용하는 이점이 줄어든다.
2. 사용자별 무효화 시각을 저장하고 JWT `iat`와 비교
   - 장점: 사용자당 값 하나로 전체 토큰을 차단할 수 있다.
   - 단점: 시간 정밀도와 경계 시점 처리가 필요하다.
3. 사용자별 토큰 버전을 비교 (채택)
   - 장점: 단순한 정수 비교로 모든 기존 토큰을 무효화할 수 있고 Redis `INCR`로 원자적 증가가 가능하다.
   - 단점: Redis 유실 시 버전 복구 수단이 없으므로 추후 DB 영속화가 필요하다.

## 실제 구현 (핵심 코드)

로그인 성공 시 버전 키가 없을 때만 `0`으로 초기화한다.

```java
String tokenVersionKey =
    RedisKeyPolicy.USER_TOKEN_VERSION.generateKey(userDto.getEmail());

redisUtil.setDataIfAbsent(tokenVersionKey, "0");

String storedTokenVersion = redisUtil.getData(tokenVersionKey);
long tokenVersion = Long.parseLong(storedTokenVersion);

String accessToken =
    jwtTokenProvider.generateAccessToken(authentication, tokenVersion);
```

`setDataIfAbsent`는 Redis의 `SETNX`를 사용한다.

```java
public boolean setDataIfAbsent(String key, String value) {
    return Boolean.TRUE.equals(
        stringRedisTemplate.opsForValue().setIfAbsent(key, value)
    );
}
```

단순히 `GET` 결과가 `null`일 때 `SET 0`을 수행하면 다음 경쟁 조건이 발생할 수 있다.

```text
로그인 요청: GET → null
권한 변경: INCR → 1
로그인 요청: SET 0 → 권한 변경 결과를 덮어씀
```

`SETNX`는 키가 이미 존재하면 값을 변경하지 않기 때문에 권한 변경으로 증가한 버전을 보존한다.

권한 변경 시 버전을 원자적으로 증가시키고 Refresh Token을 삭제한다.

```java
user.updateRole(request.getRole());

redisUtil.increment(
    RedisKeyPolicy.USER_TOKEN_VERSION.generateKey(user.getEmail())
);
redisUtil.deleteData(
    RedisKeyPolicy.REFRESH_TOKEN.generateKey(user.getEmail())
);
```

Access Token에는 발급 당시 버전을 숫자 클레임으로 포함한다.

```java
claimsBuilder.claim(TOKEN_VERSION_CLAIM, tokenVersion);
```

## 인증 및 무효화 흐름

### 로그인
1. Redis의 `access:token-version:{email}` 조회
2. 키가 없다면 `SETNX`로 `0` 초기화
3. 현재 버전을 Access Token의 `tokenVersion` 클레임에 기록
4. Refresh Token을 Redis 화이트리스트에 저장

### 일반 HTTP 인증
1. JWT 서명, 타입, 만료 검증
2. 개별 Access Token 블랙리스트 확인
3. JWT의 `tokenVersion`과 Redis의 현재 버전 비교
4. 버전이 다르면 `401 Unauthorized`
5. 버전이 같으면 DB에서 현재 사용자 권한을 조회해 SecurityContext 구성

### 권한 변경 및 계정 잠금
1. DB의 권한 또는 잠금 상태 변경
2. Redis 토큰 버전을 `INCR`
3. Refresh Token 화이트리스트 삭제
4. 기존 기기의 Access Token은 다음 요청에서 버전 불일치로 거부
5. 클라이언트는 `401`을 받으면 로컬 토큰을 제거하고 로그인 화면으로 이동

인증 실패는 로그아웃 엔드포인트 호출이 아니므로 `LogoutHandler`가 자동 실행되지 않는다.
따라서 서버는 권한 변경 시 Refresh Token을 직접 삭제하고, 클라이언트는 `401`을 로그아웃 상태로 처리해야 한다.

### WebSocket 인증
1. STOMP `CONNECT`에서 JWT와 Redis 버전 비교
2. 성공 시 이메일과 토큰 버전을 WebSocket 세션 속성에 저장
3. 이후 `SEND`, `SUBSCRIBE`, `UNSUBSCRIBE`에서도 Redis 현재 버전과 재비교
4. 권한 변경 후 기존 연결의 다음 메시지부터 인증 실패 처리

## 결과 / 검증 방법
- 최초 로그인에서 버전 `0`으로 Access Token이 발급되는지 검증
- 기존 버전 키가 로그인으로 초기화되지 않는지 검증
- 권한 변경 시 Redis `INCR`와 Refresh Token 삭제가 호출되는지 검증
- 계정 잠금 시 동일하게 기존 토큰이 무효화되는지 검증
- HTTP 요청의 버전 불일치가 `401`을 반환하는지 검증
- STOMP CONNECT 및 연결 후 요청에서 버전 불일치를 거부하는지 검증
- Refresh Token 재발급 토큰에 현재 Redis 버전이 포함되는지 검증
- 전체 테스트 실행 결과: `BUILD SUCCESSFUL`

## 배운 점 / 다음에 또 마주치면
- JWT 무효화는 토큰 자체를 수정하는 문제가 아니라 서버가 인정하는 현재 인증 세대를 관리하는 문제다.
- `GET` 후 조건부 `SET`처럼 보이는 단순 로직도 분산 환경에서는 경쟁 조건이 생기므로 Redis 원자 명령을 우선 검토해야 한다.
- 일반 로그아웃은 현재 토큰 하나를 블랙리스트로 관리하고, 권한 변경은 사용자 버전 증가로 전체 토큰을 무효화하는 방식이 역할에 맞다.
- 현재 Redis가 버전의 유일한 저장소이므로 데이터 유실 위험이 남아 있다. 추후 `User.tokenVersion`을 DB 원본으로 두고 Redis를 캐시로 전환할 예정이다.

## 관련 노트
- [[03-decisions/auth/Refresh Token 화이트리스트와 로그아웃 순서]]
- [[03-decisions/auth/계정 잠금 상태와 토큰 블랙리스트 정책]]
- [[03-decisions/auth/보안 Redis 작업 동기 처리 결정]]
