---
tags: [infra, aws, ecs, ec2, resource, deployment]
date: 2026-07-08
status: 해결됨
---

# 단일 EC2 ECS 클러스터의 task placement와 리소스 부족

## 한 줄 요약
> `t3.small` 1대에서 앱과 Redis를 함께 운영하면 CPU/memory 예약량과 배포 중 동시 task 개수를 반드시 계산해야 한다.

## 배경 / 어떤 상황이었는가
비용 절감을 위해 ECS EC2 클러스터를 `t3.small` 1대로 운영했다. 여기에 Spring Boot 앱 task와 Redis task를 함께 배치했다.

## 문제 1: memory 부족

```text
insufficient memory available
```

### 원인
앱 task와 Redis task의 memory hard limit/reservation 합이 `t3.small`의 여유 메모리를 초과했다. EC2 2GB 전체를 컨테이너에 다 줄 수는 없다. OS, Docker, ECS Agent도 메모리를 사용한다.

### 해결
Redis task/container memory를 낮춰 배치 가능하게 조정했다.

## 문제 2: CPU units 부족

```text
insufficient CPU units available
```

### 원인
Redis task가 이미 떠 있는 상태에서 앱 서비스를 force deployment하면 기존 app task와 새 app task가 동시에 배치되려고 했다.

```text
기존 app task
+ 새 app task
+ redis task
> t3.small CPU units
```

### 해결
단일 EC2에서 무중단 배포를 포기하고 먼저 내린 뒤 올리는 배포 전략으로 조정했다.

```text
Availability Zone rebalancing: off
minimum healthy percent: 0
maximum percent: 100
```

## 실제 판단
이 구성은 비용 절감을 위한 임시 운영 구성이다. 트래픽이 늘거나 QA 범위가 넓어지면 다음 중 하나가 필요하다.

| 대안 | 장점 | 단점 |
|---|---|---|
| 앱/Redis 리소스 계속 줄이기 | 비용 유지 | OOM/성능 저하 가능성 증가 |
| EC2를 `t3.medium`으로 증설 | 운영 여유 증가 | 비용 증가 |
| Redis를 ElastiCache로 분리 | 운영 안정성 증가 | 별도 비용 발생 |
| ALB + 다중 인스턴스 | 배포 안정성/확장성 증가 | 비용과 설정 복잡도 증가 |

## 결과 / 검증 방법
- Redis service와 app service가 모두 `RUNNING`인지 확인했다.
- 로그인과 API 호출로 앱/Redis 연동을 확인했다.
- CloudWatch Logs에서 CPU/memory placement 오류가 재발하지 않는지 확인했다.

## 배운 점 / 다음에 또 마주치면
- ECS task의 CPU/memory 값은 실제 사용량이 아니라 scheduler가 배치 판단에 사용하는 예약량이다.
- 단일 인스턴스에서 rolling deployment를 하면 기존 task와 새 task가 동시에 떠야 하므로 여유 리소스가 필요하다.
- 비용 최적화 구성에서는 배포 무중단성과 리소스 여유를 일부 포기하게 된다.

## 관련 노트
- [[03-decisions/infra/ECS EC2 단일 인스턴스 최소 비용 배포 결정]]
- [[03-decisions/infra/Redis를 같은 ECS EC2 클러스터의 별도 서비스로 운영하기]]
