# 이벤트 기반 아키텍처 스타일
- 확장성이 뛰어난 고성능 애플리케이션 개발에 널리 쓰이는 비동기 분산 아키텍처 스타일
- 적응성이 높음 (소규모 ~ 크고 복잡한 대규모 어플리케이션 적용 가능)
- 이벤트를 비동기 수신/처리하는 별도의 이벤트 처리 컴포넌트들로 구성

### 요청 기반 모델

<img width="454" alt="스크린샷 2024-12-31 오후 3 30 20" src="https://github.com/user-attachments/assets/a9f76939-ba77-4f04-a5aa-e8c7fcff6751" />

1. 특정 액션을 시스템에 요청 
2. 요청 오케스트레이터(주로 유저 인터페이스)가 접수 
3. 다양한 요청 프로세서에 동기적으로 요청 전달 
4. 요청 처리(ex. DB에서 정보 조회/수정 등의 작업 수행)

### 이벤트 기반 아키텍처와의 비교
- **요청 기반** : 데이터 기반의 확정적인 요청 (시스템이 반응해야할 이벤트 발생 X)
   ex. 유저의 주문 이력 검색
- **이벤트 기반** : 특정한 상황에 대응하여 이벤트에 알맞은 액션 필요
   ex. 온라인 경매 사이트에서의 입찰(가격이 발표된 직후 발생하는 이벤트) 동시 발생한 다른 입찰가와 비교 후 가장 높은 입찰가 결정


## 14.1 토폴로지
- **중재자 토폴로지** : 이벤트 처리 워크플로를 제어해야하는 경우
- **브로커 토폴로지** : 신속한 응답 및 동적인 이벤트 처리 제어가 필요한 경우

## 14.2 브로커 토폴로지
- 메시지가 경량 메시지 브로커를 통해 브로드캐스팅되는 방식으로 이벤트 프로세서 컴포넌트에 분산
- 비교적 처리 프름이 단순하고 중앙에서 이벤트를 조정할 필요가 없는 경우 유용

### 브로커 토폴로지의 컴포넌트
- 시작 이벤트: 이벤트 흐름 개시할 때 생성
- 이벤트 브로커: 시작 이벤트를 받아 프로세서에 전달
- 이벤트 프로세서: 브로커에서 시작 이벤트를 받자마자 관련된 처리 작업 수행
- 처리 이벤트: 이벤트 프로세서에서 처리 작업을 마친뒤 생성. 필요 시 부가적 처리를 위해 이벤트 브로커에 비동기 전송

<img width="467" alt="스크린샷 2024-12-31 오후 3 30 45" src="https://github.com/user-attachments/assets/a4972cfb-def9-49b5-a5b8-cff298f2eb2d" />

- 브로커 토폴로지는 파이어 앤드 포겟(fire and forget) 방식으로 비동기 브로드캐스팅 
   -> 메시지를 보내고 나서 결과를 기다리지 않음
- 브로커 토폴로지에서는 다른 이벤트 프로세서의 관심 여부와 무관하게 각 이벤트 프로세서가 자신이 한 일을 모두에게 알리는게 바람직 
   -> 이벤트 처리 과정에서 기능 추가가 필요하게 되더라도 아키텍처를 쉽게 확장할 수 있음

### 예시: 주문 입력 시스템에서 이벤트 처리 흐름
<img width="601" alt="스크린샷 2024-12-31 오후 3 31 05" src="https://github.com/user-attachments/assets/319504de-d79e-45ad-8cca-2c033e13565f" />


1. 주문이 접수되면 OrderPlacement 이벤트 프로세서는 `PlaceOrder` 시작 이벤트를 받아 데이터베이스에 주문 삽입 후 주문 ID 반환
2. 주문을 생성했음을 `order-created` 이벤트를 통해 나머지 프로세서에 통보
3. 알림, 결제, 재고 프로세서는 이벤트를 받아 각자의 임무 병렬 수행
   - 알림 프로세서: 고객에게 이메일을 보낸 후 `email-sent` 이벤트 생성. 해당 이벤트 리스닝하는 프로세서는 없지만 확장성을 위함
   - 재고 프로세서: 해당 품목의 재고 차감 후 `inventory-updated` 이벤트를 통해 완료 작업을 알림
      -> 창고 프로세서가 이를 받아 재고 상태 관리 및 부족한 품목 재주문 
   - 결제 프로세서: 결제 처리 진행. 승인 완료 시 `payment-applied`, 승인 거절 시 `payment-denied` 이벤트 생성
