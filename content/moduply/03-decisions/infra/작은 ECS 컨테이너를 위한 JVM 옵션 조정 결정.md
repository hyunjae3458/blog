---
tags: [infra, docker, jvm, memory, decision]
date: 2026-07-08
status: 채택됨
---

# 작은 ECS 컨테이너를 위한 JVM 옵션 조정 결정

## 결정
ECS EC2 `t3.small`에서 앱과 Redis를 함께 운영하기 위해 JVM 기본 옵션을 다음과 같이 조정한다.

```dockerfile
ENV JVM_OPTS="-Xms128m -Xmx384m -XX:MaxMetaspaceSize=256m -XX:+UseSerialGC"
```

## 배경 / 왜 이 결정이 필요했는가
여러 기능 QA 중 `/api/contents` 요청에서 `OutOfMemoryError: Metaspace`가 발생했다. 기존 설정은 heap과 Metaspace를 모두 매우 작게 잡고 있었다.

```dockerfile
ENV JVM_OPTS="-Xmx256m -Xms128m -XX:MaxMetaspaceSize=128m -XX:+UseSerialGC"
```

## 고려한 대안

| 대안 | 장점 | 단점 |
|---|---|---|
| 기존 설정 유지 | 메모리 사용량 최소 | Metaspace OOM 재발 |
| Metaspace 제한 제거 | OOM 가능성 감소 | 작은 EC2에서 native memory 통제 어려움 |
| `MaxMetaspaceSize=256m`, `Xmx384m` (채택) | Metaspace OOM 완화 + 전체 메모리 통제 | 트래픽 증가 시 추가 조정 필요 |
| EC2를 `t3.medium`으로 증설 | 근본적인 여유 확보 | 비용 증가 |

## 구현 방식
Dockerfile의 JVM 옵션과 주석을 수정했다.

```dockerfile
ENV JVM_OPTS="-Xms128m -Xmx384m -XX:MaxMetaspaceSize=256m -XX:+UseSerialGC"
```

각 옵션의 의도:

- `-Xms128m`: 시작 heap을 작게 잡아 컨테이너 시작 메모리 점유를 줄인다.
- `-Xmx384m`: Java heap 상한을 통제해 Redis와 같은 EC2에서 공존할 수 있게 한다.
- `-XX:MaxMetaspaceSize=256m`: Spring/Hibernate/프록시 메타데이터 공간을 기존보다 확보한다.
- `-XX:+UseSerialGC`: 작은 컨테이너에서 GC 스레드 오버헤드를 줄인다.

## 트레이드오프 / 감수한 것
- heap과 Metaspace를 완전히 넉넉하게 잡지는 않았다.
- 트래픽 증가 또는 Batch/WebSocket 사용량 증가 시 다시 메모리 조정이 필요할 수 있다.
- Serial GC는 단순하지만, 요청량이 늘면 stop-the-world 지연이 체감될 수 있다.

## 재검토 시점
- `OutOfMemoryError`가 재발할 때
- CloudWatch에서 memory utilization이 지속적으로 높게 나올 때
- Redis를 분리하거나 EC2를 증설할 때
- Actuator/metrics를 붙여 JVM memory 지표를 볼 수 있게 되었을 때

## 관련 노트
- [[02-knowledge/infra/Spring Boot 컨테이너 Metaspace OOM]]
- [[02-knowledge/infra/단일 EC2 ECS task placement와 리소스 부족]]
