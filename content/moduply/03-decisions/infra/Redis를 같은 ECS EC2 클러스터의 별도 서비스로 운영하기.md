---
tags: [infra, aws, ecs, redis, decision]
date: 2026-07-08
status: 채택됨
---

# Redis를 같은 ECS EC2 클러스터의 별도 서비스로 운영하기로 한 결정

## 결정
초기 배포 단계에서는 ElastiCache나 별도 EC2를 만들지 않고, 기존 ECS EC2 클러스터 안에 Redis용 Task Definition과 Service를 추가한다.

```text
moduply-prod-cluster
├── moduply-service
│   └── Spring Boot app task
└── moduply-redis-service
    └── Redis task
```

## 배경 / 왜 이 결정이 필요했는가
로그인 과정에서 Redis 연결이 필요했다. Redis는 refresh token, 로그아웃/블랙리스트, 임시 비밀번호, 캐시성 데이터에 사용된다. 하지만 ElastiCache를 바로 붙이면 비용이 증가하고, 배포 검증 단계에서는 과했다.

## 고려한 대안

| 대안 | 장점 | 단점 |
|---|---|---|
| EC2에 `docker run redis` 수동 실행 | 가장 빠름 | ECS 관리 밖이라 재시작/복구/기록이 지저분함 |
| 같은 ECS 클러스터에 Redis Service 추가 (채택) | ECS가 Redis task를 관리, 비용 추가 거의 없음 | 앱과 Redis가 같은 EC2 리소스를 공유 |
| 별도 EC2 Redis | 앱과 리소스 분리 | EC2 비용 추가, 운영 관리 증가 |
| ElastiCache Redis | 운영 안정성 좋음 | 비용 증가 |

## 구현 방식
Redis Task Definition:

```text
Launch type: EC2
Network mode: bridge
Image: public.ecr.aws/docker/library/redis:7-alpine
Container port: 6379
Host port: 6379
```

앱 환경변수:

```env
REDIS_HOST=<EC2_PRIVATE_IP>
REDIS_PORT=6379
```

보안그룹:

```text
EC2 SG inbound 6379
Source: EC2 SG 자기 자신
```

## 트레이드오프 / 감수한 것
- Redis와 앱이 같은 EC2 메모리/CPU를 공유한다.
- 단일 EC2 장애 시 앱과 Redis가 함께 내려간다.
- Redis persistence와 운영 안정성은 ElastiCache보다 약하다.

## 재검토 시점
- 사용자 수가 늘어 refresh token/session/cache 데이터가 증가할 때
- Redis memory warning 또는 eviction이 발생할 때
- EC2 resource placement 실패가 반복될 때
- 도메인/HTTPS 이후 운영 안정성을 높이는 단계

## 관련 노트
- [[02-knowledge/infra/ECS EC2 bridge mode에서 Redis 연결하기]]
- [[02-knowledge/infra/단일 EC2 ECS task placement와 리소스 부족]]
