# (7) 호텔 예약 시스템

## 1단계: 문제 이해 및 설계 범위 확정

- 기능 요구사항
    - 호텔 정보 페이지 표시
    - 객실 정보 페이지 표시
    - 객실 예약 지원
    - 호텔, 객실 정보를 추가/삭제/갱신하는 관리자 페이지 지원
    - 초과예약 지원
- 비기능 요구사항
    - 높은 수준이 동시성(concurrency) 지원
    - 적절한 지연 시간: 몇 초 지연 허용
- 개략적 규모 추정
    - 총 5,000개 호텔, 100만개의 객실 호텔 체인 가정
    - 평균 사용중 객실: 70%, 평균 투숙 기간: 3일
    - 일일 예상 예약 건수 = 1백만 * 0.7 / 3 = 233,333 (올림 : 약 240,000)
    - 초당 예약 건수 = 240,000/하루:10^5초 ~= 3
    초당 트랙잭션 수 (TPS)는 생각보다 적음
    - 고객이 웹사이트를 사용하는 흐름
        1. 호텔/객실 상세 페이지 (**조회 발생)**
        2. 예약 상세 정보 페이지 (**조회 발생)**
        3. 객실 예약 페이지에서 예약 (**트랜잭션** 발생) 
        - 대략 10%의 사용자가 다음 단계 진행, 90% 사용자가 최종 단계 전 이탈 가정 → TPS에서 역산하여 조회 QPS 계산
            1. 호텔/객실 상세페이지 QPS : 300
            2. 예약 상세 페이지 QPS: 30
            3. 객실 예약폐이지 QPS: 3

## 2단계: 개략적 설계안 제시 및 동의 구하기

### API설계

- 호텔 관련 API
    
    
    | API | 설명 |
    | --- | --- |
    | GET /v1/hotels/id | 호텔의 상세 정보 반환 |
    | POST /v1/hotels | 신규 호텔 추가, 호텔 직원만 사용 가능 |
    | PUT /v1/hotels/id | 호텔 정보 갱신, 호텔 직원만 사용 가능 |
    | DELETE /v1/hotels/id | 호텔 정보 삭제, 호텔 직원만 사용 가능 |
- 객실 관련 API
    
    
    | API | 설명 |
    | --- | --- |
    | GET /v1/hotels/:id/rooms/id | 특정 객실 상세 조회 |
    | POST /v1/hotels/:id/rooms | 신규 객실 추가, 호텔 직원만 사용 가능 |
    | PUT /v1/hotels/:id/rooms/id | 객실 정보 갱신, 호텔 직원만 사용 가능 |
    | DELETE /v1/hotels/:id/rooms/id | 객실 정보 삭제, 호텔 직원만 사용 가능 |
- 예약 관련 API
    
    
    | API | 설명 |
    | --- | --- |
    | GET /v1/reservations | 로그인 사용자의 예약 이력 반환 |
    | GET /v1/reservations/id | 특정 예약 상세  조회 |
    | POST /v1/reservations/id | 신규 예약 |
    | DELETE /v1/reservations/id | 예약 취소 |
    - 신규 예약 시 파라미터
    
    ```
    {
    	"startDate": "2021-04-28",
    	"endDate":"2021-04-30",
    	"hotelID":"245"
    	"roomID":"U12354673389",
    	"reservationID":"13422445"
    }
    ```
    
    - reservationID: 멱등 키(idempotent key, 이중예약방지, 예약은 단 한번)

### 데이터 모델

- 호텔 예약 시스템이 지원해야하는 질의들
    1. 호텔 상세 정보 확인
    2. 지정된 날짜 범위에 사용 가능한 객실 유형 확인
    3. 예약 정보 기록
    4. 예약 내역 또는 과거 예약 이력 정보 조회
- 기타 주의 사항
    - 시스템 평균 규모는 크지 않으나 대규모 이벤트 시 트래픽 급증 대비
- ⇒ 관계형 데이터 베이스 선택
    - 읽기 빈도가 쓰기 연산에 비해 높은 작업 흐름을 잘 지원(NoSQL: 쓰기 연산에 최적화)
    - ACID 속성(원자성, 일관성, 격리성, 영속성) 보장
    - 비즈니스 데이터 구조를 명확하게 표현 가능, 엔티티(호텔, 객실, 객실 유형 등) 간의 관계를 안정적으로 지원 가능