4. 승인 완료 시, 주문 이행 프로세서는 `payment-applied` 이벤트를 받아 포장 작업 수행. 작업 완료 시 `order-fulfilled` 이벤트 발행
   - 알림 프로세서: 주문이 이행됐고 배송 준비가 완료됐음을 고객에게 알림
   - 배송 프로세서: 적절한 배송 방법을 선택하여 주문을 배송하고 `order-shipped` 이벤트 발행
5. 승인 거절 시, 알림 프로세서는 `pay-denied` 이벤트를 받아 다른 방법으로 결제하도록 이메일로 유도

#### 브로커 토폴로지는 모든 이벤트 프로세서가 고도로 분리되어 있고 서로 독립적으로 움직임 (릴레이 경주)
- 이벤트 전달 후 더 이상 그 이벤트 처리에는 관여하지 않고 다른 시작 이벤트/처리 이벤트에 반응할 준비
- 각 이벤트 플세서는 이벤트 처리 도중 독립적으로 확장 가능

### 브로커 토폴로지의 장단점
|장점|단점|
|---|---|
|이벤트 프로세서가 디커플링됨<br>확장성 높음<br>응답성 우수<br>성능 우수<br>내고장성 뛰어남|워크플로 제어<br>에러 처리<br>복구성<br>재시작 능력<br>데이터 비일관성|

- 성능, 응답성, 확장성 측면에서 장점이 많음
- 시작 이벤트와 연관된 전체 워크플로를 제어할 수 없음 
  -> 조건에 따라 상황이 매우 유동적이고 실제 주문이 언제 끝났는지 알기 어려움 (+ 에러 처리 어려움)
  -> 처리가 실패해도 다른 파트는 그 사실을 모름 -> 비지니스 프로세스가 교착 상태에 빠지고 정체될 수 있음
- 브로커 토폴로지에서는 비지니스 트랜잭션을 재시작하는 기능(복구성) 지원되지 않음
- 시작 이벤트가 처리되는 순간부터 다른 작업들은 이미 비동기로 수행됨 -> 어디서부터 시작이벤트를 다시 넣어야할 지 알 수 없음


## 14.3 중재자 토폴로지
- 브로커 토폴로지의 단점들을 일부 보완
- 여러 이벤트 프로세서 간 조정이 필요한 시작 이벤트에 대해 워크플로를 관리/제어하는 이벤트 중재자가 존재

### 중재자 토폴로지의 컴포넌트
- 시작 이벤트
- 이벤트 큐
- 이벤트 중재자
- 이벤트 채널
- 이벤트 프로세서


<img width="702" alt="스크린샷 2024-12-31 오후 3 31 20" src="https://github.com/user-attachments/assets/67d67032-4319-49fd-9a42-7ea0348acfb5" />

1. 시작 이벤트로 전체 이벤트 프로세스를 개시 (브로커 토폴로지와 동일)
2. 시작 이벤트가 이벤트 중재자로 전달됨 (브로커 토폴로지와의 차이점)
3. 이벤트 중재자는 각각의 이벤트 채널(큐)로 전달되는 처리 이벤트 생성 (이벤트 처리에 관한 단계 정보만 가지므로)
4. 각 이벤트 프로세서는 자신의 이벤트 채널에서 이벤트 처리 후 중재자에 작업을 완료했다고 응답 (다른 프로세서에게 자신이 한 일을 알리지 않는다는 점에서 브로커 토폴로지와 차이 있음)

#### 💡 중재자 토폴로지 구현체는 대부분 특정 도메인/이벤트 그룹과 연관된 중재자가 여럿 존재 -> 단일 장애점 줄임/전체 처리량 및 성능 높임
ex. 고객 중재자(신규 고객 등록, 프로필 업데이트), 주문 중재자(장바구니 추가, 체크아웃) 나누어 이벤트 처리

### 알맞은 이벤트 중재자 구현체 선택하기
<img width="478" alt="스크린샷 2024-12-31 오후 3 31 43" src="https://github.com/user-attachments/assets/f985ac4f-3fff-4313-b98c-f48b8650e525" />

