# (9) S3와 유사한 객체 저장소

S3: AWS가 제공하는 RESTful API 기반 인터페이스로 이용 가능한 객체 저장소

## 저장소 시스템 101

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A70ab953a-fd4e-4529-aba9-145395c10119%3Aimage.png?table=block&id=32dbe4ca-d9d2-8011-838c-f49a8be10bb8&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=940&userId=&cache=v2)

### 1.  블록 저장소

- 1960년대 등장
- 원시 블록(raw block)을 서버에 볼륨(volume) 형태로 제공.
- 가장 유연하고 융통성이 높은 저장소.
    - 서버: 원시 블록을 포맷 → 파일 시스템으로 이용하거나 애플리케이션에 블록 제어권을 넘김
    - 데이터베이스나 가상머신 엔진 등의 애플리케이션: 원시 블록을 직접 제어하여 최대한의 성능 사용.
- 블록 저장소: 서버에 물리적으로 직접 연결되는 저장소게 국한x, 고속 네트워크를 사용하거나 업계 표준 연결 프로토콜인 FC(Fibre Channel)이나 iSCSI를 통해 연결
- HDD나 SSD처럼 서버에 물리적적으로 연결되는 형태의 드라이브

### 2. 파일 저장소

- 블록 저장소 위에 구현. 파일과 디렉터리를 손쉽게 다루는데 필요한, 더 높은 수준의 추상화를 제공
- 데이터는 계층적으로 구성되는 디렉터리 안에 보관
- 가장 널리 사용되는 범용 저장소 솔루션.
- SMB/CIFS나 NPS와 같은 파일 수준 네트워크 프로토콜을 사용하면 하나의 저장소를 여러 서버에 동시에 붙일 수 있음
- 파일 저장소를 사용하는 서버: 블록을 직접 제어 및 볼륨 포맷등의 까다로운 작업을 실경쓸 필요X
- 단순함 → 폴더나 파일을 같은 조직 구성원에 공유하는 솔루션으로 활용하기 좋음

### 3. 객체 저장소

- 데이터 영속성을 높이고 대규모 애플리케이션을 지원, 비용을 낮추기 위해 의도적으로 성능을 희생
- 실시간으로 갱실할 필요가 없는 상대적으로 차가운(cold) 데이터 보관에 초점을 맞춤 → 다른 유형의 저장소에 비해 상대적으로 느림
- 데이터 아카이브나 백업에 주로 사용
- 모든 데이터를 수평적 구조 내에 객체로 보관(계층적 디렉터리 구조 x)
- 데이터 접근은 주로 RESTful API를 통함
- 예시) AWS S3, Azure Blob Stroge

### 비교

| 비교 항목 | 블록 저장소 | 파일 저장소 | 객체 저장소 |
| --- | --- | --- | --- |
| 저장된 내용의 변경 가능성 | Y | Y | N (직접 변경은 불가능하지만 객체 버전을 통해 새로운 버전의 객체를 추가하는 것은 가능) |
| 비용 | 고 | 중~고 | 저 |
| 성능 | 중~고 혹은 최상 | 중~고 | 저~중 |
| 데이터 일관성 | 강력 | 강력 | 강력 |
| 데이터 접근 | SAS, iSCSI, FC | 표준 파일 접근, CIFS/SMB, NFS | RESTful API |
| 규모 확장성 | 중 | 고 | 최상 |
| 적합한 응용 | 가상 머신(VM), 데이터베이스 같은 높은 성능이 필요한 애플리케이션 | 범용적 파일 시스템 접근 | 이진 데이터, 구조화되지 않은 데이터 |

### 용어 정리

- 버킷(bucket)
    - 객체를 보관하는 논리적 컨테이너
    - 버킷 이름은 전역적으로 유일해야 함
- 객체(object)
    - 버킷에 저장하는 개별 데이터
    - 데이터(혹은 payload)와 메타데이터로 구성
    - 데이터: 어느것도 가능
    - 메타데이터: 객체를 기술하는 이름-값 쌍의 집함
- 버전(versioning)
    - 한 객체의 여러 버전을 같은 버킷 안에 둘 수 있도록 하는 기능
    - 버킷마다 별도 설정 가능.
    - 객체 복구 기능 지원
- URI(Uniform Resource Identifier)
    - 객체 저장소: 버킷과 객체에 접근할 수 있도록 하는 RESTful API 제공
    - 각 개체는 해당 API URI를 통해 고유하게 식별 가능
- SLA(Service-Level Agreement)
    - 서비스 제공자 ↔ 클라이언트 사이에 맺어지는 계약
    - 예시 - 아마존 S3 Standard-IA 저장소 클래스의 경우
        - 여러 가용성 구역에 걸쳐 99.999999999%의 객체 내구성을 제공하도록 설계
        - 하나의 가용성 구역 전체가 소실되어도 데이터 복원 가능
        - 연간 99.9%의 가용성 제공

## 1단계: 문제 이해 및 설계 범위 확정

### 기능 요구사항

- 버킷 생성
- 객체 업로드 및 다운로드
- 객체 버전
- 버킷 내 객체 목록 출력 기능 (aws s3 ls 명령어와 유사해야 함)

### 비기능 요구사항

- 100PB 데이터
- 식스 나인(six nines, 99.9999%) 수준의 데이터 내구성
- 포 나인 (four nines, 99.99%) 수준의 서비스 가용성
- 저장소 효율성: 높은 수준의 안정성과 성능은 보증하되 저장소 비용은 최대한 낮춰야 됨

### 대략적인 규모 추정

- 디스크 용량: 객체 크기가 다음 분포를 따른다 가정
    - 객체 중 20%는 10MB 미만
    - 60%는 1MB~64MB
    - 나머지 20%: 6MB 이상
