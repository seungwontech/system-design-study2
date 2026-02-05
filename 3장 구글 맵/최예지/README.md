# (3) 구글 맵

## 1단계: 문제 이해 및 설계 범위 확정

### 기능 요구사항

- 사용자 위치 갱신
- 경로 안내 서비스(ETA(예상도착시간) 서비스 포함)
- 지도 표시

### 비기능 요구사항 및 제약사항

- 정확도: 사용자에게 잘못된 경로 안내x
- 부드러운 경로 표시
- 클라이언트는 가능한 한 최소한의 데이터와 배터리 사용
- 가용성 및 규모 확장서 요구사항 만족

### 지도 101

- 측위 시스템
    
    ![image.png](https://img.notionusercontent.com/s3/prod-files-secure%2F594418fe-2b8a-40ce-8d83-fb1b120f4b8f%2Faf096c0a-dc56-440d-84b9-9a35b5ae7c1c%2Fimage.png/size/w=1310?exp=1770296406&sig=3x9ByrdgJxD3tB43WCWsetfiVswtWwp1Y-p9njRflAw&id=2fdbe4ca-d9d2-80ad-a689-c6b2221c0f0b&table=block)
    
    - 축을 중심으로 회전하는 구. 표면 상의 위치를 표현하는 체계
    - 위도(Latitude): 북쪽↔남쪽
    - 경도(Longitude): 동쪽↔서쪽
- **지도 투영법**: 3차원 위치의 2차원 변환
    - 거의 모든 투영법은 실제 지형의 기하학적 특성을 왜곡함
    - 구글맵: 웹 메르카토르(Web Mercator) 도법 사용
- 지오 코딩(Geocoding)
    - 주소를 지리적 측위 시스템의 좌표로 변환하는 프로세스
    - ex) 미국 내 주소 ‘1600 Amphitheatre Parkway, Mountain View, CA’ : (위도 37.423021, 경도 -122.083739)
- 지오해싱
    - 지도 위 특정 영역을 영문자와 숫자로 구성된 짧은 문자열에 대응시키는 인코딩 체계
    - 2차원의 평면 공간으로 표현 된 지리적 영역 위의 격자를 더 작은 격자로 재귀적으로 분할. 재귀적으로 분할한 결과로 생성 된 더 작은 격자에는 0~3 (00, 01, 10, 11) 번호를 부여
- 지도표시
    - 타일(tile): 지도를 화면에 표시하는데 가장 기본이 되는 개념
    - 클라이언트는 사용자가 조회하려는 영역에 관계 된 타일만 모자이크처럼 이어 붙여 표시함
    - 지도의 확대 수준에 따라 다른 종류의 타일을 준비해야함. (확대/축소 기능)
- 경로 안내 알고리즘을 위한 도로 데이터 처리
    - 경로 탐색 알고리즘: 그래프 자료구조로 가정
        - 교차로: 노드(node)
        - 도로: 노드를 잇는 선(edge)
    - 전 세계 도로망을 분할하는 방법:
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aee5312f0-fa15-439c-bf22-1f4b49b3c22d%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8095-aa6a-ce0fa814f905&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
        - 세계를 작은 경로 안내 타일(격자)로 나누고, 각 격자 안이 도로망을 노드와 선으로 구성 된 그래프 자료구조로 변환
        - 각 타일은 도로로 연결 된 다른 타일에 대한 참조를 유지.
    - 유의점
        - 지도 타일: PNG 이미지
        - 경로 안내 타일: 도로 데이터로 이루어진 이진 파일
    - 계층적 경로 안내 타일
        - (예시) 구체성의 정도에 따라 경로 안내 타일을 준비
            - 상: 지방도
            - 중: 간선 도로
            - 하: 주요 고속도로

### 개략적 규모 추정

