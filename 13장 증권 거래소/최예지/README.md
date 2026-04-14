# (13) 증권 거래소

## 1단계: 문제 이해 및 설계 범위 확정

### 기능 요구사항

- 새로운 지정가 주문
- 체결되지 않은 주문 취소
- 시간 외 거래 지원X
- 주문 체결 후 실시간 알림
- 호가 창 정보 실시간 갱신
- 최소 수만명의 거래자, 최소 100가지 주식 거래 가능
- 위험성 점검 기능 지원
- 사용자 지갑 관리 (주문 전 충분한 자금 확인, 체결 되지 않은 주문에 사용된 자금은 다른 주문에 사용X)

### 비기능 요구사항

- 가용성: 최소 99.99% 거래소의 가용성. 단 몇초의 장애로도 평판이 손상될 수 있다.
- 결함 내성: 프로덕션 장애의 파급을 줄이려면 결함 내성과 빠른 복구 매커니즘이 필요
- 지연 시간: 왕복 지연 시간(주문이 거래소에 들어오는 순간→ 주문의 체결 사실이 반환되는 시점까지) 은 밀리초 수준이어야 함. p99(99th 백분위수) 지연 시간이 중요.
- 보안: 새 계좌 개설 전 사용자 신원 확인을 위한 KYC(Know Your Client) 확인을 수행, 시장 데이터가 포함된 웹 페이지 등의 공개자원은 DDos 공격을 방지하는 장치가 필요

### 개략적 규모 추정

- 100가지 주식
- 하루 10억 건의 주문
- 뉴옥증권거래소는 월~금. 9:30 ~ 14:00(EST) 영업. ⇒ 청 6.5시간
- QPS: 10억 / 6.5시간 * 3600  =~ 43,000
- 최대 QPS: 5 * QPS = 215,000 거래량은 장 시작 직후, 장 마감 직전에 몰림

## 2단계: 개략적 설계안 제시 및 동의 구하기

### 증권 거래 101

- 브로커
    - 대부분의 개인 고객은 브로커 시스템을 토해 거래소와 거래, 개인 사용자가 증권을 거래, 시장 데이터를 확인할 수 있도록 편리한 사용자 인터페이스 제공
- 기관 고객
    - 전문 증권 거래 소프트웨어를 사용하여 대량으로 거래
    - 기관 고객 당 거래 시스템에 대한 요구사항
        - 연기금 :안정적 수익 목표. 낮은 거래 빈도. 많은 거래량 ⇒ 대규모 주문이 시장에 미치는 영향을 최소화 하기 위해 주문 분할 등의 기능 필요
        - 헤지 펀드: 시장 조성, 수수료 리베이트로 수익 창출 ⇒ 낮은 응답시간 요구 (모바일 앱, 웹페이지 등에서 데이터 제공 비추)
- 지정가 주문
    - 가격이 고정된 매수/매도 주문. 시장가 주문과 달리 즉시체결X, 부분체결 가능성 O
- 시장가 주문
    - 가격 지정X 주문. 시장가로 즉시 체결 됨. 체결은 보장되나 비용 면에서 손해가 있을 수 있음. 급변하는 특정 시장 상황에서 유용.
- 시장 데이터 수준
    - 미국 주식시장엔 L1, L2, L3 세가지 가격 정보 등급 존재.
    - L1
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A09311d61-5aae-46ed-96cb-00bab7dca5c9%3Aimage.png?table=block&id=340be4ca-d9d2-8084-b775-ec371abc8178&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=640&userId=&cache=v2)
        
        - 최고 매수 호가: 구매바가 주식에 지불할 의사가 있는 최고 가격
        - 최저 매도 호가: 매도자가 주식을 팔고자 하는 최저 가격
    - L2
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A673213b1-2d69-4d1a-bf85-f6e6c2c4a15c%3Aimage.png?table=block&id=340be4ca-d9d2-807e-baee-ffac215ca4ee&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
        - 깊이: 체결을 기다리는 물량의 호가
    - L3
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A4262931e-341f-4402-ac81-c0c3dd0d4016%3Aimage.png?table=block&id=340be4ca-d9d2-80bd-a594-e8a9c1a6ee46&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
        - 각 주문 가격에 체결을 기다리는 물량 정보 포함
