---
tags: [infra, aws, ecs, ec2, nginx, reverse-proxy, docker]
date: 2026-07-09
status: 진행중
---

# ECS EC2 Nginx 사이드카 리버스 프록시 흐름

## 한 줄 요약
> ECS EC2의 App Task 안에 Spring과 Nginx 컨테이너를 함께 배치하고, Nginx만 EC2의
> 80/443 포트를 사용하게 한 뒤 Docker `bridge` 링크의 `app` 별칭으로 Spring 8080
> 포트에 요청을 전달한다.

## 배경 / 어떤 상황이었는가
현재 배포 환경은 ALB 없이 단일 `t3.small` ECS EC2 인스턴스를 사용한다.

```text
ASG min/desired/max: 1/1/1
ALB: 없음
Public subnet + Public IP
App ECS Service
Redis ECS Service
```

기존에는 EC2 host 80 포트를 Spring 컨테이너의 8080 포트에 직접 연결했다.

```text
EC2 host 80 → Spring container 8080
```

OAuth와 Refresh Token의 `Secure` 쿠키를 정상 운영하려면 HTTPS가 필요하다. ALB와 ACM을
도입하지 않고 Nginx와 Let's Encrypt를 사용하기로 하면서, 외부 요청을 Nginx가 먼저
받고 Spring으로 전달하는 리버스 프록시 구조가 필요해졌다.

## 선택한 컨테이너 구조
Redis는 App 배포와 생명주기가 다르므로 별도 Service와 Task를 유지한다. Nginx는 App
트래픽을 보조하고 항상 같은 EC2에서 Spring과 함께 동작해야 하므로 App Task의
사이드카로 배치한다.

```text
App ECS Service
└─ App Task
   ├─ moduply-nginx
   └─ moduply-app

Redis ECS Service
└─ Redis Task
   └─ Redis
```

사이드카는 별도 ECS Service를 추가하는 것이 아니라 하나의 Task Definition 안에 보조
컨테이너를 함께 정의하는 방식이다. 두 컨테이너는 같이 시작·종료되고 같은 EC2에
배치된다.

## 전체 요청 흐름

```text
브라우저
  │ http://moduply.co.kr
  ▼
DNS
  │ moduply.co.kr → EC2 Public IP
  ▼
EC2 Security Group
  │ TCP 80 허용
  ▼
EC2 host port 80
  │ ECS port mapping
  ▼
Nginx container port 80
  │ proxy_pass http://app:8080
  ▼
Docker bridge network
  │ app → moduply-app 컨테이너
  ▼
Spring container port 8080
```

응답은 반대 순서로 Spring → Nginx → EC2 → 브라우저로 돌아간다. 브라우저는 내부에
컨테이너가 두 개 있다는 사실을 알지 못하며 `moduply.co.kr` 하나만 서버로 인식한다.

## DNS와 외부 포트
DNS의 `moduply.co.kr`은 EC2 Public IP를 가리킨다. 브라우저가 HTTP 주소에 접근하면
기본 포트인 80으로 요청한다.

```text
moduply.co.kr → EC2_PUBLIC_IP
HTTP 요청     → EC2_PUBLIC_IP:80
```

EC2 Security Group은 TCP 80을 허용해야 한다. HTTPS 적용 후에는 TCP 443도 허용한다.

## ECS 포트 소유권 변경
host port는 같은 EC2에서 하나의 컨테이너만 사용할 수 있다. 따라서 Spring이 사용하던
host 80 매핑을 제거하고 Nginx가 가져간다.

```text
변경 전
EC2 host 80 → Spring container 8080

변경 후
EC2 host 80  → Nginx container 80
EC2 host 443 → Nginx container 443  # 인증서 적용 후
Nginx        → Spring container 8080
```

Spring의 8080 포트는 외부 host port에 고정 매핑하지 않는다. Nginx와 Spring은 같은
Task의 Docker 내부 네트워크로 통신한다.

## bridge 링크가 컨테이너를 연결하는 방식
App Task의 네트워크 모드는 `bridge`다. Nginx 컨테이너의 `링크된 컨테이너` 설정에
다음 값을 추가했다.

```text
실제 컨테이너 이름: moduply-app
Nginx 내부 별칭: app
```

문자열 표현은 다음과 같다.

```text
moduply-app:app
```

이 설정은 Task Definition에서 다음 값에 해당한다.

```json
{
  "links": [
    "moduply-app:app"
  ]
}
```

