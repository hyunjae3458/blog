---
tags: [infra, aws, ecs, ami, awsvpc, troubleshooting]
date: 2026-07-16
status: 해결됨
---

# ECS EC2 Capacity Provider에서 오래된 AMI로 awsvpc 태스크가 배치되지 않은 문제

## 한 줄 요약
> 기존 Amazon Linux 1 계열 ECS Optimized AMI를 사용하자 ECS Agent와 Container Instance는 정상처럼 보였지만 awsvpc 태스크가 PROVISIONING/PENDING에 머물렀고, 공식 ECS Optimized Amazon Linux 2023 AMI로 교체하자 해결됐다.

## 증상

```text
EC2 instance: running
ECS Agent: running
Container Instance: ACTIVE
Capacity Provider: 정상 표시
Service event: has started 1 tasks
Task status: PROVISIONING 또는 PENDING
attachments: []
containers: []
docker ps: 실제 App/Redis 컨테이너 없음
ECS agent log: Still have 0 image states being managed
```

처음에는 Redis와 App 모두 컨테이너가 실제로 생성되지 않았다. Redis는 나중에 공식 AMI로 교체한 뒤 정상 실행됐고, App도 같은 방식으로 해결됐다.

## 원인

기존에 사용하던 AMI는 다음 계열이었다.

```text
Amazon Linux AMI amzn-ami-2018.03.20250224 x86_64 ECS HVM GP2
```

이 AMI는 ECS Optimized이긴 하지만 Amazon Linux 1 기반의 오래된 이미지다. ECS EC2 Capacity Provider, awsvpc, task ENI, 최신 ECS 서비스 배치 흐름과 조합했을 때 컨테이너 인스턴스는 정상 등록되지만 task가 실제 Docker 컨테이너까지 내려오지 않는 현상이 발생했다.

## 확인 과정

처음에는 다음 가능성을 의심했다.

```text
- CPU / Memory 부족
- Target Group 타입 불일치
- Security Group 오류
- Subnet 불일치
- S3 environment file 권한 문제
- App 환경변수 오류
```

하지만 CLI로 확인했을 때 App 인스턴스의 가용 리소스는 충분했다.

```text
App task 요구:
- CPU: 1024
- Memory: 1536

App container instance remaining:
- CPU: 2048
- Memory: 1945
```

또한 awsvpc capability도 존재했다.

```text
ecs.capability.task-eni
ecs.capability.task-eni-trunking
```

그런데 task 상세는 계속 다음 상태였다.

```json
{
  "lastStatus": "PROVISIONING",
  "desiredStatus": "RUNNING",
  "attachments": [],
  "containers": []
}
```

이 상태는 앱 실행 실패가 아니라 ECS가 아직 컨테이너 실행 단계까지 가지 못했다는 뜻이다.

## 해결

공식 ECS Optimized Amazon Linux 2023 x86_64 AMI를 찾아 Launch Template 새 버전을 만들었다.

```bash
#!/bin/bash
echo ECS_CLUSTER=moduply-prod2-cluster >> /etc/ecs/ecs.config
```

그 후 ASG를 최신 Launch Template 버전으로 변경하고 인스턴스를 재생성했다.

```text
ASG desired 0
→ 기존 EC2 종료 확인
→ ASG desired 1
→ 새 EC2가 ECS Container Instance로 등록
→ ECS Service 재배포
```

결과적으로 Redis와 App 컨테이너가 정상 실행됐다.

## 배운 점

- ECS Agent가 떠 있고 Container Instance가 ACTIVE라고 해서 task 실행 경로가 모두 정상이라는 뜻은 아니다.
- `PROVISIONING`, `attachments=[]`, `containers=[]`는 앱 코드나 환경변수보다 ECS 배치/인스턴스 실행 환경을 먼저 봐야 한다.
- ECS EC2 Capacity Provider에서는 오래된 ECS Optimized AMI를 재사용하지 말고 공식 최신 ECS Optimized AL2023 또는 최소 AL2 AMI를 사용하는 것이 안전하다.
- Launch Template을 수정해도 기존 EC2에는 적용되지 않는다. ASG 인스턴스를 재생성해야 한다.

## 다음에 같은 문제를 만나면

```text
1. describe-tasks로 lastStatus, attachments, containers 확인
2. describe-container-instances로 remainingResources와 attributes 확인
3. ECS Agent 로그에서 image pull 또는 task 수신 여부 확인
4. 오래된 AMI면 공식 ECS Optimized AL2023로 교체
5. ASG 인스턴스 재생성
```