- 봉 차트
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A306938bb-f508-4e3b-9a23-cf483cbc6e78%3Aimage.png?table=block&id=340be4ca-d9d2-8040-a30d-c84c4cd39f66&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    - 특정 기간 동안의 주가
    - 봉(캔들) 막대로 일정 시간 간격 동안 시장의 시작가, 종가, 최고가, 최저가를 표시
    - 일반적으로 지원되는 시간 간격: 1분, 5분, 1시간, 1일, 1주일, 1개월
- FIX (Financial information Exchange Protocol, 금융 정보 교환 프로토콜)
    
    ```
    // FIX로 인코딩한 증권 거래의 사례
    8=FIX.4.2 | 9=176 | 35=8 | 49=PHLX | 56=PERS | 52=20071123-05:30:00.000 | 11=ATOMN0CCC9990900 | 20=3 | 150=E | 39=E | 55=MSFT | 167=CS | 54=1 | 38=15 | 40=2 | 44=15 | 58=PHLX EQUITY TESTING | 59=0 | 47=C | 32=0 | 31=0 | 151=15 | 14=0 | 6=0 | 10=128
    ```
    
    - 증권 거래 정보 교환을 위한 기업 중립적 통신 프로토콜

### 개략적 설계안

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A6e2186ea-4254-419f-ad10-c2fcf7a4434e%3Aimage.png?table=block&id=340be4ca-d9d2-8078-8707-c387df320997&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=940&userId=&cache=v2)

- 거래 흐름(trading flow)
    - 지연 시간 요건이 엄격한, 중요 경로(critical path). 이 경로를 따라 흐르는 모든 정보는 신속하게 처리되어야 함.
    
    1단계) 고객이 브로커의 웹 또는 모바일 앱을 통해 주문
    
    2단계) 브로커가 주문을 거래소에 전송
    
    3단계) 주문이 클라이언트 게이트웨이를 통해 거래소로 이송, 클라이언트 게이트웨이는 입력 유효성 검사, 속도 제한, 인증, 정규화 등과 같은 기본적인 게이트키핑 기능을 수행 → 이후 주문을 주문 관리자에 전달
    
    4~5단계) 주문 관리자가 위험 관리자가 설정한 규칙에 따라 위험성 점검을 수행
    
    6단계) 위험성 점검 과정을 통과한 주문에 대해, 주문 관리자는 지갑에 주문 처리 자금이 충분한지 확인
    
    7~9단계) 주문이 체결 엔진으로 전송, 체결 가능 주문이 발견되면 체결 엔진은 매수 측과 매도 측에 각각 하나씩 두 개의 집행(execution, Fill)기록을 생성. 나중에 그 과정을 재생할 때 항상 결정론적으로 동일한 결과가 나오도록 보장하기 위해 시퀀서는 주문 및 집행 기록을 일정 순서로 정렬
    
    10~14단계) 주문 집행 사실을 클라이언트에 전송
    
- 시장 데이터 흐름(market data flow)
    - M1 단계: 체결 엔진: 주문이 체결되면 집행 기록 스트림(또는 충족 스트림)을 만듬.이 스트림은 시장 데이터 게시 서비스로 전송됨
    - M2 단계: 시장 데이터 게시 서비스: 집행 기록 및 주문 스트림에서 얻은 데이터를 시장 데이터로 사용하여 봉 차트와 호가 창을 구성. 이후 시장 데이터를 데이터 서비스로 전송
    - M3 단계: 시장 데이터가 실시간 분석 전용 스토리지에 저장, 브로커는 데이터 서비스를 통해 실시간 시장 데이터를 읽음. 브로커는 이 시장 데이터를 고객에게 전달
- 보고 흐름(report flow)
    - R1~R2 단계(보고 흐름): 보고 서비스. 주문 및 실행 기록에서 보고에 필요한 모든 필드의 값을 모은 다음(ex. client_id, price, quantity, order_type, filled_quantity, remaining_quantity 등) 그 값을 종합해서 만든 레코드를 데이터 베이스에 기록
- 거래 흐름은 중요 경로를 따라 진행. 시장 데이터 & 보고 흐름은 지연 시간 요구사항이 다름

### 거래 흐름

