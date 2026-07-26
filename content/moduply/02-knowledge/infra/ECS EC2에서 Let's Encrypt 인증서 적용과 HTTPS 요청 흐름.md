---
tags: [infra, aws, ecs, ec2, nginx, lets-encrypt, certbot, https, tls]
date: 2026-07-13
status: 해결됨
---

# ECS EC2에서 Let's Encrypt 인증서 적용과 HTTPS 요청 흐름

## 한 줄 요약
> 단일 ECS EC2 환경에서 Certbot으로 Let's Encrypt 인증서를 EC2 호스트에 발급하고,
> `/etc/letsencrypt`, `/var/www/certbot`을 Nginx 컨테이너에 바인드 마운트해서
> Nginx가 80은 인증서 검증/HTTPS 리다이렉트, 443은 TLS 종료 후 Spring Boot로
> 프록시하도록 구성했다.

## 배경 / 어떤 상황이었는가
운영 도메인 `moduply.co.kr`에서 OAuth 로그인과 Refresh Token 쿠키를 정상적으로
사용하려면 HTTPS가 필요했다. 이전에는 HTTP 배포 상태라 `Secure` 쿠키를 사용할 수
없었고, Google OAuth 운영 리다이렉트 URI도 HTTPS가 필요했다.

현재 인프라 구조는 다음과 같다.

```text
ALB: 없음
ACM: 사용 안 함
ECS launch type: EC2
EC2: t3.small 단일 인스턴스
App Task: Spring Boot + Nginx 사이드카
Redis Task: Redis 단독
Nginx → Spring: Docker bridge link의 app 별칭 사용
```

ALB를 쓰지 않기 때문에 ACM 인증서를 ALB에 붙이는 방식이 아니라, EC2 호스트에
Let's Encrypt 인증서를 직접 발급하고 Nginx 컨테이너가 그 파일을 읽는 방식으로
진행했다.

## Certbot과 Let's Encrypt의 역할

### Let's Encrypt
Let's Encrypt는 실제 인증서를 발급하는 공개 인증기관(CA)이다. 브라우저가 신뢰하는
TLS 인증서를 무료로 발급해준다.

### Certbot
Certbot은 Let's Encrypt와 통신하는 ACME 클라이언트다. 직접 인증서를 발급하는
기관은 아니고, 다음 절차를 자동화한다.

```text
1. Let's Encrypt에 moduply.co.kr 인증서 발급 요청
2. Let's Encrypt가 제시한 HTTP-01 challenge 토큰 수신
3. challenge 파일을 webroot 경로에 생성
4. Let's Encrypt가 해당 파일을 HTTP로 확인
5. 검증 성공 시 인증서를 /etc/letsencrypt에 저장
```

## HTTP-01 challenge 동작 원리
HTTP-01 challenge는 "이 서버가 정말 `moduply.co.kr` 도메인을 제어하고 있는가"를
검증하는 절차다.

흐름은 다음과 같다.

```text
Certbot
→ /var/www/certbot/.well-known/acme-challenge/{랜덤토큰} 파일 생성

Let's Encrypt
→ http://moduply.co.kr/.well-known/acme-challenge/{랜덤토큰} 요청

Nginx
→ /var/www/certbot에서 해당 파일을 찾아 응답

Let's Encrypt
→ 응답 내용이 기대한 값과 일치하면 도메인 제어권 검증 성공
→ 인증서 발급
```

중요한 점은 Nginx가 인증 파일을 생성하는 것이 아니라는 점이다. 인증 파일은
Certbot이 생성하고, Nginx는 그 파일을 HTTP로 외부에 제공만 한다.

## 왜 ECS Task Definition에 볼륨 마운트가 필요한가
Certbot은 EC2 호스트에서 Docker 컨테이너로 실행했고, Nginx는 ECS Task 안의 별도
컨테이너로 실행된다. 둘은 기본적으로 서로 다른 파일 시스템을 본다.

따라서 EC2 호스트의 인증서 관련 디렉터리를 Nginx 컨테이너에 같은 경로로 마운트했다.

