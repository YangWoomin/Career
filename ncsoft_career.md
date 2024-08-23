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
