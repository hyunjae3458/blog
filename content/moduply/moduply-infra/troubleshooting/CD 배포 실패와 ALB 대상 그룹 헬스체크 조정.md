---
tags: [infra, aws, ecs, cd, alb, target-group, health-check, troubleshooting]
date: 2026-07-20
status: 해결됨
---

# CD 배포 실패와 ALB 대상 그룹 헬스체크 조정

## 한 줄 요약
> GitHub Actions CD는 새 Docker 이미지를 정상 푸시하고 ECS 서비스 업데이트까지 수행했지만, ALB 대상 그룹의 헬스체크 조건이 너무 엄격해 새 task가 unhealthy로 교체되며 `services-stable` 대기 시간이 초과됐다.

## 배경

분산환경 재배포 후 App은 다음 구조로 운영 중이다.

```text
사용자
  ↓
ALB 80
  ↓
HTTPS 443 redirect
  ↓
ALB 443 listener
  ↓
Target Group: moduply-app-instance-tg
  ↓
App EC2:8080
  ↓
Spring App container
```

App task는 Google OAuth outbound 문제 때문에 `awsvpc`가 아니라 `bridge` 모드로 운영한다.

```text
Task network mode: bridge
Target group type: Instance
Host port: 8080
Container port: 8080
```

## 대상 그룹의 역할

대상 그룹(Target Group)은 ALB가 받은 요청을 어느 백엔드로 보낼지 정하는 백엔드 묶음이다.

```text
Listener
- ALB가 어떤 포트로 요청을 받을지 결정
- 예: 80, 443

Target Group
- 받은 요청을 어떤 백엔드로 보낼지 결정
- 현재는 App EC2 instance의 8080 포트를 가리킴
```

현재 구조에서는 다음 의미를 가진다.

```text
ALB 443 listener
→ moduply-app-instance-tg
→ App EC2:8080
```

대상 그룹은 트래픽 전달뿐 아니라 health check도 수행한다.

```text
Target Group
→ App EC2:8080/v3/api-docs
→ 성공 코드 범위면 healthy
→ 실패하면 unhealthy
```

ECS 서비스는 task를 실행한 뒤 해당 task/instance를 target group에 등록한다. ALB는 healthy target에게만 트래픽을 보낸다.

## 발생한 문제

GitHub Actions CD의 `Update ECS service with new task definition` 단계에서 다음 오류가 발생했다.

```text
Waiter ServicesStable failed: Max attempts exceeded
Process completed with exit code 255
```

ECS 이벤트는 다음 흐름이었다.

```text
service has started 1 tasks
service registered 1 targets in target-group moduply-app-instance-tg
task instance i-... port 8080 is unhealthy due to Health checks failed
Amazon ECS replaced 1 tasks due to an unhealthy status
service deregistered 1 targets
```

즉 CD가 새 task definition revision을 적용하는 데는 성공했지만, 새 task가 대상 그룹 health check를 통과하지 못해 ECS가 계속 task를 교체했고, GitHub Actions의 `aws ecs wait services-stable`이 제한 시간 안에 안정 상태를 보지 못했다.

## 처음 의심한 것

처음에는 task definition이나 ECR 이미지 문제를 의심했다.

하지만 새 task definition과 이전 task definition을 비교했을 때 차이는 image tag뿐이었다.

```diff
- "image": "<AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/moduply-repo:latest"
+ "image": "<AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/moduply-repo:ad11aa97e99e99ae83020b898c29d57bf2a74b5a"
```

CD는 기존 task definition을 가져와 `moduply-app` 컨테이너의 image만 새 SHA tag로 교체한다. 따라서 network mode, port mapping, environment file, log 설정은 기존과 동일했다.

## 원인

대상 그룹 health check 설정이 너무 엄격했다.

기존 설정:

```text
HealthCheckPath: /swagger-ui/index.html
Matcher: 200
Timeout: 5초
Interval: 30초
HealthyThreshold: 5
UnhealthyThreshold: 2
```

문제점:

```text
Success code가 200만 허용
→ 302/304 등도 실패로 판단될 수 있음

UnhealthyThreshold가 2
→ 2번만 실패해도 unhealthy 처리

HealthyThreshold가 5
→ 다시 healthy가 되려면 최소 5번 연속 성공 필요

Timeout 5초
→ t3.small, Spring Boot 부팅 직후, 배포 직후에는 짧을 수 있음
```

