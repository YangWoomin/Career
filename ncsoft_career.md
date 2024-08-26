# 0. Contents Table
* [1. Server Development](#1-Server-Development)
  + [1.1 System Architecture](#11-System-Architecture)
  + [1.2 Auth](#12-Auth)
  + [1.3 Lobby](#13-Lobby)
  + [1.4 Game](#14-Game)
* [2. Infrastructure](#2-Infrastructure)
  + [2.1 Statistics System](#21-Statistics-System)
  + [2.2 Log System](#22-Log-System)
  + [2.3 Monitoring System](#23-Monitoring-System)

# 1. Server Development
## 1.1 System Architecture
* 경력 기술서 내용의 이해를 돕기 위한 간단한 시스템 구성도

![server2](https://github.com/user-attachments/assets/9d7fd7c4-c20b-4a51-9c4c-1d51fc2b3576)

## 1.2 Auth
* (인증 서버는 초기 설계부터 구현 및 운영까지 작성자가 모두 작업함)
### 작업 내용
* 사용자가 외부에서 사용하는 플랫폼 계정을 게임 내 계정(account id)과 바인딩하여 외부 플랫폼 계정으로 게임을 플레이할 수 있게 지원
  + 연동한 외부 플랫폼은 사내 계정 관리 시스템(nano)과 스팀이 있음
* 인증 서버로부터 사용자가 인증된 클라이언트에게 인증 서버 외 다른 서버나 외부 시스템에서 쉽게 인증(권한 부여)할 수 있도록 내부 토큰(LLL Token) 시스템 지원
  + 클라이언트는 지급 받은 내부 토큰을 다른 서버(로비)에게 제출하여 간단한 인증 수행
* 내부(자체) 계정 시스템 지원
* 처음에 Golang + Redis로 만들었으나 (auth1) 자체 서버 프레임워크(C/C++) 기반으로 다시 만듦 (auth2)
### Trouble shooting
<details>
<summary>Trouble shooting 사례 보기</summary>

> #### 개발 환경에서의 멀티 클라이언트 및 멀티 로그인 지원을 위한 내부 계정 시스템 도입
> * 클라이언트/서버 개발자들이 하나의 로컬 PC에서 여러 개의 클라이언트를 실행하여 테스트 진행하는 경우가 잦았음
> * 사내 계정(nano)은 1인 1개씩 부여되므로 1개 nano 계정으로 밖에 로그인을 할 수 밖에 없는 상황
> * 하지만 게임 규칙 상 1개 클라이언트는 1개 계정으로만 플레이가 가능하고 서버 기술적으로도 여러개 클라이언트가 1개의 계정을 동시에 사용하는 경우는 지원되지 않았음
> * 따라서 1인당 다수의 계정이 필요한 상황임이 인지되었고 인증 서버를 개발하여 오픈하기 전에 내부 계정 시스템을 도입
> * 인증 db에 nano 계정과 게임 계정(account id)을 바인딩하는 테이블 외에 자체 계정(id/pw)을 저장하는 테이블을 추가
> * (내부)계정 관리 툴(웹 서비스)에 사용자가 직접 nano 계정으로 접근하여 내부 계정을 생성하고 지급 받음
>   + 계정 관리 툴 + 계정 관리 서비스는 별도 담당자가 작업 및 관리
>   + 계정 관리 서비스에서 인증 db에 접근하여 id/pw를 추가하거나 삭제 등의 동작 지원
> * 인증 서버는 OAuth 토큰 기반 외부 IdP(nano 및 스팀) 인증 방법 외에 내부 계정 인증 서비스를 제공
> #### 내부 계정 토큰화 지원
> * 내부 계정은 id/pw 형식이며 클라이언트에서 직접 입력하여 로그인할 수도 있지만 별도 파일에 저장하여 자동 로그인 시에도 사용 가능
>   + 자동 로그인은 사용자가 직접 id/pw를 입력할 필요 없이 클라이언트가 계정이 저장된 파일로부터 계정을 읽어들여 로그인하는 기능
> * 하지만 이로 인해 계정이 평문 그대로 파일에 노출되는 것과 id/pw를 한번 발급하면 관리(특히 회수/무효화)가 어렵다는 문제가 생김
> * 따라서 평문 노출을 없애고 발급한 계정의 만료 기간을 설정하기 위해 내부 토큰화 기능을 지원
>   + 사용자는 계정 관리 툴에서 계정을 생성한 후 토큰화를 추가로 수행하여 토큰이 저장된 파일을 로컬 PC에 다운로드
>     + id/pw를 토큰 생성 시 토큰 필드에 추가하여 encryption
>   + 클라이언트가 자동 로그인 시 다운로드한 파일로부터 토큰을 읽어들여 로그인 시도
>   + 계정 관리 서버(python flask)에서 토큰화는 인증 서버가 사용하는 토큰 라이브러리(dll)를 로드하여 토큰을 생성하고 발급
>   + 토큰은 만료시점 필드가 내부에 저장되어 있으며 인증 서버가 토큰 해석 시 토큰이 만료되었으면 로그인을 거절
> #### 내부 토큰 암호화 알고리즘의 AES 256 CBC -> AES 256 ECB 전환
> * OAuth 토큰 해석법으로, 토큰 발급자에게 토큰을 전달하여 해당 토큰의 정보를 질의하거나 토큰을 직접 해석하는 방법이 있음
> * 인증 서버는 기본적으로 토큰을 직접 해석하는 방법을 지원
> * 따라서 토큰을 생성하고 해석하는 별도 라이브러리(dll, token)를 토큰 관련 작업이 필요한 서버들이 로드하여 토큰을 생성하고 암호화하거나 복호화 수행
> * 토큰은 JWT 포맷을 따르며 암호화 알고리즘으로 AES 256 ECB를 사용
>   + cbc(cipher block chaining)를 사용하지 않고 ecb(electronic code book)를 사용한 이유는 cbc 특성상 이전에 암호화된 블럭을 (다시 XOR하여) IV로 사용하기 때문에 토큰간 의존성(순서)이 생김
>   + 따라서 스트림 데이터에 쓰이는 더욱 강력한 암호화 알고리즘인 cbc를 사용하는 대신 조금 덜 보안적이지만 의존성을 없애기 위해 ecb 사용
>   + 토큰 질의(introspection) 서비스는 외부 시스템에서 사용할 수 있게 제공할 예정이었으나 이 부분 작업 진행은 우선순위 이슈로 홀드된 상태
> * JWT 토큰 암호화는 HS256 사용
> * 토큰 암호화 키와 HMAC 키는 별개이며 모두 소스코드에 박혀져 있음
>   + 소스코드 접근은 일부 인원에게만 허용됨
> * 토큰 암호화/복호화를 위해 openssl3 라이브러리 사용
> #### 토큰 vs 세션
> * LLL 시스템은 토큰과 세션 모두를 채용
>   + 보통 토큰과 세션은 대조되는 개념으로 취급됨
>   + 사용자 인증 관련한 부분은 토큰을 사용하고 내부 시스템에서 좀 더 복잡한 유저의 상태 관리는 세션을 사용

</details>

## 1.3 Lobby
* (서버들의 출시를 위한 scale out이 가능한 구조로의 개선이 지상과제가 되면서 작성자가 아래 정리한 작업 내용을 작업함)
### 작업 내용
#### Login
* 로비 서버가 자체 세션 서버(C/C++)와 연동하여 유저 세션을 관리하던 부분을 Redis++을 사용하여 Redis와 연동하도록 수정
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
  + memory database component와 마찬가지로 librdkafka 라이브러리를 wrapping하여 별도 라이브러리(dll, message queue client component)를 구현 후 로비 서버에서 로드하여 사용
### Trouble shooting
<details>
<summary>Trouble shooting 사례 보기</summary>
  
> #### Component dll 로드 시 ABI 문제
> * 앞서 설명한 memory database component, message queue client component 등 오픈소스 라이브러리를 한번 wrapping하여 다시 dll로 뽑아냄
>   + 오픈소스의 필요한 기능만 제한적으로 사용하도록, 일관적인 코드 사용 등을 위해 wrapping함
> * 서버 빌드 시 컴파일러 버전과 옵션 등의 세팅을 모든 개발자 PC에서 동일하게 사용하도록 빌드 환경 및 시스템이 구축되어 있어서 C++ 요소들을 그대로 dll export하여도 문제가 되지 않았음
>   + 위에서 설명한 서버 구성도의 모든 서버들은 C/C++로 작성된 공통의 서버 프레임워크(dll)를 사용함
> * 하지만 vcpkg로 다운받은 redis++ 라이브러리나 librdkafka 라이브러리를 wrapping한 component 라이브러리를 서버에서 로드하여 사용할 때 ABI 문제가 발생
>   + component 라이브러리의 소스코드가 vcpkg 라이브러리 함수 호출 후 반환값(문자열 등)에 접근하면 크래시 발생
>   + 오픈소스라서 직접 소스코드를 다운받고 빌드하는 방법이 있었지만 앞으로 사용할 모든 외부 라이브러리가 오픈소스임이 보장되지 않기 때문에 이 문제를 해결해야 했음
> * ABI 문제를 피하기 위한 방법으로 컴파일러와 컴파일러 버전, 옵션 등을 맞춰서 빌드하는 방법이 있지만 다양한 외부 라이브러리를 사용할 경우 이 방법은 매우 어려운 방법이 됨
> * 따라서 ABI에 대한 표준을 정의한 C 타입을 사용하도록 component 라이브러리들을 수정
> #### Redis++의 Sync -> Async 전환
> #### 개발 환경에서의 멀티 & 자동 로그인 지원을 위한 중복 로그인 처리

</details>

## 1.4 Game
* (서버들의 출시를 위한 scale out이 가능한 구조로의 개선이 지상과제가 되면서 작성자가 아래 정리한 작업 내용을 작업함)
* (Statistics subsystem은 초기 설계부터 구현 및 운영까지 작성자가 모두 작업함)
### 작업 내용
* 로비 서버와 마찬가지로 자체 세션 서버를 사용한 유저 세션 접근 부분을 memory database component를 로드하여 사용하도록 수정
* 로비 서버와 마찬가지로 게임 서버 서로간의 연결을 제외한 나머지 서버들과의 연결 복잡도를 낮추기 위해 message queue client component를 로드하여 사용하도록 수정
* 게임 서버 상태를 모니터링하기 위해 통계 지표를 처리하는 통계 시스템(Statistics System)이 있으며 게임 서버에서 통계 시스템과 연동하기 위한 부분으로 통계 서브시스템을 구현하여 통계 시스템 연동을 지원
  + 통계 지표로 CPU와 Memory(Virtual/Physical), Game Object Count, FPS, User Count, Game Server Status 등이 있음
### Trouble shooting
<details>
<summary>Trouble shooting 사례 보기</summary>

> #### 자체 서버 프레임워크의 유저 세션과 memory database 유저 세션간 충돌
> #### 통계 서브시스템에서 다중 writer 지원을 위한 시스템 구조 개선

</details>

# 2. Infrastructure
## 2.1 Statistics System
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
<details>
<summary>Trouble shooting 사례 보기</summary>

> #### InfluxDB의 OOM에 의한 강제 재시작 현상
> #### World contents visualizer - snapshot 데이터 vs diff 데이터

</details>

## 2.2 Log System
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
<details>
<summary>Trouble shooting 사례 보기</summary>

> #### 노드당 최대 샤드 개수(max_shards_per_node) 이슈
> #### Fluent-bit -> Fluentd migration 이슈
> #### Fluentd 로그 flooding 이슈

</details>

## 2.3 Monitoring System
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

# 2.3 Development Environment
## VCS (Version control system)
* Perforce 사용 (빌드관리팀이 퍼포스 서버를 직접 운영)
* 사용자(개발자나 기획자 등)는 P4V를 사용하여 sync/submit 등의 작업 수행
## Build & Run
* 크로스 플랫폼(Windows/Linux) 지원을 위해 CMake/Make 사용
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