- 간단한 에러처리와 오케스트레이션이 필요한 경우: Apache Camel, Mule ESB, Spring Integration (워크플로를 프로그래밍 코드로 제어)
- 워크플로에 조건부 처리가 많고 동적 경로가 많아 에러처리가 복잡한 경우: Apache ODE, Oracle BPEL Process Manager
   - 이벤트 처리 단계를 기술하는 BPEL 기반으로, BPEL 아티팩트에는 에러 처리, 리다이렉션, 멀티 캐스팅 기능이 체계적으로 구현되어 있음
   - BPEL은 배우기 쉽지 않은 언어라, 일반적으로는 BPEL이 내장된 GUI 도구를 사용해 만듬
   - 이벤트 처리 중 사람이 개입하는 워크플로에는 적합 X -> BPM 엔진
- 이벤트 복잡도는 한가지 기준으로 평가되지 않으므로 단순함, 어려움, 복잡함 정도로 분류한 후 모든 이벤트가 항상 단순한 중재자(ex. Apache Camel)을 거치도록 함 -> 등급에 따라 직접 처리하거나 더 복잡한 이벤트 중재자에게 위임

 
### 예시: 주문 입력 시스템에서 이벤트 처리 흐름
<img width="590" alt="스크린샷 2024-12-31 오후 3 32 10" src="https://github.com/user-attachments/assets/31fef58b-212d-4bd1-9579-b8f220c3060e" />


- 2, 3, 4 단계의 처리 이벤트는 모두 동시에 발생하면서 단계별로 처리됨
- 3단계(주문 이행)는 4단계(주문 배송)에서 배송 준비가 끝나 고객에게 알림을 보내기 전에 반드시 완료되어 ack(확인 응답) 받아야 함

<br>

### 1단계
<img width="469" alt="스크린샷 2024-12-31 오후 3 32 24" src="https://github.com/user-attachments/assets/0cd9d616-9162-476c-816f-976cc594269a" />


1. 시작 이벤트를 접수한 이벤트 중재자는 `create_order` 이벤트를 생성
2. `create_order` 이벤트는 Order Placement Queue로 보내짐
3. Order Placement 이벤트 프로세서는 이벤트를 받아 문제 없는지 확인 후 주문 ID와 함께 중재자에게 확인 응답 보냄 
    (중재자는 해당 주문 ID를 고객에게 보내 주문 접수 사실을 알릴 수 있음)


### 2단계
<img width="469" alt="스크린샷 2024-12-31 오후 3 32 44" src="https://github.com/user-attachments/assets/19c30d7d-44bc-48f0-b045-3141088f9960" />

1. 중재자는 3개의 이벤트 `email-customer`, `apply-payment`, `adjust-inventory` 동시에 만들어 지정된 큐로 전달
2. Notification, Payment, Inventory 프로세서는 메시지를 받아 각자의 업무 수행 후 확인 응답을 중재자에게 알림
3. 중재자는 3단계 이동 전 3개의 프로세스로 부터 모두 확인 응답을 받을때까지 대기

### 3단계
<img width="469" alt="스크린샷 2024-12-31 오후 3 33 09" src="https://github.com/user-attachments/assets/9fab5882-8f5e-403d-b2ac-83644b60ec9f" />

1. 주문 이행 (`fulfill-order`, `order-stock` 이벤트 동시 발생)
2. Order Fulfillment, Warehouse 이벤트 프로세서는 이 두 이벤트를 받아 각자 할일을 한 후 확인 응답 중재자에게 보냄

### 4단계
 
<img width="469" alt="스크린샷 2024-12-31 오후 3 33 21" src="https://github.com/user-attachments/assets/0e20e358-e601-439e-ab05-9c60ee9859a2" />

1. 주문 배송 (`ship-order` 이벤트 생성)
2. 이때, 다음에 해야할 일(배송 준비 완료를 고객에게 알림) 정보가 담긴 `email-customer` 처리 이벤트도 함께 생성

### 5단계
<img width="469" alt="스크린샷 2024-12-31 오후 3 33 35" src="https://github.com/user-attachments/assets/8d174a2c-540e-4e5f-be10-085579db3bb7" />


1. 주문이 배송되었음을 고객에게 알림
2. 워크플로가 완료되어 중재자는 시작 이벤트 흐름을 완료로 마킹 후 시작 이벤트와 연관된 상태 모두 삭제

### 중재자 토폴로지의 장단점

|장점|단점|
|---|---|
|워크플로 제어<br>에러 처리<br>복구성<br>재시작 능력<br>데이터 일관성|이벤트 프로세서가 커플링됨<br>확장성 낮음<br>성능 낮음<br>내고장성 좋지 않음<br>워크플로 모델링 복잡|