새 task가 시작 직후 완전히 안정화되기 전에 health check가 실패하면 ECS는 task를 unhealthy로 판단하고 교체한다.

```text
새 task 시작
→ target group 등록
→ health check 실패
→ ECS가 task 교체
→ 새 task 재시작
→ 반복
→ services-stable timeout
```

## 확인 방법

대상 그룹 상태는 다음 명령으로 확인했다.

```bash
aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN>
```

문제 당시 상태:

```json
{
  "TargetHealth": {
    "State": "unhealthy",
    "Reason": "Target.FailedHealthChecks",
    "Description": "Health checks failed"
  }
}
```

대상 그룹 설정은 다음 명령으로 확인했다.

```bash
aws elbv2 describe-target-groups \
  --target-group-arns <TARGET_GROUP_ARN>
```

App EC2 내부에서는 후보 경로가 정상 응답했다.

```text
GET /swagger-ui/index.html → 200
GET / → 200
GET /v3/api-docs → 200
```

따라서 App 자체가 완전히 죽은 것이 아니라, 배포 직후 health check 조건과 안정화 타이밍이 문제였다.

## 해결

새 대상 그룹을 만들 필요는 없었다. 이미 bridge 구조에 맞는 `Instance` 타입 target group이었기 때문에 health check 설정만 수정했다.

수정 후 권장 설정:

```text
Health check path: /v3/api-docs
Success codes: 200-399
Timeout: 10초
Interval: 30초
Healthy threshold: 2
Unhealthy threshold: 5
```

CLI 예시:

```bash
aws elbv2 modify-target-group \
  --target-group-arn <TARGET_GROUP_ARN> \
  --health-check-path /v3/api-docs \
  --matcher HttpCode=200-399 \
  --health-check-timeout-seconds 10 \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 5
```

ECS service의 health check grace period도 길게 두는 것이 좋다.

```bash
aws ecs update-service \
  --cluster moduply-prod2-cluster \
  --service moduply-app-prod2-task-service \
  --health-check-grace-period-seconds 180
```

## CD 파일 자체에 대한 판단

현재 `.github/workflows/cd.yml`은 분산환경에서도 그대로 사용 가능하다.

다만 GitHub Actions variables는 새 분산환경 기준이어야 한다.

```text
ECS_CLUSTER = moduply-prod2-cluster
ECS_SERVICE = moduply-app-prod2-task-service
ECS_TASK_DEFINITION = moduply-app-prod-task
ECR_REPOSITORY_URI = <AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/moduply-repo
AWS_REGION = ap-northeast-2
```

CD는 task definition을 새로 설계하지 않는다. AWS에 등록된 현재 task definition을 다운로드한 뒤 image만 교체한다.

```text
현재 task definition이 bridge이면 새 revision도 bridge
현재 task definition이 awsvpc이면 새 revision도 awsvpc
```

따라서 bridge/awsvpc 전환은 CD가 아니라 ECS Service, Target Group, Task Definition을 함께 바꾸는 인프라 작업이다.

## 배운 점

- `services-stable` 실패는 GitHub Actions 문제가 아니라 ECS service가 stable 상태에 도달하지 못했다는 뜻이다.
- ECS service 이벤트의 `Task failed ELB health checks`는 Target Group health check 설정을 먼저 봐야 한다.
- Target Group type이 이미 맞다면 새로 만들 필요 없이 health check 설정만 수정하면 된다.
- 단일 App task 구조에서는 health check가 조금만 빡빡해도 배포 중 task 교체가 반복될 수 있다.
- 헬스체크 경로는 Swagger UI HTML보다 가벼운 `/v3/api-docs` 또는 추후 permitAll된 `/actuator/health`가 더 적합하다.

## 다음 개선

```text
1. /actuator/health를 운영 헬스체크용으로 permitAll 처리
2. Target Group health check path를 /actuator/health로 변경
3. App service desired count 2 구성으로 무중단 배포에 가깝게 개선
4. CD 실패 시 ECS events와 stopped task reason을 자동 출력하도록 워크플로우 보강
```