| EC2 호스트 경로 | Nginx 컨테이너 경로 | 역할 |
|---|---|---|
| `/var/www/certbot` | `/var/www/certbot` | HTTP-01 challenge 파일 공유 |
| `/etc/letsencrypt` | `/etc/letsencrypt` | 발급된 인증서와 개인키 공유 |

ECS 콘솔에서는 App Task Definition의 스토리지에서 다음 볼륨을 추가했다.

```text
볼륨 이름: certbot-webroot
구성 유형: 작업 생성 시 구성
볼륨 유형: 바인드 탑재
소스 경로: /var/www/certbot
```

```text
볼륨 이름: letsencrypt
구성 유형: 작업 생성 시 구성
볼륨 유형: 바인드 탑재
소스 경로: /etc/letsencrypt
```

그리고 Nginx 컨테이너의 마운트 지점에 연결했다.

```text
소스 볼륨: certbot-webroot
컨테이너 경로: /var/www/certbot
읽기 전용: true
```

```text
소스 볼륨: letsencrypt
컨테이너 경로: /etc/letsencrypt
읽기 전용: true
```

Spring Boot 컨테이너와 Redis 컨테이너에는 이 볼륨을 마운트하지 않는다. 인증서 파일은
Nginx만 읽으면 된다.

## EC2에서 인증서 발급 절차
먼저 EC2 호스트에 공유 디렉터리를 만들었다.

```bash
sudo mkdir -p /etc/letsencrypt
sudo mkdir -p /var/www/certbot
```

디렉터리 생성 여부는 절대경로로 확인한다.

```bash
ls -ld /etc/letsencrypt
ls -ld /var/www/certbot
```

Nginx HTTP 배포와 볼륨 마운트가 정상인지 먼저 확인했다.

```bash
curl http://moduply.co.kr/nginx-health
```

기대 응답:

```text
ok
```

그 다음 EC2 호스트에서 Certbot을 Docker 컨테이너로 실행했다.

```bash
sudo docker run --rm \
  -v /etc/letsencrypt:/etc/letsencrypt \
  -v /var/www/certbot:/var/www/certbot \
  certbot/certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email 본인_이메일@example.com \
  --agree-tos \
  --no-eff-email \
  -d moduply.co.kr
```

옵션의 의미:

| 옵션 | 의미 |
|---|---|
| `--webroot` | 이미 떠 있는 Nginx가 challenge 파일을 서빙하도록 하는 방식 |
| `--webroot-path=/var/www/certbot` | challenge 파일을 생성할 경로 |
| `-v /var/www/certbot:/var/www/certbot` | Certbot과 Nginx가 같은 challenge 파일을 보도록 공유 |
| `-v /etc/letsencrypt:/etc/letsencrypt` | 발급된 인증서를 EC2 호스트에 영속 저장 |
| `--email` | 만료/갱신 실패/보안 공지 수신용 계정 이메일 |
| `--no-eff-email` | EFF 홍보성 메일 수신 거부 |
| `-d moduply.co.kr` | 인증서를 발급받을 도메인 |

성공 후 다음 파일들이 생성된다.

```text
/etc/letsencrypt/live/moduply.co.kr/fullchain.pem
/etc/letsencrypt/live/moduply.co.kr/privkey.pem
```

확인 명령:

```bash
sudo ls -l /etc/letsencrypt/live/moduply.co.kr/
```

## Nginx default.conf 설정 설명
인증서 발급 후 Nginx 설정을 HTTP-only에서 HTTPS용으로 변경했다.

핵심은 server block을 2개로 나누는 것이다.

```text
server listen 80
→ Let's Encrypt challenge와 HTTP→HTTPS 리다이렉트 담당

server listen 443 ssl
→ TLS 종료 후 Spring Boot로 프록시
```