- 거래소의 중요 경로 상에서 진행 됨. 모든 것은 신속하게 진행되어야함. 거래흐름의 핵심은 체결 엔진
- 체결 엔진(= 교차 엔진)
    - 체결 엔진의 주요 역할
        1. 각 주식 심벌에 대한 주문서(order book) 내지 호가 창을 유지 관리. 주문서 또는 호가 창: 특정 주식에 대한 매수 및 매도 주문 목록
        2. 매수 주문과 매도 주문을 연결. = 주문 체결 결과로 두 개의 집행 기록이 만들어짐(매수 쪽에서 1건, 매도 쪽에서 1건) 체결은 신속하고 빠르게 처리되어야 함.
        3. 집행 기록 스트림을 시장 데이터로 배포
    - 가용성 높은 체결 엔진 구현체가 만드는 체결 순서는 결정론적이어야 함. 즉, 입력으로 주어지는 주문 순서가 같으면, 체결 엔진이 만드는 집행 기록 순서는 언제나 동일해야 함. ⇒ 이러한 결정론적 특성이 고가용성의 토대가 됨
- 시퀀서
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A591ad0ff-0f04-4a8b-8d5f-90e3c390e689%3Aimage.png?table=block&id=340be4ca-d9d2-80e4-87d0-ea5ac6203b67&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    - 체결 엔진을 결정론적으로 만드는 핵심 구성 요소
    - 체결 엔진에 주문을 전달하기 전에 순서 ID(sequence ID)를 붙여 보냄(입력 시퀀서), 체결 엔진이 처리를 끝낸 모든 집행 기록 쌍에도 순서 ID를 붙임(출력 시퀀서).
    - 입력 시퀀서, 출력 시퀀서 각각 고유한 순서를 유지.
    - 시퀀서가 만드는 순서 ID는 누락된 항목을 쉽게 발견할 수 있는 일련번호여야 함.
    - 순서 ID를 찍는 이유
        1. 시의성(timeliness) 및 공정성(fairness)
        2. 빠른 복구(recovery) 및 재생(replay)
        3. 정확한 1회 실행 보증(exactly-once guarantee)
    - 메시지 큐의 역할도 수행
        1. 체결엔진에 메시지(수신된 주문)를 보내는 큐 역할
        2. 주문 관리자에게 메시지(집행 기록)를 회신하는 큐 역할
    - 주문과 집행 기록을 위한 이벤트 저장소. 체결 엔진에 두 개 카프카 이벤트 스트림이 연결되어 있는 효과
        1. 입력되는 주문용
        2. 출력될 집행 기록용
           

### 주문 관리자

- 한쪽에서는 주문을 받고, 다른 쪽에서는 집행 기록을 받음. 주문상태를 관리하는게 주문 관리자의 역할.
- 주문 관리자가 클라이언트 게이트웨이를 통해 주문을 수신하고 하는 일
    - 종합적 위험 점검 담당 컴포넌트에 주문을 보내 위험성을 검토. 설계안이 지원해야하는 위험 점검 규칙: 사용자의 거래량이 하루 100만 달러 미만인지 확인하는 수준
    - 사용자의 지갑에 거래를 처리하기에 충분한 자금이 있는지 확인.
    - 주문을 시퀀서에 전달. 시퀀서는 해당 주문에 순서 ID를 찍고 체결 엔진에 보내어 처리. 새 주문에는 많은 속성이 있지만 체결엔진에 보낼 속성을 선별하여 전송하여 메시지 크기를 줄임
- 주문 관리자는 시퀀서를 통해 엔진으로부터 집행기록을 수신 → 체결된 주문에 대한 집행 기록을 클라이언트 게이트웨이를 통해 브로커에 반환
- 주문 관리자는 빠르고, 효율적이며, 정확해야 함.
- 주문의 형재상태를 유지 관리. → 다양한 상태변화를 관리해야 함. (복잡한 구현) ⇒ 이벤트 소싱 기반 설계 추천

### 클라이언트 게이트 웨이

- 거래소의 문지기. 클라이언트로부터 주문을 받아 주문 관리자에게 전송
- 클라이언트 게이트 웨이의 구성요소
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aebf2cf05-200e-4282-a86c-726a0a2bf7e9%3Aimage.png?table=block&id=340be4ca-d9d2-8020-81c4-d0c60cc03d8d&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=740&userId=&cache=v2)
    
    - 중요 경로상에 놓이며, 지연 시간에 민감. 가벼워서 가능한 빨리 목적지로 주문을 전달해야 함.
    - 어떤 기능을 클라이언트 게이트웨이에 넣을지 말지를 타협적으로 생각해야 함. → 복잡한 기능이라면 체결 엔진이나 위험 점검 컴포넌트에 맡기는 것을 권장.