Docker는 Nginx 컨테이너 내부의 `app` 이름을 `moduply-app` 컨테이너의 내부 주소로
해석한다. 따라서 Nginx는 Spring 컨테이너의 실제 IP를 몰라도 다음 주소를 사용할 수
있다.

```nginx
proxy_pass http://app:8080;
```

`links`는 Nginx 컨테이너에만 설정한다. 두 컨테이너가 같은 Task Definition에 포함되고
Task의 네트워크 모드가 `bridge`여야 한다.

## Nginx 핵심 설정

```nginx
server {
    listen 80;
    server_name moduply.co.kr;

    location = /nginx-health {
        access_log off;
        default_type text/plain;
        return 200 "ok";
    }

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location /ws {
        proxy_pass http://app:8080;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    location / {
        proxy_pass http://app:8080;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 경로별 역할
- `/nginx-health`: Spring을 거치지 않고 Nginx 생존 여부만 검사한다.
- `/.well-known/acme-challenge/`: Certbot HTTP-01 도메인 검증 파일을 제공한다.
- `/ws`: WebSocket Upgrade 헤더와 긴 timeout을 적용해 STOMP 연결을 프록시한다.
- `/`: API와 Spring이 제공하는 SPA 정적 파일을 포함한 나머지 요청을 전달한다.

## 컨테이너 리소스 배분
`t3.small`을 유지하면서 App Task의 기존 1 vCPU, 1.5GB 범위 안에서 Spring과 Nginx를
비대칭으로 배분한다.

| 컨테이너 | CPU | Soft Memory | Hard Memory |
|---|---:|---:|---:|
| Spring | 0.875 vCPU | 1GB | 1.375GB |
| Nginx | 0.125 vCPU | 0.0625GB | 0.125GB |

Nginx의 소프트 메모리 `0.0625GB`를 ECS 콘솔에서 입력할 수 없다면 생략하고 hard
limit `0.125GB`만 적용한다.

## ECS 콘솔 최종 설정값

### App Task Definition

```text
Launch type: EC2
Network mode: bridge
Task CPU: 1 vCPU
Task Memory: 1.5GB
```

Redis는 App 배포 때 같이 재시작되면 안 되므로 기존 Redis Service와 Redis Task를
유지한다. App Task에는 Spring과 Nginx만 포함한다.

### Spring 컨테이너

```text
컨테이너 이름: moduply-app
필수 컨테이너: 예
CPU: 0.875 vCPU
메모리 소프트 제한: 1GB
메모리 하드 제한: 1.375GB
컨테이너 포트: 8080
host port 80 매핑: 제거
```

기존에는 Spring이 `host 80 → container 8080` 매핑을 사용했지만, 리버스 프록시 도입
후 host 80의 소유자는 Nginx로 변경했다.

### Nginx 컨테이너

```text
컨테이너 이름: moduply-nginx
필수 컨테이너: 예
CPU: 0.125 vCPU
메모리 소프트 제한: 0.0625GB 또는 생략
메모리 하드 제한: 0.125GB
컨테이너 포트: 80
호스트 포트: 80
프로토콜: TCP
```

인증서 적용 후에는 다음 매핑을 추가한다.

```text
컨테이너 포트: 443
호스트 포트: 443
프로토콜: TCP
```

Nginx 컨테이너의 링크 설정:

```text
대상 컨테이너: moduply-app
별칭: app
```

Task Definition 표현:

```json
{
  "links": [
    "moduply-app:app"
  ]
}
```

Nginx 컨테이너의 시작 종속성:

```text
대상 컨테이너: moduply-app
조건: START
```

`START`는 Spring Boot가 요청을 받을 준비까지 끝났다는 의미가 아니라 컨테이너
프로세스가 시작됐다는 의미다. 링크 주소가 정상이라면 Spring 준비 전에는 잠시 502가
발생할 수 있지만 Nginx가 종료되지는 않는다.

### 현재 생략한 옵션

비용과 초기 설정 복잡도를 줄이기 위해 다음 항목은 현재 생략했다.

```text
ECS Nginx Health Check: 미설정
CloudWatch awslogs: 미설정
```

Nginx는 `essential=true`이므로 프로세스가 종료되면 ECS가 Task 전체를 교체한다. 다만
프로세스가 살아 있으면서 응답하지 않는 상태는 Health Check 없이는 감지하지 못한다.

로그는 Session Manager로 EC2에 접속한 뒤 확인한다.

```bash
sudo docker ps -a
sudo docker logs <nginx-container-id>
```

CloudWatch를 사용하지 않으면 Task 정리 후 과거 로그가 사라지고 Docker 로그가 EC2
디스크를 사용할 수 있으므로 로그 회전과 디스크 사용량을 별도로 확인해야 한다.

## 실제 이미지 구성
Nginx 공식 이미지를 그대로 사용하면 프로젝트의 프록시 설정이 포함되지 않는다.
별도의 Dockerfile로 설정 파일을 포함한 이미지를 만들고 ECR에 푸시한다.

```dockerfile
FROM nginx:1.27-alpine

