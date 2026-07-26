---
tags: [infra, aws, grafana, cloudwatch, observability, runbook]
date: 2026-07-21
status: 정리됨
---

# Grafana 전용 EC2 설치와 CloudWatch 연동

## 한 줄 요약
> Grafana는 ELK와 로그 역할을 겹치지 않게 두고, CloudWatch Metrics를 읽어 ALB, ECS, RDS, EC2 상태를 대시보드와 알림으로 감지하는 운영 관측 도구로 사용한다.

## 역할 분리

```text
ELK
→ 애플리케이션 로그 저장/검색/분석
→ stacktrace, requestId, 배치 실패 로그, API 예외 원인 추적

Grafana
→ 인프라/운영 지표 대시보드
→ ALB 5xx, Target unhealthy, ECS task fail, RDS CPU 감지
→ 임계치 기반 알림
```

## 현재 구성 방향

비용 절감을 위해 Grafana는 ECS Capacity Provider나 ASG에 올리지 않고, 단일 EC2에 Docker 컨테이너로 실행한다.

```text
Grafana EC2
- Instance: t3.micro 또는 Free tier eligible 타입
- OS: Amazon Linux 2023
- Storage: gp3 20 GiB
- Port: 3000
- IAM Role: CloudWatch/EC2/ELB/ECS/RDS 읽기 전용
```

## 보안그룹

```text
Inbound
- TCP 22: 내 IP만 허용
- TCP 3000: 내 IP 또는 팀원 IP만 허용

Outbound
- 0.0.0.0/0 허용
```

Grafana는 운영 대시보드이므로 처음부터 전체 공개하지 않는다.

## IAM Role

EC2에 붙일 Role 예시:

```text
Role name
- moduply-grafana-role

Policy name
- ModuplyGrafanaCloudWatchReadOnlyPolicy
```

정책은 CloudWatch 지표 조회와 AWS 리소스 목록 조회만 허용한다. `FullAccess`는 사용하지 않는다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GrafanaCloudWatchMetricsReadOnly",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:ListMetrics",
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:DescribeAlarms",
        "ec2:DescribeInstances",
        "ec2:DescribeRegions",
        "ec2:DescribeTags",
        "elasticloadbalancing:DescribeLoadBalancers",
        "elasticloadbalancing:DescribeTargetGroups",
        "elasticloadbalancing:DescribeTargetHealth",
        "ecs:ListClusters",
        "ecs:DescribeClusters",
        "ecs:ListServices",
        "ecs:DescribeServices",
        "ecs:ListTasks",
        "ecs:DescribeTasks",
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusters"
      ],
      "Resource": "*"
    }
  ]
}
```

## EC2 접속

```bash
# Grafana 전용 EC2에 SSH 접속한다.
# <키파일.pem>과 <GRAFANA_EC2_PUBLIC_IP>는 실제 값으로 바꾼다.
ssh -i <키파일.pem> ec2-user@<GRAFANA_EC2_PUBLIC_IP>
```

## Docker 설치

Amazon Linux 2023 기준 명령어다.

```bash
# 패키지 인덱스와 설치된 패키지를 최신 상태로 갱신한다.
sudo dnf update -y

# Grafana를 컨테이너로 실행하기 위해 Docker를 설치한다.
sudo dnf install -y docker

# EC2 재부팅 후에도 Docker가 자동으로 실행되도록 등록하고 즉시 시작한다.
sudo systemctl enable --now docker

# ec2-user가 sudo 없이 docker 명령을 실행할 수 있도록 docker 그룹에 추가한다.
sudo usermod -aG docker ec2-user
```

그룹 권한 반영을 위해 SSH를 한 번 종료하고 다시 접속한다.

```bash
# 현재 SSH 세션 종료
exit

# 다시 접속
ssh -i <키파일.pem> ec2-user@<GRAFANA_EC2_PUBLIC_IP>
```

Docker 동작 확인:

```bash
# Docker 데몬 상태 확인
systemctl status docker

# 현재 사용자로 docker 명령 실행 가능 여부 확인
docker version
```

## Grafana 컨테이너 실행

```bash
# Grafana 설정과 대시보드 데이터를 보존할 Docker volume을 생성한다.
docker volume create grafana-storage

# Grafana 컨테이너를 백그라운드로 실행한다.
# --restart unless-stopped: EC2 재부팅 또는 Docker 재시작 후 자동으로 다시 실행한다.
# -p 3000:3000: EC2의 3000 포트를 Grafana 컨테이너 3000 포트에 연결한다.
# -v grafana-storage:/var/lib/grafana: Grafana 데이터를 volume에 저장한다.
docker run -d \
  --name grafana \
  --restart unless-stopped \
  -p 3000:3000 \
  -v grafana-storage:/var/lib/grafana \
  grafana/grafana:latest
