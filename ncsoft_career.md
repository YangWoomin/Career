# Server Development
## System Architecture
* 경력 기술서 내용의 이해를 돕기 위한 간단한 시스템 구성도

![server2](https://github.com/user-attachments/assets/9d7fd7c4-c20b-4a51-9c4c-1d51fc2b3576)

## Auth
* (인증 서버는 초기 설계부터 구현 및 운영까지 작성자가 모두 작업함)
### 작업 내용
* 사용자가 외부에서 사용하는 플랫폼 계정을 게임 내 계정(account id)과 바인딩하여 외부 플랫폼 계정으로 게임을 플레이할 수 있게 지원
  + 연동한 외부 플랫폼은 사내 계정 관리 시스템(nano)과 스팀이 있음
* 인증 서버로부터 사용자가 인증된 클라이언트에게 인증 서버 외 다른 서버나 외부 시스템에서 쉽게 인증(권한 부여)할 수 있도록 내부 토큰 시스템 지원
  + 클라이언트는 지급 받은 내부 토큰을 다른 서버(로비)에게 제출하여 간단한 인증 수행
* 내부(자체) 계정 시스템 지원
* 처음에 Golang + Redis로 만들었으나 (auth1) 실 내부에서 사용하는 서버 프레임워크(C/C++) 기반으로 다시 만듦 (auth2)

## Lobby
* (서버들의 출시를 위한 scale out이 가능한 구조로의 개선이 지상과제가 되면서 작성자가 아래 정리한 작업 내용을 작업함)
### 작업 내용
#### Login
* 자체 세션 서버(C/C++)에서 유저 세션 관리하던 부분을 Redis++을 사용하여 메모리 데이터베이스(Redis)와 연동하도록 수정
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

## Game
* (서버들의 출시를 위한 scale out이 가능한 구조로의 개선이 지상과제가 되면서 작성자가 아래 정리한 작업 내용을 작업함)
* (Statistics subsystem은 초기 설계부터 구현 및 운영까지 작성자가 모두 작업함)
### 작업 내용
* 로비 서버와 마찬가지로 자체 세션 서버를 사용한 유저 세션 접근 부분을 memory database component를 로드하여 사용하도록 수정
* 로비 서버와 마찬가지로 게임 서버간의 연결을 제외한 나머지 서버들과의 근실시간 통신을 message queue client를 로드하여 사용하도록 수정
* 게임 서버 상태를 모니터링하기 위해 통계 지표를 처리하는 통계 시스템(Statistics System)이 있으며 게임 서버에서 통계 시스템과 연동하기 위한 부분으로 통계 서브시스템을 구현하여 통계 시스템 연동을 지원
  + 통계 지표로 CPU와 Memory(Virtual/Physical), Game Object Count, FPS, User Count, Game Server Status 등이 있음

# Statistics System
## Summary (features)
* 게임 서버 자체 상태와 월드 및 게임 오브젝트 관련 상태를 모니터링하기 위해 통계 지표를 처리하는 시스템
### Statistics db (InfluxDB)
* NoSQL 계열이며 데이터를 시간에 따라 효율적으로 저장/조회할 수 있는 시계열 데이터베이스
* 스트림(브랜치) 또는 production 환경마다 전용 bucket(rdb의 database와 대응되는 개념)으로 데이터를 나누어 관리
* InfluxQL이라는 자체 쿼리 언어로 효율적이고 다양한 방식으로 지표 조회가 가능
  +  
* 아직 database clustering 수준까지 진행하지는 못함
### Viewer (Grafana)
* Prometheus, InfluxDB, OpenTSDB 등의 시계열 데이터베이스로부터 지표를 수집하여 다양한 방식의 출력 방식(그래프 등)으로 지표를 시각화
## World contents visualize
### Summary
* 게임 서버의 게임 오브젝트들을 지도(맵) 상에서의 분포를 쉽게 파악하기 위해 자체 개발한 툴
* 통계 db로부터 게임 오브젝트 위치 데이터(지표)를 조회하여 동영상처럼 시간대 별로 맵 뷰어에 출력
* 누구나 쉽게 접근하기 위해 웹으로 서비스 제공
### Data pipeline
* 게임 서버가 통계 서브시스템을 통해 통계 db에 game object location 지표를 적재
  + 게임 서버에 있는 모든 게임 오브젝트의 현황인 snapshot을 적재하지 않고 위치 변화(diff)가 있는 게임 오브젝트만 위치 정보와 함께 적재
* 통계 db에서 자체 task를 주기적으로 돌려 실시간으로 입력되는 게임 오브젝트의 diff 지표를 snapshot 지표로 변환
  + 예를 들어 시점 1에 diff로 남은 오브젝트 A 위치 지표는 시점 2에 그 오브젝트 위치가 바뀌지 않아서 diff 지표가 남지 않더라도 시점 2 기준으로 조회를 했을 때 오브젝트 A가 계속 존재하는 것처럼 나타내기 위해 시점 2에 시점 1의 오브젝트 위치 데이터를 복제
  + diff -> snapshot 형태의 데이터 변환에 추가로 필요한 요소는 게임 서버 실행중/재시작 시점 정보가 있음
    + 서버 실행중인 상태와 종료된 상태를 인지하지 못하면 이전에 실행한 서버의 지표를 계속 snapshot으로 이어나갈 수 있는 문제가 있음
    + 서버가 계속 실행 중임을 나타내는 근거로 서버의 cpu 지표를 사용 (cpu 지표가 없으면 서버가 종료된 것으로 판단)
    + 서버가 재시작 된 것을 판단하기 위해 process id 지표를 사용 (재시작되면 process id가 달라짐)
  + 게임 오브젝트가 많아질수록 task 처리가 지연되는 현상이 있고 이는 task를 여러 개로 돌려 분산 처리할 수 있도록 작업 필요
* visualizer에서 가공된 snapshot 지표를 조회하여 시간대 별로 게임 오브젝트 현황을 파악할 수 있도록 동영상처럼 출력
### Visualizer
* frontend framework인 Vue.js 사용
* 웹 브라우저에서 맵 뷰어로 OpenLayers 사용
* UI 디자인은 Vuetify framework 사용
* 스트리밍 형식으로 데이터를 조회하여 출력
  + time bar에 설정된 전체 시간 구간에 대해 모든 데이터를 처음부터 다 조회하지 않고 time bar에서 현재 출력 시점을 나타내는 time cursor 기준으로 +5분 정도를 조회
  + time bar를 플레이하면 time cursor가 초 단위로 증가 (동영상 재생 효과)
  + time cursor가 앞서 데이터를 조회한 범위의 끝 부분에 다가오면 추가로 +5분 정도를 백그라운드로 조회 (반복)
  + time cursor를 임의의 시점으로 사용자가 직접 옮겼을 때 그 시점의 데이터가 없을 경우 그 시점 기준으로 +5분의 데이터를 조회
  + data pipeline 과정에서 통계 db의 task 처리 시간 확보를 위해 라이브 스트리밍은 1~3분 정도 딜레이를 설정
* 맵 뷰어에서 게임 오브젝트 개수를 그리드맵(grid map) 형식으로 나누어 분포도를 지역적으로 한눈에 알 수 있도록 기능 제공
* 실제 시현 영상
  + https://github.com/user-attachments/assets/7e5cd55c-d74c-4e28-be63-90c6272efe98
# Log System
## Summary (features)
* 게임 서버에서 발생한 로그를 통합 저장하고 관리할 수 있는 시스템
* CD로 배포된 서버의 로그를 쉽게 조회할 수 있도록 접근성 제공
* 개발자 또는 컨텐츠 기획자 등이 서버에서 발생한 로그를 활용(통계/분석 등)하여 필요한 정보를 획득
### Log forwarder (Fluentd)
* 게임 서버에서 남긴 로그 파일의 내용을 감시하면서 새로운 로그(행)가 추가되면 이를 추출하여 로그 수집기로 전송
* 로그 파일 이름으로 실행된 서버의 스트림(브랜치), 맵 이름, 버전, 실행 호스트 등을 추출
* 호스트에 직접 로그 forwarder(agent)를 설치하거나 컨테이너화한 후 서버가 실행되고 있는 호스트에 배포하여 서버 로그 전송
### Log aggregator (Fluentd)
* 여러 로그 전달자로부터 수집한 로그를 파싱하고 분류해서 로그 저장소에 적재
* 로그의 날짜 및 시간, 카테고리, 레벨(debug, info 등), 소스 파일과 행(line) 등을 추출
  + 서버의 어느 부분에서 로그를 많이 남기는지 파악하기 위해 소스파일 + 행 조합으로 통계를 내어 로그 관리 가능
  + 뷰어로 로그 조회 시 로그 카테고리나 레벨로 필터링하여 조회 가능
* 컨텐츠적인 로그도 따로 파싱 및 분류하여 통계를 낼 수 있도록 지원
  + 서버의 공간 정보 쿼리 성능 측정을 위해 사용
### Log storage (Elasticsearch)
* 데이터를 문서(json) 형태로 저장하는 NoSQL 계열 검색 엔진
* RESTful API로 제어가 가능하기 때문에 로그 관리 및 조회가 용이
* single node로 운영하였으며 cluster 모드로 운영해보진 못함
  + Elastic cloud라고 K8s operator 써서 cluster 배포해보긴 했는데 실제 적용 단계까지는 못해봄
* Elasticsearch 로그 관리
  + Elasticsearch에서 기본적으로 로그 1개 레코드를 document라는 단위로 관리하며 document들을 인덱스로 그룹화하여 관리
  + 그리고 인덱스를 클러스터를 이루는 여러 노드들에 샤드들로 분산하여 저장 및 관리
  + 로그 관리 작업은 대부분 인덱스 단위로 수행되며 적절히 그룹화하여 관리하기 위해 조직 환경에 맞는 용어들의 조합으로 인덱스를 네이밍하고 해당 로그를 저장
  + 또한 보편적으로 로그를 최소 일별 단위로 관리하기 때문에 년월일까지 로그 인덱스의 네이밍에 사용
  + repository는 로그 백업을 저장/복구하기 위한 논리적/물리적으로 분리된 공간이며 여러 인덱스 또는 하나의 인덱스를 하나의 repository에 저장하여 백업(snapshot) 및 복구(restore) 가능
* log retention
  + 사용자는 최근 30일 동안 남은 로그를 조회할 수 있도록 Elasticsearch에 1개월의 로그 인덱스만 로드하고 그 이전 로그는 로그 백업 파이프라인에 의해 백업되도록 자동화
* lob backup & restore pipeline
  + 로그 백업은 snapshot 개념으로 이뤄지며 repository에 여러 인덱스를 snapshot하여 백업
  + repository 생성 -> 로그 인덱스를 지정하여 snapshot 수행 -> repository 디렉토리를 통째로 압축 후 아카이빙
  + 반대로 복원은 아카이빙한 압축 파일을 미리 등록한 빈 repository에 풀고 Elasticsearch에서 해당 repository로부터 snapshot들을 사용하여 로그 인덱스를 복원
* 로그 관리 정책
  + 로그 인덱스 1개당 샤드 2개(primary + replica)
    + single node에서 replica shard는 unassigned 상태 (싱글 노드니깐)
  + 노드당 최대 샤드 개수 (max_shards_per_node)
    + 일별 최대 50개 인덱스 * 2개 샤드 * 30일 = 3000개
### Log viewer (Kibana)
* Elasticsearch에 저장되어 있는 로그를 다양한 방식으로 조회할 수 있는 웹 서비스
* 단순히 document 형식으로 조회 가능하고 다양한 graph 등을 사용하여 시각화 가능
# Monitoring System
## Summary (features)
* 실에서 운영하는 장비들과 서비스(소프트웨어)들을 모니터링하기 위한 시스템
### Metrics exporter
* 모니터링할 대상으로부터 지표를 노출시킬 에이전트
* exporter 종류
  + Node exporter (Windows/Linux)
    + 운영체제와 밀접하게 연동하여 운영체제의 여러 지표를 수집
    + cpu, memory, disk, network, 그리고 운영체제 자체 정보들을 수집
    + os마다 수집하는 지표가 다름
  + MySQL exporter
    + MySQL 데이터베이스에 MySQL 시스템적인 지표를 쿼리하여 수집
    + MySQL 메모리 사용량, 클라이언트 연결, slow queries, table locks 등
  + Elasticsearch exporter
    + Elasticsearch에 시스템적인 지표를 쿼리하여 수집
    + 클러스터 현황, JVM 메모리 사용량, cpu 및 메모리 사용량 등
### Metrics db (Prometheus)
* InfluxDB와 마찬가지로 시간에 따라 데이터를 효율적으로 저장/조회할 수 있는 시계열 데이터베이스
* 내장된 자체 언어인 PromQL로 지표를 다양하게 조회할 수 있음
### Alerter (AlertManager)
* Prometheus와 연동하여 다양한 방식으로 알람을 전송할 수 있는 소프트웨어
* e-mail, slack, teams 등 다양한 방식으로 알람 전송 가능
### Metrics viewer (Grafana)
* 시계열 데이터베이스로부터 다양한 방식의 출력 방식(그래프 등)으로 지표를 시각화
* 
# Development Environment
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
