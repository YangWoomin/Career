
## 소개
* 이 문서는 저의 오픈 소스 포트폴리오인 [ZilliaxServer](https://github.com/YangWoomin/ZilliaxServer)에 대한 설명글입니다.
* 모든 내용을 설명드리기 어려우니 **데이터 신뢰성(Data Reliability)** 위주의 내용을 정리하였습니다.
  + 메모리 데이터베이스(레디스 클러스터, 이하 레디스) 및 메시지 큐(카프카 클러스터, 이하 카프카)를 사용하여 백엔드 서버(C++/Golang)간 데이터 송수신 과정에서 **데이터 유실 및 중복 방지** 처리 (논리적 수준 포함)
  + 레디스 클러스터의 여러 노드에 저장된 데이터에 대해 **결과적 일관성(Eventually Consistency)** 방법으로 데이터 일관성을 보장
* 여기서 설명하는 데이터 신뢰성 보장을 위한 시나리오는 클라이언트별로 전송한 메시지 개수를 저장하고, 전송된 메시지별로 개수를 일관적으로 저장하는 흐름으로 진행됩니다. (레디스에 저장)
* 직접 프로젝트를 빌드하고 실행이 가능하나 번거로우실테니 이 문서를 보시면서 대략적으로 프로젝트 내용을 파악하시면 되겠습니다.
* 음슴체로 적게 된 점 양해 부탁 드립니다.

## 상세 설명

### 개요

<br/>

![zilliax_server_overview](https://github.com/user-attachments/assets/fefcc9de-039b-48bf-920d-352a9d648b3f)

<br/>

### 구성 요소별 프로젝트 모듈 관계
* Client : "network_test" (C/C++)
* Producer Server : "mq_test_producer" (C/C++)
* Cache Server : Redis Cluster, "cache" 모듈에서 접근 (프로듀서 서버가 사용, C/C++)
* Message Queue : Kafka Cluster, "mq" 모듈에서 접근 (프로듀서 서버가 사용, C/C++)
* Client Message Counter : "mq_test_consumer" (Golang)
* Message Aggregator : "mq_test_consumer" (Client Message Counter와 동일 프로젝트, Golang)

### 클라이언트
* 사용자 입력을 1줄씩 받아서 프로듀서 서버에게 전송 (1번 과정)
* mtc(massive test client) 모드로 실행시 로컬의 샘플 파일(소설)을 읽어서 프로듀서 서버에게 전송

### 프로듀서 서버
* 클라이언트로부터 데이터를 전송 받으면 캐시 서버에 클라이언트 식별자(host:port)별로 메시지를 순서대로 저장 (by 네트워크 스레드풀, 비동기, 2번 과정)
  + 클라이언트 별로 메시지에 sn(sequence number) 부여 (논리적 수준의 메시지 순서 보장 및 중복 방지)
  + 레디스 클러스터에 메시지를 분산 저장하기 위해 클라이언트 식별자를 해시슬롯(hash slot)으로 사용
* 별도 스레드들이 캐시 서버로부터 적재된 메시지들을 순차적으로 로드해와서 메시지 큐에 저장 (3,4번 과정)
* 메시지 큐에 적재 완료된 메시지들의 상태를 캐시 서버에 업데이트 (5번 과정)
* 주기적인 메시지 유실(메시지 큐 적재 실패 등) 검사를 수행하여 유실된 메시지 발견시 해당 메시지부터 재전송 (논리적 수준의 메시지 유실 방지)
* 기본적으로 enable.idempotence=true 옵션을 활성화하여 메시지 중복을 방지 (물리적 수준의 메시지 중복 방지)
* acks=all 옵션으로 데이터 분산 저장 (물리적 수준의 메시지 유실 방지)

### 메시지 큐
* 프로듀서 서버로부터 client_message 토픽에 메시지 저장
* 메시지 키로 클라이언트 식별자를 사용하여 클라이언트 메시지 순서를 보장 (논리적/물리적 메시지 순서 보장)
* 토픽들의 ISR(In-Sync Replica)은 3으로 유지하도록 설정하여 데이터 분산 저장 및 가용성 증가 (물리적 수준의 메시지 유실 방지)
* 클라이언트 식별자를 기준으로 순서가 보장되어야 하므로 토픽의 파티션 개수는 서버가 수용할 (예상하는) 클라이언트 수만큼 생성 (100개)

### 클라이언트 메시지 카운터
* 클라이언트별로 메시지 개수를 계산하기 위한 서버
* client_message 토픽으로부터 여러 컨슈머(+프로듀서)를 사용하여 메시지 조회 (6번 과정)
* 캐시 서버에 클라이언트 식별자별 메시지 개수 저장 (7번 과정)
  + 마지막으로 연산한 메시지의 sn을 기억하고 있으며 마지막 sn의 +1에 해당하는 메시지만 처리 (논리적 수준의 메시지 순서 보장 및 중복 방지)
  + 마지막 sn+1이 아닌 메시지들은 버림 (TODO: DLQ에 보관, sn+1보다 큰 메시지들은 프로듀서 서버의 메시지 유실 감지로 재전송됨)
  + 프로듀서 서버와 마찬가지로 클라이언트 식별자를 해시슬롯으로 사용
* 소비한 메시지 오프셋 커밋과 메시지 수집기(aggregator)를 위한 메시지 적재를 트랜잭션으로 처리 (물리적 메시지 유실 방지, 8번 과정)
  + message_aggregation 토픽에 적재
* 메시지 오프셋 커밋과 메시지 적재는 한 번에 모아서 하나의 트랜잭션으로 처리 (성능 향상)

### 메시지 수집기
* 메시지별로 개수를 계산하기 위한 서버
* message_aggregation 토픽으로부터 여러 컨슈머를 사용하여 메시지 조회 (9번 과정)
* 캐시 서버에 메시지별 메시지 개수를 저장 (10번 과정)
  + 레디스 클러스터에 메시지를 분산 저장하기 위해 메시지 자체(TODO: 해시)를 해시슬롯으로 사용
  + 메시지별 클라이언트 식별자와 메시지 sn으로 메시지 중복 여부를 확인 (논리적 수준의 메시지 중복 방지)
* 소비한 메시지 오프셋 커밋을 수행 (11번 과정)

## 데이터 신뢰성 테스트

### 기본 테스트

#### 환경
<details>
<summary>details</summary>

* PC
  + Windows 10 + WSL2 (Ubuntu 24.04)
  + Intel(R) Core(TM) i7-8565U CPU @ 1.80GHz
  + 16GB RAM
  + 512G SSD
* 클라이언트
  + 클라이언트(소켓) 수 : 10
* 프로듀서 서버
  + 1대
  + acks=all
  + enable.idempotence=true
  + 그 외 나머지 설정들은 모두 기본값
* 캐시 서버 (레디스 클러스터)
  + 마스터 3대 + 복제 3대
  + Redis Insight 사용
  + 모든 설정들은 기본값
* 메시지 큐 (카프카 클러스터)
  + KRaft 모드
  + 컨트롤러 3대 + 브로커 3대
  + Conduktor 사용
  + client_message 토픽 - 파티션 10개, replication factor 3, min isr 1 (옵션들은 기본값)
  + message_aggregation 토픽 - 파티션 10개, replication factor 3, min isr 1 (옵션들은 기본값)
  + 그 외 나머지 설정들은 모두 기본값
* 클라이언트 메시지 카운터 및 메시지 수집기
  + auto.offset.reset=earliest
  + enable.auto.commit=false
  + isolation.level=read_committed
  + enable.idempotence=true
  + transactional.id=[임의 문자열]
  + group.id=[임의의 문자열] (클라이언트 메시지 카운터와 메시지 수집기는 서로 다른 컨슈머 그룹)
  + 클라이언트 메시지 카운터만 프로듀서에서 트랜잭션 사용
    + 최대 100개 메시지씩 1개 트랜잭션으로 커밋하되 여유로운 상태(소비할 메시지가 없을 때)라면 바로바로 커밋
  + 프로듀서 및 컨슈머는 각각 3개씩
  + 그 외 나머지 설정들은 모두 기본값

</details>

#### 테스트 데이터
* (4422 행, 총 283052 바이트 크기) x 10 (클라이언트 수)

#### 테스트 결과 - 클라이언트 메시지 송신 결과

![image](https://github.com/user-attachments/assets/a5b51472-d057-4ab7-84ee-81bf8be3e687)

#### 테스트 결과 - 프로듀서 서버 메시지 수신 및 적재 결과

![image](https://github.com/user-attachments/assets/82265d6b-810e-43d6-afb5-789f9141be6e)
<br/>
"total processed message count : 44220" : 프로듀서 서버가 모든 클라이언트로부터 수신한 메시지 개수
<br/>
"msg manager finalized, total sent msg count : 44220" : 프로듀서 서버가 메시지 큐로 전송한 메시지 개수
<br/>
"total processed message count: 44220, size : 2830520" 이 부분이 클라이언트에서 보낸 300740 * 10과 맞지 않는 이유는 300740이 메시지 크기(4바이트)까지 포함하기 때문, (300740 - (4422 * 4)) * 10 = 2830520

#### 테스트 결과 - client_message 토픽

![image](https://github.com/user-attachments/assets/8b24c8c8-cbd5-4fd1-bdf4-bbbf9caa917b)

#### 테스트 결과 - message_aggregation 토픽

![image](https://github.com/user-attachments/assets/e862d117-f388-41f4-8489-0b31b0a507fa)
<br/>
message_aggregation 토픽의 레코드 개수는 전송한 메시지 개수인 44220보다 많은 44780인 이유는 트랜잭션으로 적재시 트랜잭션 마커(marker)도 포함되기 때문
<br/>
트랜잭션 마커는 다수의 파티션에 대해 트랜잭션이 커밋되었거나 중단되었다는 것을 표시하기 위해 마커 메시지를 사용
<br/>
참고 : https://stackoverflow.com/questions/79001842/count-mismatch-in-akhq-ui-0-24-0

#### 테스트 결과 - 데이터 일관성 확인 (레디스에 최종적으로 저장된 데이터를 전용 툴(mq_test_verifier)로 조회)

![image](https://github.com/user-attachments/assets/a828d276-c061-47de-bcec-91ad610cb574)
<br/>
각 클라이언트별로 4422개의 메시지 개수 확인 가능

![image](https://github.com/user-attachments/assets/f1f75c39-92a9-4f49-9fab-5c5b3e6f004d)
<br/>
메시지가 많아서 위에 짤렸지만 각 메시지 개수가 모두 10개로 올바른 결과 출력
<br/>
원본에 있는 모든 레코드(행)는 다른 레코드들과 중복되지 않도록 미리 작업해둠

## 메시지 유실 테스트
### 프로듀서 서버
#### 메시지 큐로부터 메시지 저장에 실패한 경우

<details>
<summary>펼쳐보기</summary>

> #### 1. 초기 메시지 2개만 적재 완료된 상황
> 
> ![before_test_in_mq](https://github.com/user-attachments/assets/cf9951d8-cf00-4c8d-b417-07de7c803667)
> 
> ![before_test_in_redis](https://github.com/user-attachments/assets/3426eddd-d650-43aa-bd3d-a2d7892d3039)
> 
> #### 2. 프로듀서 서버가 소비할 테스트 메시지를 추가 생성
> 
> ![preparing_test_in_redis](https://github.com/user-attachments/assets/3c9bb67a-f8b3-4a74-ac79-28b37a927215)
> <br/>
> 세번째 메시지 "cccccc...cc" 부터 일곱번째 메시지 "ggggg...gg" 5개 추가
> 
> #### 3. 테스트를 위해 세번째 메시지를 저장에 실패 처리한 것처럼 콜백에서 조작 후 프로듀서 서버의 재전송 결과
> 
> ![not_persisted_message_failure_result_in_producer_server](https://github.com/user-attachments/assets/cb9584b0-44e9-4bdc-929a-f47038d7435a)
> <br/>
> 세번째 메시지의 키를 "test_client"로 변경하고 이를 저장에 실패한 것처럼 유도
> <br/>
> 프로듀서는 적재에 실패한 세번째 메시지부터 다시 재전송
> 
> #### 4. 클라이언트 메시지 카운터에서 "test_client" 메시지에 대한 처리
> 
> ![not_persisted_message_failure_result_in_client_message_counter](https://github.com/user-attachments/assets/a484f35f-bde7-4794-ab07-65dff917229f)
> <br/>
> "test_client" 키에 대한 이전 데이터가 없으므로 유효하지 않은 것으로 처리
> <br/>
> 본래의 클라이언트 키에 대해 세번째 메시지가 도착하지 않았으므로 네번째 메시지 이후부터 유효하지 않은 것으로 처리
> 
> #### 5. "client_message" 토픽의 메시지 적재 결과
> 
> ![not_persisted_message_failure_result_in_client_message_mq](https://github.com/user-attachments/assets/5c444dd9-6801-43d8-b594-8853aa212e50)
> 
> #### 6. "message_aggregation" 토픽의 메시지 적재 결과
> 
> ![not_persisted_message_failure_result_in_message_aggregation_mq](https://github.com/user-attachments/assets/833e5f57-48a0-4557-ab6b-7c60426be1f7)
> <br/>
> "client_message" 토픽과 다르게 "message_aggregation" 토픽은 중복 없이 데이터 저장
> 
> #### 7. 최종적으로 클라이언트별로 저장된 메시지 개수 확인
> 
> ![not_persisted_message_failure_result_in_message_verifier_for_client](https://github.com/user-attachments/assets/dfbea426-bca5-490f-9f90-26d250d21ce2)
> 
> #### 8. 최종적으로 메시지별로 저장된 메시지 개수 확인
> 
> ![not_persisted_message_failure_result_in_message_verifier_for_message](https://github.com/user-attachments/assets/dc436395-fba7-4f05-8123-3bdb6838c74d)
> 
</details>


#### 메시지 큐로부터 메시지 저장에 대한 응답이 없어서 유실되는 경우

<details>
<summary>펼쳐보기</summary>

> #### 1. 앞서 "메시지 큐로부터 메시지 저장에 실패한 경우" 테스트 케이스와 전반적인 흐름은 동일하나 메시지 적재 실패가 아닌 유실로 인해 적재 완료 콜백이 오지 않는 경우를 시뮬레이션
>
> #### 2. 테스트를 위해 세번째 메시지를 저장 도중 유실된 것처럼 콜백에서 조작 후 프로듀서 서버의 재전송 결과
> 
> ![missed_message_failure_result_in_redis](https://github.com/user-attachments/assets/ad21a801-c8ef-47ba-a921-ca7bc5fd5f83)
> <br/>
> 레디스에서 유실된 메시지의 sn을 기다리는 다른 sn들을 "stored_msg_sn_tmp_list" sorted set에 임시 보관
> <br/>
> 2분(설정된 시간) 동안 유실된 sn에 대한 결과를 받지 못하면 유실된 sn부터 재전송
> 
> ![missed_message_failure_result_in_producer_server](https://github.com/user-attachments/assets/cb68f297-d4c1-40fe-bd57-fc5571206d8b)
> <br/>
> 프로듀서 서버가 유실된 sn부터 메시지들을 재전송
> 
> #### 3. "client_message" 토픽의 메시지 적재 결과
> 
> ![missed_message_failure_result_in_client_message_mq](https://github.com/user-attachments/assets/2fb16976-6bb7-4c10-85e1-4eda4f069245)
> 
> #### 4. "message_aggregation" 토픽의 메시지 적재 결과
> 
> ![missed_message_failure_result_in_message_aggregation_mq](https://github.com/user-attachments/assets/3ad6c9d0-9d56-4903-ae32-13398d495c38)
> 
> #### 5. 최종적으로 클라이언트별로 저장된 메시지 개수 확인
> 
> ![missed_message_failure_result_in_message_verifier_for_client](https://github.com/user-attachments/assets/79cb6c96-4ea5-4bcb-b69b-20fcf919173c)
> 
> #### 6. 최종적으로 메시지별로 저장된 메시지 개수 확인
> 
> ![missed_message_failure_result_in_message_verifier_for_message](https://github.com/user-attachments/assets/30cc8abf-9bf8-4304-9b3e-40e02862cd07)
> 
</details>

#### 프로듀서 버퍼가 가득 차서 전송에 실패하는 경우

<details>
<summary>펼쳐보기</summary>

> #### 1. 테스트 클라이언트 수를 제외한 나머지 환경은 기본 테스트와 동일 (테스트 클라이언트 10개 -> 30개)
> 
> PC 사양마다 프로듀서 버퍼가 가득 차게 되는 메시지 용량이 다를 수 있으니 개인 PC 환경에 따라 클라이언트 개수를 조절하여 버퍼가 가득 차는 클라이언트 수를 찾아야 함
> 
> #### 2. 프로듀서 서버에서 프로듀서 버퍼가 가득 차게 되어 발생하는 경고 로그 확인
>
> ![queue_full_failure_result_in_producer_server](https://github.com/user-attachments/assets/def2a8d8-1568-404d-a9fb-67848420f630)
> <br/>
> 메시지 프로듀싱하는 과정에서 -184 에러 발생
> 
> ![queue_full_failure_result_2_in_producer_server](https://github.com/user-attachments/assets/df98b9bc-40da-4ad4-8abb-b3e47997aa10)
> <br/>
> librdkafka에 정의된 -184 에러의 의미
> 
> #### 3. 프로듀서 서버에서 메시지 전송 결과
> 
> ![queue_full_failure_result_3_in_producer_server](https://github.com/user-attachments/assets/5e9b8b55-4b59-42a1-bc05-2d16ff433816)
> <br/>
> 4422 * 30 = 132,660
> 
> #### 4. "client_message" 토픽의 메시지 적재 결과
> 
> ![queue_full_failure_result_in_client_message_mq](https://github.com/user-attachments/assets/8fc48cb9-eefe-4392-a112-4463bff9cf2a)
> 
> #### 5. "message_aggregation" 토픽의 메시지 적재 결과
> 
> ![queue_full_failure_result_in_message_aggregation_mq](https://github.com/user-attachments/assets/fcd68013-56cc-4e18-adec-d9a576aa9214)
> 
> #### 6. 최종적으로 클라이언트별로 저장된 메시지 개수 확인
> 
> ![queue_full_failure_result_in_message_verifier_for_client](https://github.com/user-attachments/assets/e9b60a02-b0dd-4120-b5e8-4cfeee80e503)
> 
> #### 7. 최종적으로 메시지별로 저장된 메시지 개수 확인
> 
> ![queue_full_failure_result_in_message_verifier_for_message](https://github.com/user-attachments/assets/af3b4857-e73e-4dfd-8267-60f61821431f)
> 
</details>
  
### 메시지 큐 (준비중)
#### 우아한 브로커 재시작

<details>
<summary>펼쳐보기</summary>

> #### 1. 테스트 클라이언트 수를 제외한 나머지 환경은 기본 테스트와 동일 (테스트 클라이언트 10개 -> 50개)
>
> ![before_broker_failure_test_in_client_message_mq](https://github.com/user-attachments/assets/b85d1c31-ce9d-4d72-ab80-f7a26f197ff9)
> <br/>
> 각 파티션의 리더/팔로워 레플리카 할당 상태 확인
> <br/>
> 1번 브로커(broker-1)의 ID는 4번으로 파티션 0, 4, 8, 9의 리더 레플리카가 1번 브로커에 할당되어 있음
> 
> #### 2. 프로듀서 서버와 클라이언트 메시지 카운터, 메시지 수집기가 메시지 큐로부터 메시지 처리를 하는 동안 1번 브로커 종료
> 
> ![during_broker_failure_test_in_broker_for_preparing_shutting_down](https://github.com/user-attachments/assets/4bccac34-6b1a-49d4-b76e-3681c50a00f7)
> <br/>
> "SIGTERM" 시그널을 받아서 "STARTED" -> "PENDING_CONTROLLED_SHUTDOWN" 상태로 전환
> 
> ![during_broker_failure_test_in_broker_for_resigning_group_coordinator](https://github.com/user-attachments/assets/5f2e08e0-ecda-43e7-ac74-0cfecc9b5972)
> <br/>
> 그룹 코디네이터 사임중
> 
> ![during_broker_failure_test_in_broker_for_resigning_tx_coordinator](https://github.com/user-attachments/assets/6d1d2fe9-e837-48e0-a9fb-86dc1b749712)
> <br/>
> 트랜잭션 코디네이터 사임중
> 
> ![during_broker_failure_test_in_broker_for_shutting_down](https://github.com/user-attachments/assets/96123e2f-7dbb-4df5-bb1b-a14faca05673)
> <br/>
> "PENDING_CONTROLLED_SHUTDOWN" 상태에서 "SHUTTING_DOWN" 상태로 전환
> 
> ![during_broker_failure_test_in_producer](https://github.com/user-attachments/assets/d5b9db20-acef-4e47-ba88-be4c7a1a1fb1)
> <br/>
> 프로듀서 서버에서 브로커와 일시적 연결 끊김 로그 확인
> 
> ![during_broker_failure_test_in_consumer](https://github.com/user-attachments/assets/68359220-3536-4217-b0be-2d39f63f6513)
> <br/>
> 컨슈머(클라이언트 메시지 카운터, 메시지 수집기)에서도 연결이 단절되었다는 로그 확인
> 
> #### 3. 파티션의 모든 리더 레플리카는 1번 브로커(ID:4)로부터 다른 브로커들로 할당됨
> 
> ![during_broker_failure_test_in_client_message_mq](https://github.com/user-attachments/assets/00129448-d9c2-4837-b656-4339e8594cc7)
> <br/>
> 모든 파티션이 4번(1번 브로커)에만 할당되어 있지 않음
> <br/>
> 이와중에 메시지 적재와 소비는 문제없이 진행됨
> <br/>
> 프로듀서와 컨슈머들은 클러스터와 메타데이터 동기화를 필요할 때 하기 때문에 리더 브로커가 바뀌면(리더 재선출) 해당 브로커로 연결하여 작업을 이어 나감
> <br/>
> REPLICATION FACTOR(ISR)를 3으로 유지하고 있기 때문에 리더 브로커가 종료되더라도 다른 브로커가 리더 브로커의 역할을 매끄럽게 계속 이어나갈 수 있음
> 
> #### 4. 1분 후 1번 브로커 재시작
> 
> ![during_broker_failure_test_in_broker_for_starting](https://github.com/user-attachments/assets/8edb6a78-acfc-496a-a43f-2b15bb52141a)
> <br/>
> "SHUTDOWN" 상태에서 "STARTING" 상태로 전환되면서 브로커 시작
> 
> ![during_broker_failure_test_in_broker_for_recovery](https://github.com/user-attachments/assets/4483719b-ae1c-4371-b98c-d24387d4323d)
> <br/>
> 클러스터 메타데이터를 동기화하면서(catch up) "STARTING" 상태에서 "RECOVERY" 상태로 전환
> 
> ![during_broker_failure_test_in_broker_for_recovering](https://github.com/user-attachments/assets/ed3bf956-c5cb-491e-bdeb-4a64977f34c3)
> <br/>
> 파티션,오프셋,트랜잭션 등의 데이터를 로드하는 중 (recovering)
> 
> ![during_broker_failure_test_in_broker_for_starting_group_and_tx_coordinator](https://github.com/user-attachments/assets/1c845f1f-0a53-4813-a5e9-f61838ac7aca)
> <br/>
> 그룹 코디네이터, 트랜잭션 코디네이터 등을 시작
> 
> ![during_broker_failure_test_in_broker_for_fetching_data_from_other_brokers](https://github.com/user-attachments/assets/8c002ce6-b6ad-4e6f-89d6-385608a588e2)
> <br/>
> 다른 브로커로부터 데이터 동기화를 위해 복제 시작
>
> ![during_broker_failure_test_in_broker_for_broker_server_started](https://github.com/user-attachments/assets/deecb62a-faf8-4993-89c7-f511bdcca883)
> <br/>
> 클러스터에 브로커 참여함으로써 최종적으로 "STARTING" 상태에서 "STARTED" 상태로 전환
>  
> ![during_broker_failure_test_in_broker_for_electing_group_coordinator](https://github.com/user-attachments/assets/7617adfb-64c8-47d7-89f8-34ff5aa325ff)
> <br/>
> 그룹 코디네이터 선출
> 
> ![during_broker_failure_test_in_broker_for_electing_tx_coordinator](https://github.com/user-attachments/assets/a590ddd3-7493-46bf-9a2a-d7685a3dd8e7)
> <br/>
> 트랜잭션 코디네이터도 마찬가지로 선출
> 
> #### 5. "client_message" 토픽의 메시지 적재
> 
> ![broker_failure_test_result_in_client_message_mq](https://github.com/user-attachments/assets/6dd7fd58-2e33-4202-a745-3c8a3199c6f4)
> <br/>
> 파티션 0, 4, 8, 9번에 대한 리더 레플리카가 1번 브로커(ID:4)로 복귀
> 
> #### 6. 최종적으로 클라이언트별로 저장된 메시지 개수 확인
> 
> ![broker_failure_test_result_in_message_verifier_for_client](https://github.com/user-attachments/assets/7e2a8d08-eaf4-4383-9e4c-c64fe7e1ffc2)
> 
> #### 7. 최종적으로 메시지별로 저장된 메시지 개수 확인
>
> ![broker_failure_test_result_in_message_verifier_for_message](https://github.com/user-attachments/assets/55834c28-3bdb-46ab-9fad-1d602751e7c0)
> 
</details>

#### 강제 브로커 재시작

<details>
<summary>펼쳐보기</summary>

> #### 1. 테스트 클라이언트 수를 제외한 나머지 환경은 기본 테스트와 동일 (테스트 클라이언트 10개 -> 50개)
> 
> #### 2. 프로듀서 서버와 클라이언트 메시지 카운터, 메시지 수집기가 메시지 큐로부터 메시지 처리를 하는 동안 !!1번 브로커 강제 종료!!
> 
> ![during_broker_failure_test_2_in_producer_server](https://github.com/user-attachments/assets/dc928b8e-5027-4f7a-997d-2b692417033f)
> <br/>
> 프로듀서 서버에서 브로커와 연결이 단절되었다는 로그 확인
> 
> ![during_broker_failure_test_2_in_consumer](https://github.com/user-attachments/assets/de3f24a1-8015-4e30-b63c-8ce50191f5cb)
> <br/>
> 컨슈머(클라이언트 메시지 카운터, 메시지 수집기)에서도 연결이 단절되었다는 로그 확인
> 
> #### 3. 전반적인 작업 흐름은 우아한 브로커 재시작과 크게 다르지 않은 것 같음
> 
> 우아한 브로커 종료와 강제 브로커 종료의 차이는 종료하는 브로커가 안전하게 데이터를 디스크에 저장하느냐와 다른 브로커들이 이를 통보 받고 미리 대응을 할 수 있는지의 차이 정도로 보임
> <br/>
> 그 외 리더 레플리카가 다른 브로커에게 할당되고 (리더 재선출) 클러스터 메타데이터가 업데이트되며 프로듀서와 컨슈머들이 리디렉션되는 과정을 로그 등을 확인해봤을 때 거의 동일한 흐름으로 진행됨
> 
> #### 4. 최종적으로 메시지 큐에 저장된 데이터 및 레디스에 저장된 데이터 모두 일관성 유지
> 
> 결과는 우아한 브로커 재시작과 동일
</details>
  
### 클라이언트 메시지 카운터 & 메시지 수집기 (준비중)
#### 컨슈머 리밸런스 유도 (컨슈머 추가/재시작(장애) 등의 사유)

## 성능 테스트 (준비중)
### 기본 테스트 성능 측정
### 프로듀서 서버 설정 튜닝
### 클라이언트 메시지 카운터 & 메시지 수집기 설정 튜닝

## 더 고민해볼 것들

#### 컨슈머 서버들이 굳이 카프카 스트림즈 어플리케이션으로서 동작할 필요는 없어 보이는데 왜 그렇게 했는가?
* 사실 이 작업들은 메시지 통계만 내는 작업이므로 각자 client_message 토픽에 다른 컨슈머 그룹으로 컨슈밍해도 데이터 일관성이 보장됨
* 하지만 컨슈머 사이의 의존성으로 인해 순서가 필요할 경우를 시뮬레이션하는 것이고 실제로 순서가 필요한 작업은 아님
* 마치 결제와 아이템 지급 같은 순서 보장이 필요한 컨슈밍 작업들을 시뮬레이션하는 것으로 보아도 좋음

#### 프로듀서 서버에서 처음 레디스에 메시지 저장이 실패하고 그 이후 메시지들이 적재되면 순서가 바뀌지 않는가?
* 지금은 당장 프로듀서 서버에서 로그만 찍고 있지만 만약 순서가 매우 중요한 데이터라면 클라이언트도 서버와 연동하여 데이터 순차 처리 보장을 위한 작업을 해야 함
* 클라이언트도 메시지 적재 완료를 확인하고 유실되었음을 판단한다면 재전송하는 메커니즘 도입 필요

#### 컨슈머들은 리밸런스가 발생했을 때나 크래시 등의 예기치 않은 상황에 작업하던 내용이 유실되거나 좀비 컨슈머로서 동작해도 괜찮은가?
* 컨슈머들은 최종적으로 소비한 메시지에 대한 오프셋을 커밋하지 않는다면 다음 실행시 처리되지 않은 것으로 간주하기 때문에 재처리를 수행
* 각 컨슈머들은 레디스에 데이터를 쓸 때 작업(메시지)에 대한 sn으로 순서 보장 및 중복 처리 여부 검사를 통해 연산에 대한 멱등성을 유지함으로써 데이터는 무결성을 유지

#### 데이터 흐름의 양을 제어하고 싶다면?
* 카프카 메시지 큐를 사용할 때 중요한 점은 컨슈밍 속도가 프로듀싱 속도를 잘 따라가는지 확인하고 관리하는 것임
* 컨슈머들이 보다 적극적인 소비가 가능하도록 관련 설정이나 스레드 등의 물리적 자원을 더 투자하는 것이 가능하지만 프로듀싱하는 것을 제어하는 것으로도 어느정도 관리가 가능
* 프로듀싱하는 쪽에서 과도하게 프로듀서 버퍼에 데이터를 밀어 넣을 경우 버퍼는 금방 가득차게 되며 이를 바탕으로 프로듀서 서버는 클라이언트에게 백프레셔를 적용하여 클라이언트의 메시지 전송량을 제어하는 것도 가능

#### 왜 프로듀서 서버에서 메시지 큐에 메시지를 전달하기 전에 캐시 서버에 저장을 하는가?
* 프로듀서 서버를 stateless하게 만들고 장애로 인해 재시작을 할 경우 다른 서버 인스턴스가 이어서 메시지를 유실없이 전달하기 위함

#### 결과적 일관성이 아닌 더 엄격한 일관성 보장이 필요하다면?
* 레디스는 메모리 데이터베이스로 빠른 읽기/쓰기를 지원함으로써 2PC(2 phase commit) 등의 강력한 분산 트랜잭션 메커니즘을 지원하지 않음
* 따라서 데이터에 대한 강력한 일관성이 필요하다면 MySQL(RDB) 또는 MongoDB(NoSQL) 같은 강력한 분산 트랜잭션을 지원하는 데이터베이스를 사용해야 함

#### 메시지 유실이나 적재 실패시 왜 해당 메시지부터 전부 재전송하는가?
* 신뢰성있는 데이터 전송을 위해 먼저 간단하게 구현할 수 있는 Go-Back-N 방법을 도입하여 빠르게 구현해보았음
* 차후 고도화를 한다면 DLQ(Dead Letter Queue) + Selective-Repeat 방법을 사용하여 재전송이 필요한 메시지만 재전송할 수 있도록 작업할 예정 

#### 현재 프로듀서 서버의 장애로 인해 재시작될 경우는 신뢰성있는 데이터 전송이 보장되는가?
* 프로듀서 서버의 장애 등으로 재시작이 된다면 현재로는 신뢰성있는 데이터 전송이 어려운 상황
* 이를 달성하려면 클라이언트부터 데이터 재전송 메커니즘을 도입해야 함
* 또한 프로듀서 서버에서 클라이언트 식별을 ip:port 형식의 주소 기반으로 하고 있는데 이로써는 정확히 클라이언트를 구분할 수 없으므로 클라이언트 인증이 필요함
* 위 두가지만 해결된다면 프로듀서 서버에서는 모든 메시지를 임시로 레디스에 보관하기 때문에 프로듀서 장애 등으로 재시작되는 상황에서도 신뢰성있는 데이터 전송이 보장될 것으로 보임


(내용들을 더 모으는 중)