```

실행 확인:

```bash
# Grafana 컨테이너가 Up 상태인지 확인한다.
docker ps

# Grafana 실행 로그를 확인한다.
docker logs grafana
```

브라우저 접속:

```text
http://<GRAFANA_EC2_PUBLIC_IP>:3000
```

초기 계정:

```text
ID: admin
Password: admin
```

첫 로그인 후 비밀번호를 변경한다.

현재 변경한 접속 계정:

```text
ID: admin
Password: <GRAFANA_PASSWORD>
```

주의:

```text
- 이 비밀번호는 문서에 노출되어 있으므로 장기 운영용으로 사용하지 않는다.
- 팀 공유 후에는 Grafana UI에서 관리자 비밀번호를 다시 변경한다.
- Grafana 3000 포트는 내 IP 또는 팀원 IP로만 제한한다.
```

## 다음 단계 - CloudWatch Data Source 연결

Grafana UI에서 진행한다.

```text
Grafana
→ Connections
→ Data sources
→ Add data source
→ CloudWatch
```

설정값:

```text
Authentication Provider
→ AWS SDK Default

Default Region
→ ap-northeast-2
```

EC2에 `moduply-grafana-role`이 붙어 있으면 Access Key를 직접 입력하지 않아도 된다.

연결 확인:

```text
Save & test
→ Successfully queried the CloudWatch metrics API
```

## 다음 단계 - 대시보드 구성

대시보드 이름:

```text
Moduply Infra Overview
```

현재 대시보드에는 운영 상태를 한눈에 보기 위해 다음 패널을 생성했다.

```text
장애 여부 패널
- ALB UnHealthyHostCount
- ALB Target 5xx
- EC2 StatusCheckFailed
- RDS CPUUtilization

상태 추이 패널
- ALB RequestCount
- ALB TargetResponseTime
- RDS DatabaseConnections
- EC2 CPUUtilization
```

권장 레이아웃:

```text
상단
[ALB UnHealthyHostCount] [ALB Target 5xx] [EC2 StatusCheckFailed] [RDS CPUUtilization]

중단
[ALB RequestCount] [ALB TargetResponseTime]

하단
[RDS DatabaseConnections] [EC2 CPUUtilization]
```

상단은 장애 여부를 빠르게 판단하기 위해 `Stat` 또는 `Gauge` 중심으로 작게 배치하고, 중단/하단은 추이 확인을 위해 `Time series`로 배치한다.

처음에는 다음 패널부터 만든다.

### ALB

```text
AWS/ApplicationELB
- RequestCount
- HTTPCode_ELB_5XX_Count
- HTTPCode_Target_5XX_Count
- TargetResponseTime
- HealthyHostCount
- UnHealthyHostCount
```

### ECS

```text
AWS/ECS
- CPUUtilization
- MemoryUtilization
```

서비스 상태 관련 값은 CloudWatch 기본 지표만으로 부족하면 이후 Container Insights 또는 EventBridge 알림을 검토한다.

### RDS

```text
AWS/RDS
- CPUUtilization
- DatabaseConnections
- FreeableMemory
- FreeStorageSpace
- ReadLatency
- WriteLatency
```

### EC2

```text
AWS/EC2
- CPUUtilization
- NetworkIn
- NetworkOut
- StatusCheckFailed
```

Redis를 EC2 단일 서버로 운영하는 동안은 Redis 자체 지표보다는 Redis EC2의 CPU, 네트워크, 상태 체크를 먼저 본다. Redis 내부 지표까지 보려면 추후 Redis exporter + Prometheus 또는 CloudWatch Agent 구성을 검토한다.

## 다음 단계 - 알림 규칙

우선 다음 알림만 잡는다.

```text
ALB UnHealthyHostCount >= 1
ALB Target 5xx > 0
ALB ELB 5xx > 0
RDS CPUUtilization > 80%
RDS FreeStorageSpace 부족
EC2 StatusCheckFailed >= 1
```

알림 채널:

```text
Discord Webhook
또는 Email
```

팀 프로젝트에서는 Discord Webhook이 가장 빠르다.

## Discord 알림 연동

Discord 채널에서 Webhook을 만들고 Grafana Contact Point로 연결했다.

```text
Discord
→ 채널 설정
→ 연동
→ 웹후크
→ 새 웹후크
→ Webhook URL 복사
```

Grafana 설정:

```text
Alerting
→ Contact points
→ New contact point
→ Integration: Discord
→ Webhook URL 입력
→ Test
```

테스트 결과 Discord 알림 수신을 확인했다.

Notification Policy:

```text
Alerting
→ Notification policies
→ Default contact point: Discord contact point
```

## 등록한 알림 규칙

장애 여부를 빠르게 감지하기 위해 상단 4개 패널에 알림을 등록했다.

### 1. ALB UnHealthyHostCount

목적:

```text
ALB 대상 그룹에 unhealthy target이 생겼는지 감지한다.
```

조건:

```text
Metric: AWS/ApplicationELB - UnHealthyHostCount
Statistic: Maximum
Condition: last() OF query(A) IS ABOVE 0
Evaluation group: moduply-infra-1m
Evaluation interval: 1m
Pending period: 1m
Keep firing for: 5m
```

메시지:

```text
summary
[ALB] 대상 그룹 UnHealthy Target 발생