- 저장소 사용량
    - 메타 데이터
        - 각 지도 타일의 메타데이터. 크기가 매우 작가 본 추정에서 제외
    - 도로 정보
        - 현 가정에서는 외부에서 받은 수 TB 용량의 도로 데이터 존재
    - 세계 지도
        - 확대 수준별의 도로 타일이 한 벌 씩 필요
        
        | 확대 수준 | 필요 타일 수 | 확대 수준 | 필요 타일 수 | 확대 수준 | 필요 타일 수 | 확대 수준 | 필요 타일 수 |
        | --- | --- | --- | --- | --- | --- | --- | --- |
        | 0 | 1 | 6 | 4,096 | 12 | 16,777,216 | 17 | 17,179,869,184 |
        | 1 | 4 | 7 | 16,384 | 13 | 67,108,864 | 18 | 68,719,476,736 |
        | 2 | 16 | 8 | 65,536 | 14 | 268,435,456 | 19 | 274,877,906,944 |
        | 3 | 64 | 9 | 262,144 | 15 | 1,073,741,824 | 20 | 1,099,511,627,776 |
        | 4 | 256 | 10 | 1,048,576 | 16 | 4,294,967,296 | 21 | 4,398,046,511,104 |
        | 5 | 1,024 | 11 | 4,194,304 |  |  |  |  |
        - 파일 한 장당 100KB 저장공간 가정
            - 최대 확대시: 4.4조 * 100KB = 440PB
            - BUT 지구 표면 가운데 90%는 인간이 살지 않는 자연 그대로의 바다, 사막, 호수, 산간 지역임. 이는 높은 비율의 압축으로 대체가능 ⇒ 보수적으로 계산했을 때 80~90%의 저장 용량 절감 ⇒ 50PB
            - 확대 수준이 떨어질 때 마다 필요 타일 수가 1/4로 줄어듬
            - 모든 확대 수준의 타일 크기 합산 ⇒ **100PB**
        - 서버 대역폭
            - 서버가 처리해야 하는 요청
                - 경로 안내 요청
                - 위치 갱신 요청
                    - 실시간 교통 상황 데이터 계산에도 활용 가능
                - QPS 계산
                    - DAU: 10억
                    - 각 사용자의 주당 평균 경로 안내 사용 시간: 35분
                    - 하루에 사용되는 경로 안내시간: 50억분
                    - GPS위치 변경 데이터를 매초 서버에 전송할 때의 QPS: 300만
                    - GPS위치 변경 데이터를 모아두었다가 15초에 한 번씩 서버로 전송했을 때의 QPS: 20만

## 2단계: 개략적 설계안 제시 및 동의 구하기

### 개략적 설계안

- 제공 기능
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A332513e3-7db1-4a83-a852-bc270264d183%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8016-8c22-f8c1db12138f&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1310&userId=&cache=v2)
    
    - 위치 서비스
    - 경로 안내 서비스
    - 지도 표시
- 위치서비스
    - 클라이언트가 t초마다 자신의 위치를 전송한다 가정
        - 위치데이터로 ETA를 좀 더 정확히 산출 가능 & 위치에 따른 다른 최단 경로 갱신
        - 전송 데이터 활용 방안
            - 시스템 개선
            - 실시간 교통상황 모니터링
            - 신규 도로 및 폐쇄 도로 탐지
            - 개인화 경험 제공
    - 쓰기 요청 축소를 위해 일괄요청 활용
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A1276a8c6-ad28-43ff-b967-4fdd17ed63de%3Aimage.png?table=block&id=2fdbe4ca-d9d2-80be-8569-c29d8f9a0ace&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        
    - 높은 쓰기 요청 빈도 
    ⇒ 카산드라(Cassandra) DB 사용
    ⇒ 카프카(Kafka)등의 스트림 처리 엔진 활용
    - 통신 프로토콜
        - HTTP + keep-alive 옵션
        - 예시
            - POST /v1/locations
            인자: 
            locs: JSON으로 인코딩한 (위도, 경도, 시각) 순서쌍 배열
- 경로 안내 서비스
    - A→B 합리적으로 빠른 경로
    - 최단 시간 경로일 필요는 없으나 정확도는 보장되어야 함.
    - HTTP 요청 예시
        - GET /v1/nav?origin=1355+market+street,SF&destination=Disneyland
        - 응답 값
        
        ```json
        {
          'distance': {'text': '0.2 mi', 'value': 259},
          'duration': {'text': '1 min', 'value': 83},
          'end_location': {'lat': 37.4038943, 'lng': -121.9410454},
          'html_instructions': 'Head <b>northeast</b> on <b>Brandon St</b> toward <b>Lumin Way</b><div style="font-size:0.9em"> Restricted usage road</div>',
          'polyline': {'points': '_fhcFjbhgVuAwDsCal'},
          'start_location': {'lat': 37.4027165, 'lng': -121.9435809},
          'geocoded_waypoints': [
            {
              "geocoder_status": "OK",
              "partial_match": true,
              "place_id": "ChIJwZNMt1fawwRO2aVVVX2yKg",
              "types": ["locality", "political"]
            },
            {
              "geocoder_status": "OK",
              "partial_match": true,
              "place_id": "ChIJ3aPgQGtXawwRLYeiBMUi7bM",
              "types": ["locality", "political"]
            }
          ],
          'travel_mode': 'DRIVING'
        }
        ```
        
