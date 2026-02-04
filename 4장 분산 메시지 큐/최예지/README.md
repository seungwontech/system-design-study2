# (4) 분산 메시지 큐

### 메세지 큐를 사용하는 장점

- 결합도 완화
    - 컴포넌트 간의 강한 결합 제거 ⇒ 독립적 갱신 가능
- 규모 확장성 개선
    - 생산자(producer) ↔ 소비자(consumer) ← 시스템 규모를 독립적으로 개선 가능
- 가용성 개선
    - 컴포넌트 장애 발생 시 타 컴포넌트는 여전히 사용 가능
- 성능 개선
    - 쉬운 비동기 통신

### 분산 메시지 큐 시스템 종류

**메시지 큐 ↔ 이벤트 스트리밍 플랫폼** 간의 경계가 점차 희미해짐 (지원하는 기능이 점차 수렴)

- 이벤트 스트리밍 플랫폼
    - 아파치 Kafka, 아파치 Pulsar
    - 데이터 장기보관, 메시지 반복 소비 등의 부가 기능 지원
- 메시지큐
    - 아파치 RocketMQ, 아파치 RabbitMQ, 아파치 ActiveMQ, ZeroMQ
    - 전통적인 메시지큐:  메시지 반복 소비 지원x, 소비순서 지원x, 메시지 지속성 보관 보증x

## 1단계: 문제 이해 및 설계 범위 확정

**메시지 큐**: 생산자 > [메시지 큐] > 소비자

### 기능 요구사항

- 생산자: 메시지 큐에 메시지 발행
- 소비자: 메시지 큐에서 메시지 수신
- 메시지 반복 소비 지원
- 데이터 보관 지원. 오래된 이력 데이터(2주)는 삭제 가능
- 메시지크기: nKB
- 메시지 소비순서 지원
- 메시지 전달방식: 최소 한 번, 최대 한번, 정확히 한번 ←  설정가능

### 비기능 요구사항

- 높은 대역폭과 낮은 전송 지연 중 선택 가능
- 규모 확장성 (분산 시스템)
- 지속성 및 내구성

## 2단계: 개략적 설계안 제시 및 동의 구하기

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A527d71ec-3feb-4dfe-8328-66eb575bc103%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8003-8cca-e7a133b78cd4&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=990&userId=&cache=v2)

- 생산자는 메시지를 메시지 큐에 발행
- 소비자는 큐를 구독 & 구독한 메시지 소비
- 메시지 큐: 생산자 ↔ 소비자 결합 느슨하게 ⇒ 생산자, 소비자는 독립적인 운영 및 규모 확장 가능
- 생산자, 소비자: 클라이언트
메시지 큐: 서버

### 메시지 모델

- 일대일 모델
    - 발행 된 메시지는 오로지 한 소비자만 가능
    - 특정 소비자가 메시지를 가져가면 메시지는 메시지큐에서 삭제 됨
- 발행-구독(publish-subscribe) 모델
    - 토픽: 메시지를 주제별로 정리하는데 사용. 토픽에 전달 된 메시지는 해당 토픽을 구독하는 모든 소비자에게 전달

### 토픽, 파티션, 브로커, 소비자 그룹

- 토픽
    - 메시지는 토픽에 보관
- 파티션
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A947fb1d2-1694-4e65-9802-4e866f14d133%3Aimage.png?table=block&id=2fdbe4ca-d9d2-805c-858d-f9a385bdae71&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - 샤딩 기법. 파티션을 메시지 큐 클러스터 내의 서버에 고르게 분산 배치
    - 토픽을 여러 파티션으로 분할 & 메시지를 모든 파티션에 균등하게 전송
    - 각 토픽 파티션은 FIFO 큐처럼 동작 = 같은 파티션 안에서는 메시지의 순서(offset)가 유지
    - 같은 키를 가진 모든 메시지는 같은 파티션으로 전송 됨
    - 키가 없는 메시지는 무작위 파티션으로 전송됨
- 브로커
    - 파티션을 유지하는 서버