- 고객 유형별(개인 고객/기관 고객)로 클라이언트 게이트웨이가 다양함.
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A9b7b06f5-ce74-453f-94ae-5129d9776c09%3Aimage.png?table=block&id=340be4ca-d9d2-80a5-afdb-dfbd46c600ad&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    - 주요 고려 사항: 지연 시간, 거래량, 보안 요구사항

### 시장 데이터 흐름

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A8e9f06d3-e261-4f89-8a1b-ccb674fc88a0%3Aimage.png?table=block&id=340be4ca-d9d2-803a-b292-c82a77b19eb7&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=940&userId=&cache=v2)

- 시장 데이터: 호가 창과 봉 차트를 통칭
- 시장 데이터 게시 서비스(Market Data Publisher, MDP): 체결 엔진에서 집행 기록을 수신, 집행 기록 스트림에 호가 창과 봉 차트를 만들어 냄.
- 시장 데이터: 데이터 서비스로 전송되어 해당 서비스의 구독자(publisher)가 사용할 수 있게 됨.

### 보고 흐름

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A9eeca750-7f30-4bab-b4b0-06a8434e67e0%3Aimage.png?table=block&id=340be4ca-d9d2-8024-9d5f-e4afea862098&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=940&userId=&cache=v2)

- 거래의 중요 경로상에 있지는 않지만 시스템의 중요한 부분
- 거래 이력, 세금 보고, 규정 준수 여부 보고, 결산 등의 기능을 제공
- 보고는 효율성, 짧은 지연 시간보단 **정확성과 규정 준수가 핵심**
- 들어오는 새 주문: 주문 세부 정보
- 나가는 집행 기록: 주문 ID, 가격, 수량 및 집행 상태 정보
- 입력으로 들어오는 주문, 그 결과로 나아가는 집행 기록 모두에서 정보를 모야 속성들을 구성 → 보고서를 만드는 것이 일반적 관행

### API 설계

- 고객: 브로커를 통해 증권 거래소와 상호 작용하여 주문, 체결 조회, 시장 데이터 조회, 분석을 위한 과거 데이터 다운로드 등을 수행
- 브로커 ↔ 클라이언트 게이트웨이 간의 인터페이스 명세 작성에는 RESTful 컨벤션을 사용
- 헤지 펀드 등 기관 고객의 지연 시간 요구사항을 충족시키기 위해 공급되는 특수 소프트웨어에서는 다른 프로토콜을 사용할 수 있음. BUT 기본 기능은 동일하게 제공
- 주문
    - POST /v1/order
    
    주문을 처리. 인증이 필요
    
    - 인자
        - symbol(String): 주식을 나타내는 심벌
        - side(String): buy(매수) 또는 sell(매도)
        - price(Long): 지정가 주문의 가격
        - orderType(String): limit(지정가) 또는 market(시장가).
            - 이번장에서는 지정가만 지원
        - quantity(Long): 주문 수량
    - 응답
        - 본문(body)
            - id(Long): 주문 ID
            - creationTime(Long): 주문이 시스템에 생성된 시간
            - filledQuantity(Long): 집행이 완료된 수량
            - remainingQuantity(Long): 아직 체결되지 않은 주문 수량
            - status(Stirng): new/canceled/filled
            - 이하 입력 인자와 동일
    - 코드
        - 200: 성공
        - 40x: 인자 오류/접근 불가/권한 없음
        - 500: 서버 오류
- 집행
    - GET /v1/execution?symbol={:symbol}&orderId={:orderId}&startTime={:startTime}&endTime={:endTime}
    
    집행 정보를 질의, 인증이 필요
    
    - 인자
        - symbol: 주식 심벌
        - orderId(String, optional): 주문의 ID
        - startTime(Long): 질의 시작 시간. 기원시간(epoh) 기준.
        - endTime(Long): 질의 종료 시간. 기원 시간 기준.
    - 응답
        - 본문:
            - executions(Array): 범위 내의 모든 집행 기록의 배열
            - id(Long): 집행 기록 ID
            - orderId(Long): 주문 ID
            - symbol(String): 주식 심벌
            - side(String): buy(매수) 또는 sell(매도)
            - price(Long): 체결 가격
            - orderType(String): limit(지정가) 또는 market(시장가)
            - quantity(Long): 체결 수량
    - 코드
        - 200: 성공
        - 40x: 인자 오류/해당 자원 없음/접근 불가/권한 없음
        - 500: 서버 오류
