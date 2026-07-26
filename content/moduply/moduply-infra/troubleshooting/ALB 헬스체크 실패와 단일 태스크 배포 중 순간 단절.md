---
tags: [infra, aws, alb, ecs, health-check, sse, troubleshooting]
date: 2026-07-16
status: 해결됨
---

# ALB 헬스체크 실패와 단일 태스크 배포 중 순간 단절

## 한 줄 요약
> bridge 전환과 서비스 재배포 중 ALB target deregistration과 health check 실패로 App task가 교체되면서 순간적으로 접속이 끊겼고, 단일 App task 구조에서는 배포 중 단절이 발생할 수 있음을 확인했다.

## 증상

배포 중 다음 이벤트가 발생했다.

```text
service moduply-app-prod2-task-service deregistered 1 targets in target-group moduply-app-instance-tg
Task failed ELB health checks in target-group moduply-app-instance-tg
exit code: 137
```

애플리케이션 로그에는 다음도 함께 보였다.

```text
SpringApplicationShutdownHook
Commencing graceful shutdown
```

SSE 쪽에서는 다음 로그가 반복됐다.

```text
Broken pipe
AsyncRequestTimeoutException
No converter for ErrorResponse with preset Content-Type 'text/event-stream'
```

## 원인

직접 원인은 ALB health check 실패 또는 target 전환 중 App target이 healthy 상태가 되기 전에 기존 target이 deregister된 것이다.

```text
App service 재배포
→ 기존 target deregister
→ 새 target initial/unhealthy
→ ECS가 task를 비정상으로 판단
→ task 종료
→ 접속 순간 단절
```

`Broken pipe`는 원인이 아니라 결과에 가깝다. 브라우저나 ALB가 SSE 연결을 끊었는데, 서버가 기존 SseEmitter에 heartbeat를 보내면서 발생했다.

## 확인한 것

App EC2 내부에서는 헬스체크 후보 경로가 정상 응답했다.

```text
GET /swagger-ui/index.html → 200
GET / → 200
GET /v3/api-docs → 200
```

따라서 App 자체가 완전히 죽어 있던 것이 아니라, 재배포와 target health 상태 전환 타이밍이 문제였다.

## 적용한 설정

Target Group health check는 Swagger 경로를 사용했다.

```text
Path: /swagger-ui/index.html
Success codes: 200-399
Port: traffic port
```

bridge 구조에서는 App EC2 SG에 다음 인바운드가 필요하다.

```text
TCP 8080 from ALB SG
```

## 단일 태스크 구조의 한계

현재 상시 운영용 App은 desired count 1이다.

```text
App task 1개
Target 1개
```

이 구조에서는 배포 중 기존 task가 내려가거나 target이 deregister되면 받을 대상이 없어지는 순간이 생길 수 있다.

무중단에 가깝게 하려면 최소 2개 target이 필요하다.

```text
App ASG desired: 2
App service desired: 2
```

## 재발 방지

```text
ECS health check grace period: 180초 권장
Target group success codes: 200-399
Unhealthy threshold: 5
Deregistration delay: 30초 수준
배포 중 force deployment 반복 금지
단일 task에서는 desired 0 → 1 방식으로 의도적으로 재기동
```

## SSE 코드 개선 필요

SSE 연결 종료는 정상 상황에서도 발생한다. 현재는 끊긴 emitter에 heartbeat를 보내면서 ERROR 로그가 크게 남고, GlobalExceptionHandler가 text/event-stream 응답에 JSON ErrorResponse를 쓰려고 하면서 추가 예외가 발생한다.

추후 개선 방향:

```text
SseEmitter.send 실패 시 emitter 즉시 제거
Broken pipe / AsyncRequestTimeoutException은 낮은 로그 레벨로 처리
SSE 요청 예외는 GlobalExceptionHandler에서 JSON 응답으로 변환하지 않도록 분기
ALB idle timeout > SSE heartbeat interval 유지
```
