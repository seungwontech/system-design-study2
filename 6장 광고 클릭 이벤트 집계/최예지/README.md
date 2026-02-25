# (6) 광고 클릭 이벤트 집계

- 디지털 광고의 핵심프로세스: **RTB**(Real-Time Bidding, 실시간 경매)
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A517dc002-5d56-4d51-95ea-9435130558e2%3Aimage.png?table=block&id=30cbe4ca-d9d2-8050-9b47-c40a52f26aa3&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1310&userId=&cache=v2)
    
    - RTB 동작 절차

## 1단계: 문제 이해 및 설계 범위 확정

- 기능 요구사항
    - 지난 M분 동안의 ad_id 클릭 수 집계
    - 매분 가장 많이 클릭 된 상위 100개 광고 아이디 반환 (질의 기간, 광고 수 변경 가능)
    - 다양한 속성에 따른 집계필터링 지원
    - 데이터의 양: 페이스북 or 구글 규모
- 비기능 요구사항
    - 집계 결과 정확성: 데이터가 RTB 및 광고 가금에 사용되므로 중요
    - 지연되거나 중복된 이벤트를 적절히 처리할 수 있어야 함
    - 견고성(reliability): 부분적인 장내는 감내 가능해야 함
    - 지연 시간 요구사항: 전체 처리 시간은 최대 수 분을 넘지 않아야 함
- 개략적 추정
    - DAU: 10억 명
    - 각 사용자가 하루 평균 1개 광고를 클릭한다고 가정. 
    ⇒ 하루에 평균 10억 건의 광고 클릭 이벤트 발생
    - 광고 클릭 QPS = 10^9이벤트 / 하루 10^5초 = 10,000
    - 최대 광고 클릭 QPS를 평균 QPS의 다섯배로 가정
    ⇒ 최대 광고 클릭 QPS = 50,000QPS
    - 광고 클릭 이벤트 하나당 0.1KB의 저장 용량이 필요하다 가정
    ⇒ 일일 저장소 요구량: 0.1KB * 10억 = 100GB
    ⇒ 월간 저장 용량 요구량 = 대략 3TB

## 2단계: 개략적 설계안 제시 및 동의 구하기

### 질의 API 설계

- API 설계의 목적: 클라이언트↔서버 통신 규약
- 소비자 앱의 클라이언트: 제품의 최종 사용자
- 본 설계안의 클라이언트: 대시보드를 이용하는 데이터 과학자, 제품관리자, 광고주. → 대시보드를 이용하는 순간 집계 서비스에 질의 발생
- 기능 요구사항
    - 지난 M분 동안 각 ad_id에 발생한 클릭 수 집계
    - 지난 M분 동안 가장 많은 클릭이 발생한 상위 N개 ad_id 목록 반환
    - 다양한 속성을 기준으로 집계 결과를 필터링하는 기능 지원 ← 호출인자를 통해 지원 가능
- API 1: 지난 M분 간 각 ad_id에 발생한 클릭 수 집계
    
    
    | API | 용도 |
    | --- | --- |
    | GET /v1/ads/{:ad_id}/aggregated_count | 주어진 ad_id에 발생한 이벤트 수를 집계해 반환 |
    - API 호출에 이용하는 호출 인자
        
        
        | 인자명 | 뜻 | 자료형 |
        | --- | --- | --- |
        | from | 집계 시작 시간
        (default: 현재 시각부터 1분 전) | long |
        | to | 집계 종료 시간
        (default: 현재 시각) | long |
        | filter | 필터링 전략 식별자.
        가령 filter=001는 미국 이외 지역에서 발생한 클릭은 제외하란 뜻 | long |
    - 응답
        
        
        | 필드명 | 뜻 | 자료형 |
        | --- | --- | --- |
        | ad_id | 광고(ad) 식별자 | string |
        | count | 집계된 클릭 횟수 | long |