- ❓ 소비자 그룹
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A1d705cb0-8513-4f09-b9e8-6ce145b130a4%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8022-867e-fb186a2de01e&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - 토픽 파티션 → 그룹 내 소비자들이 분담 소비
    - 하나의 소비자 그룹은 여러 토픽을 구독 가능. → 오프셋을 별도로 관리
    - 소비자 그룹 1은 토픽 A를 구독
    - 소비자 그룹 2는 토픽A와 토픽 B를 구독
    - 토픽 A의 메시지 → 소비자 그룹 1과 소비자 그룹 2 내의 소비자에게 전달 (발행-구독 모델 지원)
    - 단점: 데이터 병렬 소비로 파티션 내의 메시지를 순서대로 소비 불가
        - 제약사항 추가로 해결: 파티션의 메시지는 한 그룹 안에서는 오직 한 소비자만 읽을 수 있도록 제한

### 개략적 설계안

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A6a310120-9131-4733-8d57-801ca9103cb0%3Aimage.png?table=block&id=2fdbe4ca-d9d2-801e-875f-cf9a5f937d08&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=990&userId=&cache=v2)

- 클라이언트
    - 생산자
    - 소비자 그룹
- 핵심 서비스 및 저장소
    - 브로커: 파티션을 유지
        - 하나의 파티션은 특정 토픽에 대한 메시지의 부분 집합 유지
    - 저장소
        - 데이터 저장소: 메시지를 파티션 내 저장소에 저장
        - 상태 저장소: 소비자 상태 저장
        - 메타데이터 저장소: 토픽 설정, 토픽 속성 등
    - 조정 서비스
        - 서비스 탐색: 생존 브로커 확인
        - 리더 선출: 브로커 중 하나는 컨트롤러를 담당. 한 클러스터에는 반드시 활성 상태 컨트롤러가 하나 있어야 함.
            - 아파치 주키퍼 or etcd가 선출 컴포넌트로 이용 됨

## 3단계: 상세 설계

### 목표

- 회전 디스크 구조 활용
- 아무 수정 없이도 전송이 가능하도록 하는 메시지 자료구조
- 일괄 처리를 우선하는 시스템

### 데이터 저장소

- 메시지 큐의 트래픽 패턴
    - 읽기와 쓰기가 빈번
    - 대부분 순차적인 읽기/쓰기
    - 갱신/삭제 연산 x
- 선택지1 데이터베이스
    - RDBMS: 토픽별로 테이블 생성. 메시지 = 레코드
    - NoSQL: 토픽별로 컬렉션 생성. 메시지 = 하나의 문서
    - 문제점: 많은 읽기/쓰기 연산 감당이 어려움
- 선택지2: 쓰기 우선 로그 (Write-Ahead log, WAL)
    - 새로운 항목이 추가되기만 하는 일반 파일, 읽기/쓰기 모두 순차적
    - 지속성을 보장해야하는 메시지의 경우 추천
    - 회전식 디스크 기반 저장장치: 저렴함
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A03d27a80-6774-4891-83cb-f0c27c5f3d59%3Aimage.png?table=block&id=2fdbe4ca-d9d2-802b-8100-fabc1c17cf55&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
    - 새로운 메시지는 파티션 꼬리부분에 추가. 오프셋은 그 결과로 점진적 증가
    - 파일의 크기는 세그먼트 단위로 나뉘어 관리.
        - 활성 상태의 세그먼트 파일에만 메시지 추가
        - 세그먼트의 크기가 한계에 도달 → 새 활성 세그먼트 파일 생성 후 비활성
        - 낡은 비활성 세그먼트 파일은 (1) 보관기한이 만료 혹은 (2)용량 한계에 도달 시 삭제가능
        - 같은 파티션에 속한 세그먼트 파일은 Partition-{:partition_id} 폴더 아래에 저장
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A744106d4-0e17-44bc-b974-ec1d598e715b%3Aimage.png?table=block&id=2fdbe4ca-d9d2-808f-a180-f5ac579ff734&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)<img width="628" height="272" alt="image" src="https://github.com/user-attachments/assets/1c5d51be-9aff-46cb-a116-007fd2fe10d4" />

        
- 디스크 성능 관련
    - 순차적 데이터 접근 패턴을 적극 활용하는 디스크 기반 자료구조 활용 → RAID로 구성된 현대적 디스크 드라이브: 수백MB/sec 수준의 읽기/쓰기 성능 달성 가능
    - OS에서 제공하는 디스크 캐시 기능을 적극적으로 활용

