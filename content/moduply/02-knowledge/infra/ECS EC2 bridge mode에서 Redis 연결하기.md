---
tags: [infra, aws, ecs, redis, bridge-network, troubleshooting]
date: 2026-07-08
status: 해결됨
---

# ECS EC2 bridge mode에서 Redis 연결하기

## 한 줄 요약
> ECS EC2 bridge mode에서 앱 컨테이너의 `localhost`는 Redis 컨테이너가 아니라 앱 컨테이너 자신이므로, Redis는 EC2 private IP와 host port로 접근해야 한다.

## 배경 / 어떤 상황이었는가
운영 배포 후 로그인 과정에서 Redis 연결 실패가 발생했다. Redis는 refresh token, 로그아웃/블랙리스트, 임시 비밀번호, 캐시성 데이터에 사용된다.

처음에는 비용을 아끼기 위해 ElastiCache나 별도 EC2를 쓰지 않고, 기존 ECS EC2 클러스터 안에 Redis용 Service를 추가했다.

## 문제
Redis 컨테이너는 떠 있었지만 앱은 계속 `localhost:6379`로 접속하려 했다.

```text
Unable to connect to localhost/127.0.0.1:6379
```

## 원인
ECS EC2 `bridge` network mode에서는 각 컨테이너의 `localhost`가 자기 자신을 의미한다.

```text
app container localhost:6379
→ app container 내부 6379
→ Redis 없음
```

Redis task는 같은 EC2 host의 `6379` host port로 열려 있으므로, 앱은 EC2 private IP로 접근해야 한다.

```text
app container
→ <EC2_PRIVATE_IP>:6379
→ redis container
```

## 실제 설정
Redis Task Definition:

```text
Launch type: EC2
Network mode: bridge
Image: public.ecr.aws/docker/library/redis:7-alpine
Container port: 6379
Host port: 6379
```

앱 S3 env file:

```env
REDIS_HOST=<EC2_PRIVATE_IP>
REDIS_PORT=6379
```

EC2 보안그룹 inbound:

```text
Custom TCP 6379
Source: EC2 security group 자기 자신
```

## 왜 private IP를 썼는가
Redis는 외부 사용자가 직접 접근할 서비스가 아니다. Refresh Token, 임시 비밀번호, 세션/캐시성 데이터가 들어갈 수 있으므로 public으로 열면 안 된다.

```text
6379 from 0.0.0.0/0 ❌
6379 from 내 IP ❌
6379 from EC2 자기 Security Group ✅
```

## 결과 / 검증 방법
- Redis service task가 `RUNNING` 상태인지 확인했다.
- 앱 service를 force deployment해 새 `REDIS_HOST`를 반영했다.
- 로그인 요청에서 Redis connection failure가 사라지고 로그인 성공을 확인했다.

## 배운 점 / 다음에 또 마주치면
- `bridge` mode에서는 컨테이너 간 통신을 `localhost`로 가정하면 안 된다.
- ECS Service를 분리하면 앱과 Redis의 생명주기를 따로 관리할 수 있어 편하다.
- 장기 운영에서는 같은 EC2에 Redis를 두는 임시 구성을 ElastiCache 또는 별도 Redis 구성으로 재검토해야 한다.

## 관련 노트
- [[03-decisions/infra/Redis를 같은 ECS EC2 클러스터의 별도 서비스로 운영하기]]
- [[02-knowledge/infra/단일 EC2 ECS task placement와 리소스 부족]]
