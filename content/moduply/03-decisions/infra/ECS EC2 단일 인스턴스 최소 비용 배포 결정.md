---
tags: [infra, aws, ecs, ec2, decision, cost]
date: 2026-07-07
status: 채택됨
---

# ECS EC2 단일 인스턴스 최소 비용 배포 결정

## 결정
초기 운영 검증 단계에서는 Fargate 대신 ECS EC2 launch type을 사용하고, `t3.small` EC2 1대에서 애플리케이션을 실행한다.

```text
ASG min/desired/max: 1/1/1
ALB: 사용 안 함
NAT Gateway: 사용 안 함
Service Auto Scaling: off
Public subnet + public IP
Host port 80 → container port 8080
```

## 배경 / 왜 이 결정이 필요했는가
ECR에 Docker 이미지를 push한 뒤 실제 AWS에 배포해야 했다. 다만 프로젝트 단계상 트래픽보다 비용 절감이 우선이었다. ALB, NAT Gateway, Fargate를 모두 쓰면 안정성은 좋아지지만 고정 비용이 커진다.

## 고려한 대안

| 대안                              | 장점                  | 단점                                             |
| ------------------------------- | ------------------- | ---------------------------------------------- |
| Fargate                         | 인스턴스 관리 불필요, 배포 단순  | 실행 시간 과금, 리소스 추가 시 비용 증가                       |
| ECS EC2 `t3.micro`              | 비용 최저               | Spring Boot + ECS Agent + Docker에 메모리 부족 가능성 큼 |
| ECS EC2 `t3.small` 단일 인스턴스 (채택) | 비용 절감과 실행 안정성의 중간점  | 무중단 배포/확장성 부족                                  |
| ALB + 다중 EC2                    | 운영 안정성/HTTPS/확장성 좋음 | 비용과 설정 복잡도 증가                                  |

## 구현 방식
- ECS cluster에 EC2 capacity provider를 연결했다.
- Auto Scaling Group은 1대 고정으로 설정했다.
- Service desired task는 1로 유지했다.
- HTTP는 host port 80으로 직접 노출했다.
- SSH는 내 IP만 허용했다.

## 트레이드오프 / 감수한 것
- 단일 EC2라 장애 시 전체 서비스가 내려간다.
- ALB가 없으므로 HTTPS와 도메인 운영이 제한적이다.
- 배포 중 기존 task와 새 task를 동시에 띄우기 어렵다.
- CPU/memory 예약량을 매우 빡빡하게 관리해야 한다.

## 재검토 시점
- 도메인/HTTPS가 필요해지는 시점
- QA 중 OOM 또는 placement 실패가 반복되는 시점
- Redis, Batch, WebSocket 사용량이 늘어나는 시점

## 관련 노트
- [[02-knowledge/infra/ECS EC2 첫 배포 트러블슈팅]]
- [[02-knowledge/infra/단일 EC2 ECS task placement와 리소스 부족]]