- 데이터 베이스 스키마
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A1979b12d-53fc-4934-bca1-684998bc8331%3Aimage.png?table=block&id=319be4ca-d9d2-8021-bfc0-e92993baa4db&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - status 필드 상태 예시: 결제 대기(pending), 결제 완료(paid), 환불 완료(refunded), 취소(canceled), 승인 실패(rejected)
    - 상태 천이도(state machine) 다이어그램
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A55a4ce40-27d0-472c-95a7-b5159dff458f%3Aimage.png?table=block&id=319be4ca-d9d2-8062-8c15-cdec207496b1&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
- room_id
    - 에어비앤비 등 특정 예약을 객실하는 경우에는 ok
    - 호텔처럼 특정 객실이 아닌 객실 유형(스탠다드 룸, 스위트 룸 등)을 예약하는 경우: 데이터 모델 변경 필요 → 상세 설계 참고

### 개략적 설계안

- 마이크로서비스 아키텍처(MSA) 사용
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A10425cdb-9269-43b3-9405-8cf7c74e449d%3Aimage.png?table=block&id=319be4ca-d9d2-80f3-b8c6-ef55df57f453&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - **사용자:** 웹 or 앱으로 객실 예약하는 당사자
    - **관리자(호텔 직원)**: 고객 환불, 예약 취소, 객실 정보 갱신 등의 작업 수행 권한 존재
    - **CDN(콘텐츠 전송 네트워크)**: 자바스크립트 코드 번들, 이미지, 동영상, HTML 등 모든 정적 콘텐츠를 캐시화, 웹사이트 로드 성능 개선
    - **공개 API 게이트웨이**: 처리율 제한(rate limiting), 인증 등의 기능 지원하는 완전 관리형 서비스, 엔드포인트 기반으로 특정 서비스에 요청 전달
    - **내부 API**: 승인된 호텔 직원만 사용 가능한 API, 내부 소프트웨어 or 웹사이트로 사용 가능. VPN(가상 시설망)등의 기술을 사용, 외부 공격으로부터 보호
    - **호텔 서비스**: 호텔과 객실에 대한 상세 정보 제공
        - 호텔, 객실 정보는 일반적으로 정적: 캐시 가능
    - **요금 서비스**: 미래의 어떤 날에 어떤 요금을 받아야 하는지 데이터 제공. 손님의 몰림도에 따라 가격 조정 가능
    - **예약 서비스**: 예약 요청을 받고 객실을 예약하는 과정을 처리, 잔여 객실 정보 갱신
    - **결제 서비스**: 고객의 결제를 맡아 처리, 예약 상태 변경 (결제 처리, 승인 실패)
    - **호텔 관리서비스**: 승인된 호텔직원만 사용가능, 임박 예약 기록 확인, 고객 객실 예약, 예약 취소 등

## 3단계: 상세 설계

### 개선된 데이터 모델