- 워크플로에 대해 잘 알고 있고 통제가 가능함 (브로커 토폴로지의 단점 보완)
- 워크플로를 직접 제어하므로 이벤트 상테를 유지하며 필요 시 에러 처리, 복구, 재시작 가능
   (ex. 3단계 결제 처리에 오류가 난 경우, 중재자는 워크플로 중단 후 요청 상태 기록 가능/결제 처리 완료 시 3단계부터 다시 시작)
- 브로커 토폴로지는 이벤트가 발행되면 프로세서에서 각자 맡은일을 하고 나머지 이벤트 프로세서가 그 액션에 반응하는 방식으로 동작(응답 확인 X)하는 반면, 
   중재자 토폴로지에서의 이벤트는 반드시 처리되어야 할 이벤트(command)
- 중재자 토폴로지는 쉽게 확장이 가능하긴 하지만, 중재자도 함께 확장해야 하므로 전체 이벤트 처리 흐름에 병목이 생기기 쉬움
- 이벤트 처리를 중재자가 제어하므로 이벤트 프로세서가 상대적으로 많이 커플링되어 있음(성능이 브로커 토폴로지보다 떨어짐)

## 14.4 비동기 통신
### 이벤트 기반 아키텍처 스타일이 다른 아키텍처와 차별화되는 점?
- 컨슈머의 응답을 받아야 하는 요청/응답 처리, 응답이 필요없는 fire and forget 처리 모두 비동기 통신만을 사용
- 이를 통해 시스템 응답성을 전반적으로 높일 수 있음

### 예시: 댓글 게시

<img width="693" alt="스크린샷 2024-12-31 오후 3 34 06" src="https://github.com/user-attachments/assets/f34ef07c-cda3-478d-9e80-8d9cb6dcac2d" />


- 댓글 서비스는 여러 파싱 엔진(비속어 검사기/문법 검사기/문맥 검사기 등)을 거침
- 보통 댓글 하나 게시하는데 3000 ms 소요
- REST로 동기 호출을 하는 경우: 서비스가 댓글 수신 50ms + 댓글 게시 3000ms + 댓글 등록됐음을 유저에게 알림 50ms = 3100ms
- 메시지 비동기 전송의 경우: 댓글 수신 25ms
   (실제 댓글 게시에 3000ms가 소요되나, 최종 유저 관점에서는 이미 댓글의 처리는 완료됨)

=> 비동기 통신을 통해 응답성을 높일 수 있음 (실제 성능을 높인 것은 아님)

### 비동기 통신에서의 문제점: 에러 처리
- 동기 호출은 댓글이 게시되었음을 최종 유저에게 보장하지만 비동기 호출은 그렇지 않음
- 최종 유저의 관점에서는 댓글이 이미 게시되었지만, 댓글에 비속어가 포함되어 있다면 게시는 거부됨 
- 응답성은 매우 개선되지만, 에러 처리가 쉽지 않기 때문에 시스템이 복잡도는 가중

## 14.5 에러 처리: 리액티브 아키텍처의 워크플로 이벤트 패턴
- 시스템이 응답성에 영향을 미치지 않고 탄쳑적으로 에러를 처리할 수 있게 만드는 패턴
- 워크플로 대리자를 통해 위임, 봉쇄, 수리 작업 진행

<img width="630" alt="스크린샷 2024-12-31 오후 3 34 42" src="https://github.com/user-attachments/assets/c814a95c-d72a-436b-82ec-eae1d8c308f8" />

1. 이벤트 프로듀서는 매시지 채널을 통해 데이터를 이번트 컨슈머에 비동기 전송
2. 이벤트 컨슈머는 처리 도중 에러가 발생하는 경우 즉시 해당 에러를 워크플로 프로세서에 위임
3. 위임 후 컨슈머는 바로 이벤트 큐의 다음 메시지로 넘어감
4. 에러를 수신한 워크플로 프로세서는 (사람의 개입 없이) 프로그래밍 방식으로 원데이터를 변경하여 긴급 조치 후 원래 큐로 돌려보냄
   (메시지의 문제를 확인할 수 없는 경우 대시보드라고 불리는 다른 큐로 보내어 담당자가 직접 메시지 확인 후 조치)
5. 이벤트 컨슈머는 이 메시지를 새로운 메시지로 간주하여 재처리 시도

=> 위 방법을 통해 컨슈머는 에러가 발생해도 바로 다음 메시지를 처리하므로 전체 응답성에 영향 X


### 예시: 거래 자문가가 대형 트레이딩 펌의 거래 주문을 대신하는 경우
<img width="677" alt="스크린샷 2024-12-31 오후 3 34 58" src="https://github.com/user-attachments/assets/e7f61f08-d139-49fd-9f42-9c607c61312f" />

