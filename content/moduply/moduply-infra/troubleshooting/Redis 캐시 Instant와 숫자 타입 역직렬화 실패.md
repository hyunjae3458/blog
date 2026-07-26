---
tags: [infra, redis, cache, jackson, spring-cache, troubleshooting]
date: 2026-07-25
status: 해결됨
---

# Redis 캐시 Instant와 숫자 타입 역직렬화 실패

## 한 줄 요약
> 운영 Redis에 저장된 `playlistDetail` 캐시를 읽는 과정에서 Jackson `PolymorphicTypeValidator`가 `Instant`, `Long` 같은 타입 역직렬화를 거부해 플레이리스트 단건 조회가 500으로 실패했고, 캐시용 허용 타입 목록 보강과 운영 캐시 삭제로 해결했다.

## 배경

운영 환경에서 플레이리스트 단건 조회 API 호출 중 500 에러가 발생했다.

```text
GET /api/playlists/{playlistId}
```

스택트레이스상 DB 조회나 권한 검증 문제가 아니라 Redis CacheManager가 캐시 값을 읽는 단계에서 실패했다.

```text
RedisCache.lookup
→ CacheInterceptor
→ PlaylistService.findById
→ PlaylistController.getPlaylist
```

## 문제

첫 번째 오류는 `PlaylistDto.updatedAt`의 `Instant` 타입에서 발생했다.

```text
Could not resolve type id 'java.time.Instant' as a subtype of `java.time.Instant`
Configured `PolymorphicTypeValidator` denied resolution
through reference chain: com.team6.moduply.playlist.dto.PlaylistDto["updatedAt"]
```

`Instant.class` 허용을 추가하고 재배포했지만, 다음에는 `PlaylistDto.subscriberCount`의 `Long` 타입에서 같은 문제가 발생했다.

```text
Could not resolve type id 'java.lang.Long' as a subtype of `java.lang.Long`
Configured `PolymorphicTypeValidator` denied resolution
through reference chain: com.team6.moduply.playlist.dto.PlaylistDto["subscriberCount"]
```

## 원인

Redis CacheManager는 `GenericJackson2JsonRedisSerializer`와 default typing을 사용한다.

```java
cacheObjectMapper.activateDefaultTyping(
    typeValidator,
    ObjectMapper.DefaultTyping.EVERYTHING,
    JsonTypeInfo.As.PROPERTY
);
```

이 설정은 Redis에 객체를 저장할 때 실제 타입 정보를 JSON에 함께 저장한다. 보안을 위해 `BasicPolymorphicTypeValidator`로 허용 타입을 제한했는데, 기존 허용 목록이 실제 캐시 DTO 필드 타입보다 좁았다.

기존 허용 목록:

```java
.allowIfSubType("com.team6.moduply.")
.allowIfSubType(ArrayList.class)
.allowIfSubType(UUID.class)
.allowIfSubType(LocalDateTime.class)
.allowIfSubType(BigDecimal.class)
```

하지만 `PlaylistDto`에는 다음 타입들이 들어간다.

```java
Instant updatedAt
Long subscriberCount
boolean subscribedByMe
Double averageRating
Integer reviewCount
```

그래서 Redis 캐시 JSON을 역직렬화할 때 타입 정보는 존재하지만, validator가 해당 타입을 허용하지 않아 차단됐다.

## 해결

`CacheConfig`의 `BasicPolymorphicTypeValidator`에 캐시 DTO에서 실제로 쓰는 타입을 추가했다.

```java
.allowIfSubType(LocalDateTime.class)
// DTO의 createdAt/updatedAt에 쓰이는 Instant 타입 캐시 역직렬화를 허용한다.
.allowIfSubType(Instant.class)
// Redis default typing은 Long/Integer/Double/Boolean 같은 wrapper 타입 정보도 저장한다.
.allowIfSubType(Long.class)
.allowIfSubType(Integer.class)
.allowIfSubType(Double.class)
.allowIfSubType(Boolean.class)
.allowIfSubType(BigDecimal.class)
```

`LocalDateTime`은 현재 캐시 DTO에서는 직접적인 원인이 아니지만, 기존 허용 목록에 있었고 향후 캐시 DTO에서 쓰일 수 있어 유지했다.

## 운영 조치

코드 수정 전 Redis에 저장된 기존 `playlistDetail` 캐시는 같은 오류를 계속 낼 수 있으므로 운영 Redis에서 삭제했다.

삭제 대상:

```text
moduply:cache:v3:playlistDetail::*
```

운영에서는 `KEYS` 대신 `SCAN`을 사용한다.

```bash
redis-cli -h <REDIS_PRIVATE_IP> -p 6379 --scan --pattern 'moduply:cache:v3:playlistDetail::*'
```

삭제:

```bash
redis-cli -h <REDIS_PRIVATE_IP> -p 6379 --scan --pattern 'moduply:cache:v3:playlistDetail::*' \
  | xargs redis-cli -h <REDIS_PRIVATE_IP> -p 6379 DEL
```

호스트 환경에서 `xargs -r` 옵션을 지원하지 않을 수 있어 `-r` 없이 실행했다. `redis-cli` 프롬프트 안에서는 파이프 명령을 실행할 수 없고, EC2 쉘에서 실행해야 한다.

### Redis 캐시 키 조회 / 삭제 절차

먼저 Redis 접속 정보가 맞는지 키 조회만 한다.