- 호텔 예약 시 객실id가 아닌 객실 유형 예약 사용 시 스키마 변경
    - 예약 API: roomID → roomTypeID
    
    ```
    POST /v1/reservations
    
    /* 호출 인자 */
    {
    	"startDate": "2021-04-28",
    	"endDate":"2021-04-30",
    	"hotelID":"245"
    	"roomTypeID":"U12354673389",
    	"reservationID":"13422445"
    	}
    ```
    
    - 갱신된 스키마
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Af139973a-df5a-4db8-9d7d-eccb9ea8d111%3Aimage.png?table=block&id=319be4ca-d9d2-80e1-8433-f7d0985d1adc&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
        - room: 객실에 관계된 정보
        - room_type_rate: 특정 객실 유형의 특정 일자 요금 정보
        - reservation: 투숙객 예약 정보
        - room_type_inventory: 호텔의 모든 객실 유형 정보
            - hotel_id(PK): 호텔 식별자
            - room_type_id(PK): 객실 유형 식별자
            - date(PK): 일자
            - total_inventory: 총 객실 수 - 일시적으로 제외 객실 수 
            ( 유지보수 등의 이유로 일부 객실 예약 기능 목록 제외 가능해야함)
            - total_reserved: 지정 된 hotel_id, room_type_id, date에 예약된 모든 객실의 수
    - 저장 용량 추정
        - 5,000개의 호텔. 각 호텔에는 20개의 객실 유형이 있다 가정
            - 저장 레코드의 수: 5000*20*2년*365일 = 7,300만개 → 하나의 데이터베이스로 충분.
                - SPOF(Single-Poin-Of-Failure) 회피, 고가용성 달성 위해 데이터 베이스 복제 필요
    - room_type_inventory 데이터 예제
        
        
        | 호텔id | room_type_id | date | total_inventory | total_reserved |
        | --- | --- | --- | --- | --- |
        | 211 | 1001 | 2021-06-01 | 100 | 80 |
        | 211 | 1001 | 2021-06-02 | 100 | 82 |
        | 211 | 1001 | 2021-06-03 | 100 | 86 |
        | 211 | 1001 | … | … | … |
        | 211 | 1001 | 2025-05-31 | 100 | 0 |
        | 211 | 1002 | 2021-06-01 | 200 | 164 |
        | 2210 | 101 | 2021-06-01 | 30 | 23 |
        | 2210 | 101 | 2021-06-02 | 30 | 25 |
        - 입력: startDate(2021-07-01), endDate(2021-07-03), roomTypeId, hotelId, numberOfRoomsToReserve
        - 출력: 해당 유형 객실 여유 있음 & 사용자가 예약 가능상태: True
                 그 외: False
        - SQL 관점 절차
            1. 주어진 기간에 해당하는 레코드 조회
                
                ```sql
                SELECT date, total_inventory, total_reserved
                  FROM room_type_inventory
                 WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
                   AND date between ${startDate} and ${endDate}
                ```
                
                - 반환 데이터 (잔여 객실 현황)
                    
                    
                    | date | total_inventory | total_reserved |
                    | --- | --- | --- |
                    | 2021-07-01 | 100 | 97 |
                    | 2021-07-02 | 100 | 96 |
                    | 2021-07-03 | 100 | 95 |
            2. 반환된 각 레코드마다 다음 조건 확인
            
            ```java
            if((total_reserved + ${numberOfRoomsToReserve}) <= total_inventory)
            /// true: 예약 가능
            
            /// 10% 초과 예약 기능 추가
            if((total_reserved + ${numberOfRoomsToReserve}) <= 110% * total_inventory)
            ```
            
- 면접관 추가 질문: 예약 데이터가 단일 데이터 베이스에 담기에 너무 크다면?
    - 현재 및 향후 예약데이터만 저장
        - 예약 이력은 자주 접근 하지 않으므로 아카이빙 or 냉동 저장소에 저장
        - 데이터 베이스를 샤딩
            - 가장 자주 사용되는 질의: 예약, 투숙객 이름으로 예약 확인
            - ⇒ 샤딩 키: hotel_id
            - ⇒ 샤딩: hash(hotel_id) % number_of_servers

### 동시성 문제

- 이중 예약 문제
    1. 같은 사용자가 예약 버튼을 여러번 누를 경우
    2. 여러 사용자가 같은 객실을 동시에 예약하려 할 경우
- 시나리오1: 같은 고객의 이중예약
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A2451448f-a68a-42db-802e-dbbf052e5332%3Aimage.png?table=block&id=319be4ca-d9d2-80c0-996f-defc4b169938&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - 클라이언트 측 구현: 클라이언트에서 요청 전송 시 예약버튼 비활성화 or hidden
    - 멱등 API: reservation_id: 를 멱등 키로 사용
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A50e40b06-0941-4072-b0a4-18d06802565d%3Aimage.png?table=block&id=319be4ca-d9d2-805d-82f5-f3238596026f&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
        1. 예약 주문서 생성: 고객이 예약 세부 정보를 입력, ‘계속’ 버튼을 누름 → 예약 서비스: 예약 주문서 생성
        2. 고객이 검토할 수 있도록 예약 주문서 반환. 반환 결과에 reservation_id 추가. (UUID이어야 함)
        3. (a) 검토가 끝난 예약 전송. reservation_id 포함
        4. (b) 사용자가 예약 완료 버튼을 중복 클릭: reservation_id가 테이블의 PK이므로 유일성 조건 위반되어 새로운 레코드 생성X