- IOPS: SATA 인터페이스를 탑재, 7200rpm을 지원하는 하드 디스크 하나가 초당 100~150회의 임의 데이터 탐색을 지원할 수 있다고 가정(100~150IOPS)
- 객체 유형별 중앙값을 사용
    - 소형 객체: 0.5MB
    - 중형 객체: 32MB
    - 대형 객체: 200MB
- 40%의 저장 공간 사용률을 유지하는 경우 저장소에 수용가능한 객체 수
    - 100PB = 100 * 1000 * 1000 * 1000MB = 10^11MB
    - (10^11 * 0.4) / (0.2 * 0.5MB + 0.6 * 32MB + 0.2 * 200MB) = 6억 8천만 개 객체
    - 모든 객체의 메타데이터 크기: 대략 1KB 가정 ⇒ 메타 데이터 저장공간: 0.68TB

## 2단계: 개략적 설계안 제시 및 동의 구하기

객체 저장소의 몇가지 속성

- 객체 불변성
    - 객체 저장소에 보관되는 객체들은 변경이 불가능. 삭제 후 새 버전 객체로 완전히 대체만 가능
- 키-값 저장소
    - 해당 객체의 URI를 사용해 객체를 가져올 수 있음. 키(URI)-값(데이터)
    
    ```json
    요청:
    GET /bucket1/object1.txt HTTP/1.1
    
    응답:
    HTTP/1.1 200 OK
    Content-Length: 4577
    
    [해당 객체의 데이터 4567 바이트]
    ```
    
- 저장은 1회, 읽기는 여러번
    - 객체 저장소에 대한 요청 가운데 95%가 읽기 요청 (출처: 링크드인)
- 소형 및 대형 객체 동시 지원
- UNIX 파일 시스템의 설계철학과 비슷
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A2466bbf5-ec4e-4350-9e98-f8a45f14dddb%3Aimage.png?table=block&id=32dbe4ca-d9d2-8000-9c02-f6f360b8f072&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    - UNIX의 경우: 파일을 로컬 파일 시스템에 저장 시
        - 파일 이름: 아이노드(inode)라 불리는 자료구조에 보관, 파일의 데이터가 실제로 보관 된 디스크 상의 위치를 가리키는 파일 블록 포인터 목록 또한 저장
        - 파일 데이터: 데스크의 다른 위치에 저장
    - 객체 저장소의 경우
        - 메타데이터 저장소: 아이노드와 비슷한 역할
            - 파일 블록 포인터 대신 네트워크로 데이터 저장소에 보관된 개체를 요청할 때 필요한 ID 저장
        - 데이터 저장소: 하드 디스크

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A48b2c3ae-5951-46df-8512-29b7ee164632%3Aimage.png?table=block&id=32dbe4ca-d9d2-80d4-9d06-c347b74de328&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=1430&userId=&cache=v2)

### 개략적 설계안

![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aeba14f64-c1c1-48e9-b598-973a9c02e191%3Aimage.png?table=block&id=32dbe4ca-d9d2-800f-a3f1-fb2cc0b9c15c&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=940&userId=&cache=v2)

- 로드밸런서: RESTful API에 대한 요청을 API 서버들에 분산하는 역할
- API 서비스: IAM(Identity & Access Management) 서비스, 메타데이터 서비스, 저장소 서비스에 대한 호출을 조율하는 역할. 무상태 서비스
- IAM 서비스: 인증(authentication), 권한 부여(authorization), 접근 제어(access control) 등을 중앙에서 맡아 처리함.
    - 인증: 호출 주체가 누구인지 확인하는 작업
    - 권한 부여: 인증된 사용자가 어떤 작업을 수행할 수 있는 지 검증하는 과정
- 데이터 저장소: 실제 데이터를 보관하고, 필요할 때마다 읽어가는 장소. 모든 데이터 관련 연산은 객체 ID(UUID)를 통함
- 메타데이터 저장소: 객체 메타데이터를 보관하는 장소

### 객체 저장소가 지원해야하는 가장 중요한 작업흐름

- 객체 업로드
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A56108c64-57b5-4841-9584-8df2f93456aa%3Aimage.png?table=block&id=32dbe4ca-d9d2-800a-a707-c203936ae7a7&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    1. 클라이언트: bucket-to-share 버킷을 생성하기 위한 HTTP PUT 요청을 전송 → API 서비스로 전달
    2. API 서비스: IAM을 호출, 해당 사용자가 WRITE 권한을 가졌는지 확인
    3. API 서비스: 메타데이터 데이터베이스 호출, 버킷 정보를 등록 → 버킷 생성 알림 메시지가 클라이언트에 전송
    4. 클라이언트: 버킷 생성 완료 후 script.txt객체를 생성하기 위한 HTTP PUT 요청
    5. API 서비스:  해당 사용자의 신원 및  WRITE 권한을 가졌는지 확인
    6. API 서비스: HTTP PUT 요청 몸체에 실린 객체 데이터를 데이터 저장소에 전송. 
    데이터 저장소: 해당 데이터를 객체로 저장 후 객체 UUID 반환
    7. API 서비스: 메타데이터 저장소를 호출, 새로운 항목을 등록. object_id(UUID), bucket_id(해당 객체가 속한 버킷), object_name 등의 정보 포함
    
    | object_name | object_id | bucket_id |
    | --- | --- | --- |
    | script.txt | 239d6b00-2d06-014e-82a1-f59e45f1ae-52948660-02f6-014e | 5f4ea1f5-9f50-5e41ae- |
    - 객체 업로드 API 호출
        
        ```java
        PUT /bucket-to-share/script.txt HTTP/1.1
        Host: foo.s3.example.org
        Date: Sun, 12 Sep 2021 17:51:00 GMT
        Authorization: [권한 문자열]
        Content-Type: text/plain
        Content-Length: 4567
        x-amz-meta-author: Alex
        
        [객체 데이터 4567 바이트]
        ```
        
