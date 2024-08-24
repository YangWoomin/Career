# Server Development
## Overview
![server2](https://github.com/user-attachments/assets/9d7fd7c4-c20b-4a51-9c4c-1d51fc2b3576)
## Auth
### Summary
* 인증 서버는 유저가 클라이언트로 게임 플레이를 하기 전에 누구인지 식별 수행
* 클라이언트가 게임 서버 내에 있는 유저의 자원을 사용하기 앞서 자원의 소유자를 식별하기 위한 계정 식별자(account id)를 제공
### Authentication
#### External IdP
* 런처나 클라이언트가 각 플랫폼 별 로그인 방식에 따라 로그인 후 OAuth 2.0 토큰 획득
  + 런처가 토큰을 획득한 경우 클라이언트를 실행할 때 토큰을 클라이언트 매개변수로 전달하거나, 토큰 파일에 저장해 놓으면 클라이언트에서 로드
* 클라이언트가 인증 서버에 연결 후 외부 플랫폼 토큰을 토큰 종류(Idp Kind)와 함께 전달
* 인증 서버는 전달받은 토큰의 종류에 따라 외부 IdP API에 해당 토큰을 전달하여 유저 식별 정보를 획득
  + 유저 식별 정보로 nano는 email address, steam은 steam id를 사용
  + 보통 https 프로토콜로 웹 서버(IdP)에게 요청하며 인증 서버에서는 openssl + libcurl을 사용하여 통신
* 토큰으로 획득한 유저 식별 정보가 인증 db에서 account id와 바인딩되어 있는지 확인 후 바인딩 정보가 있으면 로드하고 없으면 로비 서버와 연계하여 바인딩
  + account id는 게임 내 유저를 식별하기 위한 식별자
  + 인증 서버가 로비 서버에서 새로 생성한 account id를 획득하여 유저 식별 정보랑 바인딩 (인증 db와 게임 db는 분리되어 있고 접근 권한이 분리되어 있음)
  + 인증 db에서 account id와 유저 식별 정보의 바인딩 형태는 platform account id(PK) <-> account id(FK) 테이블(1번 테이블), 유저 식별 정보(PK) <-> platform account id(FK) 테이블(2번 테이블)들로 구성
    + 1번 테이블은 global하게 1개 있고 2번 테이블은 외부 플랫폼마다 1개씩 존재 (플랫폼마다 식별자 타입이 다를 수 있으므로)
* account id와 기타 정보(토큰 발행 시점, 유효 기간 등)를 같이 토큰으로 토큰화 및 암호화하여 클라이언트에게 전달
* 클라이언트는 로비 서버 접속을 위해 인증 서버에 연결되어 있는 로비 서버의 접속 정보를 요청하여 획득
#### Internal authentication
* 계정 관리 툴(account manager)에서 내부 계정 생성
  + 계정 관리 툴은 웹 브라우저로 접근 가능하며 계정 관리 서버가 서비스 제공
    + 계정 관리 서버는 다른 팀원 분이 제작 및 관리
    + 계정 관리 서버는 사내 계정(nano)으로 로그인(OAuth 2.0)이 가능하고 유저는 자신만의 내부 계정을 관리할 수 있음
    + 계정 관리 서버에서 인증 db에 직접 접근하여 내부 계정 테이블에 id/pw + 사내 계정 id(owner) 형식으로 저장
* 생성된 내부 계정 발급과 동시에 계정을 토큰화하여 토큰(내부 계정 토큰)도 같이 발급 가능
  + 토큰화는 계정 관리 서버가 token 라이브러리를 로드하기 때문에 토큰 생성이 가능함
  + id/pw를 토큰 안에 넣어서 encrypt 수행
* 발급한 내부 계정 토큰(또는 내부 계정)은 클라이언트가 로드할 수 있는 특정 경로에 파일(내부 토큰 파일)로 저장하고 클라이언트는 실행 시 이 파일로부터 토큰을 획득
* 클라이언트가 내부 계정 토큰임을 나타내는 토큰 종류와 함께 내부 계정 토큰을 인증 서버에 연결 후 전달
* 인증 서버는 내부 계정 토큰을 해석(decrypt)하여 토큰 안에 있는 id/pw를 추출하고 인증 db에서 비교하여 인증 과정 수행
  + 내부 계정 자체(id/pw)일 경우 클라 <-> 인증 서버의 패킷 종류가 다름
  + 인증 성공 시 id/pw에 바인딩된 account id를 획득
* account id와 기타 정보(토큰 발행 시점, 유효 기간 등)를 같이 토큰으로 토큰화 및 암호화하여 클라이언트에게 전달
* 클라이언트는 로비 서버 접속을 위해 인증 서버에 연결되어 있는 로비 서버 접속 정보를 요청하여 획득
  + external IdP 7번 과정과 동일
### Token
* token이라는 별도 라이브러리(dll) 형태로 토큰 생성(암호화) 및 해석(복호화) 기능을 제공하며 인증 서버와 로비 서버에서 로드하여 사용
* 토큰 자체는 JWT(Json Web Token) 형태이며 암호화 알고리즘은 HS256을 사용
* JWT 생성 및 암호화 (HS256)
  + JWT header와 body 생성
  + token payload : Base64(header) + Base64(body)
  + token signature : Bse64(hmac_sha256(token payload))
  + plain text token : token payload + token signature
  + encrypted token : aes_256_ecb(plain text token)
  + (todo) Base64 encrypted token : Base64(encrypted token)
    + (as is) readable token : HexToString(encrypted token)
* 토큰 암호화/복호화를 위해 openssl3 라이브러리 사용
  + 암호화 알고리즘으로 aes 256 ecb 사용
    + cbc(cipher block chaining)를 사용하지 않고 ecb(electronic code book)를 사용한 이유는 cbc 특성상 이전에 암호화된 블럭을 IV로 사용하기 때문에 토큰간 의존성(순서)이 생김
    + 토큰간에 의존성이 생기면 안되기 때문에 ecb 사용
  + HMAC에 sha256 사용
    + 암호화 키와 HMAC 키는 별개이며 모두 소스코드에 박혀져 있음
      + 해당 소스 코드는 접근 제한이 걸려 있음
## Lobby
* (본래 로비 서버의 담당자가 따로 있었으나 출시를 위한 scale out이 가능한 구조로의 개선이 지상과제가 되면서 작성자로 담당자가 변경된 후 관련 작업한 내용을 정리)
### Summary (features)
#### Login
* 유저 로그인 및 유저 세션 관리
#### Nickname
* 유저 닉네임 관리 및 관련 요청 처리
#### Character
* 월드 진입하여 사용할 유저의 캐릭터 관리 및 관련 요청(생성/삭제 등) 처리
### Login
* 클라이언트가 로비 서버와 연결 후 인증 서버로부터 획득한 내부 토큰을 제출
* 로비 서버는 전달받은 내부 토큰을 token 라이브러리를 사용하여 해석 후 account id를 획득
* account id를 key로 하여 세션 db(redis)에서 해당 유저의 세션을 조회
* 유저 세션 조회 후 unowned 세션이면 해당 클라이언트가 유저 세션을 소유하도록 처리
* owned 세션이면 로비 서버의 중복 로그인 정책에 따라 로그인을 거절하거나 강제로 탈취(hijack)하도록 처리
  + 개발 환경에서는 로그인을 시도한 클라이언트가 중복 로그인 결과를 전달받을 시 다음 로그인 수단을 사용하여 다시 로그인 시도
    + external IdP -> internal authentication 순서로 로그인 시도
    + 기존 접속 중인 클라이언트는 연결(세션)은 유지
    + 결과적으로 하나의 호스트에서 멀티 클라이언트 및 멀티 로그인 환경 지원 (클라이언트 각각 별도 계정 사용)
  + 프로덕션 환경에서는 세션 탈취 처리를 하며 기존 클라이언트와 연결을 유지하고 있는 서버에게 해당 클라이언트와의 연결을 종료시키도록 통지하며 로그인을 시도한 클라이언트가 로그인이 되면서 세션 소유(탈취)
    + 결과적으로 하나의 호스트에서 단일 클라이언트 및 단일 로그인 환경 지원 (하나의 호스트에서 1개 클라이언트, 1개 계정만 로그인 가능)
* 세션이 아예 없으면 새로운 세션 생성 후 해당 클라이언트에게 유저 세션 할당
### User session
* 유저 세션 구성
  + user session id
    + 유저가 로그인한 후에 세션을 생성/획득하면 클라이언트에게 session id를 전달하며, 클라이언트가 닉네임과 캐릭터 정보를 로비 서버에게 요청할 때 누구인지 식별 가능하도록 session id를 같이 전달
    + uuid 형태이며 세션 db(redis)에서 key로 사용
  + account id
    + 유저 계정을 게임 내에서 식별하기 위한 식별자
  + nickname
    + 유저가 사용 중인 닉네임
    + 게임 db에 저장되는게 원본이며 세션에서 캐싱
  + last disconnected time
    + 클라이언트가 모든 서버(로비 서버, 게임 서버)와의 연결을 맺지 않았을 때 세팅
    + 이 필드로 세션의 owned/unowned 여부를 판단
  + ttl
    + 세션 유효 시간이며 db(redis)에서 자동으로 관리
    + 세션을 실제로 유지하고 있는 서버가 클라이언트와의 연결이 유지되고 있으면 주기적으로 세션 연장을 수행 (redis에 ttl 연장 요청 수행)
  + redirect key
    + 세션을 찾을 때 세션 db에서 key scan을 하지 않고도 다른 여러 키로 찾기 위한 보조 키
    + nickname, account id가 redirect key로 존재
    + redirect key는 string(key-value) 형태이며 value에 session id가 저장됨
* 로비 서버 <-> 세션 db 연동
  + hiredis + redis++ + libuv (비동기 지원을 위한 라이브러리) 사용
  + memory database라는 별도 라이브러리(dll)로 redis++을 wrapping 했으며 로비는 wrapping한 인터페이스를 사용하여 세션 db에 접근
  + 레디스 클러스터 구성을 위한 hash slot 및 transaction 적용 등의 추가 작업이 필요한 상태
* token vs session
  + 토큰과 세션은 대조되는 개념으로 설명되기도 함
    + 토큰은 서버를 더욱 stateless하게 만드는데 도움이 되며 (클라에서 토큰을 저장하니깐 서버 확장성에 용이) 세션 저장을 위한 session storage 등의 의존성이 줄어들게 함
    + 토큰은 내부 시스템(의 db같은 민감한 저장소 등)에 접근하지 못하는 다른/협력 시스템에 저장된 내부 시스템 관련 리소스에 클라이언트가 접근할 때 권한 부여하기 용이함
      + 대표적인 예시로 외부 플랫폼(스팀) 인증이 있음
      + 일반 게임 서비스가 스팀 내부의 계정 db에 직접적으로 접근하는 대신 토큰을 발급하고 토큰을 해석하여 대신 인증(식별자를 제공)해주는 서비스를 제공
    + 세션은 서버에서 세션을 관리하기 때문에 별도 세션 저장소가 필요하며 확장성이 어렵지만 내부적으로 접근성이 좋고 보안적으로 더욱 우수함
  + LLL은 토큰과 세션을 둘다 사용하는 하이브리드 모드를 채용
    + LLL 시스템 외에 다른/협력 시스템에서 게임 관련 데이터를 접근할 경우를 대비하여 자체 토큰을 발급
    + 그 외 외부에 계정 발급을 id/pw 대신 토큰으로 하여 보안성 향상
    + LLL 시스템 내부는 유저의 휘발성 데이터에 대한 접근성과 안전성을 위하여 세션 사용
### Inter-server communication (near-real-time)
* message queue 도입 히스토리
  + 게임 서버는 보통 실시간 통신이 필요한 경우가 많고 stateful한 서버들이라서 이들은 server-to-server로 직접 연결하여 메시지를 송수신하는 방법이 적합
  + 하지만 그리 급하지 않은 일을 처리하는 서버와의 통신이면 near-real-time 수준의 동기화면 충분
  + 또한 동기화 처리가 필요한 서버가 stateless한 경우까지라면 message queue를 사용하여 서버간 동기화 처리가 가능
    + 어차피 여러 stateless한 서버들 중에 하나의 서버가 처리하면 되기 때문
  + 이러한 상황에서 근실시간 통신 수단으로 message queue를 사용
    + 유저 연결 조차 없는 온전히 stateless한 서버(팀 서버 등의 미들 서버)들은 consumer group에 추가되어 메시지를 분산 처리할 수 있음
    + 최소한 유저 연결 정도를 들고 있거나 그 이상의 stateful한 서버들은 각각 전용 큐를 할당 받고 자기에게 할당된 메시지만 수신하여 처리
    + 서버간 메시지 전달 수단으로 message queue를 사용하기 때문에 연결 관계 복잡도를 낮추는 장점이 있음
* message queue 상세
  + message queue는 kafka를 사용하며 kafka와 연동하기 위한 라이브러리로 librdkafka(c++)를 사용
  + librdkafka를 다시 한번 wrapping한 message queue client라는 라이브러리(dll)를 로비 서버와 게임 서버가 사용하여 메시지 송수신
  + 로비 서버와 게임 서버는 stateful한 서버이므로 전용 큐를 할당 받아서 사용
  + 그 외 미들 서버들은 stateless한 서버들이기 때문에 consumer group에 등록하여 하나의 message queue에 등록하고 유저 이벤트 등을 메시지로 전달 받으면 여러 서버 중 하나가 메시지를 처리
* message 종류
  + 유저 중복 로그인으로 인한 kick 처리
    + 로비 서버 <-> 로비 서버 또는 로비 서버 -> 게임 서버
  + 클라이언트의 로비 서버 복귀 또는 월드(게임 서버) 입장
    + 로비 서버 -> 게임 서버 또는 게임 서버 -> 로비 서버
    + 클라이언트의 로비 서버 복귀와 월드 입장을 왜 메시지를 사용하여 처리했는지에 대한 이유는?
      + 클라이언트의 서버간 이동을 서버간 메시지 전달을 통해 확실히 하고자 함
      + 만약 클라이언트가 서버 이동을 알리고 연결 종료가 되었다면 그게 진짜 서버 이동에 성공해서인지 이동하겠다고 하고 예기치 않게 연결이 종료된 것인지 구분할 수 없음
      + 또한 클라이언트의 서버 이동에 따른 각 서버 별 후속 처리를 위한 부분도 있음
  + 팀 관련 이벤트
    + 게임 서버 <-> 팀 서버
### Scale out
## World
# Development Environment
# Statistics System
# Log System
# Monitoring System
# CI
# CD
## K8s
## NCKUBE
# GStar
## Ansible