description
moduply-app-instance-tg에 UnHealthy Target이 발생했습니다. ECS task 상태, ALB target health, 애플리케이션 헬스체크 응답을 확인하세요.
```

### 2. ALB Target 5xx

목적:

```text
Spring App 대상에서 5xx 응답이 발생했는지 감지한다.
```

조건:

```text
Metric: AWS/ApplicationELB - HTTPCode_Target_5XX_Count
Statistic: Sum
Condition: last() OF query(A) IS ABOVE 0
Evaluation group: moduply-infra-1m
Evaluation interval: 1m
Pending period: 1m
Keep firing for: 5m
```

메시지:

```text
summary
[ALB] Target 5xx 응답 발생

description
Spring App 대상에서 5xx 응답이 발생했습니다. 애플리케이션 로그, 최근 배포 여부, RDS/Redis 연결 상태를 확인하세요.
```

### 3. EC2 StatusCheckFailed

목적:

```text
EC2 인스턴스 상태 검사 실패를 감지한다.
```

조건:

```text
Metric: AWS/EC2 - StatusCheckFailed
Statistic: Maximum
Condition: last() OF query(A) IS ABOVE 0
Evaluation group: moduply-infra-1m
Evaluation interval: 1m
Pending period: 2m
Keep firing for: 5m
```

메시지:

```text
summary
[EC2] 인스턴스 상태 검사 실패

description
EC2 StatusCheckFailed 값이 1 이상입니다. 인스턴스 상태 검사, 시스템 상태 검사, CPU/메모리/네트워크 상태를 확인하세요.
```

### 4. RDS CPUUtilization

목적:

```text
RDS CPU 사용률이 장시간 높은 상태인지 감지한다.
```

테스트 시에는 Discord 연동 확인을 위해 임계치를 낮게 잡았고, 알림 수신 확인 후 운영값으로 복구한다.

운영 조건:

```text
Metric: AWS/RDS - CPUUtilization
Statistic: Average
Condition: last() OF query(A) IS ABOVE 80
Evaluation group: moduply-infra-1m
Evaluation interval: 1m
Pending period: 5m
Keep firing for: 5m
```

메시지:

```text
summary
[RDS] CPU 사용률 80% 초과

description
RDS CPU 사용률이 80%를 초과했습니다. 느린 쿼리, 커넥션 수, 배치 작업 실행 여부, 최근 트래픽 증가 여부를 확인하세요.
```

## 알림 운영 기준

초기에는 장애 여부가 명확한 알림만 켠다.

```text
켜는 알림:
- ALB UnHealthyHostCount
- ALB Target 5xx
- EC2 StatusCheckFailed
- RDS CPUUtilization

대시보드로만 관찰:
- ALB RequestCount
- ALB TargetResponseTime
- RDS DatabaseConnections
- EC2 CPUUtilization
```

응답시간, DB 커넥션, EC2 CPU는 서비스 평소 기준선을 파악한 뒤 임계치를 정한다.

## 운영 주의사항

- Grafana 3000 포트는 전체 공개하지 않는다.
- Grafana admin 초기 비밀번호는 즉시 변경한다.
- AWS Access Key를 Grafana에 직접 저장하지 않고 EC2 IAM Role을 사용한다.
- Grafana는 로그 저장소가 아니다. 로그 분석은 ELK에서 수행한다.
- Grafana EC2가 꺼지면 대시보드와 알림도 멈추므로, 발표 전에는 반드시 인스턴스 상태와 컨테이너 상태를 확인한다.

## 장애 확인 명령어

```bash
# Grafana 컨테이너 실행 상태 확인
docker ps

# Grafana 로그 확인
docker logs grafana

# Grafana 컨테이너 재시작
docker restart grafana

# EC2 3000 포트 리스닝 확인
sudo ss -lntp | grep 3000

# Docker daemon 상태 확인
systemctl status docker
```

## 결론

Grafana는 App 서버와 분리된 전용 EC2에서 실행하고, CloudWatch Metrics를 읽어 운영 지표 대시보드와 알림을 담당한다. ELK가 애플리케이션 로그 원인 분석을 맡고, Grafana는 인프라 이상 감지와 운영 상태 시각화에 집중한다.