### 메시지 자료 구조

- 생산자-메시지 큐-소비자 사이의 계약
- 메시지가 큐를 거쳐 소비자에게 전달되는 과정에서 불필요한 복사가 일어나지 않게 설계 → 높은 대역폭 달성
- 메시지 자료 구조의 스키마 사례
    
    
    | 필드 | 데이터 타입 |
    | --- | --- |
    | key | byte[] |
    | value | byte[] |
    | topic | string |
    | partition | integer |
    | offset | long |
    | timestamp | long |
    | size | integer |
    | crc | integer |
- 메시지 키
    - 파티션을 정할 때 사용
    - 키는 비즈니스 관련 정보가 담기는게 보통. (숫자, 문자열)
    - 클라이언트에게 노출되어선 안됨
    - 파티션 결정 식: hash(key) & numPartitions
        - (생산자가 직접 정의도 가능)
    - 키가 주어지지 않으면 무작위 결정
    - 주의: 키-값 저장소의 키와는 다른 개념 ⇒ 메시지 마다 고유할 필요x, not null x
- 메시지 값
    - = 메시지의 내용. 페이로드(payload)
    - 일반 텍스트, 이진 블록 가능
- 토픽: 메시지가 속한 토픽의 이름
- 파티션: 메시지가 속한 파티션의 ID
- 오프셋
    - 파티션 내 메시지의 위치
    - (토픽, 파티션, 오프셋) ⇒ 메시지 찾기 가능
- 타임스탬프: 메시지가 저장된 시각
- 크키: 메시지의 크기
- CRC: 순환 중복 검사(Cyclic Redundancy Check) 데이터의 무결성 보장에 이용
- +) 기타 선택적필드: ex.태그 (메시지 필터링)

### 일괄 처리

메시지 큐 내부의 메시지 일괄 처리

- 성능개선에 중요한 이유
    - 값비싼 네트워크 왕복 비용 제거
    - ❓ 브로커가 여러 메시지를 한 번에 로그에 기록 → 더 큰 규모의 순차 쓰기 연산 → 디스크 캐시에서 더 큰 규모의 연속된 공간을 점유 → 더 높은 디스크 대역폭
    - ❓낮은 응답 지연이 중요한 경우(전통적 메시지 큐) → 일괄 처리 메시지양 낮춤 → 디스크 성능이 낮아짐 → 처리량을 늘려야한다면 토픽당 파티션 수를 늘림 → 낮아진 순차 쓰기 연산 대역폭을 벌충

### 생산자 측 작업 흐름

- 라우팅 계층
    - 적절한 브로커에 메시지를 보내는 역할
    - 브로커를 복제하여 운용하는 경우의 적절한 브로커: 리더 브로커
    - 흐름
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aaed19ba5-9714-4910-af5e-9fd56c7971b5%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8091-a710-c8529237350f&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
        1. 생산자 > [메시지] > 라우팅 계층
        2. 라우팅 계층: 메타데이터 저장소에서 사본 분산 계획을 읽어 자기 캐시에 보관. 이를 통해 메시지를 파티션-1의 리더 사본에 전송
        3. 리더 사본 → 리더를 따르는 사본에 데이터 전달
        4. 충분한 수의 사본이 동기화 → 리더: 데이터를 디스크에 기록. 데이터가 소비 가능 상태가 됨. 기록 종료 후 생산자에게 회신
    - 단점
        - 거쳐야 할 네트워크 노드가 증가 → 네트워크 전송 지연 증가
        - 일괄 처리 고려x
    - 수정안
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A4b8c48ec-8080-4dfa-b9fb-f6bf035aea71%3Aimage.png?table=block&id=2fdbe4ca-d9d2-80f0-bdad-f024a64b2b05&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
        - 라우팅 계층을 생산자 내부로 편입, 버퍼 도입
        - 장점
            - 네트워크 노드 감소 → 전송 지연 감소
            - 생산자: 파티션 결정 로직 커스텀 가능
            - 전송할 메모리를 버퍼 메모리에 보관 → 일괄 전송 → 대역폭 증가
        - 일괄처리 메시지 양 선택
            
            ```
            지연 ↑
             │
             │        /
             │      /
             │    /
             │  /
             └──────────► 대역폭
            
            ```
            
            - 대역폭 ↔ 응답 지연 사이에서 타협점을 찾아야함