- 지도 표시
    - 클라이언트의 위치, 확대 수준에 따라 서버에서 지도 타일을 가져옴
    - 선택지1
        - 클라이언트의 위치, 확대 수준에 따라 지도타일을 즉석에서 생성
        - 단점: 지도 타일을 동적 생성하는 클러스터에 부하, 캐시 활용 어려움
    - 선택지2
        - 확대 수준별로 미리 생성해 둔 지도 타일(정적)을 클라이언트에 전달
            
            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A2890c567-630c-46ce-b4a2-d715fa9d8825%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8051-813b-fe3d09ac8fa8&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=540&userId=&cache=v2)
            
            1. 단말 사용자는 지오해시 URL을 통해 지도 타일 요청을 CDN에 전송
            2. 해당 타일이 CDN을 통해 서비스된 적 없는 경우, 서버에서 원본 파일을 가져와 캐시 ⇒ 사용자에게 반환
            3. 이후 같은 파일을 요청 ⇒ 캐시의 사본을 반환
        - 사용자 데이터 사용량
            - 사용자 이동속도: 3km/h
            - 확대 영역 사이즈: 200m * 200m
            - 평균 이미지 크기: 100KB
            - 1km * 1km: 이미지 25장 필요 ⇒ 2.5MB
            - 시간당 75MB, 분당 1.25MB
        - CDN 데이터 사용량
            - 매일 50억 분 가량의 경로 안내
            - 분당 1.25MB
            - ⇒ 50억 * 1.25MB = 6.25PB/일
            - 초당 전송 지도 데이터: 62,500MB
            - 전 세계의 200개 POP가 있다 가정.
            - 각 POP의 트래픽처리량: 312.5MB
        - 타일 요청 URL 예시 (지오해시사용)
            - https://cdn.map-provider.com/tiles/919hvu.png
            - 지오해시 계산을 클라이언트에서 할 때 단점: 인코딩 교체가 어려움
            - 해결방안: 서버에 (위도,경도,확대수준) ⇒ 타일 URL 변환 서비스 구현

## 3단계: 상세 설계

### 데이터 모델

- (1) 경로 안내 타일
    - 대용량(TB), 애플리케이션이 지속적으로 수집한 사용자 위치 데이터를 통해 끊임없이 개선
    - 데이터⇒경로 안내 타일 처리서비스: 오프라인 데이터 가공 파이프라인을 주기적으로 실행. ⇒경로 안내 타일로 변환
    - 그래프 데이터: 메모리에 **인접 리스트 형태**로 보관하는 것이 일반적
    - BUT 본 설계안의 그래프 데이터는 너무 방대함.
    - ⇒ S3 등의 객체 저장소에 파일을 보관(지오해시로 분류, 이진파일) ⇒ 경로 안내 서비스에서 캐싱(인접리스트)
- (2) 사용자 위치 데이터
    - 대량의 쓰기연산 처리, 수평적 규모확장 ⇒ 카산드라
    
    | user_id | timestamp | user_mode | driving_mode | location |
    | --- | --- | --- | --- | --- |
    | 101 | 1635740977 | active | driving | (20.0, 30.5) |
- (3) 지오코딩 데이터베이스
    - 주소 → (위도, 경도)
    - 많은 읽기 연산, 드문 쓰기 연산
    - 레디스 등의 키-값 저장소
- (4)미리 만들어둔 지도 이미지
    - 클라우드 저장소(Ex. 아마존 S3)
    - CDN에서 캐싱하여 사용

### 서비스

- 위치 서비스
    - 사용자 위치 데이터
        - 초당 백만 건의 업데이트 ⇒ 쓰기 연산 지원에 탁월한 DB 필요
        - 중요도: 데이터 일관성 < 가용성, 분할 내성
        - ⇒ NoSQL 키-값 데이터베이스 or 열-중심 데이터베이스가 적합
        - ⇒ 카산드라
        - 키 (user_id, timestamp)
            - user_id: 파티션 키
            - timestamp: 클러스터링 키
- 사용자 위치 데이터 이용
    - 이용방안
        - 신규 / 페쇄 도로 감지
        - 지도 데이터의 정확성 개선
        - 실시간 교통 현황 파악
    - 이용 방법
        - 사용자 위치: 메시지 큐(ex. 카프카)에 로깅
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A45320240-f9ee-4749-b668-12b16a2073ff%3Aimage.png?table=block&id=2fdbe4ca-d9d2-8064-97df-f4aecff4a392&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1250&userId=&cache=v2)
        

### 지도 표시