- 시나리오 2: 여러 사용자가 중복 예약
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A718d444d-c4d6-4f00-a897-48d86093c924%3Aimage.png?table=block&id=319be4ca-d9d2-8054-a0cf-d05ead43c751&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    1. 데이터베이스 트랜잭션 격리수준이 가장 높은 수준(직렬화 가능 수준, serializable)으로 설정되어 있지 않다고 가정
        1. 사용자 1의 트랜잭션 1
        2. 사용자 2의 트랜잭션 2
        3. 호텔 전체 객수 100, 예약 중 객실 99
    2. 트랜잭션 2: if((total_reserved + ${numberOfRoomsToReserve}) <= total_inventory) 검사
    ⇒True 반환
    3. 트랜잭션 1: if((total_reserved + ${numberOfRoomsToReserve}) <= total_inventory) 검사
    ⇒ True 반환
    4. 트랜잭션 1이 먼저 객실 예약 & 예약 현황 갱신
    ⇒ reserved_room = 100
    5. 트랜잭션 2가 객실 예약 시도. 트랜잭션 1이 변경한 데이터는 1이 완료(commit) 전에는 트랜잭션 2에 보이지 않음
    ⇒ 트랜잭션 2 관점의 total_reserved = 99
    ⇒ 이중 에약 발생
    6. 트랜잭션 1이 변경사항을 성공적으로 데이터베이스에 반영
    7. 트랜잭션 2가 변경사항을 성공적으로 데이터베이스에 반영
    - 객실 예약 SQL 질의문 의사코드
        
        ```sql
        #1: 객실 재고 여유 확인
        SELECT date, total_inventory, total_reserved
        FROM room_type_inventory 
        WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
          AND date BETWEEN ${startDate} AND ${endDate}
        
        #1단계에서 반환되는 모든 객실에 다음사항 확인
        IF ((total_reserved + ${numberOfRoomsToReserve}) > 110% * total_inventory) {
          ROLLBACK
        }
        
        #2: 객실 예약
        UPDATE room_type_inventory
        SET total_reserved = total_reserved + ${numberOfRoomsToReserve}
        WHERE room_type_id = ${roomTypeId}
          AND date BETWEEN ${startDate} AND ${endDate}
        
        COMMIT
        
        ```
        
    - 해결 방법: LOCK
        - 비관적 락
        - 낙관적 락
        - 데이터베이스 제약조건(constraint)
- **방안 1: 비관적 락**
    - 사용자가 레코드를 갱신하려고 하는 순간 즉시 락을 걸어 업데이트를 방지하는 기술. 레코드를 갱신하려는 사용자는 앞선 사용자가 변경을 마치고 락을 해제할 때까지 기다려야 함
    - MySQL의 경우: “SELECT … FOR UPDATE” → 반환한 레코드에 ROCK
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Acaaa1d79-f678-4bd5-8541-853d1afd8455%3Aimage.png?table=block&id=319be4ca-d9d2-80ce-abee-cc6b02495383&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
    - 장점
        - 이중 갱신 방지
        - 구현이 쉽고 모든 갱신 연산을 직렬화하여 충돌 방지 가능, 데이터에 대한 경합이 심할 때 유용
    - 단점
        - 여러 레코드에 락을 걸면 교착상태(deadlock)가 발생할 수 있음. 교착상태 방지코드 작성이 까다로움
        - 확장성이 낮음: 트랜잭션의 수명이 길거나 많은 엔티티에 관련된 경우 데이터베이스 성능에 심각한 결함