- API 2: 지난 M분 간 가장 많은 클릭이 발생한 상위 N개 ad_id 목록
    
    
    | API | 용도 |
    | --- | --- |
    | GET /v1/ads/poular_ads | 지난 M분간 가장 많은 클릭이 발생한 상위 N개 광고 목록 반환 |
    - API 호출에 이용하는 호출 인자
        
        
        | 인자명 | 뜻 | 자료형 |
        | --- | --- | --- |
        | count | 상위 몇 개의 광고를 반환할 것인가 | integer |
        | window | 분 단위로 표현된 집계 윈도 크기 | integer |
        | filter | 필터링 전략 식별자 | long |
    - 응답
        
        
        | 필드명 | 뜻 | 자료형 |
        | --- | --- | --- |
        | ad_ids | 광고 식별자 목록 | array |

### 데이터 모델

- 원시 데이터 (raw data)
    - 원시데이터의 사례
        
        ```
        [AdClickEvent] ad001, 2021-01-01 00:00:01, user 1, 207.148.22.22, USA
        ```
        
    - 구조화된 형식(structured way)으로 표현
        
        
        | ad_id | click_timestamp | user_id | ip | country |
        | --- | --- | --- | --- | --- |
        | ad001 | 2021-01-01 00:00:01 | user1 | 207.148.22.22 | USA |
        | ad001 | 2021-01-01 00:00:02 | user1 | 207.148.22.22 | USA |
        | ad002 | 2021-01-01 00:00:02 | user2 | 209.153.56.11 | USA |
- 집계 결과 데이터 (aggregated)
    - 매분 광고 클릭 이벤트가 집계 됨을 가정했을 때의 집계 결과
        
        
        | ad_id | click_minute | count |
        | --- | --- | --- |
        | ad001 | 202101010000 | 5 |
        | ad001 | 202101010001 | 7 |
    - 광고 필터링 지원을 위해 filter_id 추가
        - 필터를 사용해 집계한 데이터
            
            
            | ad_id | click_minute | filter_id | count |
            | --- | --- | --- | --- |
            | ad001 | 202101010000 | 0012 | 2 |
            | ad001 | 202101010000 | 0023 | 3 |
            | ad001 | 202101010001 | 0012 | 1 |
            | ad001 | 202101010001 | 0023 | 6 |
        - 필터 테이블
            
            
            | filter_id | region | ip | user_id |
            | --- | --- | --- | --- |
            | 0012 | US | 0012 | * |
            | 0013 | * | 0023 | 123.1.2.3 |
    - 지난 M분간 가장 많이 클릭 된 상위 N개의 광고 질의용 테이블
        
        
        | most_clicked_ads | 자료형 | 설명 |
        | --- | --- | --- |
        | window_size | integer | 분 단위로 표현된 집계 윈도 크기 |
        | update_time_minute | timestamp | 마지막으로 갱신된 타임스탬프 (1분 단위) |
        | most_clicked_ads | array | JSON 형식으로 표현된 ID 목록 |
- 비교
    
    
    |  | 원시 데이터만 보관하는 방안 | 집계 결과 데이터만 보관하는 방안 |
    | --- | --- | --- |
    | 장점 |   • 원본 데이터를 손실 없이 보관
      • 데이터 필터링 및 재계산 지원 |   • 데이터 용량 절감
      • 빠른 질의 성능
     |
    | 단점 |   • 막대한 데이터 용량
      • 낮은 질의 성능 |   • 데이터 손실. 원본 데이터가 아닌 계산/유도된 데이터를 저장하는 데서 오는 결과. 예를 들어 10개의 원본 데이터는 1개의 데이터로 집계/축약될 수 있다. |
    - 결론: 둘 다 저장하는 걸 추천
        - 문제가 발생하면 디버깅에 활용하도록 원시 데이터 보관.
        버그로 집계 데이터가 손상 되면 버그 수정 후 원시 데이터에서 재집계 가능
        - 원시 데이터는 양이 막대하므로 직접 질의는 비효율적 → 집계 결과 데이터에 질의하는게 바람직함
        - 원시 데이터는 백업 데이터로 활용. 오래된 원시 데이터는 냉동 저장소(cold storage)로 옮겨 비용 절감
        - 집계 결과 데이터: 활성 데이터(active data)구실. 질의 성능을 높이기 위해 튜닝하는 것이 보통

### 올바른 데이터베이스의 선택

