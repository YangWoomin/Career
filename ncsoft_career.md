# Server Development
## Overview
![server2](https://github.com/user-attachments/assets/9d7fd7c4-c20b-4a51-9c4c-1d51fc2b3576)
## Auth
### Summary
* 인증 서버는 유저가 클라이언트로 게임 플레이를 하기 전에 유저가 누구인지 식별
* 클라이언트가 로비 서버 및 게임 서버 내에 있는 유저의 자원을 사용하기 앞서 자원의 소유자를 식별하기 위한 수단으로 계정 식별자(account id)를 포함한 내부 토큰 제공

<details>
<summary>Details</summary>

> ### Authentication
> #### External IdP
> * (inhouse)런처나 클라이언트가 각 플랫폼 별 로그인 방식에 따라 로그인 후 OAuth 2.0 토큰 획득
>   + 런처가 토큰을 획득한 경우 클라이언트를 실행할 때 토큰을 클라이언트 매개변수로 전달하거나, 토큰 파일에 저장해 놓으면 클라이언트에서 로드
>   + (런처나 클라이언트 쪽 작업은 각각 별도 작업자가 작업함)
> * 클라이언트가 인증 서버에 연결 후 외부 플랫폼 토큰을 토큰 종류(Idp Kind)와 함께 전달
> * 인증 서버는 전달받은 토큰의 종류에 따라 외부 IdP API에 해당 토큰을 전달하여 유저 식별 정보를 획득
>  + 유저 식별 정보로 nano는 email address, steam은 steam id를 사용
>  + 보통 https 프로토콜로 웹 서버(IdP)에게 요청하며 인증 서버에서는 openssl + libcurl을 사용하여 통신
> * 토큰으로 획득한 유저 식별 정보가 인증 db에서 account id와 바인딩되어 있는지 확인 후 바인딩 정보가 있으면 로드하고 없으면 로비 서버와 연계하여 바인딩
>  + account id는 게임 내 유저를 식별하기 위한 식별자
>  + 인증 서버가 로비 서버에서 새로 생성한 account id를 획득하여 유저 식별 정보랑 바인딩 (인증 db와 게임 db는 분리되어 있고 접근 권한이 분리되어 있음)
>  + 인증 db에서 account id와 유저 식별 정보의 바인딩 형태는 platform account id(PK) <-> account id(FK) 테이블(1번 테이블), 유저 식별 정보(PK) <-> platform account id(FK) 테이블(2번 테이블)들로 구성
>    + 1번 테이블은 global하게 1개 있고 2번 테이블은 외부 플랫폼마다 1개씩 존재 (플랫폼마다 식별자 타입이 다를 수 있으므로)
> * account id와 기타 정보(토큰 발행 시점, 유효 기간 등)를 같이 묶어서 토큰화 및 암호화하여 내부 토큰을 클라이언트에게 전달
> * 클라이언트는 로비 서버 접속을 위해 인증 서버에 연결되어 있는 로비 서버의 접속 정보를 요청하여 획득
> * 클라이언트는 로비 서버 연결 후 어떤 유저인지 확인 가능하도록 로비 서버에게 내부 토큰을 제출
>   + 이후 진행은 "Lobby" 참고 
> #### Internal authentication
> * 계정 관리 툴(account manager)에서 내부 계정 생성
>  + 계정 관리 툴은 웹 브라우저로 접근 가능하며 계정 관리 서버가 서비스 제공
>    + (계정 관리 서버는 다른 팀원 분이 제작 및 관리)
>    + 계정 관리 서버는 사내 계정(nano)으로 로그인(OAuth 2.0)이 가능하고 유저는 자신만의 내부 계정을 관리할 수 있음
>    + 계정 관리 서버에서 인증 db에 직접 접근하여 내부 계정 테이블에 id/pw + 사내 계정 id(owner) 형식으로 저장
> * 생성된 내부 계정 발급과 동시에 계정을 토큰화하여 토큰(내부 계정 토큰)도 같이 발급 가능
>  + 토큰화는 계정 관리 서버가 token 라이브러리를 로드하기 때문에 토큰 생성이 가능함
>  + id/pw를 토큰 안에 넣어서 encrypt 수행
> * 발급한 내부 계정 토큰(또는 내부 계정)은 클라이언트가 로드할 수 있는 특정 경로에 파일(내부 토큰 파일)로 저장하고 클라이언트는 실행 시 이 파일로부터 토큰을 획득
> * 클라이언트가 내부 계정 토큰임을 나타내는 토큰 종류와 함께 내부 계정 토큰을 인증 서버에 연결 후 전달
> * 인증 서버는 내부 계정 토큰을 해석(decrypt)하여 토큰 안에 있는 id/pw를 추출하고 인증 db에서 비교하여 인증 과정 수행
>  + 내부 계정(id/pw)을 사용할 경우 클라 <-> 인증 서버의 인증 요청 패킷이 다름
>  + 인증 성공 시 id/pw에 바인딩된 account id를 획득
> * account id와 기타 정보(토큰 발행 시점, 유효 기간 등)를 같이 묶어서 토큰화 및 암호화하여 내부 토큰을 클라이언트에게 전달
> * 클라이언트는 로비 서버 접속을 위해 인증 서버에 연결되어 있는 로비 서버 접속 정보를 요청하여 획득
> * 클라이언트는 로비 서버 연결 후 어떤 유저인지 확인 가능하도록 로비 서버에게 내부 토큰을 제출
>   + 이후 진행은 "Lobby" 참고 
> ### Token
> * token이라는 별도 라이브러리(dll) 형태로 토큰 생성(암호화) 및 해석(복호화) 기능을 제공하며 인증 서버와 로비 서버에서 로드하여 사용
> * 토큰 자체는 JWT(Json Web Token) 형태이며 암호화 알고리즘은 HS256을 사용
> * JWT 생성 및 암호화 (HS256)
>  + JWT header와 body 생성
>  + token payload : Base64(header) + Base64(body)
>  + token signature : Bse64(hmac_sha256(token payload))
>  + plain text token : token payload + token signature
>  + encrypted token : aes_256_ecb(plain text token)
>  + (as is) readable token : HexToString(encrypted token)
>    + (to do) Base64 encrypted token : Base64(encrypted token)
> * 토큰 암호화/복호화를 위해 openssl 3 라이브러리 사용
>  + 암호화 알고리즘으로 aes 256 ecb 사용
>    + cbc(cipher block chaining)를 사용하지 않고 ecb(electronic code book)를 사용한 이유는 cbc 특성상 이전에 암호화된 블럭을 IV로 사용하기 때문에 토큰간 의존성(순서)이 생김
>    + 토큰간에 의존성이 생기면 안되기 때문에 ecb 사용
>  + HMAC에 sha256 사용
>    + 암호화 키와 HMAC 키는 별개이며 모두 소스코드에 박혀져 있음
>      + 해당 소스 코드는 접근 제한이 걸려 있음

</details>

## Lobby
* (원래 로비 서버의 담당자가 따로 있었으나 출시를 위한 scale out이 가능한 구조로의 개선이 지상과제가 되면서 작성자로 담당자가 변경된 후 이미 있는 기능을 보완하거나 추가로 작업한 내용들을 정리)
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
* 클라이언트는 특정 로비 서버와 연결(connection)을 맺고 있지만 어느 로비 서버(instance)로든 동일한 서비스를 제공받을 수 있기 때문에 로비 서버가 완전하게 stateless 하거나 stateful 하지 않음
* 클라이언트와 연결을 맺은 서버는 지속적인 요청/응답을 주고 받는 면에서 stateful 하지만, 다른 어느 서버든 그 역할을 동일하게 수행할 수 있고 실제 데이터가 저장된 데이터베이스에 클라이언트의 요청 데이터를 입출력하는 대리자로서 역할을 수행하기 때문에 stateless 하다고 볼 수 있음
* 즉, persistent layer는 memory database(redis) 또는 rdbms(MySQL)이기 때문에 연결을 유지하는 것만 제외하면 stateless함
* 위와 같은 특징 때문에 로비 서버 앞단에 로드 밸런서 등을 위치시켜서 클라이언트의 연결을 여러 로비 서버에 분산시켜 처리할 수 있음
## World
* (원래 월드 서버의 담당자가 따로 있었으나 출시를 위한 scale out이 가능한 구조로의 개선이 지상과제가 되면서 작성자로 담당자가 변경된 후 이미 있는 기능을 보완하거나 추가로 작업한 내용들을 정리)
### Summary (features)
* 게임 서버들이 월드 서버에 등록 요청을 하며 월드 서버는 게임 서버 매니저 역할을 담당
* 게임 서버들이 모여서 하나의 월드를 구성
* LLL은 단일 월드 체제
#### User handover
* 로비에서 인증된 클라이언트에게 월드 진입을 위해 접속 가능한 게임 서버의 접속 정보를 로비에게 전달
* 클라이언트는 게임 서버 접속 정보를 사용하여 게임 서버에 접속
#### Game server management
* 게임 서버가 아래 필드들과 함께 월드 서버에 등록 요청
  + server id
    + 게임 서버 식별자
  + cluster
    + 월드(서버군)를 구분하기 위한 식별자
  + address (ip/port)
    + 클라이언트가 게임 서버에 접속하기 위한 접속 정보
  + map name
    + 클라이언트가 필요로 하는, 접속할 게임 서버의 맵 이름
  + status
    + 게임 서버의 상태
    + loaded / ready (enterable) / closing
    + 게임 서버 status가 업데이트되면 월드 서버에 갱신 요청
* 게임 서버가 등록/제거되거나 상태(status)가 갱신되면 연결된 모든 로비 서버들에게 broadcasting
  + 현재는 월드 서버에 등록된 게임 서버 정보를 로비 서버들에게 단순히 전달만 하는 수준
  + (to do) 클라이언트가 월드 진입 요청 시 로비 서버는 월드 서버로부터 게임 서버 접속 정보를 획득하여 클라이언트에게 전달하는 구조로 변경 필요
  + (to do) 월드 서버의 single point of failure를 대비하여 게임 서버 등록 정보를 메모리 데이터베이스(redis)에 저장하도록 개선 필요
* cluster 필드는 월드를 구분하기 위한 group key
  + LLL은 단일 월드 체제이고 멀티 월드와 서버 채널링에 대한 고려가 없는 상태
  + 월드의 논리적 그룹화를 통한 유저풀 분리를 목적으로 멀티 월드를 자체적으로 구현
  + 월드는 유저가 흔히 느끼는 서버(도시섭, 시골섭 등) 개념에 해당되며 게임 내에서 플레이어블한 공간을 논리적으로 분리한 형태
  + 개발 환경에서는 게임 서버를 제외한 나머지 서버들의 공용화에 따라 작업자의 서버들 별로 고유의 월드를 보장하기 위해 멀티 월드 지원
    + 차후 기획적으로 멀티 월드 지원을 요구할 수 있어서 미리 준비한 부분도 있음
  + 채널링 도입은 고려되지 않은 상태 (서버팀 내부적으로 언급만 된 상태)
    + 채널링은 월드의 맵을 지리(region)적으로 나눈 뒤 해당 지역을 여러 차원으로 instantiating한 개념이 파티션(partition)
    + 어느 특정 지역에 대해 여러 파티션으로 구성하여 유저풀을 나누어 처리하도록 분산 처리가 고려된 시스템
## Game
* (게임 서버 담당자는 따로 있으며 아래 Summary에 정리된 내용은 서버 기능과 역할을 설명하기 위해 작성)
* (위에서 언급한 로비 서버의 scale out 가능한 구조를 게임 서버에도 반영하기 위해 유저 세션 작업을 위한 세션 db 연동과 message queue 연동 작업은 작성자가 작업함)
* (그 외 통계 시스템과 연동하기 위한 Statistics subsystem도 작성자가 작업함)
### Summary (features)
* 게임 서버는 lllgame이라는 컨텐츠(원래는 별도 라이브러리)를 돌려 유저가 플레이할 월드를 서비스하는 서버
  + lllgame은 서버와 클라이언트의 공통 코드 또는 게임 관련한 코드(컨텐츠 로직)가 들어간 platform independent pure c++ 작업 공간
* 앞서 월드 서버에서 설명한 파티션들을 실행하는 주체가 되며 파티션에서 발생하는 모든 일을 처리
* 지역적으로 서로 인접한 파티션은 겹쳐지며 겹친 부분은 경계선 영역이 되고 이 영역은 파티션간 게임 오브젝트의 원본과 복제 개념으로 동기화 수행
  + 즉, 경계선 영역을 포함하고 있는 파티션들에서 게임 오브젝트들은 각각 원본과 복제본으로 나타나게 되며 파티션 간에 밀접한 통신으로 동기화 수행
  + 현재 게임 서버 내부의 파티션 간에 동기화가 되고 있는 상태이며, 그것도 최대 2개, 1:1로만 가능한 상태
    + inter-server를 통한 파티션들 사이의 동기화는 구현해야 하는 상황
      + 이 작업이 되어야 게임 서버의 scale out(보다는 분산 처리가 더 정확한 표현)이 가능
### Statistics subsystem
* 게임 서버 상태와 월드 및 게임 오브젝트 관련 상태를 모니터링하기 위해 통계 지표를 처리하는 통계 시스템이 있으며 게임 서버에서 통계 시스템과 연동하기 위한 부분이 통계 서브시스템
* 통계 서브시스템은 게임 서버에서 발생하는 다양한 지표를 모아서 통계 시스템의 통계 db에 적재하는 작업을 수행
* 통계 지표 종류
  + CPU & Memory
    + 게임 서버 프로세스의 CPU와 메모리 사용량
    + 메모리는 virtual(가상 메모리 상에서의 사용량)과 physical(물리 메모리(RAM) 상에서의 사용량)로 구성
  + Game object count
    + 파티션(viewer에서는 shard로 표시) 단위 게임 오브젝트 개수
  + Game object count by type
    + 파티션 단위 게임 오브젝트들의 타입별 개수
  + Game object location
    + 게임 서버에 있는 모든 게임 오브젝트의 위치 및 타입
    + world contents visualize(map viewer)에서 사용
  + FPS
    + 파티션 단위 초당 파티션 tick 호출 횟수
  + User count
    + 게임 서버에 인증된 유저의 수
  + Connection count
    + 게임 서버에 연결된 클라이언트 수
  + Game server status
    + 게임 서버의 status
  + Send/Recv packet size
    + 게임 서버에 연결된 모든 클라이언트와의 초당 송수신 패킷 크기
  + Game play task count
    + 게임 서버의 scheduler가 파티션 단위 초당 처리하는 task 개수
* 지표 처리 방식
  + metrics table
    + table
      + 같은 성격의 지표를 모아놓은 테이블
      + rdb의 테이블과 대응되는 요소
    + tag
      + 지표의 부가 정보를 담고 있는 필드
    + field
      + 실제 측정하고자 하는 필드
  + 기본 처리 방식
    + 지표는 1개 레코드(n tags + m fields) 단위로 수집
    + 외부로부터 큐에 담겨 전달된 지표는 통계 테이블에 time index 단위로 batching되어 저장
      + 테이블에 설정된 연산 종류에 따라 적절한 연산을 취하여 batching
      + 예를 들어 초당 송신 패킷 크기를 측정할 경우 연산 종류는 sum이며 2024-07-17 10:05:00 ~ 10:05:01 time index(epoch로 표현됨)에 발생하는 모든 지표는 이 time index안에 합하여 저장됨
        + time index 1721179700, send packet size(sum) : 1024
        + time index 1721179701, send packet size(sum) : 64
        + ...
  + 연산 종류
    + raw
      + 어떠한 연산도 적용하지 않고 날 것 그대로 time index와 함께 저장
    + sum
      + 설정된 주기 (e.g. 1초) 별로 발생한 모든 지표를 time index 단위로 batching한 후 각각 합하여 출력
      + batching한 결과, 아무 데이터가 없으면 0을 출력하도록 설정 가능 (항상 양의 실수가 입력된다는 전제, default는 값 자체가 없음)
    + update
      + 설정된 주기 별로 발생한 모든 지표를 time index 단위로 batching한 후 각각 마지막으로 전달된 지표를 출력
      + batching한 결과, 아무 데이터가 없으면 0을 출력하도록 설정 가능 (항상 양의 실수가 입력된다는 전제, defualt는 값 자체가 없음)
    + accumulative sum
      + 이전 time index에 batching하여 연산(합)한 결과를 다음 time index에 이어서 연산
      + batching한 결과, 아무 데이터가 없으면 이전 time index 값을 출력
    + accumulative update
      + 이전 time index에 batching하여 연산(갱신)한 결과를 다음 time index에 이어서 연산
      + batching한 결과, 아무 데이터가 없으면 이전 time index 값을 출력
    + max
      + 설정된 주기 별로 발생한 모든 지표를 time index 단위로 batching하여 그 중 최댓값 출력
      + batching한 결과, 아무 데이터가 없으면 0을 출력하도록 설정 가능 (항상 양의 실수가 입력된다는 전제, default는 값 자체가 없음)
    + 평균 연산 등 그 외 복잡한 연산도 추가하려고 했으나 통계 db에서 쉽게 계산할 수 있도록 쿼리가 지원되어서 추가하지 않음
    + 통계 서브시스템에서는 최대한 지표를 압축(batching)하는데 집중
  * scope cycle
    + 주로 특정 함수 등의 수행 시간과 호출 횟수를 측정하기 위한 통계 서브시스템 인터페이스
      + sum 연산을 적용한 counter와 duration 필드로 수집
      + 초당 함수 호출 횟수와 초당 함수 처리 시간을 계산
    + scope cycle 전용 클래스가 있으며 생성자에서 생성 시점을 기록하고 소멸자에서 생성 시점으로부터 걸린 시간을 계산하여 지표 수집기에 전달
      + counter는 소멸자에서 지표 수집기에 1 전달 (1 증가)
      + 측정할 함수에서는 scope cycle 클래스를 RAII 패턴으로 사용하며 측정할 block에 매크로로 클래스를 선언하여 측정
* 실제 지표 측정 현황
  + ![game_server_statistics_1](https://github.com/user-attachments/assets/3353822f-5a63-499c-8d12-cbf04df42b25)
  + ![game_server_statistics_2](https://github.com/user-attachments/assets/0ec9f7ca-08a7-41c0-8c1e-c399e79d827e)
# Statistics System
## Summary (features)
* 게임 서버 자체 상태와 월드 및 게임 오브젝트 관련 상태를 모니터링하기 위해 통계 지표를 처리하는 시스템
### Statistics db (InfluxDB)
* NoSQL 계열이며 데이터를 시간에 따라 효율적으로 저장/조회할 수 있는 시계열 데이터베이스
* 스트림(브랜치) 또는 production 환경마다 전용 bucket(rdb의 database와 대응되는 개념)으로 데이터를 나누어 관리
* InfluxQL이라는 자체 쿼리 언어로 효율적이고 다양한 방식으로 지표 조회가 가능
  + ![influx_ql_example](https://github.com/user-attachments/assets/0c7049d2-8b18-4735-ac7a-4e910816055b)
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
* ![monitoring](https://github.com/user-attachments/assets/edc57fa5-ee77-484a-8808-e9090d75416a)
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
