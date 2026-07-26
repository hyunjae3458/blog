---
tags: [infra, elk, kibana, filebeat, logstash, ecs, troubleshooting]
date: 2026-07-23
status: 해결됨
---

# Kibana에 애플리케이션 로그가 보이지 않은 문제

## 한 줄 요약
> Kibana에 운영 로그가 보이지 않은 원인은 하나가 아니라, ECS 컨테이너 로그 디렉터리 미공유, 호스트 로그 파일 권한, Logstash grok 패턴, Logstash 검증 중 리소스 고갈, Logstash timezone 오설정이 순차적으로 드러난 복합 문제였다.

## 최종 결론

최종 정상 흐름:

```text
Spring App 컨테이너
→ /app/.logs/moduply.log 기록
→ ECS host bind mount
→ App EC2 호스트 /app/.logs/moduply.log
→ Filebeat
→ Logstash 5044
→ grok/date 파싱
→ Elasticsearch moduply-logs-YYYY.MM.dd
→ Kibana Discover
```

최종 핵심 설정:

```text
ECS bind mount
- Host path: /app/.logs
- Container path: /app/.logs

Filebeat
- input path: /app/.logs/moduply.log
- output: <ELK_PRIVATE_IP>:5044

Logstash
- elasticsearch output: http://localhost:9200
- date timezone: UTC

Kibana
- dateFormat:tz: Asia/Seoul
- sort: @timestamp desc
```

## 장애 흐름

### 1. Kibana에 테스트 로그만 보임

초기에는 Kibana Discover에서 운영 애플리케이션 로그가 보이지 않았고, Elasticsearch에는 테스트로 직접 넣은 로그 1건만 보였다.

확인:

```bash
curl "http://localhost:9200/moduply-logs-*/_search?pretty&size=3&sort=@timestamp:desc"
```

결과에는 다음 테스트 로그만 있었다.

```text
log_message: filebeat-logstash-test
log.file.path: /app/.logs/moduply.log
```

판단:

```text
Filebeat → Logstash → Elasticsearch 흐름은 동작한다.
하지만 Filebeat가 실제 운영 앱 로그 파일을 읽지 못하고 있다.
```

### 2. ECS 컨테이너 내부 로그를 호스트 Filebeat가 읽지 못함

App EC2 호스트에서는 Filebeat가 `/app/.logs/*.log`를 읽고 있었다.

```yaml
paths:
  - /app/.logs/moduply.log
  - /app/.logs/moduply-error.log
```

하지만 실제 Spring 로그는 ECS 컨테이너 내부 `/app/.logs`에만 존재했다.

```text
ECS 컨테이너 내부
/app/.logs/moduply.log
→ 운영 로그 존재

App EC2 호스트
/app/.logs/moduply.log
→ 운영 로그 없음 또는 테스트 로그만 존재
```

해결:

```text
ECS task definition에 host bind mount 추가

Volume
- Type: Bind mount / Host
- Name: app-logs
- Source path: /app/.logs

Container mount point
- Container: moduply-app
- Source volume: app-logs
- Container path: /app/.logs
- Read only: false
```

교훈:

```text
EC2 호스트에서 실행되는 Filebeat는 컨테이너 내부 파일을 자동으로 읽지 못한다.
호스트 Filebeat 방식이면 컨테이너 로그 디렉터리를 host bind mount로 공유해야 한다.
```

### 3. bind mount 후 Logback Permission denied 발생

bind mount 적용 후 Spring Boot가 시작 단계에서 실패했다.

```text
ERROR in ch.qos.logback.core.rolling.RollingFileAppender[FILE]
openFile(.logs/moduply.log,true) call failed.
java.io.FileNotFoundException: .logs/moduply.log (Permission denied)
```

원인:

```text
컨테이너가 /app/.logs/moduply.log에 쓰는 파일은 실제로 App EC2 호스트 /app/.logs/moduply.log다.
호스트 디렉터리나 기존 로그 파일이 root 소유/쓰기 제한 상태면 컨테이너 앱 유저가 append하지 못한다.
```

특히 테스트 과정에서 `sudo tee -a /app/.logs/moduply.log`로 만든 파일은 root 소유로 남을 수 있다.