- 객체 다운로드
    - 디렉터리 계층 구조 지원X, 다만 버킷 이름 - 객체 이름 연결 시 폴더 구조를 흉내 내는 논리적 계층 생성 가능
        - 예) 객체 이름을 bucket-to-share/script.txt와 같이 작성
    - 객체 다운로드 코드
    
    ```java
    GET /bucket-to-share/script.txt HTTP/1.1
    Host: foo.s3.example.org
    Date: Sun, 12 Sept 2021 18:30:01 GMT
    Authorization: [권한 문자열]
    ```
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ad66817a3-07f8-40cf-b7bf-cce4c4794457%3Aimage.png?table=block&id=32dbe4ca-d9d2-806f-89ca-c3df26971a0c&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    1. 클라이언트: GET /bucket-to-share/script.txt 요청을 로드밸런서로 보냄
    로드밸런서: 해당 요청을 API 서버로 보냄
    2. API 서비스: IAM을 질의, 사용자가 해당 버킷에 READ 권한을 가지고 있는지 확인
    3. API 서비스: 해당 객체의 UUID를 메타 데이터 저장소에서 가져옴
    4. API 서비스: UUID로 데이터 저장소에서 객체 데이터를 읽음
    5. API 서비스: HTTP GET요청에 대한 응답으로 객체 데이터를 반환
- 객체 버전 및 버킷 내 모든 객체 목록의 출력 (상세 설계에서 계속)

## 3단계: 상세설계

---

### 데이터 저장소

- 객체 업로드/다운로드를 처리하기 위해 API 서비스와 데이터 저장소가 연동하는 방법
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aa1660675-63b6-4f67-9555-4c421ef87c30%3Aimage.png?table=block&id=32ebe4ca-d9d2-8059-9372-c825b500d4a5&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    

### 데이터 저장소의 개략적 설계

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A11bbf9ce-8154-4737-ab8a-ebd44b9d5b82%3Aimage.png?table=block&id=32ebe4ca-d9d2-8054-a42f-e467ce6b651e&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=940&userId=&cache=v2)

- 데이터 라우팅 서비스
    - 데이터 노드 클러스터에 접근하기 위한 RESTful 또는 gRPC 서비스를 제공. 무상태 서비스
    - 배치 서비스를 호출, 데이터를 저장할 최적의 데이터 노드를 판단
    - 데이터 노드에서 데이터를 읽어 API서비스에 반환
    - 데이터 노드에 데이터 기록
- 배치 서비스
    - 어떤 데이터 노드에 데이터를 저장할지 결정하는 역할
    - 내부적으로 가상 클러스터 지도를 유지.
        - 가상 클러스터 지도: 클러스터의 물리적 혀앙 정보가 보관
        - 배치 서비스: 지도에 보관되는 데이터 노드의 위치정보를 이용하여 데이터 사본이 물리적으로 다른 위치에 놓이도록 (높은 데이터 내구성 제공)
    - 가상 클러스터 지도
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aee2c5454-82c6-4987-bc2a-f5b399bf83d8%3Aimage.png?table=block&id=32ebe4ca-d9d2-80f8-b5ca-ca235865b9ee&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
    - 배치 서비스: 모든 데이터 노드와 지속적으로 박동 메시지를 주고 받으며 상태를 모니터링 함. → 15초의 유예기간 동안 박동 메시지에 응답하지 않는 데이터 노드를 죽은(down)노드로 표시
    - 5~7개의 노드를 가지는 배치 서비스 클러스터: 팩서스(Paxos)나 래프트(Raft) 같은 합의 프로토콜을 사용하여 구축할 것을 권장.
        - 일부 노드에 장애가 생겨도 건강한 노드 수가 클러스터 크기의 절반 이상이면 서비스를 지속할 수 있도록 보장
- 데이터 노드
    - 실제 객체가 보관되는 장소
    - 다중화 그룹: 여러 노드에 데이터를 복제해 데이터의 안정성과 내구성을 보증
    - 각 데이터 노드: 배치 서비스에 주기적으로 박동메시지를 보내는 서비스데 데몬이 돔.
    - 박동 메시지로 전달되는 정보
        - 해당 데이터 노드에 부착된 디스크 드라이브(HDD/SSD) 수
        - 각 드라이브에 저장된 데이터의 양
    - 배치 서비스가 못 보던 데이터 노드에서 박동 메시지를 처음 받을 경우
        - 해당 노드에 ID를 부여, 가상 클러스터 지도에 추가
        - 아래 정보 반환
            - 데이터 노드에 부여한 고유 식별자
            - 가상 클러스터 지도
            - 데이터 사본을 보관할 위치

### 데이터 저장 흐름

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A3293ecb4-4ceb-40bb-9472-313a11dc6bf9%3Aimage.png?table=block&id=32ebe4ca-d9d2-80da-a637-f07a42908ece&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=940&userId=&cache=v2)

1. API 서비스: 객체 데이터를 데이터 저장소로 포워드
2. 데이터 라우팅 서비스: 해당 객체에 UUID 할당, 배치 서비스에 객체를 보관할 데이터 노드를 질의
배치 서비스: 가상 클러스터 지도를 확인하여 주 데이터 노드를 반환
    - 주로 안정해시를 사용