#### ⚠️ 주의할 점
- 에러가 발생한 후 메시지를 다시 생성하면 처리 순서가 변경됨
- 특정 계정의 모든 주식거래는 순서대로 처리되어야 하므로 주어진 콘텍스트에서 메시지 순서를 유지하도록 해야함
   1. 거리 처리 서비스에서 에러 발생한 거래의 계좌 번호를 큐에 담아 보관
   2. 해당 계좌번호의 거래 이벤트가 발생하면 임시 큐에 저장
   3. 에러난 거래가 조치되면 임시 큐에서 꺼내 순차적으로 처리

## 14.6 데이터 소실 방지
이벤트 기반 아키텍처는 데이터가 소실될 만한 곳이 많음

### 데이터가 소실되는 일반적인 경우
[예시] 이벤트 프로세서 A가 큐에 메시지를 비동기로 전송 후 프로세서 B에서 해당 메시지를 받아 DB 삽입 작업을 하는 경우
<img width="677" alt="스크린샷 2024-12-31 오후 3 35 13" src="https://github.com/user-attachments/assets/6fc80ff0-d98c-4697-8fef-1ce15651e0f6" />

1.  이벤트 프로세서 A에서 메시지가 큐로 전달되지 않음 or 전달되었지만 B 프로세서에 도달하기 전에 브로커가 다운됨
2. 이벤트 프로세서 B가 큐에서 다음 메시지를 꺼낸 후 이벤트를 처리하기 전 장애 발생
3. 데이터에 에러가 있어 프로세서 B에서 DB에 저장할 수 없음

### 1번 이슈 해결법
- 동기 전송과 퍼시스턴스 메시지 큐 이용
- 동기 전송은 브로커가 메시지를 저장했다는 확인 응답을 줄때까지 프로듀서를 차단하여 기다리게 함
- 퍼시스턴스 메시지 큐는 전달 보장 지원(브로커가 메시지 수신 시 메모리/물리적 데이터 저장소에 메시지 저장) -> 브로커가 다운되어도 다시 살아나면 메시지 처리 계속할 수 있음

### 2번 이슈 해결법
- 클라이언트 확인 응답 모드라는 메시지 기술 이용
- 메시지는 원래 큐에서 빠져나가는 즉시 삭제가되지만, 클라이언트 확인 응답 모드는 모든 메시지를 큐에 보관한 채 다른 컨슈머는 메시지를 읽을 수 없도록 client ID를 메시지에 부착
- 해당 방법을 사용하면 프로세서 B가 잘못되어도 메시지는 큐에 남아있기 때문에 데이터의 소실을 방지할 수 있음

### 3번 이슈 해결법
- 트랜잭션의 커밋으로 해결: ACID(원자성, 일관성, 격리성, 내구성) 보장
- 최종 참여자 지원(LPS)를 사용하면 메시지 처리가 끝나 데이터베이스에 저장됐음을 확인한 후 큐에서 메시지가 삭제됨



## 14.7 브로드캐스팅
- 이벤트 기반 아키텍처는 메시지를 누가 받든, 그 메시지로 무슨 일을 하든 상관없이 이벤트를 브로드캐스트할 수 있음
- 메시지 프로듀서는 자신이 보낸 메시지를 어느 이벤트 프로세서가 수신할 지, 메시지를 받아 무슨 일을 할 지 알 수 없음
- 즉, 브로드캐스팅은 여러 이벤트 프로세서를 높은 수준으로 디커플링하는 수단

## 14.8 요청 - 응답
- 이벤트 프로세서에서 응답이 필요한 동기 통신을 해야하는 경우, **요청-응답 메시징 방식** 수행 (ex. 도서 주문 후 주문 ID가 필요한 경우)
- 요청-응답 메시징 방식에서 이벤트 채널은 요청 큐, 응답 큐로 구성되어 있음

<img width="677" alt="스크린샷 2024-12-31 오후 3 35 31" src="https://github.com/user-attachments/assets/b55f15e8-409c-48e8-9a46-dbcc171ec972" />


1. 처음 정보를 요청하면 요청 큐에 비동기 전송된 후 메시지 프로듀서에게 제어권이 반환
2. 메시지 프로듀서는 응답 큐에 응답이 도착하길 기다리며 차단 대기 상태가 됨
3. 메시지 컨슈머가 메시지를 받아 처리한 후 응답 큐에 응답을 보냄
4. 이벤트 프로듀서는 응답 데이터가 포함된 메시지 수신 후 대기 풀림