- 평가 사항
    - 데이터는 어떤 모습? 관계형 데이터? 문서 데이터? 이진 대형 객체(BLOB)?
    - 작업 흐름이 읽기 중심? 쓰기 중심? 둘 다?
    - 트랜잭션을 지원?
    - 질의 과정에서 SUM이나 COUNT 같은 온라인 분석 처리(OLAP)함수를 많이 사용?
- 원시 데이터
    - 일상적인 작업에서는 질의x. 데이터 과학자나 기계 학슴 엔지니어가 사용자 반응 예측, 행동 타기팅, 관련성 피드백 등을 연구하는 경우에 사용
    - 평균 쓰기 QPS: 10,000. 최대 쓰기 QPS: 50,000 ⇒ 쓰기 중심 시스템
    - 읽기는 백업, 재계산 용도로만 사용해 연산 빈도가 낮음
    - 쓰기 및 시간 범위 질의에 최적화 된 카산드라, InfluxDB를 사용하는게 바람직
    - ORC, 파케이, AVRO 등의 칼럼형 데이터 형식을 사용해 아마존 S3에 데이터를 저장하는 방법도 존재
        - 각 파일의 최대 크기를 제한(ex 10GB) → 원시 데이터 기록 담당 스트림 프로세서: 최대 크기에 도달하면 자동으로 새파일 생성
- 집계 데이터
    - 본질적으로 시계열 데이터.
    - 읽기/쓰기 연산 모두 빈번
        - (읽기) 매 분마다 데이터베이스에 질의를 던져 고객에게 최신 집계 결과를 제시
        - (쓰기) 집계 서비스가 데이터를 매 분 집계하고 그 결과를 기록

### 개략적 설계안

입력: 원시 데이터(무제한 데이터 스트림) 출력: 집계 결과

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A4339b59d-9efe-4bee-bef8-7d0e41d96d82%3Aimage.png?table=block&id=30cbe4ca-d9d2-8069-98ad-cdb82d979acc&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1360&userId=&cache=v2)

- 비동기 처리
    - 데이터를 동기식으로 처리할 경우 트래픽이 갑자기 증가하여 발생하는 이벤트 수가 소비자의 처리용량을 훨씬 넘어서는 경우, 메모리 부족 오류등의 예기치 않은 문제갑 발생할 수 있음 ⇒ 동기식 시스템의 경우 특정 컴포넌트의 장애 → 전체 시스템 장애로 이어질 수 있음
    - 카프카 등의 메시지 큐를 도입하여 생산자↔소비자 결합을 끊는 방법 → 프로세스가 비동기 방식으로 동작 → 생산자, 소비자가 독립적으로 규모확장 가능
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A6975f368-2255-4e85-ae40-ec4e2940d3db%3Aimage.png?table=block&id=30cbe4ca-d9d2-80b4-93a9-c3d5425d31a7&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1310&userId=&cache=v2)
    
    - 로그 감시자, 집계 서비스, 데이터베이스: 두 개의 메시지 큐로 분리
    - 첫 번째 메시지 큐에 입력되는 데이터
        
        
        | ad_id | click_timestamp | user_id | ip | country |
        | --- | --- | --- | --- | --- |
    - 두 번째 메시지 큐에 입력되는 데이터
        - 분 단위로 집계된 광고 클릭 수
            
            
            | ad_id | click_minute | count |
            | --- | --- | --- |
        - 분 단위로 집계한, 가장 많이 클릭한 상위 N개 광고
            
            
            | update_time_minute | most_clicked_ads |
            | --- | --- |
    - 집계 결과를 데이터베이스에 바로 기록하지 않는 이유
        - 정확하게 한 번 (exactly once) 데이터를 처리하기 위해 (atomic commit, 원자적 커밋) 카프카 시스템이 두번째 메시지 큐로 도입되어야 함

### 집계 서비스

- 맵리듀스(MapReduce) 프레임 워크 사용
    - 유향 비순환 그래프(directed acyclic graph, DAG)
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ac552d176-934a-45fc-bdcc-9708025769bd%3Aimage.png?table=block&id=30cbe4ca-d9d2-80e3-99ad-e3391e0f4b6d&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
        - 핵심: 시스템을 맵/집계/리듀스 노드 등의 작은 컴퓨팅 단위로 세분화