### 소비자 측 작업 흐름

![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aabad9406-2ae4-4a10-8182-d022736f480b%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8064-bea6-d02ac1696d60&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=990&userId=&cache=v2)

- ❓소비자는 특정 파티션의 오프셋을 주고, 해당 위치에서 부터 이벤트를 묶어 가져옴

### 푸시 vs 풀

브로커가 소비자에게 전송 vs 소비자가 브로커에게서 가져감

- 푸시 모델
    - 장점: 낮은 지연
    - 단점
        - 소비자의 메시지를 처리하는 속도가 느릴 경우 → 소비자에게 큰 부하
        - 생산자가 데이터 전송 속도를 결정 → 소비자는 이에 맞는 처리가 가능한 컴퓨티 자원을 준비해 두어야함
- 풀 모델 (대부분의 메시지 큐가 선택)
    - 장점
        - 생산자가 메시지 소비 속도를 결정 → 각 소비자 별로 실시간/일괄 소비 선택 가능
        - 메시지를 소비하는 속도가 느려질 경우 소비자를 늘리거나 기다리거나 선택 가능
        - 일괄 처리에 적합: 오프셋을 통해 마지막으로 가져간 로그 위치 다음에 오는 모든 메시지 or 설정된 최대 개수 만큼 가져올 수 있음
    - 단점
        - 브로커에 메시지가 없어도 소비를 시도할 수 있음 → 소비자의 컴퓨팅 자원 낭비
            - 극복 방법: 롤 폴링 모드 지원. 가져갈 메시지가 없으면 일정시간 대기
    - 흐름도
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A068efce0-44a1-4e74-856f-71977d345b48%3Aimage.png?table=block&id=2fdbe4ca-d9d2-80bc-9404-ecf7b260a316&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
        1. 같은 그룹의 모든 소비자는 같은 브로커(전담 코디네이터)에 접속
            - 코디네이터: 소비자 그룹의 조정 작업
            - 조정 서비스: 브로커 클러스터 조정작업
        2. 코디네이터: 신규 소비자를 그룹에 참여시킴 & 파티션을 소비자에 할당 
            - (❓파티션 배치정책: 라운드 로빈, 범위 기반 정책 등)
        3. 소비자: 마지막으로 소비한 오프셋 이후 메시지 조회
            - 오프셋 정보: 상태저장소에 저장
        4. 소비자: 메시지를 처리 후 새로운 오프셋을 브로커에게 전송

### 소비자 재조정

어떤 소비자가 어떤 파티션을 책임지는지 재 선택하는 프로세스. 
소비자 측의 규모 확장성과 결함 내성을 보장

- 시작 이유: 신규 소비자 합류or기존 소비자가 그룹 탈퇴or특정 소비자에게 장애 발생or파티션들이 조정
- 코디네이터: 소비자로부터 오는 박동(heartbeat) 메시지를 살피고 각 소비자의 파티션 내 오프셋 정보를 관리
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A47597f4a-c71e-4a0f-83ab-78d69255b6c1%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8006-b1ce-ee7c11849ab6&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - 같은 그룹의 소비자는 같은 코디네이터에 연결 (그룹 이름을 해싱해 배정)
    - 소비자 목록에 변화가 생기면 코디네이터는 그룹의 새 리더를 선출
    - 새 리더: 새 파티션 배치 계획을 만들어 코디네이터에 전달 → 코디네이터는 계획을 그룹 내 다른 소비자에게 전달
- 재조정 시나리오들 (그룹 내 소비자 수:2, 토픽의 파티션:4 가정)
    - 새로운 소비자 B가 그룹에 합류한 경우
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Abcecba5d-93c6-4c1d-b14b-773fb8afa687%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8059-994d-cb76017d6ee6&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
    - 기존 소비자 A가 그룹을 떠나는 경우
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Abe547d80-bddd-4b07-abcd-5b789247e01d%3Aimage.png?table=block&id=2fdbe4ca-d9d2-80a5-bbbe-ddbda3861575&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
    - 소비자 A가 비정상적으로 가동을 중단한 경우
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ab745bc14-466a-4138-9750-ca1a233c7f6c%3Aimage.png?table=block&id=2fdbe4ca-d9d2-80a2-9a06-ed8ec013e936&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        