해결:

```bash
sudo mkdir -p /app/.logs
sudo touch /app/.logs/moduply.log /app/.logs/moduply-error.log
sudo chmod 777 /app/.logs
sudo chmod 666 /app/.logs/moduply.log /app/.logs/moduply-error.log
```

시작 템플릿 User Data에도 반영:

```bash
#!/bin/bash
set -e

grep -q '^ECS_CLUSTER=' /etc/ecs/ecs.config \
  || echo 'ECS_CLUSTER=moduply-prod2-cluster' >> /etc/ecs/ecs.config

mkdir -p /app/.logs
touch /app/.logs/moduply.log /app/.logs/moduply-error.log
chmod 777 /app/.logs
chmod 666 /app/.logs/moduply.log /app/.logs/moduply-error.log
```

주의:

```text
User Data는 새 EC2 인스턴스에만 적용된다.
이미 떠 있는 App EC2에는 수동으로 권한을 보정해야 한다.
```

### 4. Logstash grok 파싱 실패

운영 로그가 Elasticsearch에 들어오기 시작했지만 다음 필드가 붙었다.

```text
tags: _grokparsefailure
parse_error: true
```

예시:

```text
26-07-23 06:10:25.043 [main] INFO  c.t.m.Sb10ModuplyTeam6Application    [ |  | ] - The following 1 profile is active: "prod"
```

원인:

```text
logger와 [requestId | method | uri] 사이의 공백이 여러 개인데,
기존 grok 패턴은 고정 공백에 가까운 형태라 일부 로그를 매칭하지 못했다.
```

해결:

```conf
grok {
  match => {
    "message" => "%{DATA:timestamp} \[%{DATA:thread}\] %{LOGLEVEL:level}%{SPACE}%{NOTSPACE:logger}%{SPACE}(\[%{DATA:requestId} \| %{DATA:requestMethod} \| %{DATA:requestUri}\]%{SPACE})?- %{GREEDYDATA:log_message}"
  }
}
```

검증 결과:

```text
level: INFO
logger: c.t.moduply.Test
thread: test-thread
requestId: test-request
requestMethod: GET
requestUri: /elk-test-6
log_message: elk-live-test-6
```

새 로그에서는 `_grokparsefailure`가 사라졌다.

### 5. logstash -t 실행 중 ELK 서버가 멈춤

Logstash 설정 검증을 위해 다음 명령을 실행했다.