- 맵 노드
    - 데이터 출처에서 읽은 데이터를 필터링, 변환하는 역할
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ab2fc974a-998d-4980-9234-fc70f03ee103%3Aimage.png?table=block&id=30cbe4ca-d9d2-801e-b089-dc8624ff0f63&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1310&userId=&cache=v2)
    
    - ad_id % 2 = 0의 조건을 만족하는 데이터 → 노드 1
    비만족 데이터 → 노드 2
    - 맵 노드가 필요한 이유
        - 입력 데이터를 정리, 정규화해야 하는 경우 필요
        - 생성되는 방식에 대한 제어권이 없는 경우 → 동일한 ad_id를 갖는 이벤트가 서로 다른 카프카 파티션에 입력될 수 있음
- 집계 노드
    - ad_id별 광고 클릭 이벤트 수를 매 분 메모리에서 집계
    - 리듀스 프로세스의 일부 (사실 상 맵-집계-리듀스 프로세스 = 맵-리듀스-리듀스 프로세스)
- 리듀스 노드
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A77b4a905-fbb7-4acc-ba2c-6bbc3c9317ae%3Aimage.png?table=block&id=30cbe4ca-d9d2-805a-8135-ccf868d3f2c8&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1310&userId=&cache=v2)
    
    - 모든 ‘집계’ 노드가 산출한 결과를 최종 결과로 축약
- DAG: 맵-리뷰스 패러다임을 표현하기 위한 모델.
빅데이터 입력 → 병렬 분산 컴퓨팅 자원활용 → 작거나 일반적인 크기의 데이터로 변환
- 중간 데이터는 메모리에 저장.
- 노드 간 통신
    - TCP로 처리: 노드들이 서로 다른 프로세스에서 실행되는 경우
    - 공유 메모리로 처리: 노드들이 서로 다른 스레드에서 실행되는 경우
- 주요 사용 사례
    - 사례1: 클릭 이벤트 수 집계
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Acc417983-c6c0-4dc5-b11e-5433a3541c52%3Aimage.png?table=block&id=30cbe4ca-d9d2-8019-a58c-fe9530970d77&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
        맵 노드: ad_id % 3을 기준으로 분배
        
    - 사례2: 가장 많이 클릭된 상위 N개 광고 반환
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A2049f0cc-a18c-46af-b672-55e06ad63c7f%3Aimage.png?table=block&id=30cbe4ca-d9d2-801b-b54f-cf334d8917fa&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
        - 입력 이벤트: ad_id 기준으로 분배 → 각 집계노드: 힙을 내부적으로 사용, 상위 3개 광고 효율적 식별 → 리듀스 노드: 전달 받은 9개 광고 가운데 지난 1분간 가장 많이 클릭된 광고 3개 선별
    - 사례3: 데이터 필터링
        - 스키마 기법
            - 데이터 웨어하우스에서 널리 사용되는 기법.
            - 필터링에 사용되는 필드: 차원(dimension)이라 칭함
            - 장점
                - 이해하기 쉽고 구축이 간단
                - 기존 집계 서비스를 재사용하여 스타 스키마에 더 많은 차원 생성 가능. 추가 컴포넌트 필요X
                - 결과를 미리 계산, 필터링 기준에 따라 데이터에 빠르게 접근 가능
            - 한계
                - 많은 버킷(buckeet)과 레코드가 생성됨.
        - “미국 내 광고 ad001에 대해 집계된 클릭 수 만 표시” 등의 사전 정의 된 필터링 기준에 따라 집계된 예시
            
            
            | ad_id | click_minute | country | count |
            | --- | --- | --- | --- |
            | ad001 | 202101010001 | USA | 100 |
            | ad001 | 202101010001 | GPB | 200 |
            | ad001 | 202101010001 | others | 3000 |
            | ad002 | 202101010001 | USA | 10 |
            | ad002 | 202101010001 | GPB | 25 |
            | ad002 | 202101010001 | others | 12 |

## 3단계: 상세 설계

