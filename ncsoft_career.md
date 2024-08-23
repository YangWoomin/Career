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
    + 계정 관리 서버는 다른 팀원 분이 관리
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
### Summary (features)
#### Login
#### Nickname
#### Character
### Login
### User session
### Inter-server communication (near-real-time)
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