### 요청-응답 메시징의 주요 기술 1) 메시지 헤더에 상관 ID 사용
상관 ID는 응답 메시지의 필드로 ,대부분 원요청 메시지의 메시지 ID로 세팅됨
<img width="477" alt="스크린샷 2024-12-31 오후 3 35 46" src="https://github.com/user-attachments/assets/7f48fac0-beda-4361-9056-991ee5c0e5e6" />

1. 이벤트 프로듀서가 요청 큐에 메시지를 보내고 고유 메시지 ID 기록(124)
2. 이벤트 프로듀서는 응답 큐 차단 대기 (이때 상관 ID 124 설정)
3. 이벤트 컨슈머가 124 메시지를 받아 요청 처리
4. 이벤트 컨슈머는 응답 메시지를 생성하고 상관 ID를 124로 설정한 이벤트 생성(고유 ID는 857)
5. 이벤트 컨슈머는 새 메시지(고유 ID: 857, 상관 ID: 124) 응답 큐로 보냄
6. 이벤트 프로듀서는 상관 ID가 124인 메시지가 있으므로 해당 메시지 수신

### 요청-응답 메시징의 주요 기술 1) 임시 큐 사용
임시 큐는 지정된 요청에서만 사용되며, 요청이 들어오면 생성되고 종료 시 삭제
<img width="471" alt="스크린샷 2024-12-31 오후 3 36 08" src="https://github.com/user-attachments/assets/de29f7f1-7223-4582-afcc-bc1a08dc9f27" />

1. 이벤트 프로듀서는 임시 큐를 생성하고 reply-to 헤더에 임시 큐 이름을 세팅(Q998)하여 요청 큐에 메시지 보냄
2. 이벤트 프로듀서는 임시 응답 큐를 차단 대기하면서 응답이 도착하길 기다림
3. 이벤트 컨슈머는 메시지를 받아 요청을 처리한 후 reply-to 헤더에 세팅된 이름을 가진 응답 큐에 메시지를 보냄
4. 이벤트 프로듀서는 메시지를 수신 후 임시 큐 삭제

#### 💡 대체적으로 상관 ID를 사용하는 방법이 권장됨
기술적으로는 임시 큐가 훨씬 단순하지만, 메시지 브로커가 요청 시마다 임시 큐 생성/폐기를 반복해야 하므로 성능과 응답성에 영향이 있기 때문


## 14.9 요청 기반이냐, 이벤트 기반이냐
### 이벤트 기반 모델이 요청 기반보다 좋은 점
- 동적인 유저 컨텐츠의 응답성이 좋음
- 확장성, 탄력성 우수
- 민첩성과 변화 관리 우수
- 적응성과 **확장성 뛰어남**
- **응답성과 성능 좋음**
- 실시간 의사결정 가능
- 반응성 좋음

### 단점
- 최종 일관성만 지원됨
- **처리 흐름을 제어하기 곤란**함
- 이벤트 흐름의 **결과를 예측하기 어려움**
- 테스팅, 디버깅이 어려움

## 14.10 하이브리드 이벤트 기반 아키텍처
- 이벤트 기반 아키텍처와 다른 아키텍처 스타일을 함께 사용하는 방법
   (ex. 이벤트 기반 아키텍처를 마이크로서비스 아키텍처, 공간 기반 아키텍처의 일부로 사용)
- 이벤트 기반 아키텍처를 추가하면 **병목 지점 제거 할 수 있음 + 이벤트 요청을 백업하는 배압 지점 확보 가능 + 응답성 보장**

## 14.11 아키텍처 특성 등급
<img width="378" alt="스크린샷 2024-12-31 오후 3 36 38" src="https://github.com/user-attachments/assets/342db471-8794-4c3b-9be7-ac785f64f192" />

- 이벤트 기반 아키텍처에서는 특정 도메인이 여러 이벤트 프로세서에 분산되어 있음 (중재자, 큐, 토픽을 통해 서로 묶여 있음) 
  -> 도메인 분할 아키텍처 X, 기술 분할 아키텍처
- 성능, 확장성, 내고장성이 높음
  -> 이벤트 프로세서는 프로그래밍 방식으로 로드 밸런싱 가능 + 확장성이 뛰어남 (요청 수 증가 시 이벤트 프로세서 추가 쉬움)
- 비결정적, 동적인 이벤트 흐름 때문에 단순성과 시험성은 상대적으로 낮음

