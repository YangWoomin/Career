# 0. Contents Table
* [1. Server Development](#1-Server-Development)
  + [1. System Architecture](#11-System-Architecture)
  + [2. Auth](#12-Auth)
* [2. Infrastructure](#2-Infrastructure)

# 1. Server Development
## 1.1 System Architecture
* 경력 기술서 내용의 이해를 돕기 위한 간단한 시스템 구성도

![server2](https://github.com/user-attachments/assets/9d7fd7c4-c20b-4a51-9c4c-1d51fc2b3576)

## 1.2 Auth
* (인증 서버는 초기 설계부터 구현 및 운영까지 작성자가 모두 작업함)
### 작업 내용
* 사용자가 외부에서 사용하는 플랫폼 계정을 게임 내 계정(account id)과 바인딩하여 외부 플랫폼 계정으로 게임을 플레이할 수 있게 지원
  + 연동한 외부 플랫폼은 사내 계정 관리 시스템(nano)과 스팀이 있음
* 인증 서버로부터 사용자가 인증된 클라이언트에게 인증 서버 외 다른 서버나 외부 시스템에서 쉽게 인증(권한 부여)할 수 있도록 내부 토큰 시스템 지원
  + 클라이언트는 지급 받은 내부 토큰을 다른 서버(로비)에게 제출하여 간단한 인증 수행
* 내부(자체) 계정 시스템 지원
* 처음에 Golang + Redis로 만들었으나 (auth1) 자체 서버 프레임워크(C/C++) 기반으로 다시 만듦 (auth2)
### Trouble shooting

## 3) Lobby
* (서버들의 출시를 위한 scale out이 가능한 구조로의 개선이 지상과제가 되면서 작성자가 아래 정리한 작업 내용을 작업함)
### 작업 내용
#### Login
* 로비 서버가 자체 세션 서버(C/C++)와 연동하여 유저 세션 관리하던 부분을 Redis++을 사용하여 메모리 데이터베이스(Redis)와 연동하도록 수정
  + Redis++을 wrapping한 별도 라이브러리(dll, memory database component)를 구현 후 로비 서버에서 로드하여 사용
* 위 작업으로 인한 유저 로그인 처리 로직 대부분을 수정
#### Nickname & Character
* 인게임에서 사용할 닉네임과 캐릭터 관련 준비 작업(In-game Preparation)하는 부분이 원래 로비 서버가 별도 서버들과 연동하여 수행했던 부분들을 모두 라이브러리화하여 연동하도록 수정
  + 로비에서 닉네임 생성/수정 또는 캐릭터 생성/삭제 등의 작업이 가능함
  + 계정 서버와 캐릭터 서버가 별도로 존재했었음 (당시 서버 구조가 MSA를 추구)
  + 별도 서버들을 모두 라이브러리(dll)로 변환하여 로비 서버에서 로드하고 사용하도록 수정
#### Inter-server Communication 
* 서버들 사이의 연결 복잡도를 줄이고 통신 방법을 표준화하기 위하여 메시지 큐(Kafka) 도입
  + 근실시간 동기화 수준 정도면 충족되는 서버들 사이의 연결에 한함
  + Redis++와 마찬가지로 librdkafka 라이브러리를 wrapping하여 별도 라이브러리(dll, message queue client)를 구현 후 로비 서버에서 로드하여 사용
### Trouble shooting
* Component dll 로드 시 ABI 문제
* Redis++의 Sync -> Async 전환
* (TODO) Redis 클러스터 모드에 따른 대응

## 4) Game
* (서버들의 출시를 위한 scale out이 가능한 구조로의 개선이 지상과제가 되면서 작성자가 아래 정리한 작업 내용을 작업함)
* (Statistics subsystem은 초기 설계부터 구현 및 운영까지 작성자가 모두 작업함)
### 작업 내용
* 로비 서버와 마찬가지로 자체 세션 서버를 사용한 유저 세션 접근 부분을 memory database component를 로드하여 사용하도록 수정
* 로비 서버와 마찬가지로 게임 서버 서로간의 연결을 제외한 나머지 서버들과의 근실시간 통신을 위해 message queue client를 로드하여 사용하도록 수정
* 게임 서버 상태를 모니터링하기 위해 통계 지표를 처리하는 통계 시스템(Statistics System)이 있으며 게임 서버에서 통계 시스템과 연동하기 위한 부분으로 통계 서브시스템을 구현하여 통계 시스템 연동을 지원
  + 통계 지표로 CPU와 Memory(Virtual/Physical), Game Object Count, FPS, User Count, Game Server Status 등이 있음
### Trouble shooting
* 자체 서버 프레임워크의 유저 세션과 memory database 유저 세션간 충돌
* 통계 서브시스템에서 다중 writer 지원을 위한 시스템 구조 개선

# 2. Infrastructure
## 1. Statistics System
* (통계 시스템 관련 모든 작업은 작성자가 전담함)
### 1) 사용 목적
* 게임 서버 상태를 모니터링하기 위한 지표 수집 및 저장 관리
### 2) 구성 요소
#### Statistics db (InfluxDB)
* 스트림(브랜치) 또는 production 환경마다 전용 bucket(rdb의 database와 대응되는 개념)으로 데이터를 나누어 관리
* InfluxQL 자체 쿼리 언어를 사용하여 게임 관련 지표 조회
#### Viewer (Grafana)
* 개발자들이 게임 서버 상태를 쉽게 모니터링하기 위해 웹 서비스 형태로 서비스 제공
* InfluxQL 언어를 사용하여 지표를 시각화 (그래프)
#### World contents visualize
##### 개요
* 게임 서버의 게임 오브젝트들을 지도(맵) 상에서의 분포를 쉽게 파악하기 위해 자체 개발한 툴
* 통계 db로부터 게임 오브젝트 위치 데이터(지표)를 조회하여 동영상처럼 시간대 별로 맵 뷰어에 출력
* 누구나 쉽게 접근하기 위해 웹으로 서비스 제공
##### Visualizer
* frontend framework인 Vue.js 사용
* 웹 브라우저에서 맵 뷰어로 OpenLayers 사용
* UI 디자인은 Vuetify framework 사용
* 스트리밍 형식으로 데이터를 조회하여 동영상처럼 재생하며 출력
* 맵 뷰어에서 게임 오브젝트 개수를 그리드맵(grid map) 형식으로 나누어 분포도를 지역적으로 한눈에 알 수 있도록 기능 제공
* 실제 시현 영상
  + https://github.com/user-attachments/assets/7e5cd55c-d74c-4e28-be63-90c6272efe98
### 3) Trouble shooting
* InfluxDB의 OOM에 의한 강제 재시작 현상
* World contents visualizer - snapshot 데이터 vs diff 데이터

## 2. Log System
* (로그 시스템 관련 모든 작업은 작성자가 전담함)
### 1) 사용 목적
* 게임 서버에서 발생한 로그를 통합 저장하고 관리
* 개발자 또는 컨텐츠 기획자 등이 서버에서 발생한 로그를 분석하기 위한 통계 및 시각화 등의 기능 제공
### 2) 구성 요소
#### Log forwarder (Fluentd)
* 게임 서버에서 남긴 로그 파일의 내용을 감시하면서 새로운 로그(행)가 추가되면 이를 추출하여 로그 수집기로 전송
* 로그 파일 이름으로 실행된 서버의 스트림(브랜치), 맵 이름, 버전, 실행 호스트 등을 추출
* 호스트에 직접 로그 forwarder(agent)를 설치하거나 컨테이너화한 후 서버가 실행되고 있는 호스트에 배포하여 서버 로그 전송
#### Log aggregator (Fluentd)
* 여러 로그 전달자로부터 수집한 로그를 파싱하고 분류해서 로그 저장소에 적재
* 로그의 날짜 및 시간, 카테고리, 레벨(debug, info 등), 소스 파일과 행(line) 등을 추출
  + 소스파일 + 행 조합 통계로 불필요한 로그 추적에 용이
  + 뷰어로 로그 조회 시 로그 카테고리나 레벨로 필터링하여 조회 가능
* 컨텐츠적인 로그도 따로 파싱 및 분류하여 통계를 낼 수 있도록 지원
  + 서버의 공간 정보 쿼리 성능 측정을 위해 사용
#### Log storage (Elasticsearch)
* 로그 관리를 위해 스트림 + 유저(호스트네임) + 년월일 단위로 인덱스를 관리
* TeamCity에서 Elasticsearch RESTful API로 접근하여 로그 백업 및 관리
  + 사용자는 최근 1개월 동안 남은 로그를 조회할 수 있도록 Elasticsearch에 1개월의 로그 인덱스만 로드하고 그 이전 로그는 로그 백업 파이프라인에 의해 백업되도록 자동화
* single node로 운영하였으며 cluster 모드로 운영해보진 못함
  + Elastic cloud라고 K8s operator 써서 cluster 배포해보긴 했는데 실제 적용 단계까지는 못해봄
#### Log viewer (Kibana)
* 개인 서버가 아닌 (배포된) 공용 서버의 로그에 다른 개발자들이 쉽게 접근할 수 있도록 웹 서비스 제공
* 로그 저장소에 저장된 로그들을 다양한 방식으로 조회 및 통계 작업이 가능하도록 지원
### 3) Trouble shooting
* 노드당 최대 샤드 개수(max_shards_per_node) 이슈

## 3. Monitoring System
* (모니터링 시스템 관련 모든 작업은 작성자가 전담함)
### 1) 사용 목적
* 실에서 운영하는 장비들과 서비스(소프트웨어)들을 모니터링
### 2) 구성 요소
#### Metrics exporter
##### Node exporter (Windows/Linux)
* 호스트 종료 및 리소스(CPU, Memory, Disk) 과용 탐지
  + 특히 각 호스트의 디스크 부족 현상에 의한 불능 상태를 미리 탐지
##### MySQL exporter
* MySQL 메모리 사용량, 클라이언트 연결, slow queries, table locks 등
  + DBA가 주로 이용
##### Elasticsearch exporter
* 클러스터 현황, JVM 메모리 사용량, cpu 및 메모리 사용량 등
#### Metrics db (Prometheus)
* PromQL로 지표를 다양하게 조회하거나 알람 조건을 설정
#### Alerter (AlertManager)
* 장비의 리소스 과용 시 teams 채널에 메일로 알람 전송
#### Metrics viewer (Grafana)
* Prometheus로부터 다양한 지표를 그래프 등으로 시각화하여 출력
### 3) Trouble shooting

# 3. Development Environment
## VCS (Version control system)
* Perforce 사용 (빌드관리팀이 퍼포스 서버를 직접 운영)
* 사용자(개발자나 기획자 등)는 P4V를 사용하여 sync/submit 등의 작업 수행
## Build & Run
* 크로스플랫폼(Windows/Linux) 지원을 위해 CMake/Make 사용
* CMake를 사용하여 프로젝트 별로 Visual studio 프로젝트 파일 및 make 파일 생성
  + 윈도우 환경에서 Visual studio를 사용하여 작업 (주로 디버깅 용도로 사용)
  + Visual studio code도 사용 (주로 소스코드 작업할 때 사용)
* Make를 사용하여 빌드
  + Linux 환경은 원래 Make 명령어로 빌드가 가능하지만 윈도우 환경에서는 Visual studio 프로젝트에서 빌드 (또는 msbuild.exe 사용)
  + 빌드를 Make로 통일하기 위해 Make 스크립트를 사용하여 빌드
* 컴파일러는 윈도우 환경에서 MSVC를, 리눅스 환경에서 Clang 사용
* 실행
  + 서버 소스코드를 내려받아서 빌드를 돌리지 않아도 기본적으로 서버 CI에 의해 자동 서밋되는 서버 바이너리들로 서버 실행이 가능
  + 서버 실행에만 필요한 설치 프로그램으로 vc_redist.x64(2022), mysql-odbc-connector(v8.0.28)가 있음
## Database
* (아래와 같은 평소 db 관련 작업은 보통 작성자가 아닌 DBA 작업자가 수행하며 작성자는 MySQL 설치, 초기 세팅, 모니터링, 운영 등의 작업에 관여함)
* 별도 로컬 데이터베이스 운영을 지원하지 않고 공용 데이터베이스만 운영
* DBA가 db 작업이 있을 때만 로컬 db에 직접 세팅해서 테스트 해보고 소스파일 서밋할 때 공용 db에도 같이 적용
  + 상당히 비효율적이고 db 연관 작업을 진행하기가 쉽지 않음 (항상 DBA를 끼고 작업해야 하기 때문)
  + 하지만 기획자나 개발자들의 작업이 주로 db보다는 (정적인) 데이터 파일 위주의 작업이 많은 관계로 이 방식으로 운영이 가능 
* 스트림마다 소스파일 및 바이너리를 merge할 때마다 공용으로 운영하는 해당 스트림의 db에도 맞춰서 업데이트 수행
* DBA가 주로 db migration tool을 가지고 스크립트 실행
  + db migration tool은 python으로 작성됨
## CI
* Teamcity에서 job(build configuration)으로 서버 모듈 별로 CI를 돌림
  + Teamcity에 VCS로 P4를 지정하고 P4에서 각 서버별 소스 파일들 변경이 생기는지 감시하다가 변경이 발생하면 Teamcity가 트리거링 되어 CI 빌드 수행
# CD
## Inhouse Launcher
## K8s
## NCKUBE
# GStar
## Ansible
