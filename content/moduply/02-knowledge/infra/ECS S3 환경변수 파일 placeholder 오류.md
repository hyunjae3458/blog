---
tags: [infra, aws, ecs, s3, environment-variable, troubleshooting]
date: 2026-07-07
status: 해결됨
---

# ECS S3 환경변수 파일과 placeholder 오류

## 한 줄 요약
> ECS는 S3 env file을 읽었지만, 파일 안의 변수명/값 형식이 Spring 설정과 맞지 않으면 `${POSTGRES_URL}` 같은 placeholder가 그대로 남아 부팅에 실패한다.

## 배경 / 어떤 상황이었는가
ECS Task Definition에서 S3 environment file을 연결해 운영 환경변수를 주입했다.

```json
"environmentFiles": [
  {
    "value": "arn:aws:s3:::moduply-bucket-<AWS_ACCOUNT_ID>-ap-northeast-2-an/env/.env",
    "type": "s3"
  }
]
```

## 문제
애플리케이션이 `prod` profile로 뜨긴 했지만 datasource URL이 정상 주입되지 않았다.

```text
Driver org.postgresql.Driver claims to not accept jdbcUrl, ${POSTGRES_URL}
```

OAuth URL도 같은 방식으로 실패했다.

```text
Could not resolve placeholder 'OAUTH_FAILURE_URL'
```

cron 설정도 따옴표 때문에 실패했다.

```text
invalid @Scheduled method ... cron expression ""0 0 2 * * *""
```

## 원인
S3 env file을 읽는 권한 문제는 아니었다. ECS agent 로그에 env file resource가 생성된 것이 확인됐다.

```text
envfile_moduply-app status="CREATED"
```

실제 원인은 Spring 설정에서 기대하는 변수명과 S3 env file의 변수명이 맞지 않거나, env 값에 따옴표가 포함된 것이었다.

## 실제 설정
`application-prod.yml`은 다음 변수명을 기대한다.

```yaml
spring:
  datasource:
    url: ${RDS_URL:${POSTGRES_URL}}
    username: ${RDS_USERNAME:${POSTGRES_USER_NAME}}
    password: ${RDS_PASSWORD:${POSTGRES_PASSWORD}}
```

따라서 S3 env file은 둘 중 하나로 통일해야 한다.

```env
RDS_URL=jdbc:postgresql://<RDS_ENDPOINT>:5432/postgres
RDS_USERNAME=<RDS 사용자>
RDS_PASSWORD=<RDS 비밀번호>
```

또는:

```env
POSTGRES_URL=jdbc:postgresql://<RDS_ENDPOINT>:5432/postgres
POSTGRES_USER_NAME=<RDS 사용자>
POSTGRES_PASSWORD=<RDS 비밀번호>
```

## env file 작성 시 주의한 점
ECS environment file은 일반 shell script가 아니다. `KEY=value` 형식으로 단순하게 작성해야 한다.

```env
CONTENT_IMPORT_BATCH_CRON=0 0 2 * * *
```

다음처럼 쓰면 값에 따옴표가 포함될 수 있다.

```env
CONTENT_IMPORT_BATCH_CRON="0 0 2 * * *"
```

SPA hash route에 들어가는 `#`는 주석으로 해석될 수 있으므로 `%23`으로 인코딩했다.

```env
OAUTH_SUCCESS_URL=http://<EC2_PUBLIC_IP>/%23/contents
OAUTH_FAILURE_URL=http://<EC2_PUBLIC_IP>/%23/sign-in
```

## 결과 / 검증 방법
- CloudWatch Logs에서 `The following 1 profile is active: "prod"`를 확인했다.
- datasource URL placeholder가 문자열 그대로 남지 않는지 로그로 확인했다.
- cron parsing 오류가 사라지는지 확인했다.

## 배운 점 / 다음에 또 마주치면
- `prod` profile이 잡혔다고 해서 모든 env 값이 정상 주입됐다고 단정하면 안 된다.
- S3 env file read 권한 문제라면 task가 시작 전 `ResourceInitializationError`로 실패한다.
- 컨테이너가 실행되고 Spring placeholder 오류가 난다면 env file 권한보다는 변수명/값 형식 문제를 먼저 봐야 한다.

## 관련 노트
- [[02-knowledge/infra/ECS EC2 첫 배포 트러블슈팅]]