- 호가 창/주문서
    - GET /v1/marketdata/orderBook/L2?symbol={:symbol}&depth={:depth}
    
    주어진 주식 심벌, 주어진 깊이 값에 대한 L2 호가 창 질의 결과를 반환
    
    - 인자
        - symbol: 주식 심벌
        - depth(Int): 반환할 창의 호가 깊이
        - startTime(Long): 질의 시작 시간. 기원시간(epoh) 기준.
        - endTime(Long): 질의 종료 시간. 기원 시간 기준.
    - 응답
        - 본문
            - bids(Array): 가격과 수량 정보를 담은 배열.
            - asks(Array): 가격과 수량 정보를 담은 배열
    - 코드
        - 200: 성공
        - 40x: 인자 오류/해당 자원 없음/접근 불가/권한 없음
        - 500: 서버 오류

### 데이터 모델

- 상품, 주문, 집행
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A7532b24b-2105-4c10-8fae-fc93266ad45f%3Aimage.png?table=block&id=340be4ca-d9d2-80c4-a3d1-f90206574785&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    - 상품
        - 거래 대상 주식(=심벌)이 가진 속성으로 정의
        - 상품의 유형, 거래에 쓰이는 심벌, UI에 표시될 심벌, 결산에 이용되는 통화 단위, 매매 수량 단위(lot size), 호가 가격 단위(tick size) 등
        - 자주 변경되지 않는 데이터로 주로 UI표시를 위한 데이터
        - 아무 데이터베이스에나 저장 가능, 캐시 적용하기 좋음
    - 주문
        - 매수 또는 매도를 실행하라는 명령
    - 집행 기록(execution)
        - 체결이 이루어진 결과.
        - 충족(fill)이라고도 부름
        - 모든 주문이 집행되지는 않음.
        - 체결 엔진은 하나의 주문 체결에 관여한 매수 행위와 매도 행위를 나타내는 두 개의 집행 기록을 결과로 출력ㅇ유
    - 주문&집행 기록은 거래소가 취급하는 가장 중요한 데이터.
        - 중요 거래 경로는 주문과 집행 기록을 데이터베이스에 저장X. 성능을 높이기 위해 메모리에서 거래를 체결 → 하드디스크나 공유 메모리를 활용하여 주문과 집행 기록을 저장 및 공유. 빠른 복구를 위해 시퀀서에 저장, 데이터 보관은 장 마감 이후에 실행 함.
        - 보고 서비스는 조정이나 세금 보고 등을 위해 데이터베이스에 주문 및 집행 기록을 저장.
        - 집행 기록은 시장 데이터 프로세서로 전달되어 호가 창/주문서와 봉 차트 데이터 재구성에  쓰임
