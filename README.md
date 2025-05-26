# Vector

이 문서는 Ubuntu 환경에서 Vector 에이전트를 이용해 로그를 수집하고 Kafka로 전송하기 위한 설정 및 실행 방법을 안내합니다.

## 목차

* [아키텍처](#아키텍처)
* [요구 사항](#요구-사항)
* [디렉토리 구조](#디렉토리-구조)
* [설치 및 실행](#설치-및-실행)
* [필수 수정 항목 요약](#필수-수정-항목-요약)
* [운영 및 확장](#운영-및-확장)

## 아키텍처

* **Vector Agent (docker-compose)**

  * **로그 소스**: `/var/log`
  * **로그 수집**: Vector File Source
  * **로그 전송**: Kafka의 `raw-logs` 토픽

## 요구 사항

* Ubuntu 20.04 LTS 이상
* Docker & Docker Compose 설치
* Kafka 브로커 접근 권한 (호스트, 포트, 인증 정보)

## 디렉토리 구조

```
├─ docker-compose.yml       # Vector Agent 서비스 정의
└─ vector-agent.toml        # Vector 설정 파일
```

## 설치 및 실행

1. 저장소 클론

   ```bash
   git clone <repo-url>
   cd <repo-dir>
   ```

2. 설정 파일 수정

   ### docker-compose.yml

   * `image`: `timberio/vector:latest-alpine` 대신 특정 버전을 고정하려면 태그 변경
   * `container_name`: 필요에 따라 서비스 이름 변경
   * `volumes`:

     * 호스트의 로그 디렉토리 경로(`/var/log`)가 맞는지 확인
     * 프로젝트 내 `vector-agent.toml` 경로가 올바른지 확인

   ```yaml
   services:
     vector-agent:
       image: timberio/vector:<TAG>
       container_name: vector-agent
       entrypoint: ["vector", "-c", "/etc/vector/vector.toml"]
       volumes:
         - /var/log:/var/log:ro      # 호스트 로그 경로
         - ./vector-agent.toml:/etc/vector/vector.toml:ro
   ```

   ### vector-agent.toml

   * `sources.file.include`: 수집할 로그 파일 패턴을 환경에 맞게 수정

     * 예: `/var/log/syslog`, `/var/log/nginx/*.log`, `/custom/path/*.log`
   * `ignore_older`: 파일 변경이 없을 때 무시할 최대 시간(초)
   * `sinks.kafka_raw.bootstrap_servers`: 실제 Kafka 브로커 주소(호스트:포트) 및 포트 리스트로 수정
   * `sinks.kafka_raw.topic`: 전송할 Kafka 토픽 이름 설정

   ```toml
   [sources.file]
   type         = "file"
   include      = ["/var/log/syslog", "/var/log/nginx/*.log"]
   ignore_older = 86400            # 24시간 이상 변경 없으면 건너뜀

   [sinks.kafka_raw]
   type              = "kafka"
   inputs            = ["file"]
   bootstrap_servers = [
     "broker1.example.com:9092",
     "broker2.example.com:9092"
   ]
   topic             = "raw-logs"
   encoding.codec    = "json"
   compression       = "none"
   # key_field = "host"         # 메시지 키로 사용할 필드 or 로그 메타
   # batch.max_size = 500         # 배치 크기 조정
   # batch.timeout_secs = 1       # 배치 타임아웃
   ```

3. 컨테이너 실행

   ```bash
   docker-compose up -d
   ```

4. 상태 확인

   ```bash
   docker ps | grep vector-agent
   docker logs -f vector-agent
   ```

## 필수 수정 항목 요약

| 파일                 | 설정 항목              | 설명                                    |
| ------------------ | ------------------ | ------------------------------------- |
| docker-compose.yml | image              | Vector 버전 태그 (`:latest-alpine` 대체 가능) |
|                    | volumes 호스트 경로     | 호스트 로그 디렉토리 경로 맞춤                     |
| vector-agent.toml  | include            | 로그 수집 대상 경로 및 패턴                      |
|                    | ignore\_older      | 무시할 오래된 파일 최대 시간 (초)                  |
|                    | bootstrap\_servers | Kafka 브로커 주소 목록                       |
|                    | topic              | Kafka 토픽 이름                           |

## 운영 및 확장

* **다른 OS 지원**:

  * **CentOS / RHEL**: `yum install docker docker-compose` 명령 사용, 로그 경로(`/var/log/messages`, `/var/log/secure`)로 설정
  * **Alpine Linux**: `apk add docker docker-compose`, Vector 바이너리 Alpine 패키지로 설치 후 `/var/log/*` 패턴 사용
  * **Debian**: Ubuntu와 동일한 APT 명령, 시스템 서비스는 `systemctl`으로 관리
  * **Windows (WSL2 또는 Windows Server)**: Docker Desktop 설치 후 볼륨 마운트(`//var/log`) 대신 `C:\logs` 경로 매핑

* **수평 확장**: 여러 Vector 에이전트를 배포하여 로그 수집 용량 확장

  * Kubernetes 환경: DaemonSet으로 배포하여 모든 노드 로그 수집
  * Docker Swarm: 서비스 모드로 배포, 볼륨 스코프 `global`로 설정

* **모니터링**: Vector 메트릭(`prometheus` 모듈) 수집 후 Grafana나 Datadog으로 시각화

* **업데이트 관리**: Docker 이미지 태그로 버전 관리, GitOps(CI/CD) 파이프라인으로 자동 배포

* **보안 및 네트워크**:

  * TLS 인증: Kafka, Elasticsearch 등 대상 시스템과의 암호화 통신 설정
  * 방화벽 설정: UFW, firewalld, Windows Defender 방화벽 규칙으로 포트 제어

* **고가용성(HA)**: 로그 전송 실패 시 로컬 디스크에 버퍼링, 재시도 로직 활성화

* **유연한 설정 관리**: ConfigMap, Secrets 활용하여 OS별 환경 변수 및 자격증명 분리