3. 데이터 라우팅 서비스: 저장할 데이터를 UUID와 함께 주 데이터 노드에 직접 전송
4. 주 데이터 노드: 데이터 저장 및 부 데이터 노드에 다중화. 성공 후 데이터 라우팅 서비스에 응답 전송
    - 모든 노드에 강력한 일관성 보장 → 지연 시간 측면에서는 ㄴ손해.
    - 데이터 일관성과 지연 시간 사이의 타협적 관계
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A2bcf1595-6b29-4342-9c67-dff9ecf60cf3%3Aimage.png?table=block&id=32ebe4ca-d9d2-80b2-9bf2-ef05af7c128f&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
5. 라우팅 서비스: API 서비스에 UUID 반환

### 데이터는 어떻게 저장되는가

- 각각의 객체를 개별 파일로 저장하는 방법: 파일이 많아지면 생기는 문제
    - 낭비되는 데이터 블록 수가 늘어남
    - 아이노드의 용량 한계를 초과하는 문제
- 작은 객체들은 큰 파일 하나로 모아서 저장하는 방법
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A7e0a2980-661a-40dc-adb2-4d3973d853f8%3Aimage.png?table=block&id=32ebe4ca-d9d2-80f1-a4b6-ebc2c10f9b1a&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    - 개념적으로는 WAL(Write-Ahead Log)와 같이 객체를 저장할 때 이미 존재하는 파일에 추가하는 방식. 용량 임계치에 도달한 파일은 읽기 전용으로 변경하고 새로운 파일 생성
    - 순차적 쓰기 연산, 객체를 파일에 일렬로 저장 ⇒ 다중 코어를 가진 현대식 서버 시스템의 경우 대역폭이 크게 감소 → 코어별로 전담 읽기/쓰기 파일을 두어 해결

### 객체 소재 확인

- 파일에 다중 객체 저장 방법 선택 시 객체 위치를 찾을 때 필요한 정보
    - 객체가 보관된 데이터 파일
    - 데이터 파일 내 객체 오프셋
    - 객체 크기
- 소재 확인 작업에 필요한 데이터베이스 스키마
    
    
    | **object_mapping** |
    | --- |
    | **object_id**
    
    file_name
    start_offset
    object_size |
    
    | 필드 | 설명 |
    | --- | --- |
    | object_id | 객체의 UUID |
    | file_name | 객체에 저장한 파일의 이름 |
    | start_offset | 파일 내 객체의 시작 주소 |
    | object_size | 객체의 바이트 단위 크기 |
- 정보 저장 방법
    1. RocksDB와 같은 파일 기반 키-값 저장소를 이용
        - SSTable에 기반한 방법. 
        높은 쓰기 성능, 낮은 읽기 성능
    2. 관계형 데이터베이스를 이용 👍🏻
        - 보통 B+트리 기반저장 엔진을 이용. 
        높은 읽기 성능, 낮은 쓰기 성능.
- 데이터 노드에 저장되는 위치 데이터를 다른 데이터 노드와 공유할 필요X → 데이터 노드마다 관계형 데이터 베이스 설치. (ex. SQLite사용. 파일 기반 관계형 데이터 베이스)

### 개선된 데이터 저장 흐름

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A78a8c706-5ad7-4642-9695-827293018251%3Aimage.png?table=block&id=32ebe4ca-d9d2-8071-a3c5-f52435141155&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=940&userId=&cache=v2)

1. API 서비스: 새로운 객체(4)를 저장하는 요청을 데이터 노드 서비스에 전송
2. 데이터 노드 서비스: 객체4를 읽기-쓰기 파일의 마지막 부분에 추가
3. 해당 객체에 대한 새로운 레코드를 object_mapping 테이블에 추가
4. 데이터 노드 서비스: API 서비스에 해당 객체의 UUID 반환

### 데이터 내구성

- 하드웨어 장애와 장애 도메인
    - 내구성을 높이는 검증된 방법: 데이터를 여러 대의 하드 드라이브에 복제. (→ 장애 발생 시 전체 데이터 가용성에 영향X)
    - 가정: 회전식 드라이브의 연간 장애율 0.81%
        - →데이터 3중 복제시 내구성: 1 - 0.0081^3 =~ 0.999999
    - 현대적인 데이터센터: 서버는 보통 rack에 설치됨
        - 랙: 특정한 열(row)/층(floor)/방(room)에 위치
        - 랙 내의 모든 서버: 랙에 부설된 네트워크 스위치, 파워 서플라이, 마더보드, CPU, HDD 드라이브 공유 → 해당 랙이 나타내는 장애 도메인 안에 있음
    - 대규모의 장애 도메인 사례: 데이터센터의 가용성 구역(Availability Zone, AZ)
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A48cad286-b3b4-45ee-9429-6d90e45c5f4b%3Aimage.png?table=block&id=32ebe4ca-d9d2-8007-990e-e2c25450ea01&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
        - 가용성 구역: 다른 데이터 센터와 물리적 인프라를 공유하지 않는 독립적 데이터 센터
        - 데이터를 여러 AZ에 복제 → 장애 여파를 최소화