- **방안 2: 낙관적 락**
    - 여러 사용자가 동시에 같은 자원을 갱신하려 시도하는 것을 허용
    - 버전 번호, 타임스탬프의 두가지 방법으로 구현
    - 버전 번호로 구현한 낙관적 락의 예시
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ab6f281d9-ae78-4471-8031-50d6f70a2929%3Aimage.png?table=block&id=319be4ca-d9d2-8038-88c5-d419053c6e1e&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
        1. 데이터베이스 테이블에 version 열을 추가
        2. 사용자가 데이터베이스 레코드를 수정하기 전, 애플리케이션이 해당 레코드의 버전 번호를 읽음
        3. 사용자가 레코드를 갱신할 때, 애플리케이션은 버전 번호에 1을 더한 다음 데이터베이스에 다시 기록
        4. 이때 유효성 검사 → 다음 버전 번호는 현재 버전 번호보다 1만큼 큰 값이어야 함.
        유효성 검사 실패 시 트랜잭션 중단, 사용자는 2단계부터 재반복
    - 낙관적 락은 일반적으로 비관 락보다 속도가 빠르지만 동시성 수준이 높으면 성능이 급격히 나빠짐 → 많은 사용자가 동시에 시도 시 재시도 횟수 반복 증가
    - 장점
        - 애플리케이션이 유효하지 않은 데이터를 편집하는 일을 막음
        - 데이터베이스 자원에 락을 걸 필요X, **버전 번호를 통해 데이터 일관성을 유지할 책임은 애플리케이션에 있음**
        - 데이터에 대한 경쟁이 치열하지 않은 상황에 적합. 그 경우 락을 관리하는 비용이 없이 트랜잭션을 실행할 수도 있음
    - 단점
        - 데이터에 대한 경쟁이 치열한 상황에서는 낮은 성능 (무한 재시도)
- **방안 3: 데이터베이스 제약 조건**
    - 낙관적 락과 유사한 방법
    - room_type_inventory 테이블에 제약 조건 추가. 해당 제약조건 위반 시 ROLLBACK
        
        ```sql
        	CONSTRAINT 'check_room_count' CHECK(('total_inventory - total_reserved' >=0))
        ```
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A444270eb-8bad-4089-adc1-f5e07e193e0e%3Aimage.png?table=block&id=319be4ca-d9d2-8099-89ac-eabc82154ff9&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
    - 장점
        - 구현이 쉬움
        - 데이터에 대한 경쟁이 심하지 않을 때 잘 동작
    - 단점
        - 낙관적 락과 마찬가지로 데이터에 대한 경쟁이 심하면 실패 연산 수가 기하급수적으로 증가
        - 버전을 통제하기는 어려움
        - 제약조건을 허용하지 않는 데이터베이스가 존재해 데이터베이스 교체가 어려움

### 시스템의 규모 확장

- 면접관 추가질문: “호텔 예약 시스템이 해당 호텔 예약 페이지 뿐만 아니라 아고다, 부킹, 익스페디아 등의 유명 예약 웹사이트와 연동되어야 할경우는?”
    - ⇒QPS가 기하급수적으로 늘 수 있음
    - 시스템의 부하가 높을 때에는 무엇이 병목이 될 수 있을지 파악하는 게 중요.
    - 현재 설계안의 모든 서비스: 무상태 서비스 ⇒ 서버 증설은 용이
