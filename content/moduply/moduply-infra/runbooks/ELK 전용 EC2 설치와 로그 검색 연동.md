---
tags: [infra, aws, elk, elasticsearch, logstash, kibana, filebeat, runbook]
date: 2026-07-23
status: 정리됨
---

# ELK 전용 EC2 설치와 로그 검색 연동

## 한 줄 요약
> ELK 서버는 단일 EC2에 Elasticsearch, Logstash, Kibana를 systemd 서비스로 설치하고, App EC2의 Filebeat가 애플리케이션 로그를 Logstash로 전송하는 구조로 운영한다.

## 역할

```text
ELK EC2
- Elasticsearch: 콘텐츠 검색 인덱스와 로그 인덱스 저장
- Logstash: Filebeat 로그 수신, grok 파싱, Elasticsearch 적재
- Kibana: 로그 조회 UI

App EC2
- Filebeat: 애플리케이션 로그 파일을 읽어 Logstash로 전송
```

## 인덱스 분리

```text
콘텐츠 검색
- contents

애플리케이션 로그
- moduply-logs-YYYY.MM.dd
```

검색용 데이터와 로그 데이터는 같은 Elasticsearch 서버를 사용하되, 인덱스를 분리한다.

## EC2 스펙

```text
Name: moduply-elk-server
AMI: Amazon Linux 2023
Instance type: t3.medium
Storage: gp3 50GB 이상
```

## 보안그룹

```text
Inbound
- TCP 22: 내 IP
- TCP 9200: App/ECS 인스턴스 보안그룹
- TCP 5044: App/ECS 인스턴스 보안그룹
- TCP 5601: 내 IP

Outbound
- All traffic: 0.0.0.0/0
```

포트 역할:

```text
22: SSH
9200: Elasticsearch API
5044: Logstash Beats input
5601: Kibana UI
```

`9200`, `5044`, `5601`은 전체 공개하지 않는다.

## Elastic 8.x repository 등록

Amazon Linux 기본 repository에는 Elasticsearch, Logstash, Kibana 패키지가 없으므로 Elastic 공식 repository를 추가한다.

```bash
# Amazon Linux 패키지를 최신 상태로 갱신한다.
sudo dnf update -y

# Elastic 8.x RPM 패키지를 받을 공식 repository를 등록한다.
sudo tee /etc/yum.repos.d/elastic.repo > /dev/null <<'EOF'
[elastic-8.x]
name=Elastic repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```

설정 의미:

```text
baseurl: Elastic 8.x RPM 패키지 저장소
gpgcheck: 패키지 서명 검증 활성화
gpgkey: Elastic 공식 공개키
enabled: dnf install/update 시 repository 사용
```

## Elasticsearch 설치

```bash
# Elasticsearch 패키지를 설치한다.
sudo dnf install -y elasticsearch
```

설정 파일:

```bash
# Elasticsearch 단일 노드 운영 설정을 수정한다.
sudo vi /etc/elasticsearch/elasticsearch.yml
```

운영 초기 설정:

```yaml
cluster.name: moduply-elk
node.name: moduply-elk-1
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
xpack.security.enabled: false
```

주의:

```text
xpack.security.enabled: false는 내부망 단순 구성을 위한 선택이다.
대신 보안그룹에서 9200 접근을 App/ECS 인스턴스 보안그룹으로 제한한다.
```

OS 설정:

```bash
# Elasticsearch가 사용하는 mmap 영역 수 제한을 올린다.
echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-elasticsearch.conf

# sysctl 설정을 즉시 반영한다.
sudo sysctl --system
```

Heap 설정:

```bash
# t3.medium 기준 Elasticsearch heap을 1GB로 제한한다.
sudo vi /etc/elasticsearch/jvm.options.d/heap.options
```

```text
-Xms1g
-Xmx1g
```

서비스 시작:

```bash
# systemd unit 변경사항을 반영한다.
sudo systemctl daemon-reload

# 부팅 시 Elasticsearch가 자동 실행되도록 등록한다.
sudo systemctl enable elasticsearch

# Elasticsearch를 시작한다.
sudo systemctl start elasticsearch

# Elasticsearch 실행 상태를 확인한다.
sudo systemctl status elasticsearch
```

동작 확인:

```bash
# Elasticsearch HTTP API가 응답하는지 확인한다.
curl http://localhost:9200

# 9200 포트가 외부 인터페이스에서 listen 중인지 확인한다.
sudo ss -lntp | grep 9200
```

## Logstash 설치

```bash
# Logstash 패키지를 설치한다.
sudo dnf install -y logstash
```

Pipeline 설정:

```bash
# Moduply 애플리케이션 로그 파이프라인을 생성한다.
sudo vi /etc/logstash/conf.d/moduply-logstash.conf
```

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => {
      "message" => "%{DATA:timestamp} \[%{DATA:thread}\] %{LOGLEVEL:level}%{SPACE}%{NOTSPACE:logger}%{SPACE}(\[%{DATA:requestId} \| %{DATA:requestMethod} \| %{DATA:requestUri}\]%{SPACE})?- %{GREEDYDATA:log_message}"
    }
  }

  if "_grokparsefailure" not in [tags] {
    date {
      match => ["timestamp", "yy-MM-dd HH:mm:ss.SSS"]
      target => "@timestamp"
      timezone => "UTC"
    }

    mutate {
      strip => ["requestId", "requestMethod", "requestUri"]
      remove_field => ["timestamp", "host", "agent"]
    }
  }

  if "_grokparsefailure" in [tags] {
    mutate {
      add_field => { "parse_error" => "true" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "moduply-logs-%{+YYYY.MM.dd}"
  }
}
```

주의:

```text
repo의 elk/logstash/logstash.conf는 로컬 Docker Compose 기준이라 hosts가 http://elasticsearch:9200이다.
운영 EC2 직접 설치 환경에서는 http://localhost:9200으로 바꾼다.
운영 앱 로그 원문 시간은 UTC로 찍히므로 Logstash date timezone은 UTC로 둔다.
Kibana 화면 표시는 dateFormat:tz를 Asia/Seoul로 설정해 한국 시간으로 본다.
```

Logstash heap 설정:

```bash
# t3.medium에서 Elasticsearch/Kibana와 함께 사용하기 위해 Logstash heap을 512MB로 제한한다.
sudo vi /etc/logstash/jvm.options
```

```text
-Xms512m
-Xmx512m
```

권한 보정:

```bash
# Logstash queue 디렉터리를 생성한다.
sudo mkdir -p /var/lib/logstash/queue

# Logstash 데이터 디렉터리를 logstash 사용자 소유로 바꾼다.
sudo chown -R logstash:logstash /var/lib/logstash

# Logstash 데이터 디렉터리 권한을 제한한다.
sudo chmod -R 750 /var/lib/logstash

# Logstash 로그 디렉터리를 생성한다.
sudo mkdir -p /var/log/logstash

# Logstash 로그 디렉터리를 logstash 사용자 소유로 바꾼다.
sudo chown -R logstash:logstash /var/log/logstash

# Logstash 로그 디렉터리 권한을 제한한다.
sudo chmod -R 750 /var/log/logstash
```

Pipeline 문법 검사:

```bash
# Logstash 설정과 pipeline 문법이 유효한지 검사한다.
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

주의:

```text
t3.medium 단일 EC2에서 Elasticsearch, Logstash, Kibana가 함께 실행 중일 때 logstash -t는 테스트용 Logstash JVM을 하나 더 띄운다.
메모리/CPU가 부족하면 SSH/Kibana까지 멈춘 것처럼 보일 수 있으므로 운영 중에는 가급적 사용하지 않는다.
설정 변경 후에는 systemctl restart logstash와 journalctl로 확인하는 방식을 우선 사용한다.
```

서비스 시작:

```bash
# 부팅 시 Logstash가 자동 실행되도록 등록한다.
sudo systemctl enable logstash

# Logstash를 시작한다.
sudo systemctl start logstash

# Logstash 실행 상태를 확인한다.
sudo systemctl status logstash
```

포트 확인:

```bash
# Logstash가 5044 포트를 listen 중인지 확인한다.
# *:5044 또는 0.0.0.0:5044면 정상이다.
sudo ss -lntp | grep 5044
```

## Kibana 설치

```bash
# Kibana 패키지를 설치한다.
sudo dnf install -y kibana
```

설정 파일:

```bash
# Kibana가 외부 접속을 받고 localhost Elasticsearch에 연결하도록 설정한다.
sudo vi /etc/kibana/kibana.yml
```

```yaml
server.host: "0.0.0.0"
server.port: 5601
elasticsearch.hosts: ["http://localhost:9200"]
```

서비스 시작:

```bash
# 부팅 시 Kibana가 자동 실행되도록 등록한다.
sudo systemctl enable kibana

# Kibana를 시작한다.
sudo systemctl start kibana

# Kibana 실행 상태를 확인한다.
sudo systemctl status kibana
```

확인:

```bash
# Kibana가 5601 포트를 listen 중인지 확인한다.
sudo ss -lntp | grep 5601

# ELK 서버 내부에서 Kibana HTTP 응답을 확인한다.
curl http://localhost:5601
```

브라우저 접속:

```text
http://<ELK_EC2_PUBLIC_IP>:5601
```

## App 서버 Filebeat 설치

App EC2에도 Elastic repository를 등록한 뒤 Filebeat를 설치한다.

```bash
# Elastic 8.x repository를 App EC2에도 등록한다.
sudo tee /etc/yum.repos.d/elastic.repo > /dev/null <<'EOF'
[elastic-8.x]
name=Elastic repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

# Filebeat 패키지를 설치한다.
sudo dnf install -y filebeat
```

Filebeat 설정:

```bash
# 기존 filebeat.yml을 백업한다.
sudo cp /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.bak

# Moduply 운영 로그 수집 설정으로 교체한다.
sudo vi /etc/filebeat/filebeat.yml
```

```yaml
filebeat.inputs:
  - type: filestream
    id: moduply-app-logs
    enabled: true
    paths:
      - /app/.logs/moduply.log
    fields:
      log_type: app
    fields_under_root: false

  - type: filestream
    id: moduply-error-logs
    enabled: true
    paths:
      - /app/.logs/moduply-error.log
    fields:
      log_type: error
    fields_under_root: false

output.logstash:
  hosts: ["<ELK_PRIVATE_IP>:5044"]

logging.level: info
```

주의:

```text
Filebeat는 output을 하나만 사용할 수 있다.
기본 output.elasticsearch와 output.logstash가 동시에 있으면 more than one namespace configured accessing 'output' 에러가 발생한다.
운영에서는 output.logstash만 남긴다.
```

설정 검사:

```bash
# Filebeat 설정 문법을 검사한다.
sudo filebeat test config

# Logstash 5044에 연결 가능한지 검사한다.
sudo filebeat test output
```

서비스 시작:

```bash
# 부팅 시 Filebeat가 자동 실행되도록 등록한다.
sudo systemctl enable filebeat

# Filebeat를 재시작한다.
sudo systemctl restart filebeat

# Filebeat 실행 상태를 확인한다.
sudo systemctl status filebeat
```

## ECS Task Definition 로그 볼륨 마운트

Filebeat는 App EC2 호스트에서 실행되므로, 컨테이너 내부 로그 파일을 직접 읽을 수 없다.

ECS task definition에 host bind mount를 추가한다.

```text
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

이 설정 후 컨테이너가 `/app/.logs/moduply.log`에 쓰는 로그가 EC2 호스트의 `/app/.logs/moduply.log`에 저장되고, Filebeat가 해당 파일을 읽을 수 있다.

## Spring 앱 Elasticsearch 환경변수

운영 앱이 검색용 Elasticsearch에 붙도록 환경변수를 설정한다.

```env
ELASTICSEARCH_URIS=http://<ELK_PRIVATE_IP>:9200
```

앱이 기본값 `http://localhost:9200`으로 실행되면 ECS 컨테이너 자기 자신으로 접속하므로 `Connection refused`가 발생한다.

## 로그 보관 정책

현재 로그 인덱스는 날짜별로 생성된다.

```text
moduply-logs-YYYY.MM.dd
```

Elasticsearch는 콘텐츠 검색 인덱스 `contents`도 함께 사용하므로, 로그 인덱스를 무제한 보관하지 않는다.

초기 운영 정책:

```text
보관 기간: 14일
삭제 대상: moduply-logs-*
삭제 방식: ELK EC2 crontab
실행 시각: 매일 03:00
```

삭제 스크립트 생성:

```bash
# 운영 스크립트 디렉터리를 생성한다.
sudo mkdir -p /opt/moduply/scripts

# 14일 전 로그 인덱스를 삭제하는 스크립트를 생성한다.
sudo vi /opt/moduply/scripts/delete-old-elk-logs.sh
```

스크립트 내용:

```bash
#!/bin/bash
set -e

RETENTION_DAYS=14
ELASTICSEARCH_URL="http://localhost:9200"

TARGET_DATE=$(date -u -d "${RETENTION_DAYS} days ago" +"%Y.%m.%d")
INDEX_NAME="moduply-logs-${TARGET_DATE}"

echo "Deleting old ELK log index: ${INDEX_NAME}"
curl -s -X DELETE "${ELASTICSEARCH_URL}/${INDEX_NAME}"
echo
```

실행 권한 부여:

```bash
# 스크립트를 실행 가능하게 만든다.
sudo chmod +x /opt/moduply/scripts/delete-old-elk-logs.sh
```

수동 테스트:

```bash
# root 권한으로 스크립트를 실행하고 결과를 로그 파일에 기록한다.
sudo sh -c '/opt/moduply/scripts/delete-old-elk-logs.sh >> /var/log/delete-old-elk-logs.log 2>&1'

# 삭제 스크립트 실행 로그를 확인한다.
sudo tail -n 50 /var/log/delete-old-elk-logs.log
```

아직 14일 전 인덱스가 없으면 다음 응답이 나올 수 있다.

```text
index_not_found_exception
status: 404
```

이 경우는 삭제 대상 인덱스가 아직 없다는 뜻이므로 정상이다.

crontab 설치:

```bash
# Amazon Linux에 crontab 명령이 없으면 cronie를 설치한다.
sudo dnf install -y cronie

# cron 데몬을 부팅 시 자동 실행하도록 등록하고 즉시 시작한다.
sudo systemctl enable crond
sudo systemctl start crond
sudo systemctl status crond
```

cron 등록:

```bash
# root crontab을 편집한다.
sudo crontab -e
```

매일 새벽 3시에 실행:

```cron
0 3 * * * /opt/moduply/scripts/delete-old-elk-logs.sh >> /var/log/delete-old-elk-logs.log 2>&1
```

등록 확인:

```bash
# root crontab에 삭제 작업이 등록되었는지 확인한다.
sudo crontab -l
```

## 상태 확인

ELK 서버:

```bash
# Elasticsearch 상태 확인
curl http://localhost:9200

# 인덱스 목록 확인
curl http://localhost:9200/_cat/indices?v

# 로그 인덱스 확인
curl http://localhost:9200/_cat/indices/moduply-logs-*?v

# 콘텐츠 검색 인덱스 확인
curl http://localhost:9200/_cat/indices/contents?v

# 콘텐츠 검색 문서 수 확인
curl http://localhost:9200/contents/_count?pretty

# 최신 로그 문서 확인
curl "http://localhost:9200/moduply-logs-*/_search?pretty&size=3&sort=@timestamp:desc"
```

App EC2:

```bash
# Filebeat 상태 확인
sudo systemctl status filebeat

# Filebeat 최신 로그 확인
sudo journalctl -u filebeat -n 100 --no-pager

# App EC2 호스트에 실제 애플리케이션 로그가 쌓이는지 확인
ls -l /app/.logs/
tail -n 50 /app/.logs/moduply.log
```

Kibana:

```text
Stack Management
→ Data Views
→ Create data view
→ Name: moduply-logs
→ Index pattern: moduply-logs-*
→ Timestamp field: @timestamp
```

시간대 표시 설정:

```text
Stack Management
→ Advanced Settings
→ dateFormat:tz
→ Asia/Seoul
```

조회:

```text
Discover
→ Data view: moduply-logs
→ 시간 범위: Today 또는 Last 24 hours
→ 정렬: @timestamp desc
```

## 현재 확인된 결과

```text
Logstash 5044 listen: 정상
Filebeat test output: 정상
moduply-logs-* 인덱스 생성: 정상
Kibana Discover 조회: 정상
Spring 콘텐츠 검색 자동 재색인: 정상
contents 문서 수: 134
```