- 호가 창
    
    특정 증권 또는 금융 상품에 대한 매수 및 매도 주문 목록. 가격 수준별로 정리되어 있음. 체결 엔진이 빠른 주문 체결을 위해 사용하는 핵심 자료 구조.
    
    - 호가 창의 자료 구조는 다음 요구 사항을 만족할 수 있는, 효율 성이 높은 것이어야 함.
        - 일정한 조회 시간: 특정 가격 수준의 주문량 조회, 특정 가격 범위 내의 주문량 조회 등이 포함
        - 빠른 추가/취소/실행 속도: 가급적 O(1) 시간 복잡도를 만족해야 함. 새 주문 넣기, 기존 주문 취소하기, 주문 체결하기 등이 포함
        - 빠른 업데이트: 주문 교체 등 포함
        - 최고 매수 호가/최저 매도 호가 질의
        - 가격 수준 순회(iteration)
    - 호가 창 예제
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aa7e3bbcf-65df-4319-8f75-80f30ebb8082%3Aimage.png?table=block&id=340be4ca-d9d2-808b-8443-d229a40309fb&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
        위 예제에서 애플 주식 2,700주에 대한 대량 시장가 매수 주문의 처리 흐름
        
        - 최저 매도 호가 큐의 모든 매도 주문과 체결된 후에 호가 100.11 큐의 첫번째 매도 주문과 체결되며 거래가 끝남.
        - 이 대량 주문의 체결 결과로 매수/매도 호가 스프레드, 즉 둘 간의 가격 차이가 넓어지고 주식 가격이 한 단계 상승(= 최저 매도 호가가 100.11로 변경)
    
    ```java
    class PriceLevel {
        private Price limitPrice;
        private long totalVolume;
        private List<Order> orders;
    }
    
    class Book<Side> {
        private Side side;
        private Map<Price, PriceLevel> limitMap;
    }
    
    class OrderBook {
        private Book<Buy> buyBook;
        private Book<Sell> sellBook;
        private PriceLevel bestBid;
        private PriceLevel bestOffer;
        private Map<OrderID, Order> orderMap;
    }
    ```
    
    - 지정가 주문을 추가/취소 하는 시간 복잡도가 O(1)인가?: NO
        - 연결 리스트(private List<Order> orders)를 사용하고 있어 불가
        - 보다 효율적인 호가 창을 만들려면 orders의 자료구조를 이중 연결리스트로 변경하여 모든 삭제 연산(주문 취소나 체결 처리에 필요)이 O(1)에 처리되도록 해야 함.
        - 시간 복잡도가 O(1)가 되는 순서
            
            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A13a16fb1-28ba-4629-8015-b8a4d4d59086%3Aimage.png?table=block&id=342be4ca-d9d2-80bd-ba34-d886d156cfcb&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=710&userId=&cache=v2)
            
            1. 새 주문을 넣는다: PriceLevel 리스트 마지막(tail)에 새 Order을 추가하는 것을 의미. → 이중 연결 리스트에서 이 연산은 O(1)에 처리 됨.
            2. 주문을 체결한다: PriceLevel 리스트의 맨 앞(head)에 있는 Order를 삭제한다. → 이중 연결 리스트의 경우 시간 복잡도 O(1)
            3. 주문을 취소한다: 호가 창(OrderBook)에서 Order를 삭제한다는 뜻. OrderBook에 포함되어 있는 도움(helper) 자료 구조 Map<OrderID, Order> orderMap을 활용하면 이중 연결 리스트를 사용하면 Order안에 이전 주문을 가리키는 포인터가 있어 O(1) 시간 내에  주문을 취소할 수 있음. 
- 봉 차트
    - 시장 데이터 프로세서가 시장 데이터를 만들 때 호가창과 더불어 사용하는 핵심 자료 구조
    - 봉 차트를 모델링하기 위해 Candlestick 클래스와 CandlestickChart 클래스를 사용. 하나의 봉이 커버하는 시간 범위가 경과 → 다음 주기를 커버할 새 Candlestick 클래스 객체를 생성하여 CandlestickChart 객체 내부 연결 리스트에 추가
        
        ```java
        class Candlestick {
            private long openPrice;
            private long closePrice;
            private long highPrice;
            private long lowPrice;
            private long volume;
            private long timestamp;
            private int interval;
        }
        
        class CandlestickChart {
            private LinkedList<Candlestick> sticks;
        }
        ```
        
    - 봉 차트에서 많은 종목의 가격 이력을 다양한 시간 간격을 사용해 추적하려면 메모리가 많이 필요 → 최적화할 방법 두가지
        1. 미리 메모리를 할당해 둔 링(ring) 버퍼에 봉을 보관하여 새 객체 할당 횟수를 줄임
        2. 메모리에 두는 봉의 개수를 제한, 나머지는 디스크에 보관
    - 시장데이터는 일반적으로 실시간 분석을 위해 메모리 상주 칼럼형 데이터베이스(ex. KDB)에 보관. → 시장이 마감된 후 이력 유지 전용 DB에도 저장

## 3단계: 상세 설계

일부 대형 거래소는 하나의 거대 서버로 모든 것을 운영.

### 성능

- 지연 시간은 거래소에 아주 중요한 문제. 평균 지연 시간은 낮고, 전반적인 지연 시간 분포도는 안정적이어야 함.
- 전반적인 지연 시간이 안정적인지 보는 척도: p99(99% 백분위수) 지연시간
- 지연 시간을 구성 요소별로 분할하는 공식
    
    $$
    지연시간 = \sum 중요 경로상의 컴포넌트 실행시간
    $$
    