### 상태 저장소

- 저장 되는 내역
    - 소비자에 대한 파티션의 배치 관계
    - 각 소비자 그룹이 각 파티션에서 마지막으로 가져간 메시지의 오프셋
- 소비자 상태 정보 데이터가 이용되는 패턴
    - 적은 양의 빈번한 읽기/쓰기.
    - 빈번한 데이터 갱신. 드문 삭제
    - 무작위적 패턴의 읽기/쓰기 연산
    - 데이터의 일관성이 중요
- ⇒ 주키퍼(Apache ZooKeepr) 같은 키-값 저장소를 사용하는게 바람직
    - (+) 카프카는 오프셋 저장소를 주키퍼→카프카 브로커로 이전

### 메타데이터 저장소

- 토픽 설정이나 속성 정보를 보관 ( 파티션 수, 메시지 보관 기간, 사본 배치 정보 등)
- 적은 양의 드문 변경
- 높은 일관성을 요구
- ⇒ 주키퍼가 적절

### 주키퍼

- 계층적 키-값 저장소 (hierarchical key-value store)
- 분산 시스템에 필수적인 서비스
- 분산 설정 서비스, 동기화 서비스, 이름 레지스트리 등으로 이용됨
- 주키퍼를 활용한 설계안
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A5e95550e-2986-47ad-ad8b-145cab596941%3Aimage.png?table=block&id=2fdbe4ca-d9d2-807c-bbfe-e68f3759d1b9&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - 메타데이터, 상태 저장소 ⇒ 주키퍼
    - 조정 서비스 ⇒ 주키퍼
    - 브로커: 데이터 저장소만 유지

### 복제

하드웨어 장애에 대비, 높은 가용성을 보장

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A6f87a358-53cd-4c0b-919f-4f0507c9b187%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8039-bde8-da499163ba71&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=990&userId=&cache=v2)

- 각 파티션의 사본들은 서로 다른 브로커 노드에 분산
- 짙은 색 사본: 파티션의 리더
- 메시지를 완전히 동기화한 사본의 갯수가 지정된 임계 값을 넘으면: 리더 > 생산자 응답
- 사본 분산계획
    - 사본을 파티션에 어떻게 분산할지 기술하는 것
    - 조정 서비스로 선출 된 리더 브로커 노드가 사본 분산 계획을 생성 → 메타 데이터 저장소에 보관

### 사본 동기화

- 동기화 된 사본(In-Sync Replicas, ISR): 리더와 동기화 된 사본
- 동기화되었다?: 토픽 설정에 따라 달라짐
    - ❓replica.lag.max.message: 4
    단순 사본에 보관 된 메시지 개수 ↔ 리더 사이의 차이: 3이면 해당 사본은 여전히 ISR
    - 리더는 늘 ISR 상태
    - 합의 오프셋: 이전에 기록된 모든 메시지는 ISR 집합 내 모든 사본에 동기화 완료 됨
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ab7af0f42-af1b-4fb5-9cde-168230190c57%3Aimage.png?table=block&id=2fdbe4ca-d9d2-80d0-ab07-c669b3559340&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
- ISR이 필요한 이유
    - 성능↔영속성 사이의 타협점

### 메세지 수신응답

- ACK = all
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ae943a2fb-3f38-4544-af1c-cfe8d31bbf8d%3Aimage.png?table=block&id=2fdbe4ca-d9d2-80d2-b1a9-ea10a424b45a&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - 생산자는 모든 ISR이 메시지를 수신한 뒤에 ACK 응답을 받음
    - 높은 영속성, 낮은 성능
- ACK = 1
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A1896e1af-1208-4c2b-b004-344ec241a5f3%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8090-b436-f2c8a9f44082&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - 생산자는 리더가 메시지를 저장하고 나면 바로 ACK 응답을 받음
    - 높은 성능, 낮은 영속성
