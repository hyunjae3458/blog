---
tags: [infra, docker, jvm, spring-boot, memory, troubleshooting]
date: 2026-07-08
status: 해결중
---

# Spring Boot 컨테이너의 Metaspace OOM

## 한 줄 요약
> `OutOfMemoryError: Metaspace`는 heap 부족이 아니라 JVM의 클래스 메타데이터 영역 제한 문제였고, Spring/JPA/QueryDSL 조합에서 `128MB`는 너무 작았다.

## 배경 / 어떤 상황이었는가
Redis 연결 이후 로그인은 성공했지만, 여러 기능 QA 중 콘텐츠 목록 조회에서 반복적으로 Metaspace OOM이 발생했다.

```text
GET /api/contents
java.lang.OutOfMemoryError: Metaspace
```

## 문제
API 요청 처리 중 DispatcherServlet에서 handler dispatch가 실패했다.

```text
jakarta.servlet.ServletException: Handler dispatch failed: java.lang.OutOfMemoryError: Metaspace
```

## 원인
Dockerfile의 JVM 옵션에서 Metaspace 최대 크기를 128MB로 제한하고 있었다.

```dockerfile
ENV JVM_OPTS="-Xmx256m -Xms128m -XX:MaxMetaspaceSize=128m -XX:+UseSerialGC"
```

Metaspace는 Java heap이 아니다. 다음 정보들이 들어가는 네이티브 메모리 영역이다.

- 클래스 메타데이터
- Spring Bean/Configuration 메타데이터
- Hibernate 엔티티/쿼리 메타데이터
- Spring AOP, CGLIB 프록시 클래스
- 리플렉션/어노테이션 관련 정보

현재 애플리케이션은 Spring Boot, Spring Security, JPA/Hibernate, QueryDSL, WebSocket, OAuth2, Batch, AOP/Retry를 함께 사용한다. 이 조합에서 `128MB`는 QA 중 여러 기능을 호출하면서 한계에 닿았다.

## 고민한 대안들

| 대안 | 장점 | 단점 |
|---|---|---|
| `MaxMetaspaceSize` 제한 제거 | OOM 가능성 감소 | 작은 EC2에서 네이티브 메모리 사용량 통제가 어려움 |
| `MaxMetaspaceSize=256m`로 상향 | 현재 OOM 원인 완화 + 컨테이너 메모리 통제 유지 | 트래픽 증가 시 다시 부족할 수 있음 |
| EC2를 `t3.medium`으로 증설 | 전체 메모리 여유 증가 | 비용 증가 |
| Redis를 외부로 분리 | 앱 컨테이너/호스트 여유 증가 | ElastiCache 등 추가 비용 |

우선은 `MaxMetaspaceSize=256m`로 상향하고 heap은 `384MB`로 제한해 앱+Redis 동시 운영 메모리 안에서 균형을 맞추기로 했다.

## 실제 구현

```dockerfile
# JVM Configuration (ECS EC2 t3.small + Redis 동시 운영 기준)
# -Xms128m : 초기 힙 메모리를 128MB로 설정
# -Xmx384m : 최대 힙 메모리를 384MB로 제한
# -XX:MaxMetaspaceSize=256m : Metaspace 최대 크기를 256MB로 제한
# -XX:+UseSerialGC : Serial GC를 사용
ENV JVM_OPTS="-Xms128m -Xmx384m -XX:MaxMetaspaceSize=256m -XX:+UseSerialGC"
```

옵션 의미:

- `-Xms128m`: JVM 시작 시 초기 heap을 128MB로 잡아 시작 메모리 점유를 줄인다.
- `-Xmx384m`: 객체가 저장되는 heap 최대 크기를 384MB로 제한한다.
- `-XX:MaxMetaspaceSize=256m`: Spring/Hibernate/프록시 클래스 메타데이터 공간을 256MB까지 허용한다.
- `-XX:+UseSerialGC`: 작은 CPU/메모리 컨테이너에서 GC 스레드 오버헤드를 낮추기 위해 Serial GC를 사용한다.

## 결과 / 검증 방법
Dockerfile 수정 후 다음 순서로 검증한다.

```text
Docker image rebuild
ECR push
Task Definition 새 revision
ECS app service force deployment
/api/contents QA 재실행
CloudWatch Logs에서 Metaspace OOM 재발 여부 확인
```

## 배운 점 / 다음에 또 마주치면
- Java OOM은 heap만 보는 게 아니다. `Metaspace`, thread stack, direct memory 같은 native 영역도 컨테이너 메모리를 사용한다.
- Spring Boot 애플리케이션은 프레임워크 메타데이터와 프록시가 많아 Metaspace를 너무 작게 잡으면 기능 호출 중 터질 수 있다.
- 작은 EC2에서 JVM 옵션은 비용 절감과 안정성 사이의 트레이드오프다. 운영 지표를 보면서 `t3.medium` 또는 Redis 분리도 재검토해야 한다.

## 관련 노트
- [[03-decisions/infra/작은 ECS 컨테이너를 위한 JVM 옵션 조정 결정]]
- [[02-knowledge/infra/단일 EC2 ECS task placement와 리소스 부족]]