- 데이터베이스의 규모를 늘리는 방법
    1. 데이터베이스 샤딩
        - 데이터베이스를 여러 대 두고 각각의 데이터의 일부만 보관
        - 샤딩 키 : hotel_id
        - 예시: QPS가 30,000이라 가정, 16개의 샤드로 분산 할 시
        ⇒각 샤드의 QPS: 30,000/16 = 1875QPS
    2. 캐시
        - 호텔 잔여 객실 데이터의 특성: 과거의 데이터가 중요하지 않음.
        - 낡은 데이터는 자동으로 소멸되도록 TTL을 설정할 수 있으면 바람직.
        - 이력 데이터: 다른 데이터베이스를 통해 질의하도록
        - 레디스(Redis) 사용 추천: TTL과 LRU(Leat Recently Used) 캐시 교체 정책을 사용하여 메모리를 최적으로 활용할 수 있음
        - 데이터 베이스 앞에 캐시 계층을 두고 잔여 객실 확인 및 객실 예약 로직이 해당 계층에서 실행 → 일부만 데이터베이스가 처리, 나머지는 캐시가 처리
            - ⇒ 데이터 로딩 속도 개선 및 데이터 베이스 확장성 보장
            - 주의: 캐시 데이터에 잔여 객실이 충분해도 데이터베이스를 다시 한 번 확인할 필요가 있음
            
            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A174d7294-9599-4091-8f8f-cf8e3d79eb0c%3Aimage.png?table=block&id=319be4ca-d9d2-80d8-94be-d2b4f8e536e4&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=820&userId=&cache=v2)
            
            - **예약 서비스**
                - 지정된 호텔과 객실 유형, 주어진 날짜 범위에 이용 가능한 객실의 수를 질의
                - 객실을 예약하고 total_reserved의 값을 1 증가
                - 고객이 예약을 취소하면 잔여 객실 수 갱신
            - **잔여 객실 캐시**
                - 대부분의 읽기 연산을 담당
                - 레디스로 구현
                    - 키: hotelID_roomTypeID_{날짜}
                    - 값: 주어진 호텔 ID, 객실 유형 ID, 날짜에 맞는 잔여 객실 수
            - **잔여 객실 데이터베이스: 잔여 객실 수에 대한 가장 믿을 만한 정보가 보관**
        - 캐시가 주는 새로운 과제
            - 캐시 계층이 추가되면 시스템의 확장성과 처리량이 대폭 증가하지만 데이터베이스 ↔ 캐시 사이의 데이터 일관성 유지에 관한 새로운 과제
            - 사용자가 객실을 예약할 때 이루어지는 작업
                1. 잔여 객실 수를 질의하여 충분한지 확인 (캐시에서 실행)
                2. 잔여 객실 데이터를 갱신: 데이터베이스가 먼저 갱신 → 비동기적으로 캐시에 변경 내역 반영
                    - 애플리케이션이 수행할 때: 애플리케이션이 데이터베이스에 데이터를 저장한 다음 → 캐시데이터를 수정
                    - 변경 데이터 감지(Change Data Capture, CDC) 매커니즘을 활용: 데이터베이스에서 발생한 변화를 감지 → 다른  시스템ㅁ에 적용
                    - 드베지움(Debezium): 보편적으로 많이 사용되는 솔루션
                        - 소스 커넥터: 데이터베이스에서 발생한 변경 내역을 감지 → 레디스 등 캐시 시스템에 반영
        - 우선 캐시 질의로 잔여 객실 확인 → 이후 데이터베이스에 쓰기 요청 시 유효성 체크로 이중 체크
            - 장점
                - 읽기 질의를 캐시가 처리 → 데이터베이스의 부하가 대폭 줄어듬
                - 읽기 질의를 메모리에서 실행 → 높은 성능 보장
            - 단점
                - 데이터베이스↔캐시 사이의 데이터 일관성 유지가 어려움. 불일치가 사용자 경험에 어떤 영향을 끼치는지 따져보아야 함.

### MSA에서의 데이터 일관성 문제에 대한 해결 방안

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A9cdb09b1-e65b-45ac-bcb3-958188fea07a%3Aimage.png?table=block&id=31abe4ca-d9d2-8009-82e0-c16d344e5afc&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=990&userId=&cache=v2)

- 전통적인 모노리스 아키텍쳐: 데이터의 일관성을 보장하기 위해 관계형 데이터베이스를 공유하는 것이 기본.
- 마이크로 서비스 기반 아키텍처를 활용한 본 설계안: 하이브리드 접근법 채택
    - 예약서비스: 예약 및 잔여 객실 API를 모두 담당.
    - 예약 테이블과 잔여 객실 테이블을 동일한 관계형 데이터베이스에 저장
    
    ⇒관계형 데이터베이스의 ACID 속성을 활용, 동시성 문제를 효과적으로 처리 가능
    
- BUT **.** MSA 순수주의자: 각 마이크로서비스가 독자적인 데이터베이스를 갖추어야한다고 생각
    - ⇒ 교조주의적 접근은 다양한 데이터 일관성 문제가 존재
    - 모노리스 아키텍처의 경우
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A3d695dba-e0d9-4b03-9cf0-e131746c60b6%3Aimage.png?table=block&id=31abe4ca-d9d2-80b7-ac9c-f14b0a370e3f&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
        - 연산을 하나의 트랜잭션으로 묶어 ACID 속성 만족 보장
    - 각 서비스가 독자적인 데이터베이스르 사용
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ac8c79276-b2cd-453d-945a-b02bfa7576e3%3Aimage.png?table=block&id=31abe4ca-d9d2-80d3-9135-f8c01960ce36&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
        - 논리적으로는 하나의 원자적 연산이 여러 데이터베이스에 걸쳐 진행되는 일을 피할 수 없음 → 하나의 트랜잭션으로 데이터 일관성을 보증하는 기법을 사용할 수 없음
        - 해결 방법
            - 2단계 커밋(2-phase commit, 2PC)
                - 여러 노드에 걸친 원자적 트랜잭션을 보증하는 데이터베이스 프로토콜. 모든 노드가 성공 or 실패 → 둘 중 하나로 트랜잭션이 마무리되도록 보장
                - 단점: 비중단 실행이 불가능한 프로토콜, 어느 한 노드에 장애가 발생하면 장애가 복구될 때 까지 진행이 중단 → 성능이 뛰어나진 않음
            - 사가(Saga)
                - 각 노드에 국지적으로 발생하는 트랜잭션을 하나로 엮은 구조
                - 각각의 트랜잭션이 완료 → 다음 트랜잭션을 시작하는 트리거로 쓰일 메시지를 만들어 보냄
                - 어느 한 트랜잭션이라도 실패 → 이전 트랜잭션의 결과를 전부 되돌리는 트랜잭션을 순차적으로 실행
                - 결과적 일관성에 의존
        - 결과적으로 각서비스 당 각 데이터베이스를 배정하는 건 설계의 복잡성을 크게 증가시키므로 설계자는 가치 결정을 해야함.