- 스트리밍 vs 일괄 처리
    - 스트림 처리: 데이터를 오는 대로 처리 & 거의 실시간으로 집계된 결과를 생성하는데 사용
    - 일괄 처리: 이력데이터를 백업 시
    
    | 구분 | 서비스 (온라인 시스템) | 일괄 처리 시스템 (오프라인 시스템) | 스트리밍 시스템 (실시간에 가깝게 처리하는 시스템) |
    | --- | --- | --- | --- |
    | 응답성 | 클라이언트에게 빠르게 응답 | 클라이언트에게 응답할 필요가 없음 | 클라이언트에게 응답할 필요가 없음 |
    | 입력 | 사용자의 요청 | 유한한 크기를 갖는 입력, 큰 규모의 데이터 | 입력에 경계가 없음 (무한 스트림) |
    | 출력 | 클라이언트에 대한 응답 | 구체화 뷰, 집계 결과 지표 등 | 구체화 뷰, 집계 결과 지표 등 |
    | 성능 측정 기준 | 가용성, 지연 시간 | 처리량 | 처리량, 지연 시간 |
    | 사례 | 온라인 쇼핑 | 맵리듀스 | 플링크(Flink) |
    - 일괄+스트리밍 처리 동시 지원 시스템의 아키텍처를 람다(lambda)라고 함
        - 단점: 유지 관리해야할 코드가 두 벌
    - ⇒ 카파 아키텍쳐: 일괄 처리 및 스트리밍 처리 경로를 하나로 결합하여 해결
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Af48fd722-e11e-4c07-abe3-ed6678c6ad42%3Aimage.png?table=block&id=312be4ca-d9d2-80f6-98cf-e07fda5d8606&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1310&userId=&cache=v2)
    
    - 데이터 재계산
        - 따로 이미 집계한 데이터를 다시 계산해야 하는 경우 ⇒ 데이터 재처리
        - 집계 서비스에 중대한 버그가 발생 시 데이터 재계산의 흐름
            
            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Af2bda3ab-3ec2-4cfb-a900-583894397e82%3Aimage.png?table=block&id=312be4ca-d9d2-804e-8f22-c6846013a418&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1200&userId=&cache=v2)
            
            1. 재계산 서비스: 원시 데이터 저장소에서 데이터를 검색. 일괄 처리 프로세스를 따름
            2. 추출된 데이터 → 전용 집계 서비스로 전송 (전용 집계 서비스 사용 이유: 실시간 데이터 처리 과정이 과거 재처리 프로세스와 간섭하는 일 방지)
            3. 집계 결과 → 두 번째 메시지 큐로 전송되어 집계 결과 데이터베이스에 반영
- 시간
    - 집계 시에는 타임스탬프가 필요. 타임스탬프의 두가지 위치
        - 이벤트 시각: 광고 클릭이 발생한 시각
        - 처리 시각: 집계 서버가 클릭 이벤트를 처리한 시스템 시각
    - 네트워크 지연이나 비동기적 처리 환경으로 이벤트 시각 ↔ 처리 시각 사이의 격차가 커질 수 있음
    - ⇒이벤트가 발생한 시각을 집계에 활용하는 경우: 지연된 이벤트 처리 문제를 잘 해결해야 함.
    - ⇒처리 시각을 집계에 사용하는 경우: 집계 결과가 부정확할 수 있음을 고려해야 함.
    
    | 구분 | 장점 | 단점 |
    | --- | --- | --- |
    | 이벤트 발생 시각 | 광고 클릭 시점을 정확히 아는 것은 클라이언트이므로 집계 결과가 보다 정확 | 클라이언트가 생성한 타임스탬프에 의존하는 방식이므로 클라이언트에 설정된 시각이 잘못되었거나 악성 사용자가 타임스탬프를 고의로 조작하는 문제에서 자유로울 수 없음 |
    | 처리 시각 | 서버 타임스탬프가 클라이언트 타임스탬프보다 안정적 | 이벤트가 시스템에 도착한 시각이 한참 뒤인 경우에는 집계 결과가 부정확해짐 |
    - 데이터 정확도가 중요하므로 이벤트 발생 시각을 사용할 것을 추천 ⇒ 지연된 이벤트 처리하려면?
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A79928db4-de6d-49dd-b83d-7fb250f52888%3Aimage.png?table=block&id=312be4ca-d9d2-8014-9334-f58b057484a3&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
        - 1분 단위로 끊어지는 텀블링 위도를 사용해 광고 클릭 이벤트를 집계
            - 윈도 1은 이벤트 2를 집계 실패, 윈도 3은 이벤트 5 집계 실패 문제 있음
    - 워터마크 사용
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A9e17cb34-3cc0-4048-9a63-a37933fbbfbc%3Aimage.png?table=block&id=312be4ca-d9d2-808f-b4a7-ed923f422073&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
        - 워터마크: 집계 윈도의 확장
    - 워터마크의 크기는 비즈니스 요구사항에 따라 정함.