실제 설정의 핵심 구조는 다음과 같다.

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

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

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name moduply.co.kr;

    ssl_certificate /etc/letsencrypt/live/moduply.co.kr/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/moduply.co.kr/privkey.pem;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_protocols TLSv1.2 TLSv1.3;

    server_tokens off;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location = /nginx-health {
        access_log off;
        default_type text/plain;
        return 200 "ok";
    }

    location /ws {
        proxy_pass http://app:8080;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    location /api/sse {
        proxy_pass http://app:8080;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;

        proxy_buffering off;
        proxy_cache off;
    }

    location / {
        proxy_pass http://app:8080;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### 80 포트 server block
80 포트의 역할은 HTTPS 제공이 아니라 "인증서 발급/갱신을 위한 HTTP 경로 유지"와
"일반 HTTP 요청을 HTTPS로 보내는 것"이다.

```nginx
location /.well-known/acme-challenge/ {
    root /var/www/certbot;
}
```

이 설정 때문에 Let's Encrypt가 다음 URL로 접근했을 때 Certbot이 만든 파일을 받을 수
있다.

```text
http://moduply.co.kr/.well-known/acme-challenge/{토큰}
```

그 외 HTTP 요청은 전부 HTTPS로 리다이렉트한다.

```nginx
location / {
    return 301 https://$host$request_uri;
}
```

예시:

```text
http://moduply.co.kr/api/auth/me
→ 301
→ https://moduply.co.kr/api/auth/me
```

### 443 포트 server block
443 포트는 실제 HTTPS 요청을 처리한다.

```nginx
ssl_certificate /etc/letsencrypt/live/moduply.co.kr/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/moduply.co.kr/privkey.pem;
```

`fullchain.pem`은 서버 인증서와 중간 인증서 체인을 포함한다. 브라우저는 이 체인을
통해 서버 인증서가 신뢰 가능한 CA인 Let's Encrypt로 이어지는지 검증한다.

`privkey.pem`은 서버 개인키다. 외부로 유출되면 안 되므로 EC2 호스트와 Nginx 컨테이너
안에서만 읽히도록 관리해야 한다.

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

오래된 TLS 버전을 비활성화하고 현대 브라우저가 사용하는 TLS 1.2/1.3만 허용한다.

### `/nginx-health`
Nginx가 직접 응답한다.

```nginx
location = /nginx-health {
    access_log off;
    default_type text/plain;
    return 200 "ok";
}
```

Spring Boot를 거치지 않기 때문에 이 엔드포인트는 "Nginx 컨테이너가 요청을 받을 수
있는가"만 확인한다. Spring Boot 애플리케이션의 readiness를 보장하지는 않는다.

### `/ws`
WebSocket/STOMP 요청은 HTTP 연결을 WebSocket으로 업그레이드해야 하므로 별도 설정을
둔다.

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

처음에는 `Connection "upgrade"`를 하드코딩했지만 CodeRabbit 피드백을 받고 `map`을
사용하도록 변경했다. 이 프로젝트의 WebSocket 엔드포인트는 Spring 설정에서
`.withSockJS()`를 사용한다. 따라서 `/ws` 아래로 순수 WebSocket upgrade 요청뿐 아니라
SockJS HTTP fallback 요청도 들어올 수 있다.

예를 들면 다음 요청들은 WebSocket upgrade가 아니라 일반 HTTP 요청이다.

```text
/ws/info
/ws/{server-id}/{session-id}/xhr
/ws/{server-id}/{session-id}/xhr_streaming
/ws/{server-id}/{session-id}/eventsource
```

이런 요청에까지 무조건 `Connection: upgrade`를 전달하면 upstream인 Spring Boot가
요청 의미를 잘못 해석할 여지가 있다. `map`을 사용하면 요청에 `Upgrade` 헤더가 있을
때만 `Connection: upgrade`를 전달하고, 없으면 `Connection: close`를 전달한다.

```text
Upgrade 헤더 있음
→ WebSocket upgrade 요청
→ Connection: upgrade

Upgrade 헤더 없음
→ SockJS HTTP fallback 또는 일반 HTTP 요청
→ Connection: close
```

`map`은 `location`이나 `server` 블록 안에 둘 수 없다. 현재 파일
`nginx/conf.d/default.conf`는 공식 Nginx 이미지의 `http {}` 블록 안에서 include되므로
파일 최상단에 두는 것이 맞다.

또한 WebSocket은 장시간 연결이 유지되므로 timeout을 길게 잡았다.

```nginx
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
```

### `/api/sse`
SSE 연결은 일반 HTTP 요청이지만 연결이 오래 유지되고 서버가 이벤트를 즉시 흘려보내야
한다. 그래서 `/api/sse`는 일반 `/` 프록시와 분리했다.

```nginx
location /api/sse {
    proxy_pass http://app:8080;
    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;

    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;

    proxy_buffering off;
    proxy_cache off;
}
```

`proxy_buffering off`를 둔 이유는 Nginx가 SSE 이벤트를 버퍼에 쌓아두지 않고
클라이언트에 즉시 전달하게 하기 위해서다.

### `/`
그 외 모든 요청은 Spring Boot로 전달한다. API와 SPA 정적 파일 요청이 이 블록을 통해
전달된다.

```nginx
location / {
    proxy_pass http://app:8080;
}
```

`app`은 외부 DNS가 아니라 ECS bridge link로 설정한 Spring Boot 컨테이너 별칭이다.

```text
moduply-nginx 컨테이너
→ app:8080
→ moduply-app 컨테이너의 8080 포트
```

SSE는 `/api/sse` location을 별도로 분리했고, WebSocket/SockJS는 `/ws` location에서
처리한다.

## 클라이언트 요청부터 Spring Boot 전달까지 전체 흐름

### 일반 HTTP 요청

```text
브라우저
→ http://moduply.co.kr
→ DNS가 EC2 Public IP로 해석
→ EC2 Security Group TCP 80 허용
→ ECS host port 80
→ Nginx container port 80
→ location / 매칭
→ 301 https://moduply.co.kr 로 리다이렉트
```

### HTTPS 요청

```text
브라우저
→ https://moduply.co.kr
→ DNS가 EC2 Public IP로 해석
→ EC2 Security Group TCP 443 허용
→ ECS host port 443
→ Nginx container port 443
→ Nginx가 fullchain.pem / privkey.pem으로 TLS handshake 수행
→ HTTPS 연결 성립
→ HTTP 요청 내용을 Nginx가 읽음
→ location / 또는 /ws 매칭
→ proxy_pass http://app:8080
→ Docker bridge network
→ Spring Boot moduply-app 컨테이너 8080
```

즉 외부 구간은 HTTPS로 암호화되고, Nginx와 Spring Boot 사이의 같은 EC2 내부 Docker
bridge 구간은 HTTP로 통신한다. 이것이 TLS termination 구조다.

### WebSocket 요청

```text
브라우저
→ wss://moduply.co.kr/ws
→ Nginx 443에서 TLS 처리
→ /ws location 매칭
→ Upgrade / Connection 헤더 유지
→ http://app:8080/ws 로 전달
→ Spring WebSocket/STOMP 핸들러가 처리
```

브라우저 입장에서는 `wss://` 보안 WebSocket이지만, 내부적으로는 Nginx가 TLS를
종료한 뒤 Spring Boot에 일반 HTTP WebSocket upgrade 요청으로 전달한다.

### SockJS HTTP fallback 요청

```text
브라우저
→ https://moduply.co.kr/ws/info 또는 /ws/.../xhr
→ Nginx 443에서 TLS 처리
→ /ws location 매칭
→ Upgrade 헤더가 없으므로 Connection: close 전달
→ http://app:8080/ws/... 로 일반 HTTP 요청 전달
→ Spring SockJS 핸들러가 처리
```

이 흐름 때문에 `Connection "upgrade"` 하드코딩보다 `$connection_upgrade` 변수를 쓰는
것이 더 정확하다.

## ECS와 Docker 이미지 변경
Nginx Dockerfile에는 443 포트를 문서화했다.

```dockerfile
FROM nginx:1.27-alpine

COPY conf.d/default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80 443
```

`EXPOSE`는 실제 포트를 여는 설정이 아니라 이미지 문서화에 가깝다. 실제 외부 공개는
ECS Task Definition의 포트 매핑과 EC2 Security Group이 결정한다.

Nginx 컨테이너 포트 매핑:

```text
80  TCP, port name: http,  app protocol: HTTP
443 TCP, port name: https, app protocol: HTTP
```

443도 앱 프로토콜은 `HTTP`로 둔다. ECS의 앱 프로토콜은 컨테이너가 HTTP 계열 트래픽을
처리한다는 힌트이고, 실제 TLS 처리는 Nginx가 수행한다.

Apple Silicon에서 빌드하므로 ECR에는 `linux/amd64` 이미지로 푸시해야 한다.

```bash
docker buildx build \
  --platform linux/amd64 \
  -f nginx/Dockerfile \
  -t "$ECR_URI:latest" \
  --push \
  nginx
```

## 검증 방법

배포 후 다음을 확인한다.

```bash
curl -I http://moduply.co.kr
```

기대 결과:

```text
HTTP/1.1 301 Moved Permanently
Location: https://moduply.co.kr/...
```

HTTPS health check:

```bash
curl https://moduply.co.kr/nginx-health
```

기대 결과:

```text
ok
```

인증서 파일 확인:

```bash
sudo ls -l /etc/letsencrypt/live/moduply.co.kr/
```

브라우저 확인:

```text
https://moduply.co.kr
```

주소창에 자물쇠가 표시되고 인증서 발급자가 Let's Encrypt 계열로 보이면 정상이다.

## 주의할 점
- `/etc/letsencrypt` 마운트가 빠지면 Nginx는 시작 시 인증서 파일을 읽지 못해 바로
  종료된다.
- 80 포트는 HTTPS 적용 후에도 닫으면 안 된다. Let's Encrypt 인증서 갱신의 HTTP-01
  challenge에 계속 필요하다.
- 현재 방식은 단일 EC2 인스턴스에서는 단순하고 적절하지만, EC2 인스턴스를 여러 대로
  늘리면 각 인스턴스의 로컬 `/etc/letsencrypt` 상태가 달라질 수 있다. 다중 인스턴스
  구조에서는 ALB + ACM 또는 EFS 공유 볼륨을 검토해야 한다.
- HTTPS 적용 후에는 임시로 제거했던 Refresh Token 쿠키의 `Secure` 옵션을 복구해야
  한다.
- Google/Kakao OAuth redirect URI도 `https://moduply.co.kr/login/oauth2/code/...`로
  맞춰야 한다.
- Spring Security가 프록시 뒤의 원본 scheme을 제대로 인식하지 못하면 OAuth redirect
  URI를 `http`로 생성할 수 있으므로 `X-Forwarded-Proto` 전달과 Spring의 forwarded
  header 설정을 함께 확인해야 한다.
- HSTS(`Strict-Transport-Security`)는 브라우저가 해당 도메인을 HTTPS로만 접속하도록
  캐시하는 정책이다. `includeSubDomains`까지 적용하면 하위 도메인도 HTTPS가 강제되므로
  운영 HTTPS가 안정화된 뒤 유지 여부를 결정해야 한다.

## 배운 점 / 다음에 또 마주치면
- Certbot은 인증서를 발급하는 CA가 아니라 Let's Encrypt와 통신하는 ACME 클라이언트다.
- HTTP-01 challenge는 "파일을 특정 HTTP URL에서 제공할 수 있는가"로 도메인 제어권을
  확인한다.
- ECS 컨테이너와 EC2 호스트는 파일 시스템이 분리되어 있으므로, Certbot이 만든 파일을
  Nginx가 보려면 바인드 마운트가 필요하다.
- Nginx의 80 포트는 HTTPS 적용 후에도 challenge와 redirect 용도로 유지해야 한다.
- 외부 HTTPS를 Nginx에서 종료하고 내부 Spring Boot로 HTTP 전달하는 구조를 TLS
  termination이라고 부른다.
- SockJS를 사용하는 `/ws` 엔드포인트에서는 WebSocket upgrade 요청과 HTTP fallback
  요청이 함께 들어올 수 있으므로 `Connection: upgrade`를 하드코딩하지 않고 `map`으로
  분기하는 것이 정확하다.
- SSE는 버퍼링이 켜져 있으면 이벤트 전달이 지연될 수 있으므로 별도 location에서
  `proxy_buffering off`를 명시하는 것이 안전하다.

## 관련 노트
- [[02-knowledge/infra/ECS EC2 Nginx 사이드카 리버스 프록시 흐름]]
- [[02-knowledge/auth/HTTP 배포에서 OAuth Refresh Token 쿠키가 누락된 원인]]
- [[03-decisions/infra/ECS EC2 단일 인스턴스 최소 비용 배포 결정]]