- 소거 코드
    - 데이터를 작은 단위로 분할, 다른 서버에 배치. 일부가 소실되었을 때 복구하기 위한 패리티(parity) 정보를 만들어 중복성(redundancy) 확보
    - 소거 코드를 활용한 데이터 복구
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A5089ec36-771c-41ed-b378-5c5396caaaad%3Aimage.png?table=block&id=32ebe4ca-d9d2-80d8-927c-f1cafc05007d&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
        1. 데이터를 네 개의 같은 크기 단위로 분할(d1, d2, d3, d4)
        2. 수학 공식을 사용하여 패리티 p1, p2를 계산
            
            단순예제: p1 = d1 + 2*d2 - d3 + 4*d4
            p2 = -d1 + 5*d2 + 3d - 3*d4
            
        3. 데이터 d3, d4가 장애로 소실
        4. 남은 값 d1, d2, p1, p2와 패리티 계산 수식을 활용해 d3, d4 복구
    - (8 + 4) 소거 코드 예시
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A3394352b-3b87-431d-a0fa-ee1e1d962d93%3Aimage.png?table=block&id=32ebe4ca-d9d2-8017-8f6d-d7d433b3a103&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
        - 최대 4대 노드의 장애 발생 시 복원 지원
    - 소거 코드의 단점: 최대 8개의 건강한 노드에서 데이터를 읽어와야함. (다중화: 하나의 건강한 노드면 충분)
    - 높은 응답지연, 내구성 향상, 낮은 저장소 비용
        - 저장소 비용 계산
            
            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aae16e852-df40-4638-b6eb-c82cc5e5d0d7%3Aimage.png?table=block&id=32ebe4ca-d9d2-801f-81c3-c40364ee1514&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=710&userId=&cache=v2)
            
            - 2개 데이터 블록, 하나의 패리티 블록 → 오버헤드: 50%
            - 3중 복제 다중화 방식 채택 시 오버헤드: 200%
    - 소거 코드의 데이터 내구성
        - 노드의 연간 장애 발생률: 0.81% 가정 → 11-nine 내구성 달성 (출처: 백블레이즈 사)
- 다중화 vs 소거코드
    
    
    |  | 다중화 | 소거코드 | 승자 |
    | --- | --- | --- | --- |
    | 내구성 | 99.9999% (3중 복제의 경우) | 99.999999999% (8+4 소거 코드를 사용하는 경우) | 소거 코드 |
    | 저장소 효율성 | 200%의 저장 용량 오버헤드 | 50%의 저장 용량 오버헤드 | 소거 코드 |
    | 계산 지원 | 계산이 필요 없음. | 패리티 계산에 많은 계산 자원 소모 | 다중화 |
    | 쓰기 성능 | 데이터를 여러 노드에 복제. 추가로 필요한 계산x | 데이터를 디스크에 기록하기 전 패리티 계산이 필요. 쓰기 연산 응답지연 증가 | 다중화 |
    | 읽기 성능 | 장애가 발생하지 않은 노드에서 데이터를 읽음 | 데이터를 읽어야 할 때마다 클러스터 내의 여러 노드에서 데이터를 가져와야 함. 장애가 발생한 경우 빠진 데이터를 먼저 복원 해야함 → 지연시간 증가 | 다중화 |
    - 응답 지연이 중요한 애플리케이션: 다중화
    - 저장소 비용이 중요한 애플리케이션: 소거 코드

### 정확성 검증

- 대규모 시스템의 경우 데이터 훼손 문제는 디스크 뿐만 아니라 메모리까지도 조종 일어남.
- 메모리 데이터 훼손 문제 > 프로세스 경계에 데이터 검증을 위한 체크섬(checksum)을 두어 해결
- 체크섬: 데이터 에러를 발견하는 데 사용되는 작은 크기의 데이터 블록
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A122d8b88-7f98-4ced-aed5-c5aea9337532%3Aimage.png?table=block&id=32ebe4ca-d9d2-807b-8a83-f9e6451c4e2e&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    - 새로 계산한 체크섬이 원본 체크섬과 다르면 데이터가 망가진 것
    - 같은 경우에는 아주 높은 확률로 온전한 데이터.
- 체크섬 알고리즘: MD5, SHA1, HMAC …

![image.png](https://possible-telephone-196.notion.site/image/attachment%3Aaa49f070-5d3c-4bc2-b2b0-f18d2c37ccf7%3Aimage.png?table=block&id=32ebe4ca-d9d2-801d-95d9-f7739411dcf4&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=940&userId=&cache=v2)

- 파일을 읽기 전용으로 전환하기 직전, 전체 파일의 체크섬을 계산한 후 파일 말미에 추가
- (8+4) 소거 코드와 체크섬 확인 매커니즘을 동시에 활용하는 경우에 객체 데이터를 읽을 시 수행하는 절차
    1. 객체 데이터와 체크섬을 가져옴
    2. 수신된 데이터의 체크섬을 계산
        1. 두 체크섬이 일치 > 데이터에 에러가 없다고 간주
        2. 두 체크섬이 불일치 → 데이터가 망가졌다 판단. 다른 장애 도메인에서 데이터를 가져와 복구 시도
    3. 데이터 8조각을 전부 수신할 때까지 1과 2를 반복. → 원본 객체를 복원 후 클라이언트에게 전송

### 메타데이터 데이터 모델

- 해당 데이터베이스의 스키마는 3가지 질의를 지원할 수 있어야 함
    1. 객체 이름으로 객체 ID 찾기
    2. 객체 이름에 기반하여 객체 삽입 또는 삭제
    3. 같은 접두어를 갖는 버킷 내의 모든 객체 목록 확인
    
    | buket |
    | --- |
    | bucket_name
    bucket_id
    owner_id
    enable_versioning |
    
    | buket |
    | --- |
    | bucket_name
    object_name
    object_version
    object_id |
- bucket 테이블의 규모 확장
    - 보통 한 사용자가 만들 수 있는 버킷의 수에는 제한이 있음.
    - 전체 테이블의 크기는 최신 데이터베이스 서버 한 대에 충분히 저장할 수 있으나 읽기 요청 처리를 위한 CPU 용량이나 네트워크 대역폭의 한계가 있음 → 데이터베이스 사본을 만들어 부하를 분산
- object 테이블의 규모 확장
    - 객체의 메타데이터를 보관
    - 객체 메타데이터를 데이터베이스 한 대에 보관하기 불가능. 샤딩 필요
    - bucket_id를 기준으로 삼아 같은 버킷 내 객체는 같은 샤드에 배치되도록 함 → 단점: 버킷 안에 수십억 개의 객체가 있는 핫스팟 샤드 지원 불가
    - object_id를 기준으로 샤딩: 부하를 균등하게 분산.→ 단점: 질의 1과 2를 효율적으로 지원 불가(URI를 기준으로 질의)
    - bucket_name과 object_name을 결합하여 샤딩 → 세번째 질의가 다소 애매함
    
    ### 버킷 내 객체 목록 확인
    
    - 객체는 s3://<버킷 이름>/<객체 이름>의 수평적 경로로 접근.
    - ex) s3://mybucket/abc/d/e/f/file.txt
        - mybucket: 버킷 이름
        - abc/d/e/f/file.txt: 객체 이름
    - 접두어: 객체 이름의 시작부분 문자열. 디렉터리와 비슷하게 데이터 정리 가능
        - abc/d/e/f: 접두어
    - 목록 출력 명령어
        1. 어떤 사용자가 가진 모든 버킷 목록 출력
            
            ```bash
            aws s3 list-buckets
            ```
            
        2. 주어진 접두어를 가진, 같은 버킷 내 모든 객체 목록 출력
            
            ```bash
            aws s3 ls s3://mybucket/abc/
            ```
            
            주어진 접두어 다음에 오는 슬래시 나머지 부분이 잘림.
            
            예시
            
            ```bash
            --버킷 내 객체 목록
            CA/cities/losangeles.txt
            CA/cities/sanfranciso.txt
            NY/cities/ny.txt
            federal.txt
            ```
            
            ‘/’를 접두어로 주고 버킷을 질의 결과
            
            ```bash
            -- 질의
            aws s3 ls s3://mybucket/
            
            --결과
            CA/
            NY/
            federal.txt
            
            -- CA/와 NY/를 접두어로 갖는 모든 객체가 전부 CA/와 NY/로만 표시
            ```
            
        3. 주어진 접두어를 가진 같은 버킷 내 모든 객체를 재귀적으로 출력
            
            ```bash
            aws s3 ls s3://mybucket/abc/ --recursive
            
            --결과
            CA/cities/losangeles.txt
            CA/cities/sanfranciso.txt
            ```
            