- 지도 타일 사전 계산
    - 최적화: 벡터 사용(WebGL 기술 사용)
        - 이미지 대신 경로(path)와 다각형(polygon)등의 벡터 정보 제공 ⇒ 수신 된 값으로 지도를 생성
        - 장점: 월등한 압축률, 매끄러운 지도 확대 경험

### 경로 안내 서비스

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A5503c12c-c195-4ac0-a3d5-dd5d5583fffe%3Aimage.png?table=block&id=2fdbe4ca-d9d2-80d6-8977-e931f9d1bd7a&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1360&userId=&cache=v2)

- 지오코딩 서비스 (구글 지오코딩 API의 사례)
    - 요청
        - [**https://maps.googleapis.com/maps/api/geocode/json?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA**](https://maps.googleapis.com/maps/api/geocode/json?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA)
    - JSON응답
    
    ```json
    {
    	"results" : [
    		{
    			"formatted_address" : "1600 Amphitheatre Parkway, Mountain View, CA 94043, USA",
    			"geometry" : {
    				"location" : {
    					"lat" : 37.4224764,
    					"lng" : -122.0842499
    				},
    				"location_type" : "ROOFTOP",
    				"viewport" : {
    					"northeast" : {
    						"lat" : 37.4238253802915,
    						"lng" : -122.0829009197085
    					},
    					"southwest" : {
    						"lat" : 37.4211274197085,
    						"lng" : -122.0855988802915
    					}
    				}
    			},
    			"place_id" : "ChIJ2eUgeAK6j4ARbn5u_wAGqWA",
    			"plus_code" : {
    				"compound_code": "CWC8+W5 Mountain View, California, United States",
    				"global_code": "849VCWC8+W5"
    			},
    			"types" : [ "street_address" ]
    		}
    	],
    	"status" : "OK"
    }
    ```
    
- 경로 계획 서비스
    
    현재 교통 상황, 도로상태에 입각 → 최적화 된 경로 제안
    
    - 최단 경로 서비스
        - 입력값: 출발지와 목적지의 위도/경도
        - 응답값: k개의 최단 경로
        - 교통, 도로 상황 고려x 도로 구조에만 의존
        - 그래프는 거의 정적이므로 캐시 추천
        - 객체 저장소에 저장된 경로 안내 타일에 대해 A* 경로 탐색 알고리즘 실행
            1. 입력값: 출발지와 목적지의 위도/경도 ⇒ 지오해시로 변환
            2. 지오 해시 ⇒ 경로 안내 타일 획득
            3. 출발지 타일→목적지 타일: 그래프 자료구조를 탐색 이 때, 필요한 주변 타일은 객체 저장소 or 캐시에서 가져 옴
            알고리즘을 통해 같은 지역의 다른 확대 수준 타일의 연결도 조회 가능
                
                ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A4cfd7809-0786-4e3a-aafe-856ea8d78448%3Aimage.png?table=block&id=2fdbe4ca-d9d2-806d-920e-e4c4b4c0d4ba&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1140&userId=&cache=v2)
                
            4. 3번 반복
    - 예상 도착 시간 서비스
        - 최단 경로 목록 수신 → ETA 서비스 호출
        - 기계 학습을 활용해 현재 교통 상황 및 과거 이력에 근거하여 예상 도착 시간을 계산
        - 앞으로 10-20분 뒤 교통 상황도 예측 필요 → 알고리즘 사용
    - 순위 결정 서비스
        - ETA 예상치 계산 → 순위 결정 서비스 호출
        - 사용자가 정의한 필터링 조건(유료 도로 제외, 고속도로 제외 등)을 적용 후 최단 시간 경로 K개 반환
- 중요 정보 갱신 서비스
    - 카프카 위치 데이터 스트림을 구독, 비동기적으로 중요 데이터를 업데이트해 최신화
    - 실시간 교통 정보, 경로 안내 타일(신규/폐쇄 도로) 등

### 적응형 ETA와 경로 변경 기능 추가

- Qustion
    - 현재 경로 안내를 받고 있는 사용자 추적방안
    - 수백만 경로 가운데 교통 상황 변화에 영향을 받는 경로, 사용자 계산 방법