- 지연 시간을 줄이는 두 가지 방법
    1. 중요 경로에서 실행할 작업 수를 줄인다.
        - 중요 매매 경로에 포함되는 컴포넌트:
            
            게이트웨이 → 주문관리자 → 시퀀서 → 체결 엔진
            
        - 중요 경로에는 꼭 필요한 구성 요소만 둠. 로깅도 지연 시간을 줄이기 위해 경로에서 제거
    2. 각 작업의 소요 시간을 줄인다.
        1. 네트워크 및 디스크 사용량 경감
        2. 각 작업의 실행 시간 경감
        - 핵심 경로의 구성요소: 네트워크를 통해 연결된 개별 서버에서 실행. 왕복 네트워크 지연 시간은 약 500마이크로초 ⇒ 핵심 경로에 네트워크를 통해 통신하는 컴포넌트가 많으면 총 네트워크 지연 시간이 한 자릿수 밀리초까지 늘어날 수 있음.
        - 시퀀서: 이벤트를 디스크에 저장하는 이벤트 저장소 → 순차적 쓰기 성능의 이점을 이용한다 해도 디스크 액세스 지연 시간: 수십 밀리초 단위.
        - 네트워크 및 디스크 액세스 지연 시간을 모두 고려하면 총 단대단(end-to-end) 지연 시간이 수십 밀리초에 달함.  → 현재 시점에서는 만족스럽지 않은 지연 속도
        - ⇒ 모든 것을 단일서버에 구현하여 네트워크를 통하는 구간을 제거하여 지연시간 개선. 같은 서버 내 컴포넌트 간 통신은 이벤트 저장소인 mmap을 통함.
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A62380b54-6d32-4d70-9d8b-df364fff10bc%3Aimage.png?table=block&id=342be4ca-d9d2-806b-a983-cc3fdfb9b622&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
        - 애플리케이션 루프: while 순환문을 통해 실행할 작업을 계속 폴링(polling). 목적 달성에 가장 중요한 작업만 순환문 안에서 처리해야 함.
        - 각 구성 요소의 실행 시간을 줄여 전체적인 실행 시간이 예측가능하도록 보장하는 것.
        - 다이어그램의 각 상자: 컴포넌트(서버의 프로세스)
        - CPU 효율성을 극대화 하기 위해 애플리케이션 루프(주 처리 루프)는 단일 스레드로 구현 & 특정 CPU 코어에 고정.
            - 주문관리자 컴포넌트의 예시
                
                ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A79c1c246-bd76-409f-9d58-f44539553bd1%3Aimage.png?table=block&id=342be4ca-d9d2-805e-b6f4-f63d25ebfb6c&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=520&userId=&cache=v2)
                
        - 애플리케이션 루프를 CPU에 고정할 때의 이점
            1. 문맥 전환(context switch)가 없음. → CPU1이 주문 관리자의 애플리케이션 루프 처리에 온전히 할당 됨
            2. 상태를 업데이트 하는 스레드가 하나 뿐 → 락을 사용할 필요 x, 잠금 경합x
        - CPU를 고정하는 방법의 단점: 코딩이 복잡해짐 → 각 작업이 애플리케이션 루프 스레드를 너무 오래 점유하지 않도록 각 작업에 걸리는 시간을 신중하게 분석해야 함.
- mmap
    - 파일을 프로세스의 메모리에 매핑하는 mmap(2)라는 이름의 POSIX 호환 UNIX 시스템 콜(system call)
    - 프로세스 간 공성능 메모리 공유 매커니즘을 제공.
    - 메모리에 매핑할 파일이 /dev/shm에 있을 때 성능 이점이 더욱 커짐
    - /dev/shm: 메모리 기반 파일 시스템. 이곳에 위치한 파일에 mmap(2)을 수행하면 공유 메모리에 접근해도 디스크 I/O가 발생하지 않음.
    - 최신 거래소는 이를 활용하여 중요 경로에서 가능한 한 디스크 접근이 일어나지 않도록 함. ⇒ mmap(2)를 사용하여 중요 경로에 놓인 구성 요소가 서로 통신할 때: 메시지 버스를 구현 → 네트워크나 디스크에 접근X. 메시지 전송에 마이크로 초 미만 소요.
- 이벤트 소싱
    - 전통적 애플리케이션은 상태를 데이터베이스에 유지

---