- ACK = 0
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ab2dd49eb-521b-4f4c-b7c0-bf44e589704c%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8033-a528-f630c49b6008&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - 생산자는 보낸 메시지에 대한 수신 확인 메시지를 기다리지 않음.
    - 재시도 하지 않음
    - 매우 높은 성능, 낮은 영속성
    - 메시지의 양이 많고, 데이터 손실이 괜찮은 경우(지표 수집이나 데이터 로깅 등)에 사용
- 소비자 측면
    - 보통은 리더 사본에서 메시지를 읽어감
        - 특정 파티션의 메시지는 같은 소비자 그룹 내 오직 한 소비자만 읽으므로 연결이 생각보다 많지않음
        - 아주 인기 있는 토픽이 아니라면 연결 수가 많지 않음
        - 아주 인기 있는 토픽의 경우 파티션 및 소비자 수를 늘려 규모 확장
    - 리더 사본에서 메시지를 읽어가는게 바람직 하지 않는 경우 존재
        - ex. 소비자의 위치가 리더 사본이 존재하는 데이터 센터와 멀음

### 규모 확장성

- 생산자
    - 단순 추가/삭제 가능
- 소비자
    - 소비자 그룹은 서로 독립적 ⇒ 단순 추가/삭제 가능
    - 소비자 그룹 내의 소비자가 새로 추가/삭제, 장애로 제거 ⇒ 재조정 매커니즘이 처리
- 브로커
    - 브로커 노드의 장애
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aaeaaaf80-a124-4569-a9d0-aebe46026fb2%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8035-8466-fc55bb0e85a7&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
        - 브로커 컨트롤러는 브로커-3이 사라졌음을 감지하고 새로운 파티션 분산 계획을 생성
        - 리더가 손실 된 경우 본산 된 사분이 리더로 변화
        - 새로 추가 된 사본은 단순 사본. 리더에 보관 된 메세지를 따라 잡는 동작 개시
    - 브로커의 결함 내성을 높이기 위한 고민
        - 메세지가 성공적으로 합의(committed) 됨의 판단: 얼마나 많은 사본에 메시지가 반영 되어야?
        - 같은 파티션의 사본들을 같은 노드에 두면 안됨.
        - 파티션의 모든 사본에 문제가 생기면 해당 파티션의 데이터는 영원히 손실 됨
            - 사본은 여러 데이터 센터에 분산하는 게 안전함
            - 동기화로 인한 응답 지연과 비용에 대해 고민해보아야 함
    - 브로커의 규모 확장성
        - 브로커 노드가 추가/삭제 될 때 사본을 재배치
        - 브로커 컨트롤러로 하여금 한시적으로 시스템에 설정 된 사본 수 보다 많은 사본을 허용 (채택)
            
            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A99bcfb37-6c99-41f1-9eb0-1f4bcee4b9b8%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8017-89ce-c806a29b4072&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=820&userId=&cache=v2)
            
            - 새로 추가 된 브로커 노드가 기존 브로커 상태를 따라잡고 나면 필요없는 노드는 제거
- 파티션
    - 토픽의 규모를 늘리거나, 대역폭을 조정하거나, 가용성↔대역폭 사이의 균형을 맞추는 등의 이유로 파티션 수를 조정
    - 생산자: 브로커와 통신할 때 통지 받음
    - 소비자: 재조정을 시행
    - 파티션 추가
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A00eaf4d4-ea86-4c8f-9ff4-3c8bbaf30723%3Aimage.png?table=block&id=2fdbe4ca-d9d2-80b3-ba6f-ee201536d163&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
        - 지속적으로 보관 된 메시지는 기존 파티션에 존재. 이동X
    - 파티션 삭제
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A29ca2410-2eac-4395-9b33-1e5867450f2d%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8007-963a-fa583807c7cf&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
        - 구독하는 소비자가 존재할 수 있으므로 퇴역 된 파티션은 일정시간 유지 후 삭제
        - 파티션이 완전히 삭제되면 소비자는 재조정 작업을 시작

### 메세지 전달 방식

