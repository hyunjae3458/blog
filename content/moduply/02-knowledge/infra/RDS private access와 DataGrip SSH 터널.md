---
tags: [infra, aws, rds, postgresql, datagrip, ssh-tunnel]
date: 2026-07-07
status: 해결됨
---

# RDS private access와 DataGrip SSH 터널

## 한 줄 요약
> RDS를 public access 없이 운영할 때는 같은 VPC의 EC2를 SSH tunnel 경유지로 사용하고, RDS 보안그룹은 EC2 보안그룹을 source로 5432를 열어야 한다.

## 배경 / 어떤 상황이었는가
RDS PostgreSQL을 public access off로 만들었다. 로컬 DataGrip에서 바로 RDS endpoint로 접근할 수 없어서 ECS EC2 인스턴스를 SSH tunnel 경유지로 사용했다.

## 문제
처음에는 DataGrip 연결이 실패했다.

```text
SSH tunnel creation failed
Connection refused
timeout
password authentication failed
```

## 원인과 해결

### SSH user 문제
Amazon Linux 2023의 기본 SSH user는 `ec2-user`다. 다른 user를 쓰면 SSH 접속이 실패한다.

```text
SSH host: <EC2_PUBLIC_IP>
SSH user: ec2-user
SSH port: 22
```

### RDS 보안그룹 문제
RDS inbound에 PostgreSQL 5432가 열려 있지 않거나 source가 잘못되어 EC2에서 RDS로 접근하지 못했다.

올바른 규칙은 RDS 보안그룹에 추가한다.

```text
Type: PostgreSQL
Port: 5432
Source: ECS EC2 인스턴스의 Security Group ID
```

EC2 보안그룹에 5432를 여는 것이 아니라, RDS 보안그룹이 EC2 보안그룹에서 오는 5432 요청을 허용해야 한다.

### DB user/password 문제
DataGrip의 user/password는 AWS IAM 계정이 아니라 PostgreSQL 내부 사용자다. RDS 생성 시 설정한 master username/password를 사용했다.

## DataGrip 설정

```text
SSH Tunnel
- SSH host: <EC2_PUBLIC_IP>
- SSH user: ec2-user
- Auth: key pair

PostgreSQL
- Host: <RDS_ENDPOINT>
- Port: 5432
- Database: postgres
- User: <RDS master username>
- Password: <RDS master password>
```

## 스키마 생성
빠른 배포 확인을 위해 RDS 기본 `postgres` DB에 `schema.sql`을 실행했다. 앱 전용 `moduply` DB도 만들었지만, 스키마가 `postgres`에 생성되어 우선 `postgres` DB를 사용했다.

## 결과 / 검증 방법
EC2에서 RDS 5432 접근을 확인했다.

```bash
nc -vz <RDS_ENDPOINT> 5432
```

DataGrip으로 `postgres` DB에 접속하고 테이블 생성 여부를 확인했다.

## 배운 점 / 다음에 또 마주치면
- RDS public access off는 정상적인 운영 선택이다. 대신 같은 VPC의 EC2, VPN, bastion, SSM 등을 통해 접근 경로를 만들어야 한다.
- 보안그룹 source는 IP보다 security group ID로 묶는 편이 안전하고 관리하기 쉽다.
- 기본 `postgres` DB 사용은 빠른 검증용으로는 가능하지만, 최종 운영에서는 앱 전용 DB로 분리해야 한다.

## 관련 노트
- [[03-decisions/infra/ECS EC2 단일 인스턴스 최소 비용 배포 결정]]