- 재귀적 보관
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aba1b8ceb-7e13-48a0-a323-605ef27d6e62%3Aimage.png?table=block&id=2fdbe4ca-d9d2-801a-a09e-cf4a9882b41a&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1310&userId=&cache=v2)
    
    - 경로 안내를 받는 사용자(user_1) 각각의 현재 경로 안내타일(r_1) ⇒ 포함한 상위 타일(확대수준이 더 낮은타일) super(r_1)⇒ 상위 타일 super(super(r_1)) ⇒ … ⇒ 출발-목적지가 모두 포함된 타일
    재귀적으로 더하여 보관
    - user_1, r_1, super(r_1), super(super(r_1)), …
    - 특정 타일의 교통상황이 변화 → 경로안내 ETA가 달라지는 사용자: 데이터베이스 레코드 마지막 타일에 그 타일이 속하는 ㄴ사용자
    - 검색시간 복잡도: O(n)
    - 한계점: 교통 상황이 개선 되었을 때 알림 방식 구현 필요
        - 방법: 사용자가 ETA를 주기적으로 재계산
- 전송 프로토콜(서버 ⇒ 클라이언트)
    - 모바일 푸시 알림 (비추천)
        - 보낼 수 있는 메시지 크기가 매우 제한적(iOS의 경우 최대 4,096바이트)
        - 웹 어플리케이션 지원X
    - 롱 폴링 (비추천)
    - 웹소켓
        - 낮은 서버 부담
        - 양방향 통신 지원
    - 서버 전송 이벤트(SSE)
- 최종 설계안

![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aa4f086f2-94d6-436e-be17-5cb89f27a3a3%3Aimage.png?table=block&id=2fdbe4ca-d9d2-80e2-9bc4-ddd3a75df3b9&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1360&userId=&cache=v2)

## 추가 공부

### 1. Partition Key vs Clustering Key

| **구분** | **파티셔닝 (Partitioning)** | **클러스터링 (Clustering)** |
| --- | --- | --- |
| **정의** | 테이블을 물리적으로 여러 작은 조각(파티션)으로 분할 | 파티션 내에서 관련 데이터(유사한 값)를 물리적으로 한곳에 묶어 저장 |
| **작동 방식** | 주로 날짜나 범위(Range) 기준 (예: `WHERE date='2023-01-01'`) | 특정 열(Column)의 데이터를 정렬하여 저장 |
| **주 목적** | 불필요한 데이터 전체 스캔 방지, **스캔 데이터 양 감소** | 지정된 열의 데이터 검색 속도 향상, **관련 데이터 근접성 향상** |
| **구조** | 물리적인 디렉토리/파일 분리 (가장 굵은 단위) | 파티션 내부의 데이터 정렬 (세부 단위) |
| **사용 시기** | 특정 기간(Date)이나 범위 기반의 조회 시 | 파티셔닝 후에도 데이터가 너무 많거나 세부 필터링이 필요할 때 |

**Partition Key**: 데이터 분산 담당

- 해시값으로 **노드 결정** (같은 PK → 같은 노드)
- 파티션 크기 균등 유지 중요 (핫스팟 방지)
- 쿼리 시 **WHERE PK=값**으로 노드 1개만 타겟

**Clustering Key**: 파티션 내 정렬 담당

- 같은 파티션 안에서 **순서 결정** (ASC/DESC)
- 범위 쿼리 최적화 (예: 날짜 내림차순)
- PK 뒤에 오는 컬럼들

### 2. Raster 이미지 vs Vector 이미지

| 구분 | Raster (픽셀 기반) | Vector (수학 방정식 기반) |
| --- | --- | --- |
| 구성 | 픽셀 배열 (JPG, PNG) | 선·곡선·포인트 (SVG, AI) [adobe+1](https://www.adobe.com/creativecloud/file-types/image/comparison/raster-vs-vector.html) |
| 확대 축소 | 확대시 픽셀 깨짐 | 무한 확대해도 선명 |
| 용량 | 고해상도일수록 큼 | 항상 작음 |
| 적합 | 사진, 복잡한 색상 | 로고, 아이콘, 그래픽 |
| 편집 | 픽셀 단위 | 형태·색상 단위 |

예시: Raster=사진(확대시 흐림), Vector=지도(줌인해도 선명)

### 3. Long Polling & Server-Sent Events (SSE)

**Long Polling**: 클라이언트가 요청 → 서버 **기다림** → 데이터 생기면 응답 → 재요청

```
클라 GET /data ─────┐
서버          │← 데이터 대기 (수초~수십초)
             └──── 응답 → 클라 즉시 재요청

```

- 장점: 일반 HTTP, 즉시성 좋음
- 단점: 빈번한 연결 생성/종료, 서버 부하

**SSE**: 단일 연결로 서버→클라 **일방향 푸시**

```
클라 GET /events ───┐
서버         │← 지속 연결 유지
            └──── 데이터 생길 때마다 푸시

event: message\ndata: {"status": "done"}
```

- 장점: 자동 재연결, 브라우저 내장, 서버 효율적
- 단점: 단방향(클라→서버 별도), IE 미지원