- 최대 한 번
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ac2a3e331-a829-4884-a395-260818c0d847%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8093-9189-cbbb081478fb&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - 생산자: 토픽에 비동기적으로 메시지를 전송, 수신 응답을 기다리지 않음(ACK=0). 메시지 전달이 실패해도 재시도 하지 않음
    - 소비자: 오프셋을 먼저 갱신 후 메시지를 읽고 처리. 오프셋 갱신 이후 소비자가 장애로 죽으면 메시지는 다시 소비 불가능
    - 지표 모니터링 등 소량의 데이터 손실은 감수할 수 있는 경우 사용
- 최소 한 번
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A68cebdcc-ebb8-44d7-8ba8-fb6744f39dc2%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8097-a50c-c109102de4bd&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - 생산자: 메시지를 동기/비동기 적으로 전송, ACK = 1 또는 ACK=all 사용 → 메시지가 브로커에게 전달되었음을 확인. 메시지 전달이 실패하거나 타임아웃이 발생하면 재시도
    - 소비자: 데이터를 성공적으로 처리한 뒤에만 오프셋을 갱신. 메시지 처리가 실패한 경우 메시지를 다시 가져옴.
    메시지 처리 후 오프셋 갱신 전 장애로 소비자가 죽으면 메시지를 중복처리할 수 있음
    - 메시지는 브로커나 소비자에게 한 번 이상 전달 가능
    - 메시지 중복이 큰 문제가 아니거나 or 자체 중복 제거 기능이 있는 애플리케이션에서 사용
- 정확히 한 번
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aa0d6b1ad-01c7-475f-bbe8-76d64b1c5f6f%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8005-be57-e9271a770d7d&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=930&userId=&cache=v2)
    
    - 구현 복잡도가 높음
    - 중복을 허용하지 않으며, 메세지 손실을 허용하지 않음.
    - 같은 입력에 항상 같은 결과를 내놓도록 구현되어 있지 않은 애플리케이션에서 사용

### 고급기능

- 메시지 필터링
    - 토픽: 같은 유형의 메시지를 담아 처리하기 위해 도입 된 논리적 개념
    - 토픽의 세부/하위 유형 메시지를 필터링
    - 예시: 주문시스템
        - 주문시스템 > 주문과 관련된 모든 걸 주문 토픽에 전송
        - 지불시스템 > 주문 토픽 중 결재, 환불 관련 메시지만 필요
        - 지불 전용 토픽을 새로 분리할 경우
            - 같은 메시지를 여러 토픽에 저장하는 것은 자원 낭비
            - 새로운 소비자 측 요구사항이 나올 때마다 생산자 구현을 바꿔야할 수 있음 → 생산자↔소비자 간 결합도 상승
    - 필터링 방법
        - 소비자가 모든 메세지를 받은 후 필요 없는 걸 버림
            - 높은 유연성
            - BUT 불필요한 트래픽 증가→시스템 성능 저하
        - 브로커에서 메시지를 필터링
            - 필터링을 위해 복호화 or 역질렬화 등이 필요하면 브로커 성능이 저하
            - 메시지에 민감 데이터 포함 시 메시지 큐에서 조회 불가능
            - ⇒ 브로커에서 구현할 필터링 로직은 메시지의 payload를 추출하면 안됨
            - 필터링에 사용할 데이터는 메타 데이터 영역에 저장
            - 예시: 메시지마다 태그
            
            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A13faac6b-663b-47fa-95cb-943bce70c7a5%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8068-86f7-c63a04804cec&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=820&userId=&cache=v2)
            
- 메시지의 지연 전송 및 예약 전송
    - 지연 전송 예시: 결제 확인 메시지
        - 주문 시점에 메시지 전송. 소비자에겐 30분 뒤에 전달 되도록 설정
        - 결재 미완료→ 취소 / 결재 완료 → 무시
        - 지연 전송 메시지는 브로커 내부의 임시 저장소에 저장 후 일정 시간 뒤 토픽으로 이동
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A7b632384-40e6-4b22-96dc-9910e02f529a%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8084-9eb6-cd50848695b5&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=880&userId=&cache=v2)
        
    - 하나 이상의 특별 메시지 토픽을 임시 저장소로 활용 가능
    - 타이밍 기능
        - 메시지 지연 전송 전용 메시지 큐 (1초, 5초, 10초, … 20분, 30ㅇ분, 한시간, 두시간 …)
        - 계층적 타이밍 휠을 사용
    - 메시지 예약 전송
        - 지정된 시간에 소비자에게 메세지 전송. 지연 전송 시스템과 유사함