### 단일 데이터베이스 서버

- 단일 데이터베이스 서버로 목록 출력 명령어를 지원하는 방법
    
    ```sql
    --특정 사용자가 가진 모든 버킷 출력
    SELECT * FROM bucket WHERE owner_id = {id}
    
    -- 같은 접두어를 갖는, 버킷 내 모든 객체를 출력
    -- bucket_id의 값이 123이며, abc/를 공통 접두어로 갖는 모든 객체를 찾음
    SELECT * FROM object
    WHERE bucket_id = "123" AND object_name LIKE 'abc/%'
    ```
    

### 분산 데이터베이스

- 메타데이터 테이블을 샤딩하면 어떤 샤드에 데이터가 있는지 모름 → 목록 출력 기능을 구현하기 어려움
- ⇒ 해결책: 검색 질의를 모든 샤드에 돌린 다음 결과를 취합
    1. 메타데이터 서비스는 모든 샤드에 다음 질의를 돌림
        
        ```sql
        SELECT * FROM object
        WHERE bucket_id = "123" AND object_name LIKE 'a/b/%'
        ```
        
    2. 메타데이터 서비스는 각 샤드가 반환한 객체들을 취합, 그 결과를 호출 클라이언트에 반환
    - 단점: 페이징 기능 구현이 복잡함.
    - 단일 데이터베이스 서버의 페이징 구현 법
        
        ```sql
        -- 페이지당 10개 객체 목록을 반환하는 SELECT 질의
        SELECT * FROM object
        WHERE bucket_id = "123" AND object_name LIKE 'a/b/%'
        ORDER BY object_name OFFSET 0 LIMIT 10
        
        --- 두번째 페이지에 대한 질의
        SELECT * FROM object
        WHERE bucket_id = "123" AND object_name LIKE 'a/b/%'
        ORDER BY object_name OFFSET 10 LIMIT 10
        -
        ---전체 객체목록의 순회가 끝났음을 알리는 특별한 커서를 서버가 결과를 실어 보내면 끝
        
        ```
        
    - 데이터베이스 샤딩 시 페이지 나눔 기능을 구현하기 어려운 이유
        - 객체가 나뉘어있어 샤드마다 반환하는 객체 수가 제 각각
        - 애플리케이션 코드는 모든 샤드의 질의 결과를 받아 취합한 다음 정렬하여 그 중 10개만 추려야 함 → 반환 페이지에 포함되지 못한 객체는 다음에 다시 고려 → 샤드마다 추적해야하는 오프셋이 달라짐 → 샤드가 nnn개면 어플리케이션이 추적해야하는 오프셋도 nnn개
        - 해결법: 버킷ID로 샤딩하는 별도 테이블에 목록 데이터를 비정규화 하는 것도 방법 (최적화 포기)

### 객체 버전