- 집계 윈도 (aggregation window)
    - 윈도의 종류: 텀블링 윈도(=고정윈도), 호핑 윈도, 슬라이딩 윈도, 세션 윈도텀블링 윈도
    - 텀블링 윈도
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A0fc36700-46ab-4c90-8794-ad4b4eb83699%3Aimage.png?table=block&id=312be4ca-d9d2-803a-b05e-f3b11903cfa7&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
        - 시간을 같은 크기의 겹치지 않는 구간으로 분할 ⇒ 매 분 발생한 클릭 이벤트를 집계하기에 적합
    - 슬라이딩 윈도
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A253b6207-b321-4044-a593-cf348fdf1c3e%3Aimage.png?table=block&id=312be4ca-d9d2-80d3-8a95-e896829b6d08&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
        - 데이터 스트림을 미끄러져 나아가면서 같은 시간 구간 안에 있는 이벤트를 집계, 서로 겹칠 수 있음 ⇒ 지난 M분간 가장 많이 클릭 된 상위 N개 광고를 알아내기 적함
- 전달 보장(delivery guarantee)
    - 집계 결과는 과금 등에 활용될 수 있어 데이터의 정확성, 무결성이 중요
        - 이벤트의 중복 처리를 어떻게 피할 수 있는가?
        - 모든 이벤트의 처리를 어떻게 보장할 수 있는가?
    
    ⇒ “정확히 한 번” 방식 권장
    
    - 데이터 중복 제거
        - 데이터 중복의 두가지 사례
            - 클라이언트 측: 한 클라이언트가 같은 이벤트를 여러 번 보내는 경우 ⇒ 광고 사기/ 위험 제거 컴포넌트가 적합
            - 서버 장애: 집계 도중 집계 서비스 노드에 장애 발생 → 업스트림 서비스가 이벤트 메시지에 대해 응답을 받지 못해 이벤트가 재전송 되어 중복 집계
                
                ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Accaa2f89-ec99-4d1a-af41-1b988b09f542%3Aimage.png?table=block&id=312be4ca-d9d2-809d-9de9-cde5499be983&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1140&userId=&cache=v2)
                
                - 해결책 1: HDFS나 S3같은 외부 파일 저장소에 오프셋을 기록
                    
                    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Abfcb4d50-8a91-4809-95f4-e16d76057ea7%3Aimage.png?table=block&id=312be4ca-d9d2-80e7-a64e-ff54266e1ff0&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1080&userId=&cache=v2)
                    
                    - 문제점: 3단계 이후 장애가 발생하여 4단계를 완료하지 못할 시 → 데이터 손실 발생
                    - 해결책: 다운스트림에서 집계 결과 수신 확인 응답을 받은 후 오프셋을 저장
                    
                    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A5ff92560-a35f-40b2-8c0c-189c709f456a%3Aimage.png?table=block&id=312be4ca-d9d2-80e9-8705-ca0b5b74549c&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1080&userId=&cache=v2)
                    