COPY conf.d/default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
```

ECS Task Definition의 Nginx 컨테이너에는 해당 ECR Image URI를 지정한다.

### 이미지 아키텍처
개발 머신이 Apple Silicon이라 기본 빌드 결과는 `linux/arm64`였다. ECS의 `t3.small`은
`linux/amd64`이므로 ARM 이미지를 실행할 때 다음 오류가 발생했다.

```text
CannotPullContainerError:
no matching manifest for linux/amd64 in the manifest list entries
```

ECS용 이미지는 플랫폼을 명시해 ECR로 직접 빌드·푸시한다.

```bash
docker buildx build \
  --platform linux/amd64 \
  -f nginx/Dockerfile \
  -t "$ECR_URI:latest" \
  --push \
  nginx
```

이미지 플랫폼은 다음 명령으로 확인한다.

```bash
docker buildx imagetools inspect "$ECR_URI:latest"
```

결과에 `linux/amd64`가 있어야 한다. Nginx 이미지가 실행되지 않아 필수 컨테이너가
종료되면 ECS가 같은 Task의 Spring 컨테이너에도 종료 신호를 보낸다. 이때 Spring 종료
과정에서 발생하는 JPA 예외는 근본 원인이 아니라 Task 종료에 따른 2차 증상일 수 있다.

## HTTPS 적용 후 흐름
인증서 적용 후에는 Nginx가 443에서 TLS를 종료하고 같은 Docker 내부 HTTP 연결로
Spring에 전달한다.

```text
브라우저 HTTPS 443
→ EC2 host 443
→ Nginx container 443에서 TLS 처리
→ http://app:8080
→ Spring
```

외부 구간은 HTTPS로 암호화하고, 같은 EC2의 Docker bridge 내부 구간은 HTTP를
사용하는 일반적인 TLS termination 구조다.

## 검증 방법
최초 HTTP Nginx 배포 후 다음 순서로 확인한다.

```text
http://moduply.co.kr/nginx-health → 200, body: ok
http://moduply.co.kr/             → SPA 화면
http://moduply.co.kr/api/...      → Spring API 응답
/ws                               → WebSocket Upgrade 성공
```

문제가 발생하면 다음을 순서대로 확인한다.

1. DNS가 현재 EC2 Public IP를 가리키는가
2. Security Group이 80을 허용하는가
3. Nginx가 host 80을 사용하고 있는가
4. Spring의 기존 host 80 매핑을 제거했는가
5. Nginx 링크가 `moduply-app:app`으로 설정됐는가
6. 실제 Spring 컨테이너 이름이 `moduply-app`과 일치하는가
7. Spring이 컨테이너 내부 8080에서 실행 중인가

## 배운 점 / 다음에 또 마주치면
- DNS는 EC2까지 찾아오는 역할이고, ECS port mapping은 host 요청을 Nginx
  컨테이너로 전달하는 역할이다.
- Nginx의 `proxy_pass`는 외부 도메인이 아니라 Docker 내부의 컨테이너 별칭을
  대상으로 한다.
- 하나의 host port는 한 컨테이너만 사용할 수 있으므로 프록시 도입 시 포트 소유권을
  먼저 정리해야 한다.
- 같은 Task에 넣는 이유는 리소스 절감보다 공통 생명주기와 안정적인 내부 네트워크
  연결에 있다.

## 관련 노트
- [[02-knowledge/infra/ECS EC2 bridge mode에서 Redis 연결하기]]
- [[02-knowledge/infra/ECS EC2 첫 배포 트러블슈팅]]
- [[02-knowledge/auth/HTTP 배포에서 OAuth Refresh Token 쿠키가 누락된 원인]]
- [[03-decisions/infra/ECS EC2 단일 인스턴스 최소 비용 배포 결정]]