![image.png](https://possible-telephone-196.notion.site/image/attachment%3A39b361ad-f621-49e1-b45e-f8ea89b82dbd%3Aimage.png?table=block&id=32fbe4ca-d9d2-804e-8cee-f1d803fddb3c&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=940&userId=&cache=v2)

1. 클라이언트: script.txt객체를 업로드하기 위한 HTTP PUT 요청을 보냄
2. API 서비스: 사용자의 신원을 확인, 해당 사용자가 해당 버킷에 쓰기 권한을 가지고 있는지 확인
3. 문제가 없으면 API 서비스: 데이터를 데이터 저장소에 업로드.
데이터 저장소: 새 객체를 만들어 데이터를 영속적으로 저장, API 서비스에 새로운 UUID를 반환
4. API 서비스: 메타데이터 저장소를 호출, 새 객체의 메타데이터 정보를 보관
5. 메타데이터 저장소의 객체 테이블에는 object_version 이라는 버전기능을 지원하는 열이 있음.
    - 기존 레코드를 덮어쓰는 대신, bucket_id와 object_name은 같지만 object_id(UUID), object_version(TIMEUUID)은 새로운 값인 레코드를 추가.
    - 같은 object_name을 갖는 항목 가운데 object_version에 기록된 TIMEUUID값이 가장 큰 것이 최신 버전
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A9381ad5a-42f0-4c1d-a53f-75804856f8d1%3Aimage.png?table=block&id=32fbe4ca-d9d2-80e4-a099-f972e9968e17&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
    - 삭제 시 버전기능
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A8407bca9-0c45-4013-a3a1-c99714488d30%3Aimage.png?table=block&id=32fbe4ca-d9d2-80bb-b840-f46d026a5cc1&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
        - 삭제 표식 삽입으로 TIMEUUID 검색 제외

### 큰 파일의 업로드 성능 최적화

- 큰 객체를 통째로 버킷에 직접 업로드 시 단점: 오랜 시간, 업로드 중간 네트워크 오류 발생 시 처음부터 다시 시도해야 함
- 큰 객체를 작게 쪼갠 다음 독립적으로 업로드하는 방법: → 모든 조각이 업로드되고 나면 조각을 모아서 원본 객체를 복원 (multipart 업로드방법)
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A82c1ac65-53f2-49d1-9d38-afd21b73aaff%3Aimage.png?table=block&id=32fbe4ca-d9d2-80f9-8aa4-ff317cf51b54&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    1. 클라이언트: 멀티파트 업로드를 시작하기 위해 객체 저장소 호출
    2. 데이터 저장소: uploadID 반환. (업로드를 유일하게 식별할 ID)
    3. 클라이언트: 파일을 작은 객체로 분할, 업로드 시작
    가정: 파일 크기: 1.6GB, 8조각으로 나눌 시 조각당 200MB
    uploadID와 함께 데이터저장소에 올림
    4. 데이터 저장소: 조각 하나가 업로드 될 때 마다 ETag를 반환. 
    ETag: 조각에 대한 MD5 해시 체크섬. 멀티파트 업로드가 정상적으로 되었는지 검사 용도
    5. 클라이언트: 모든 조각 업로드 후 멀티파트 업로드 요청 전송(uploadID, 조각 번호 목록, ETag 목록 포함)
    6. 데이터 저장소: 전송 받은 조각 번호 목록을 사용해 원본 객체를 복원. (수 분 소요 가능) 완료 후 성공 메시지 반환 
    - 객체 조립이 끝난 뒤의 조각들은 삭제하여 저장 용량을 확보

### 쓰레기 수집 (garbage collection)

- 더이상 사용되지 않는 데이터에 할당된 저장 공간을 자동으로 회수하는 절차
- 9장 시스템에서 쓰레기 데이터가 생기는 경우
    - 객체의 지연된 삭제(lazy object deletion): 삭제했다고 표시하지만 실제로 지우지 않는 경우
    - 갈 곳 없는 데이터(orphaned data): 반쯤 업로드 된 데이터, 혹은 취소 된 멀티파트 업로드 데이터
    - 훼손 된 데이터(corrupted data): 체크섬 검사에 실패한 데이터
- 쓰레기 수집기
    - 정리 매커니즘을 주기적으로 실행하여 삭제
    - 사용되지 않는 사본에 할당 된 저장 공간을 회수
- 쓰레기 수집기의 정리 매커니즘
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A6d135bb1-a800-47f2-8add-55b794e83188%3Aimage.png?table=block&id=32fbe4ca-d9d2-8028-87f0-ed63daf4abf3&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    1. 쓰레기 수집기: /data/b의 객체를 /data/d로 복사. 삭제된 객체 플래그 값이 참인 객체(2, 5)는 제외
    2. 쓰레기 수집기: 모든 객체 복사 후 object_mapping 테이블을 갱신. 
        - ex: 객체 3: object_id, object_size 유지. file_name, start_offset 수정
        - 데이터의 일관성을 보장하기 위해 file_name, start_offset에 대한 갱신 연산은 같은 트랜잭션 안에서 수행하는 것이 바람직
    
    

---

## 저장소 시스템 101
![image.png](https://possible-telephone-196.notion.site/image/attachment%3A70ab953a-fd4e-4529-aba9-145395c10119%3Aimage.png?table=block&id=32dbe4ca-d9d2-8011-838c-f49a8be10bb8&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=940&userId=&cache=v2)

- **블록**: 최고 성능, DB/VM에 최적 (고비용)
- **파일**: 범용 공유, 폴더 구조 (중비용)
- **객체(S3)**: 대용량 저비용, REST API (저비용/최고 확장성)

### 핵심 비교표

| 비교항목 | 블록 저장소 | 파일 저장소 | 객체 저장소(S3) |
| --- | --- | --- | --- |
| **변경 가능성** | ✅ Y | ✅ Y | ❌ N (버전으로 대체) |
| **비용** | 고 | 중~고 | **저** ← 승자 |
| **성능** | 중~고, 최상 | 중~고 | 저~중 |
| **접근 방식** | SAS/iSCSI/FC | SMB/NFS | **REST API** |
| **확장성** | 중 | 고 | **최상** ← 승자 |
| **용도** | DB/VM | 파일 공유
범용적 파일 시스템 접근 | **백업/아카이브
이진 데이터, 구조화x 데이터** |

**핵심 메시지**: "S3는 비용/확장성 최강이지만 실시간 고성능엔 부적합!"

## 다중화 vs 소거코드

|  | 다중화 | 소거코드 | 승자 |
| --- | --- | --- | --- |
| 내구성 | 99.9999% (3중 복제의 경우) | 99.999999999% (8+4 소거 코드를 사용하는 경우) | 소거 코드 |
| 저장소 효율성 | 200%의 저장 용량 오버헤드 | 50%의 저장 용량 오버헤드 | 소거 코드 |
| 계산 지원 | 계산이 필요 없음. | 패리티 계산에 많은 계산 자원 소모 | 다중화 |
| 쓰기 성능 | 데이터를 여러 노드에 복제. 추가로 필요한 계산x | 데이터를 디스크에 기록하기 전 패리티 계산이 필요. 쓰기 연산 응답지연 증가 | 다중화 |
| 읽기 성능 | 장애가 발생하지 않은 노드에서 데이터를 읽음 | 데이터를 읽어야 할 때마다 클러스터 내의 여러 노드에서 데이터를 가져와야 함. 장애가 발생한 경우 빠진 데이터를 먼저 복원 해야함 → 지연시간 증가 | 다중화 |

## 멀티파트 업로드

![image.png](attachment:82c1ac65-53f2-49d1-9d38-afd21b73aaff:image.png)

1. 클라이언트: 멀티파트 업로드를 시작하기 위해 객체 저장소 호출
2. 데이터 저장소: uploadID 반환. (업로드를 유일하게 식별할 ID)
3. 클라이언트: 파일을 작은 객체로 분할, 업로드 시작
가정: 파일 크기: 1.6GB, 8조각으로 나눌 시 조각당 200MB
uploadID와 함께 데이터저장소에 올림
4. 데이터 저장소: 조각 하나가 업로드 될 때 마다 ETag를 반환. 
ETag: 조각에 대한 MD5 해시 체크섬. 멀티파트 업로드가 정상적으로 되었는지 검사 용도
5. 클라이언트: 모든 조각 업로드 후 멀티파트 업로드 요청 전송(uploadID, 조각 번호 목록, ETag 목록 포함)
6. 데이터 저장소: 전송 받은 조각 번호 목록을 사용해 원본 객체를 복원. (수 분 소요 가능) 완료 후 성공 메시지 반환 
- 객체 조립이 끝난 뒤의 조각들은 삭제하여 저장 용량을 확보

## **AWS S3 객체 목록 조회 (aws s3 ls)**

1. **S3의 핵심 개념**
- **S3는 "디렉터리"가 없다**
    - 파일시스템처럼 폴더 구조가 없음
    - 대신 접두어**(prefix)**를 사용해 폴더처럼 구성
    - 실제로는 모든 객체가 평면 구조로 저장됨
- **객체 경로 구조**
    
    ```
    s3://mybucket/abc/d/e/f/file.txt
    
    ├── mybucket: 버킷 이름
    
    └── abc/d/e/f/file.txt: 객체 이름(key) - 문자열 전체
    
    └── abc/d/e/f: 접두어 (디렉터리처럼 보이는 부분)
    ```
    
1. **실제 버킷 구조 예시**
- **버킷에 이런 객체들이 있다고 가정:**
    
    ```
    CA/cities/losangeles.txt
    CA/cities/sanfranciso.txt
    NY/cities/ny.txt
    federal.txt
    ```
    
1. **명령어 비교**
    
    **3.1 한 단계 목록만 보기 (디렉터리 구조)**
    
    ```bash
    -- S3 명령어:
    
    aws s3 ls s3://mybucket/
    
    -- 출력 결과:
    
    2021-01-01  CA/
    2021-01-01  NY/
    2021-01-01  federal.txt
    ```
    
    - `CA/cities/...` 객체들은 모두 `CA/` 로만 표시됨
    - `NY/cities/...` 객체들은 모두 `NY/` 로만 표시됨
    - 슬래시 뒤의 나머지 부분은 잘려서 표시되지 않음
    - 마치 디렉터리 구조만 보는 것처럼 느껴짐
    - 유사한리눅스 쉘 명령어 ****
        - **ls /dir/**
    
    **3.2 재귀적으로 모든 객체 보기**
    
    ```bash
    -- S3 명령어:
    
    aws s3 ls s3://mybucket/ --recursive
    
    -- 출력 결과:
    
    2021-01-01  CA/cities/losangeles.txt
    2021-01-01  CA/cities/sanfranciso.txt
    2021-01-01  NY/cities/ny.txt
    2021-01-01  federal.txt
    ```
    
    - 모든 객체의 전체 경로가 그대로 출력됨
    - 깊이 상관없이 모든 파일을 표시
    - 유사한 리눅스 쉘 명령어:
        - **ls -R /dir/**
        - **find /dir/**
2. **접두어(Prefix) 사용**
    - **접두어를 지정하면 그 아래의 객체들만 조회:**
    
    ```bash
    -- 명령어:
    
    -- 한 단계 목록:
    aws s3 ls s3://mybucket/CA/
    
    2021-01-01  CA/cities/
    
    - 재귀적 전체:
    
    aws s3 ls s3://mybucket/CA/ --recursive
    
    2021-01-01  CA/cities/losangeles.txt
    2021-01-01  CA/cities/sanfranciso.txt
    ```
    
3. 요약표
    
    
    | s3 명령어 | (유사)쉘 명령어 | 동작 |
    | --- | --- | --- |
    | `aws s3 ls s3://bucket/` | `ls /dir/` | 한 단계 목록 (폴더 구조) |
    | `aws s3 ls s3://bucket/ --recursive` | `ls -R /dir/` 또는 `find /dir/` | 전체 파일 재귀 조회 |
    | `aws s3 ls s3://bucket/prefix/` | `ls /dir/subdir/` | 특정 접두어의 한 단계 |
    | `aws s3 ls s3://bucket/prefix/ --recursive` | `ls -R /dir/subdir/` | 특정 접두어 아래 전체 |