- 시스템의 규모 확장
    
    간략한 추정: 사업은 매년 30%씩 성장, 트래픽은 3년마다 두 배 → 일 때 시스템을 규모확장 하는 법
    
    - 메시지 큐의 규모 확장 (4장에 상세설명 되어있음)
        - 생산자: 생산자 인스턴스 수에는 제한X, 확장성을 쉽게 달성할 수 있음
        - 소비자: 소비자 그룹 내의 재조정 매커니즘: 노드 추가/삭제를 통해 그 규모를 쉽게 조정
            - 소비자를 추가하는 작업은 시스템 사용량이 많지 않은 시간에 실행하는 걸 권장
        - 브로커
            - 해시 키
                - ad_id를 해시키로 활용하여 같은 ad_id를 갖는 이벤트를 같은 파티션에 저장 ⇒ 집계 서비스는 같은 ad_id를 갖는 이벤트를 같은 파티션에서 구독할 수 있음
            - 파티션의 수
                - 사전에 충분한 파티션을 확보 권장. (파티션의 수가 변하면 같은 ad_id를 갖는 이벤트가 다른 파티션에 기록될 수 있음)
            - 토픽의 물리적 샤딩
                - 지역에 따라(topic_north_america, topic_asia…), 사업유형에 따라(topic_web_ads, topic_mobile_ads…) 토픽을 나눌 수도 있음
                    - 장점: 시스템의 처리 대역폭을 높일 수 있음. 단일 토픽에 대한 소비자 수가 줄면 소비자 그룹의 재조정 시간도 단축
                    - 복잡성 증가 ⇒ 유지 관리 비용 증가
    - 집계 서비스의 규모 확장
        - 집계서비스는 본질적으로 맵리듀스 연산으로 구현
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ab559ca2f-a6c9-48ac-9b2e-e52a4c3baa9d%3Aimage.png?table=block&id=312be4ca-d9d2-80b6-933c-ffb4ace908d4&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
        - 노드의 추가/삭제를 통해 수평적으로 규모 조정 가능
        - 집계 서비스의 처리 대역폭을 높이는 방법
            - 방안 1: ad_id마다 별도의 처리 스레드 구현
                
                ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ac604080d-e86b-4094-89fe-11fc0e039205%3Aimage.png?table=block&id=312be4ca-d9d2-80e4-837b-da9555ea771d&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1140&userId=&cache=v2)
                
                - 구현이 더 쉬움. 자원 공급자에 대한 의존 관계X
            - 방안 2: 집계 서비스 노드를 아파치 하둡 YARN과 같은 자원 공급자에 배포. 다중 프로세싱 활용
                - 실제로 많이 쓰이는 바업. 더 많은 컴퓨팅 자원을 추가하여 시스템 규모를 확장할 수 있음
    - 데이터 베이스의 규모확장
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A0d04bf5d-9bad-4e60-9f4a-7a8ef5a9d9ed%3Aimage.png?table=block&id=312be4ca-d9d2-80fe-bbcc-ef39ef0e7a6b&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
        - 카산드라는 기본적으로 안정 해시와 유사한 방식으로 수평적인 규모 확장을 지원
        - 데이터는 각 노드에 균등히 분산. 사본도 적당한 수만큼 만들어 분산
        - 각 노드:  해시 링 위의 특정 해시 값 구간의 데이터 & 다른 가상 노드의 데이터 사본 보관
        - 클러스터에 새 노드를 추가하면 가상 노드 간의 균형이 자동으로 조정 됨.
- 핫스팟 문제
    - 핫스팟: 다른 서비스나 샤드보다 더 많은 데이터를  수신하는 서비스나 샤드
    - 큰 회사는 수백만 달러에 달하는 광고 예산을 집행, 해당 ad_id에 더 많은 클릭 발생 ⇒ 핫스팟 문제
        - ⇒ 더 많은 집계 서비스 노드를 할당하여 완화
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A809dbc9e-c97b-4419-9ace-f531cf30d67d%3Aimage.png?table=block&id=312be4ca-d9d2-80f1-afc1-e61e865b3b1c&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
        1. 집계 서비스 노드에 300개 이벤트가 도착. 한 노드가 감당할 수 있는 양 초과 ⇒ 자원 관리자에 추가 자원 요청
        2. 자원 관리자: 해당 서비스 노드에 과부하가 걸리지 않도록 추가 자원을 할당
        3. 원본 집계 서비스노드: 추가된 각 서비스 노드가 100개씩의 이벤트를 처리할 수 있도록 이벤트를 세 개 그룹으로 분할
        4. 집계가 끝나 축약된 결과는 다시 원본 집계 서비스 노드에 기록
- 결함 내성
    - 집계는 메모리에서 이루어지므로 집계 노드에 장애가 생기면 집계 결과도 손실 됨
    - ⇒ 업스트림 카프카 브로커에서 이벤트를 다시 받아와 숫자를 다시 만들어 냄