```bash
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

이후 SSH와 Kibana가 응답하지 않는 상태가 됐다.

원인:

```text
t3.medium 단일 EC2에서 Elasticsearch, Logstash, Kibana가 이미 실행 중이었다.
logstash -t는 테스트용 Logstash JVM을 추가로 띄우므로 순간적으로 CPU/메모리 압박이 커졌다.
```

복구:

```text
EC2 Reboot가 정상 처리되지 않아 Stop → Start로 복구했다.
```

복구 후 상태 확인:

```bash
sudo systemctl status elasticsearch
sudo systemctl status kibana
sudo systemctl status logstash
free -h
```

추가 안정화:

```text
Logstash heap
- Xms256m
- Xmx512m
```

운영 중 설정 확인 방식 변경:

```bash
sudo systemctl restart logstash
sudo journalctl -u logstash -n 100 --no-pager
sudo ss -lntp | grep 5044
```

`logstash -t`는 운영 중 사용을 피한다.

### 6. Filebeat connection reset by peer

Filebeat 로그에 다음 오류가 남았다.

```text
failed to publish events:
write tcp <APP_PRIVATE_IP>:<PORT>-><ELK_PRIVATE_IP>:5044:
write: connection reset by peer
```

판단:

```text
보안그룹 문제라면 timeout이 날 가능성이 높다.
connection reset by peer는 Logstash가 연결을 받았다가 끊은 상태다.
```

실제 원인은 Logstash 재시작/중단 타이밍에 Filebeat가 batch를 전송하려다 연결이 끊긴 것으로 판단했다.

확인:

```bash
sudo systemctl status logstash
sudo journalctl -u logstash -n 200 --no-pager
sudo ss -lntp | grep 5044
```

Logstash는 다음 상태로 정상 기동했다.

```text
Starting input listener {:address=>"0.0.0.0:5044"}
Pipeline started
Pipelines running
```

Filebeat 재시작 후 테스트 로그가 정상 적재됐다.

### 7. Kibana 최신순에서 요청 로그가 보이지 않음

요청할 때마다 Elasticsearch `documents` 수는 증가했지만, Kibana Discover 최신순에서 방금 요청 로그가 보이지 않았다.

App EC2 로그 파일에는 실제 요청 로그가 있었다.

```text
26-07-23 07:21:19.659 [http-nio-8080-exec-9] INFO  c.t.m.c.controller.ContentController [58649d48220245ad98fa69e2d29cbc3a | GET | /api/contents/fff2b50b-59a9-4a69-bfce-098f92be18e9] - 콘텐츠 단건 조회 요청 수신...
```

원인:

```text
운영 앱 로그 원문 시간은 UTC로 찍히고 있었다.
하지만 Logstash date filter가 timezone => "Asia/Seoul"로 설정되어 있었다.
따라서 UTC 원문 시간을 KST로 오해해 @timestamp를 9시간 과거로 저장했다.
```

잘못된 설정:

```conf
date {
  match => ["timestamp", "yy-MM-dd HH:mm:ss.SSS"]
  target => "@timestamp"
  timezone => "Asia/Seoul"
}
```

최종 수정:

```conf
date {
  match => ["timestamp", "yy-MM-dd HH:mm:ss.SSS"]
  target => "@timestamp"
  timezone => "UTC"
}
```

Kibana 표시 시간대:

```text
Stack Management
→ Advanced Settings
→ dateFormat:tz
→ Asia/Seoul
```

정리:

```text
Logstash date timezone
→ UTC
→ 원문 로그 시간이 UTC임을 알려줌

Kibana dateFormat:tz
→ Asia/Seoul
→ 화면에는 한국 시간으로 표시

Discover 정렬
→ @timestamp desc
```

기존에 잘못 들어간 로그는 자동 수정되지 않고, 수정 이후 새 로그부터 최신순으로 정상 표시된다.

## 최종 검증

테스트 로그:

```text
26-07-23 16:15:00.000 [test-thread] INFO  c.t.moduply.Test [test-request | GET | /elk-test-6] - elk-live-test-6
```

Elasticsearch 적재 결과:

```text
requestUri: /elk-test-6
log_message: elk-live-test-6
level: INFO
logger: c.t.moduply.Test
@timestamp: 2026-07-23T07:15:00.000Z
tags: beats_input_codec_plain_applied
```

정상 기준:

```text
_grokparsefailure 없음
parse_error 없음
requestUri/log_message 필드 추출됨
Kibana Discover에서 Last 24 hours, @timestamp desc 기준 조회됨
```

## 이번 과정에서 함께 기록할 가치가 있는 항목

이번 장애 문서에 포함한 항목:

```text
- ECS 컨테이너 로그 디렉터리와 호스트 Filebeat 경로 불일치
- host bind mount 후 로그 디렉터리/파일 권한 문제
- Filebeat output.elasticsearch/output.logstash 동시 설정 충돌
- Logstash queue/log 디렉터리 권한 문제
- Logstash grok 공백 패턴 문제
- logstash -t로 인한 t3.medium 리소스 고갈
- Filebeat connection reset by peer
- Logstash date timezone과 Kibana 표시 timezone 분리
```

별도 문서로 이미 기록한 항목:

```text
- ELK 전용 EC2 설치 절차: [[ELK 전용 EC2 설치와 로그 검색 연동]]
- 퍼블릭 인터넷 스캐너 봇 로그 점령: [[퍼블릭 인터넷 노출로 인한 자동 스캐너 봇 로그 점령]]
- Grafana 운영 대시보드: [[Grafana 전용 EC2 설치와 CloudWatch 연동]]
```

추가로 별도 문서화할 후보:

```text
- Logstash 인덱스 보관 정책, 예: moduply-logs-* 7일 또는 14일 보관
- ELK 단일 EC2 메모리 튜닝 기준
- 운영용 Filebeat/Logstash 설정 파일을 repo에 prod example로 분리
```
