---
tags: [infra, aws, ecs, ec2, docker, troubleshooting]
date: 2026-07-07
status: 해결됨
---

# ECS EC2 첫 배포 트러블슈팅

## 한 줄 요약
> Fargate 대신 ECS EC2 launch type으로 배포하면서, EC2 capacity provider, task placement, GPU requirement, 이미지 digest 고정, HTTP 접속 방식을 단계적으로 정리했다.

## 배경 / 어떤 상황이었는가
애플리케이션 Docker 이미지를 ECR에 push한 뒤, 비용을 아끼기 위해 Fargate가 아니라 ECS EC2 방식으로 배포했다. ALB와 NAT Gateway 없이 `t3.small` EC2 1대에 앱 컨테이너를 올리는 것이 목표였다.

## 문제 1: 클러스터에 등록된 EC2 인스턴스가 없음
서비스 생성 시 EC2 launch type을 선택했는데 클러스터에 등록된 container instance가 없다는 메시지가 나왔다.

### 원인
ECS 클러스터에 capacity provider는 있었지만, 연결된 Auto Scaling Group의 desired capacity가 `0`이라 실제 EC2 인스턴스가 떠 있지 않았다.

### 해결
Auto Scaling Group의 desired capacity를 `1`로 올렸다.

```text
ASG minimum capacity: 1
ASG desired capacity: 1
ASG maximum capacity: 1
```

## 문제 2: GPU 리소스 부족으로 task placement 실패

```text
insufficient GPU resource available
```

### 원인
Task Definition에서 GPU requirement가 들어가 있었다. `t3.small`은 GPU 인스턴스가 아니고, Spring Boot 백엔드 서버도 GPU를 필요로 하지 않는다.

### 해결
GPU 값을 `0`으로 넣는 것이 아니라 GPU requirement 항목 자체를 제거했다.

## 문제 3: ECR에 새 이미지를 push했는데 ECS가 이전 이미지를 계속 사용
Task Definition의 image가 tag가 아니라 digest로 고정되어 있었다.

```text
<AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/moduply-repo@sha256:...
```

### 원인
Digest는 특정 이미지 내용을 완전히 고정한다. `latest` 태그를 새로 push해도 기존 digest는 바뀌지 않는다.

### 해결
배포 확인 중에는 image URI를 tag 기반으로 바꿨다.

```text
<AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/moduply-repo:latest
```

운영 추적성을 생각하면 `latest`보다 git commit hash나 `20260707-1` 같은 명시적 태그가 더 낫다.

## 문제 4: EC2 public IP 접속 시 무한 로딩처럼 보임
브라우저 자동완성 때문에 `https://<EC2_PUBLIC_IP>`로 접속하고 있었다.

### 원인
현재 구성은 ALB와 ACM 인증서가 없고, EC2 host port `80`으로 Spring Boot HTTP만 열려 있다.

### 해결
명시적으로 HTTP로 접속했다.

```text
http://<EC2_PUBLIC_IP>
```

## 실제 구성

```text
ECS Cluster: moduply-prod-cluster
Launch type: EC2
EC2 instance: t3.small 1대
Network mode: bridge
Container port: 8080
Host port: 80
Load balancer: 없음
NAT Gateway: 없음
Service desired tasks: 1
```

## 결과 / 검증 방법
- ECS task가 `PENDING → RUNNING` 상태로 전환되는지 확인했다.
- CloudWatch Logs로 Spring Boot 기동 로그를 확인했다.
- 브라우저에서 `http://<EC2_PUBLIC_IP>`로 접속해 정적 프론트와 API 호출 흐름을 확인했다.

## 배운 점 / 다음에 또 마주치면
- ECS EC2에서 capacity provider가 있다는 것과 실제 container instance가 등록되어 있다는 것은 다르다.
- 단일 EC2 인스턴스 환경에서는 task placement 오류가 대부분 CPU/memory/port/GPU requirement 중 하나로 좁혀진다.
- digest 고정은 재현성에는 좋지만, 빠른 배포 확인 중에는 새 이미지 반영 여부를 헷갈리게 만든다.

## 관련 노트
- [[03-decisions/infra/ECS EC2 단일 인스턴스 최소 비용 배포 결정]]
- [[02-knowledge/infra/ECS S3 환경변수 파일 placeholder 오류]]