---

## 추가 공부

### 멱등키

- **정의**: API 요청 중복 방지용 클라이언트 생성 고유키 (UUID 등)
- **용도**: POST/PUT 등에서 네트워크 오류 재시도 시 동일 작업 여러번 실행 방지. 몇 번을 호출에도 같은 결과를 내게하는 키
- **동작**

```
text요청: Idempotency-Key: abc123 → POST /reservations
서버:
  - 처음: 처리 → 결과 저장 (24h)
  - 재요청: 저장된 결과 즉시 반환
```

- **언제 사용?**
    
    
    | 메서드 | 멱등키 |
    | --- | --- |
    | GET | X |
    | POST | O (필수) |
    | PUT | △ |
    | DELETE | △ |
- **핵심**: 결제/주문 생성 API에서 이중처리 방지

### 아카이빙 vs 냉동저장소

- 데이터 생명주기 관리에서 장기 보관 단계
    
    
    | 항목 | 아카이빙 (Archiving) | 냉동 저장소 (Cold Storage) |
    | --- | --- | --- |
    | **목적** | 역사적/법적 기록 보존, 참조용 원본 이동 | 비활성 데이터 백업, 필요시 복구 |
    | **접근 속도** | 매우 느림 (시간~일 단위, 검색/복원 필요) | 중간 (분~시간 단위, 즉시 마운트 가능) |
    | **비용** | 최저가 (테이프: $0.01/GB, Glacier: $0.004/GB) | 저가 (S3 Glacier Deep: $0.00099/GB, 테이프) |
    | **데이터 형태** | 압축/암호화/인덱싱, 불변 저장 | 읽기 전용 스냅샷, 주기적 갱신 |
    | **용량** | 페타바이트 규모 가능 | 테라~엑사바이트 |
    | **복구 시간** | RPO: 주~월, RTO: 시간~일 | RPO: 일, RTO: 분~시간 |
    | **예시** | 완결 프로젝트 문서, 로그 아카이브 | DR 백업, 규제 준수용 DB 덤프 |
- 실제활용
    
    ```
    활성(Hot) → 온(Hot) → 냉동(Cold) → 아카이빙(Archive)
      DB     → 파일서버  → S3 IA     → Glacier Deep Archive
    ```
    
    **호텔 예약 DB**: 3년 지난 예약 데이터는 냉동 → 7년 후 아카이빙
    

## UUID

**전 세계적으로 고유한 식별자**(Universally Unique Identifier)

- **핵심 특징**
    - **크기**: 128비트 (32자리 16진수, `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`)
    - **형식**: `550e8400-e29b-41d4-a716-446655440000`
    - **중복 확률**: 사실상 0 (우주에 50억 년 동안 초당 10억 개 생성해도 안정적)
- **버전별 특징**
    
    
    | 버전 | 생성 방식 | 특징 |
    | --- | --- | --- |
    | v1 | 시간+MAC | 순서 보장, 프라이버시 취약 |
    | **v4** | **랜덤** | **가장 많이 사용** (멱등키 등) |
    | v5 | SHA-1 해시 | 이름 기반, 결정적 |
- **예시**
    
    ```
    호텔 예약 ID: 550e8400-e29b-41d4-a716-446655440000
    멱등키: f47ac10b-58cc-4372-a567-0e02b2c3d479
    파일명: room_image_550e8400.jpg
    ```
