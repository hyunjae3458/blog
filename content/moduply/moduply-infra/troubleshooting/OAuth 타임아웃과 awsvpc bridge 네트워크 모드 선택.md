---
tags: [infra, aws, ecs, oauth, awsvpc, bridge, nat, troubleshooting]
date: 2026-07-16
status: 해결됨
---

# OAuth 타임아웃과 awsvpc bridge 네트워크 모드 선택

## 한 줄 요약
> App task를 awsvpc + public IP disabled로 실행하자 NAT Gateway가 없는 환경에서 Google OAuth 토큰 요청이 외부로 나가지 못했고, 비용 절감을 위해 App만 bridge 모드로 전환해 EC2의 public egress를 사용하도록 했다.

## 증상

OAuth 로그인 시 다음 오류가 발생했다.

```text
An error occurred while attempting to retrieve the OAuth 2.0 Access Token Response:
I/O error on POST request for "https://www.googleapis.com/oauth2/v4/token":
Operation timed out
```

로그인 요청 자체는 App까지 도달했지만, App이 Google OAuth token endpoint로 나가는 outbound 요청에서 timeout이 발생했다.

## 원인

처음 App service는 다음 구조였다.

```text
App task network mode: awsvpc
Assign public IP: disabled
NAT Gateway: 없음
```

awsvpc 모드의 ECS task는 task ENI를 통해 네트워크를 사용한다. public IP가 없고 NAT Gateway도 없으면 외부 인터넷으로 나갈 수 없다.

```text
사용자 → ALB → App
```

인바운드는 정상이어도,

```text
App → Google OAuth
```

아웃바운드는 실패한다.

## 선택지

| 대안 | 장점 | 단점 |
|---|---|---|
| NAT Gateway 추가 | 실무형 private subnet 구조에 가장 적합 | 상시 운영 비용 증가 |
| NAT Instance 구성 | NAT Gateway보다 저렴할 수 있음 | 운영/장애 관리 부담 |
| App task를 bridge로 전환 | 비용 증가 없이 EC2 public egress 사용 가능 | task 단위 SG 분리 약화, target group 변경 필요 |

현재는 부트캠프 포트폴리오용 상시 운영 비용 절감이 우선이므로 App만 bridge로 전환했다.

## 최종 선택

```text
Spring App
- network mode: bridge
- host port: 8080
- target group: Instance 타입
- ALB → EC2:8080 → App container

Redis
- network mode: awsvpc 유지
- Redis task private IP를 App 환경변수에 임시 사용
```

bridge 모드에서는 App 컨테이너가 EC2 인스턴스의 네트워크 뒤에 붙는다. EC2에 public IPv4와 인터넷 라우트가 있으므로 Google OAuth 같은 외부 API 호출이 가능하다.

## 변경한 ALB / Target Group 설정

기존 awsvpc 구조:

```text
Target group type: IP
ALB → App task private IP:8080
```

bridge 전환 후:

```text
Target group type: Instance
ALB → App EC2 private IP:8080
```

Target Group은 생성 후 type 변경이 불가능하므로 새로 만들었다.

```text
moduply-app-instance-tg
- Target type: Instances
- Protocol: HTTP
- Port: 8080
- Health check path: /swagger-ui/index.html
- Success codes: 200-399
```

## 주의점

bridge + host port 8080 구조에서는 한 EC2에 App task를 여러 개 띄울 수 없다.

```text
App EC2 1대당 App task 1개
```

발표 전 1만 트래픽 테스트 때는 App ASG와 App service desired count를 같이 늘린다.

```text
App ASG desired: 2~3
App service desired: 2~3
```

## 추후 개선

실무형 구조로 가려면 App을 다시 awsvpc/private subnet 구조로 돌리고 NAT Gateway 또는 VPC Endpoint를 붙인다.

```text
Private App Subnet
→ NAT Gateway
→ Google OAuth / S3 / 외부 API
```

상시 운영 비용이 허용되면 이 방식이 더 권장된다.