```bash
redis-cli -h <REDIS_PRIVATE_IP> -p 6379 --scan --pattern 'moduply:cache:v3:playlistDetail::*'
```

여기서 `<REDIS_PRIVATE_IP>`는 실제 Redis private IP로 바꿔야 한다. 꺾쇠까지 그대로 입력하면 쉘에서 리다이렉션 문자처럼 해석될 수 있다.

예시:

```bash
redis-cli -h 10.0.12.34 -p 6379 --scan --pattern 'moduply:cache:v3:playlistDetail::*'
```

삭제는 같은 패턴을 `xargs`로 넘겨 `DEL` 한다.

```bash
redis-cli -h 10.0.12.34 -p 6379 --scan --pattern 'moduply:cache:v3:playlistDetail::*' \
  | xargs redis-cli -h 10.0.12.34 -p 6379 DEL
```

키가 없을 때 `xargs`가 빈 `DEL`을 실행하는 게 싫으면 환경에 따라 `xargs -r`를 쓸 수 있다.

```bash
redis-cli -h 10.0.12.34 -p 6379 --scan --pattern 'moduply:cache:v3:playlistDetail::*' \
  | xargs -r redis-cli -h 10.0.12.34 -p 6379 DEL
```

단, 일부 환경에서는 `xargs -r` 옵션을 지원하지 않는다. 그때는 `-r` 없이 실행하거나 shell loop를 쓴다.

```bash
for key in $(redis-cli -h 10.0.12.34 -p 6379 --scan --pattern 'moduply:cache:v3:playlistDetail::*'); do
  redis-cli -h 10.0.12.34 -p 6379 DEL "$key"
done
```

### redis-cli 프롬프트 안에서 할 때

프롬프트가 아래처럼 보이면 이미 `redis-cli` 내부다.

```text
127.0.0.1:6379>
```

이 상태에서는 쉘 파이프 명령을 실행할 수 없다.

```bash
redis-cli --scan --pattern 'moduply:cache:v3:playlistDetail::*' | xargs redis-cli DEL
```

위 명령을 Redis 프롬프트 안에서 치면 Redis가 `redis-cli`를 명령어로 해석해 `unknown command`가 난다. 이때는 `exit`로 나와 EC2 쉘에서 실행한다.

Redis 프롬프트 안에서 직접 확인하려면 Redis 명령만 사용한다.

```text
SCAN 0 MATCH moduply:cache:v3:playlistDetail::* COUNT 100
```

나온 키가 적으면 직접 삭제한다.

```text
DEL moduply:cache:v3:playlistDetail::<실제키>
```

### Docker 권한 문제

Redis 컨테이너에 `docker exec`로 들어가려다 아래 오류가 날 수 있다.

```text
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

이건 `redis-cli` 권한 문제가 아니라 현재 Linux 사용자가 Docker daemon socket에 접근할 권한이 없다는 뜻이다. 가능한 선택지는 둘이다.

```bash
sudo docker ps
sudo docker exec -it <redis-container-name-or-id> redis-cli
```

또는 Docker를 거치지 않고 Redis host/port로 직접 접속한다.

```bash
redis-cli -h <REDIS_PRIVATE_IP> -p 6379
```

이번 작업에서는 prefix가 명확했기 때문에 DataGrip이나 SSH 터널을 쓰지 않고, EC2 쉘에서 `redis-cli --scan`으로 조회/삭제하는 방식이 더 단순했다.

## 검증

로컬에서 컴파일을 확인했다.

```bash
./gradlew compileJava
```

결과:

```text
BUILD SUCCESSFUL
```

운영 재배포 후 확인할 것:

```text
1. GET /api/playlists/{playlistId} 첫 조회
   - DB 조회 후 Redis에 새 캐시 저장
2. 같은 URL 두 번째 조회
   - Redis cache hit
   - SerializationException 미발생
```

## 남은 주의점

`PlaylistDto.contents`나 `tags`가 `stream().toList()` / `List.of()`로 생성되면 JDK 불변 리스트 구현체 타입이 Redis JSON에 남을 수 있다. 만약 다음에 `java.util.ImmutableCollections...` 계열 타입 차단 오류가 발생하면 다음 중 하나로 대응한다.

```text
1. DTO 생성 시 List 필드를 new ArrayList<>(...)로 고정
2. validator에 필요한 컬렉션 구현체 허용 추가
```

보안 관점에서는 모든 타입을 허용하는 방식은 피하고, 캐시에 실제 저장되는 DTO와 값 타입만 명시적으로 허용하는 쪽이 낫다.

## 배운 점

- Redis 캐시 직렬화에 default typing을 쓰면 DTO 필드 타입과 컬렉션 구현체까지 역직렬화 허용 목록에 영향을 준다.
- `PolymorphicTypeValidator`를 좁게 잡는 것은 보안상 맞지만, 캐시 대상 DTO의 실제 필드 타입을 기준으로 테스트해야 한다.
- 운영 캐시 장애는 코드 수정만으로 끝나지 않고, 이미 저장된 기존 캐시 키 삭제까지 배포 절차에 포함해야 한다.

## 관련 코드

```text
src/main/java/com/team6/moduply/common/config/CacheConfig.java
src/main/java/com/team6/moduply/playlist/dto/PlaylistDto.java
src/main/java/com/team6/moduply/playlist/service/PlaylistService.java
```