## 4단계: 마무리

### 추가 이야기 거리

- 프로토콜
    - 노드 사이에 오고가는 데이터에 관한 규칙, 문법, API를 규정
    - 분산 메시지 큐 시스템의 프로토콜
        - 메세지 생산, 소비, 박동 메시지 교환 등의 모든 활동을 설명
        - 대용량 데이터를 효과적으로 전송할 방법을 설명
        - 데이터의 무결성을 검증할 방법을 기술
    - 유명한 프로토콜
        - AMQP, 카프카 프로토콜
- 메세지 소비 재시도
    - 새로 몰려드는 메시지들이 제대로 처리되지 못하는 걸 방지하려면 어떻게 재시도?
    - ex. 재시도 전용 토픽
- 이력 데이터 아카이브
    - 이미 삭제 된 메시지를 다시 원하는 사용자 → 시간 기반 or 용량 기반 로그 보관 매커니즘
    - 오래된 데이터는 HDFS 같은 대용량 저장소 or 객체 저장소에 보관

---

## 추가공부

### RAID

**Redundant Array of Independent Disks**

여러 개의 독립 디스크를 **하나의 논리적 디스크** 처럼 사용하는 기술

- 성능 향상 (병렬 I/O)
- 데이터 보호 (중복/패리티)
- 용량 효율성

3가지 기본기술

1. Striping (스트라이핑): 데이터 분할 → 속도↑
ABCDE → 디스크1:AC / 디스크2:BE
2. Mirroring (미러링): 데이터 복제 → 안전↑
디스크1:ABCDE / 디스크2:ABCDE
3. Parity (패리티): 오류복구 정보 → 1개 고장 OK
디스크1:AB / 디스크2:CD / 디스크3:P(복구정보)

### CRC란?

### 조정서비스 vs 코디네이터

- **조정서비스**: **클러스터 전체 조정** (외부/독립 서비스)
- **코디네이터**: **특정 기능 조정** (브로커 내부 컴포넌트)
- 비교표
    
    
    | **구분** | **조정서비스** | **코디네이터** |
    | --- | --- | --- |
    | **위치** | 외부(ZooKeeper/KRaft) | Kafka 브로커 내부 |
    | **역할** | 클러스터 메타데이터, 리더 선출 | 컨슈머그룹/트랜잭션 |
    | **예시** | ZooKeeper, KRaft Controller | Group Coordinator, Tx Coordinator |
    | **스코프** | **전역** (전체 클러스터) | **국소** (그룹/트랜잭션) |

### 브로커 컨트롤러 vs 리더 브로커

- **브로커 컨트롤러**: **클러스터 전체 관리자** (하나만 존재)
- **리더 브로커**: **특정 파티션의 데이터 담당자** (파티션당 하나)
- 역할 비교
    
    
    | 구분 | 브로커 컨트롤러 | 리더 브로커 |
    | --- | --- | --- |
    | **수량** | 클러스터당 **1개** | 파티션당 **1개** (수백~수천) |
    | **주요 역할** | 리더 선출, 메타데이터 배포 | **읽기/쓰기 처리** |
    | **통신 대상** | 모든 브로커 | 프로듀서/컨슈머 |
    | **장애 시** | 새 컨트롤러 선출 | ISR에서 새 리더 선출 |
    | **ZooKeeper** | `/controller` 노드 | 파티션 리더 정보 |
- 동작 흐름
    
    ```
    text1. 컨트롤러 선출 (ZooKeeper)
       브로커A ← /controller 에포크 획득
    
    2. 리더 배정
       컨트롤러 → 각 파티션별 리더 결정
       브로커B → 파티션0 리더
       브로커C → 파티션1 리더
    
    3. 장애 발생
       브로커B 다운 → 컨트롤러가 파티션0 새 리더 선출
    ```
    

### 의문

메시지란 무엇인가… 소비자 그룹을 생성하면 모든 메시지를 다 못읽는게 아닌가… 클라이언트별로 하나의 소비자 그룹을 가지는것인가…
