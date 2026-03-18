# (8) 분산 이메일 서비스

## 1단계: 문제 이해 및 설계 범위 확정

### 기능 요구사항

- 이메일 발송/수신
- 모든 이메일 가져오기
- 읽음 여부에 따른 이메일 필터링
- 검색 기능(제목, 발신인, 메일 내용)
- 스팸 및 바이러스 방지 기능

### 비기능 요구사항

- **안정성**: 이메일 데이터는 소실X
- **가용성**: 이메일과 사용자 데이터를 여러 노드에 자동으로 복제하여 가용성 보장
- **확장성**: 사용자 수가 늘어나도 감당 할 수 있어야 함. 시스템 성능 저하X
- **유연성과 확장성**: 새 컴포넌트를 더하여 쉽게 기능 추가 및 성능 개선이 가능한 유연하고 확장성 높은 시스템이어야 함.
    - ⇒ ⚠️ POP, IMAP 등의 기존 이메일 프로토콜은 이에 대한 기능이 제한적

### 개략적인 규모 추정

- 10억 명의 사용자
- **가정1**: 한 사람이 하루에 보내는 평균 이메일 수: 10건
    - 이메일 전송 QPS = (10^9 * 10) / (10^5) = 100,000
- **가정2**: 한 사람이 하루에 수신하는 이메일 수는 평균 40건. 이메일 하나의 메타데이터(이메일에 대한 모든 정보, 첨부 파일은 포함X)는 평균 50KB
- **가정3**: 메타데이터는 데이터베이스에 저장한다고 가정.
- ⇒ 1년 간 메타데이터를 유지 하기 위한 스토리지 요구량: 10억 명 사용자 * 하루 40건 이메일 * 365일 * 50KB = 730PB
- **가정4**: 첨부 파일을 포함하는 이메일의 비율: 20%, 첨부파일 평균크기: 500KB
- ⇒ 1년 간 첨부 파일을 보관하는데 필요한 저장 용량: 10억 명 사용자 * 하루 40개 이메일 * 365일 * 20% * 500KB = 1,460PB
- **결론**: 많은 데이터를 처리해야 함. 분산 데이터베이스 솔루션이 필요.

## 2단계: 개략적 설계안 제시 및 동의 구하기

### 이메일 101

- 이메일 프로토콜
    - **SMTP**: Simple Mail Transfer Protocol. 이메일을 한 서버 > 다른 서버로 보내는 표준 프로토콜
    - 이메일 수신 목적으로 가장 널리 사용되는 프로토콜: **POP, IMAP**
    - **POP**: 이메일 클라이언트가 원격 메일 서버에서 이메일을 수신, 다운 시 사용하는 표준 프로토콜. 
    단말로 다운로드 된 이메일은 서버에서 삭제 됨 > 한 대 단말에서만 이메일 읽기 가능. 
    이메일 확인 시 무조건 전체 내려받기만 가능 > 첨부 파일 용량이 클 시 속도 문제
    - **IMAP**: 개인 이메일 계정에서 가장 널리 사용되는 프로토콜
    읽음 시 메일 서버에서 지워지지 않음. > 여러 단말에서 읽기 가능
    이메일을 실제로 열기 전에는 헤더만 다운로드 > 클릭하지 않으면 메시지가 다운로드 되지 않음. > 속도 개선
    - **HTTPS**: 메일 전송 프로토콜은x. 웹 기반 이메일 시스템의 메일함 접속에 이용 됨. (ex. 마이크로 소프트의 아웃룩: 액티브싱크(ActiveSync) HTTPS기반 자체 프로토콜 사용)
- 도메인 이름 서비스(DNS)
    - 수신자 도메인의 메일 교환기 레코드(Mail Exchange, MX) 검색에 DNS가 사용됨.
    - 명령행으로 gmail.com의 DNS 레코드 검색 시 아래 MX 레코드가 표시 됨
        
        ```bash
        > set qmx
        192.168.65.1
        Address: 192.168.65.153
        
        Non-authoritative answer:
        gmail.com          mail exchanger = 20 alt2.gmail-smtp-in.l.google.com.
        gmail.com          mail exchanger = 40 alt3.gmail-smtp-in.l.google.com.
        gmail.com          mail exchanger = 30 alt1.gmail-smtp-in.l.google.com.
        gmail.com          mail exchanger = 50 alt4.gmail-smtp-in.l.google.com.
        gmail.com          mail exchanger = 10 gmail-smtp-in.l.google.com.
        ##                       (MX)    = (우선순위)  (메일 서버 목록)
        
        ```
        
        - 우선순위: 선호도. 낮을 수록 선호 됨
        - 송신 측 메일 서버는 우선순위 순서대로 연결을 시도함
- 첨부 파일
    - 일반적으로 Base64 인코딩 사용, 첨부 파일 크기 제한 있음
    - **다목적 인터넷 메일 확장(Multi-purpose Internet Mail Extension, MIME)**: 인터넷을 통해 첨부 파일을 전송할 수 있도록 하는 표준 규격

### 전통적 메일 서버

보통 서버 한 대로 운용되는, 사용자가 많지 않을 때 잘 동작하는 시스템

- 전통적 메일 서버 아키텍처
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Afbf6fe32-a3a9-4067-85fc-4dedd2e61eea%3Aimage.png?table=block&id=325be4ca-d9d2-8041-a0fa-e49b39727ca0&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    1. **앨리스**: 아웃룩 클라이언트에 로그인 후 이메일 작성 & [보내기] 버튼 클릭 > **SMTP 프로토콜**을 통해 이메일이 아웃룩 메일 서버로 전송 됨. 
    2. **아웃룩 메일 서버**: DNS 질의를 통해 수신사 SMTP 서버 주소를 찾은 뒤 메일 서버로 **SMTP 프로토콜**을 통해 이메일을 보냄. 
    3. 지메일 서버는 이메일을 저장. 수신자가 이메일을 읽어갈 수 있도록 함
    4. 밥이 지메일에 로그인 > **지메일 클라이언트**: IMAP/POP 서버를 통해 새 이메일을 가져옴
- 저장소
    - 전통적 메일 서버는 이메일을 파일 시스템의 디렉토리에 저장.
        - 각각의 이메일은 고유한 이름을 가진 별도 파일로 보관
        - 각 사용자의 설정 데이터 & 메일함: 사용자 디렉토리에 보관 (ex. Maildir)
            
            ```
            
            ## Maildir
            home
            ├── user1
            │   └── Maildir
            │       ├── cur
            │       ├── new
            │       └── tmp
            └── user2
                └── Maildir
                    ├── cur
                    ├── new
                    └── tmp
            ```
            
        - 사용자가 많지 않을 때는 잘 동작 하나 수십억 개의 이메일을 검색, 백업 목적으로는 부적합(I/O 병목 등의 문제 발생) & 가용성, 안정성 문제 요구사항 충족 불가능
            
            ⇒ 안정적인 분산 데이터 저장소 계층이 필요해짐
            

### 분산 메일 서버

현대적 사용 패턴을 지원, 확장성과 안정선 문제를 해결

- 이메일 API
    - = 메일 클라이언트. 이메일 생명주기 단계마다 달라질 수 있음
        - 모바일 단말 클라이언트를 위한 SMTP/POP/IMAP API
        - 송신 측 메일 서버와 수신 측 메일 서버 간의 SMTP 통신
        - 대화형 웹 기반 이메일 애플리케이션을 위한 HTTP 기반 RESTful API
    1. **POST /v1/message 엔드포인트**
        - To, Tc, Bcc 헤더에 명시된 수신자에게 메시지를 전송
    2. **GET /v1/folders 엔드포인트**
        - 주어진 이메일 계정에 존재하는 모든 폴더를 반환한다.
        - 응답형식
            
            ```json
            {
              "id": "string",   // 고유한 폴더 식별자 
              "name": "string",  // 폴더이름: "RFC7159에 따르면, 이는 객체의 회원이며 값이 문자열인 것과 같습니다. All, Archive, Drafts, Flagged, Junk, Sent와 같은 사전 정의된 라벨이 있으며 사용자 정의 라벨 ID도 있습니다.",
              "user_id": "string" //계정 소유자ID
            }
            ```
            
    3. **GET /v1/folders/{:folder_id}/messages 엔드포인트**
        - 주어진 폴더 아래의 모든 메시지를 반환한다.
        - 응답형식
            - 메시지 객체 목록
    4. **GET /v1/messages/{:message_id} 엔드포인트**
        - 주어진 특정 메시지에 대한 모든 정보를 반환.
        - 메시지는 이메일 애플리케이션의 핵심 구성정보 요소: 발신자, 수신자, 메시지 제목, 본문, 천부 파일 등의 정보로 구성 됨
        - 응답 형식
            
            ```json
            {
              "user_id": string,  // 계정주의 ID
              "from": {
                "name": string,
                "email": string //발신자의 <이름, 이메일> 쌍
               },
              "to": {
                "name": string,
                "email": string
              },  // 수신자 (이름, 이메일)
              "subject": string,  // 제목
              "body": string,  // 본문
              "is_read": Boolean  // 읽음 여부
            }
            
            ```
            
- 분산 이메일 서버 아키텍처
    - 개략적 설계안
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A9f5bd8e2-cdca-4a0a-9f3c-8eab2f8aea50%3Aimage.png?table=block&id=325be4ca-d9d2-80a7-88e0-d5856d1adb24&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        
        - **웹메일(webmail)**: 사용자는 웹브라우저를 사용해 메일을 받고 보낸다
        - **웹서버**: 사용자가 이용하는 요청/응답 서비스로, 로그인, 가입, 사용자 프로파일등에 대한 관리 기능을 담당. 
        본 설계안:이메일 발송, 폴더 목록 확인, 폴더 내 모든 메시지 확인 등의 모든 이메일 API요청이 웹서버를 통함
        - **실시간 서버**: 새로운 이메일 내역을 클라이언트에 실시간으로 전달.
            - 지속성 연결을 맺고유지해야 하므로 stateful 서버임.
            - 실시간 통신 지원 방안: 롱 폴링, 웹소켓(아파치 제임스)
        - **메타데이터 데이터베이스**: 이메일 제목, 본문, 발신인, 수신인 목록 등의 메타 데이터를 저장
        - **첨부 파일 저장소**: 아마존 S3 같은 객체 저장소를 사용. 이미지나 동영상 등의 대용량 파일을 저장하는데 적합한, 확장이 용이한 저장소 인프라
            - 카산드라 등의 컬럼 기반 NoSQL 데이터베이스는 적합하지 않음
                - 카산드라가 BLOB자료형을 지원하고, 해당 자료형의 최대 크기가 2GB지만 실질적으로는 1MB 이상 지원이 어려움
                - 카산드라에 첨부 파일 저장 시 레코드 캐시 활용 어려움 > 첨부 파일이 너무 많은 메모리를 잡아먹음
        - **분산 캐시**: 최근에 수신된 메일은 자주 읽을 가능성이 높으므로 메모리에 캐시해두면 좋음 → list 같은 다양한 기능을 제공 및 규모확장이 용이한 레디스 사용
        - **검색 저장소**: 분산 문서 저장소로 고속 텍스트 검색을 지원하는 역 인데스를 자료구조로 사용

### 이메일 발송 및 수신 흐름

- **이메일 전송절차**
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A74063d75-2350-49b0-86ec-3e06e62b47a6%3Aimage.png?table=block&id=325be4ca-d9d2-80bf-988d-fc339a9133ad&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    1. 사용자: 웹 메일 환경에서 메일을 작성 및 [전송] > 요청이 로드밸런서로 전송 됨
    2. 로드밸런서: 처리율 제한 한도를 넘지 않는 선에서 요청을 웹서버로 전달
    3. 웹 서버
        1. 기본적인 이메일 검증: 이메일 크기 등의 사전 정의된 규칙을 사용하여 이메일 검사
        2. 수신자 이메일 주소 도메인이 송신자 이메일 주소 도메인과 같은지 검사 
            1. 같다면 이메일 내용의 스팸 여부와 바이러스 감염여부 검사 
            2. 통과 시 이메일은 송신인의 ‘보낸 편지함’, 수신인의 ‘받은 편지함’에 저장 됨
            3. 수신 측 클라이언트는 RESTful API를 통해 이메일을 가져올 수 있음 (4 이후 생략)
    4. 메시지 큐
        1. 기본 검증을 통과한 이메일은 외부 전송 큐로 전달. 
            - 큐에 넣기에 첨부 파일이 너무 큰 이메일은 객제 저장소에 첨부 파일을 따로 저장, 큐에 전달하는 이메일 안에 해당 저장 위치에 대한 참조 정보를 보관
        2. 기본 검증에 실패한 이메일은 에러 큐에 보관
    5. 외부 전송 담당 SMTP 작업 프로세스: 외부 전송 큐에서 메시지를 꺼내어 이메일의 스팸 및 바이러스 감염 여부를 확인
    6. 검증 절차를 통과한 이메일은 저장소 계층 내의 ‘보낸 편지함’에 저장
    7. 외부 전송 담당 SMTP 작업 프로세스: 수신자의 메일 서버로 메일을 전송
- 외부 전송 큐의 크기를 모니터링 중요: 메일이 처리되지 않고 큐에 오랫동안 남아있으면 이유를 분석해야 함
    - 수신자 측 메일 서버에 장애 발생: 나중에 메일을 다시 전송해야함. 지수적 백오프가 좋은 전략일 수 있음
    - 이메일을 보낼 큐의 소비자 수가 불충분: 더 많은 소비자를 추가하여 처리시간을 단축하는 방법 필요
- **이메일 수신 절차**
    
    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A9e975cc4-3a54-40fc-8936-a3022c8612b7%3Aimage.png?table=block&id=325be4ca-d9d2-8093-89b9-fbcdb1609179&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=860&userId=&cache=v2)
    
    1. 이메일이 SMTP 로드밸런서에 도착
    2. 로드밸런서: 트래픽을 여러 SMTP 서버로 분산. 이메일 수락 정책을 구성하여 유효하지 않은 이메일은 반송하도록 할 수 있음 (이후 처리 불필요)
    3. 이메일의 첨부 파일이 큐에 들어가기 너무 큰 경우: 첨부 파일 저장소(S3)에 보관
    4. 이메일을 수신 이메일 큐에 넣음.: 메일 처리 작업 프로세스와 SMTP 서버간 결합도를 낮춰 각자 독립적으로 규모확장이 가능하도록 함. 수신되는 이메일 양이 폭증하는 경우 버퍼 역할도 가능
    5. 메일 처리 작업 프로세스(worker): 스팸 메일 필터, 바이러스 차단 등의 역할
    6. 이메일을 메일 저장소, 캐시, 객체 저장소 등에 보관
    7. 수신자가 온라인 상태인 경우 이메일을 실시간 서버로 전달
    8. 실시간 서버: 수신자 클라이언트가 새 이메일을 실시간으로 받을 수 있게 하는 웹소켓 서버
    9. 오프라인 상태의 사용자의 이메일 > 저장소 계층에 보관. 이후 사용자가 온라인 상태가 되면 웹메일 클라이언트가 웹 서버에 RESTful API를 통해 연결
    10. 웹서버: 새로운 이메일을 저장소 계층- → 클라이언트에 반환
    
    ## 3단계: 상세 설계
    
    ### 메타데이터 데이터베이스
    
    - 이메일 메타데이터의 특성
        - 이메일 헤더: 일반적으로 작고, 빈번하게 사용됨
        - 이메일 본문: 다양한 크기, 사용빈도가 낮음 (일반적으로 사용자는 1회만 읽음)
        - 이메일 가져오기, 읽은 메일로 표시, 검색 등의 이메일 관련 작업: 사용자별로 격리 수행되어야함.
        - 데이터의 신선도: 데이터 사용 패턴에 영향을 끼침. 사용자는 보통 최근 메일만 읽음. 만들어진지 16일 이하 데이터에 발생하는 읽기 질의 비율: 전체 질의의 82%
        - 데이터의 높은 안정성 보장 필요. 데이터 손실 용납X
    - 올바른 데이터베이스의 선정
        
        지메일, 아웃룩 규모의 메일 서비스는 초당 입/출력 연산 빈도(Input/Output Operations Per Second, IOPS)를 낮추기 위해 맞춤제작 데이터베이스를 사용함.
        
        - **관계형 데이터베이스**
            - 이메일을 효율적으로 검색할 수 있음 (이메일 헤더, 본문에 대한 인덱스 활용)
            - 단점: 데이터 크기가 작을 때 적합 → 이메일은 nKB ~ 100KB 크기
                - 비정형 BLOB 자료형 데이터에 대한 검색 질의 성능이 좋지 않음
        - **분산 객체 저장소**: 이메일의 원시데이터를 그대로 아마존S3와 같은 객체 저장소에 보관
            - 장점: 백업 데이터를 보관하기 좋음
            - 단점: 이메일의 읽음 표시, 키워드 검색, 이메일 타래 기능 구현이 어려움
        - **NoSQL 데이터베이스**
            - 지메일: 구글 빅테이블(Bigtable)
                - ⇒ 오픈소스로 공개되어 있지 않아 이메일 검색을 어떻게 구현했는지는 미지수. 카산드라가 대안
        - 결론: 완벽히 지원하는 데이터 베이스는 없다고 봐도 무방함.
        - 데이터베이스가 충족해야 하는 조건
            - 어떤 단일 칼럼의 크기는 한 자릿수 MB 정도일 수 있음
            - 강력한 데이터 일관성이 보장되어야 함
            - 디스크 I/O가 최소화되도록 설계
            - 가용성이 아주 높아야 함.
            - 일부 장애를 감내할 수 있어야 함.
            - 증분 백업(incremental backup)이 쉬워야 함.
    - 데이터 모델
        - 데이터를 저장하는 방법: user_id를 파티션 키로 사용하여 특정 사용자의 데이터는 항상 같은 샤드에 보관
        - 기본 키
            - 파티션 키: 데이터를 여러 노드에 균등하게 분산하는 역할.
            - 클러스터 키: 같은 파티션 속 데이터를 정렬하는 역할
        - 이메일 서비스의 데이터 계층이 지원해야하는 질의
            - **질의 1: 특정 사용자의 모든 폴더 질의**
                
                ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A0f2bb37b-979d-4e23-be73-16188c7696e9%3Aimage.png?table=block&id=327be4ca-d9d2-8021-976c-e8b184354096&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=630&userId=&cache=v2)
                
            - **질의 2: 특정 폴더에 속한 모든 이메일표시**
                
                ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A451a8289-8494-422f-b1fd-45d50509e7be%3Aimage.png?table=block&id=327be4ca-d9d2-809f-a2fa-ceb8e7bad175&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=630&userId=&cache=v2)
                
                - 같은 폴더에 속한 모든 이메일이 같은 파티션에 속하도록 하려면: <user_id, folder_id> 형태의 복합 파티션 키를 활용
                - email_id: 이메일을 시간순으로 정렬
            - **질의 3: 이메일 생성/삭제/수신**
                
                ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A6b902870-9793-4ce2-8ce5-aae87e632a4f%3Aimage.png?table=block&id=327be4ca-d9d2-80a5-be5f-c4befdf54967&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=630&userId=&cache=v2)
                
                ```sql
                SELECT * FROM emails_by_user WHERE email_id = 123;
                ```
                
            - **질의 4: 읽은, 또는 읽지 않은 모든 메일**
                - 관계형 데이터 베이스를 사용 시
                    
                    ```sql
                    -- 읽은 메일 전부 읽기
                    SELECT * FROM emails_by_folder
                     WHERE user_id = <user_id> AND folder_id = <folder_id> AND is_read = true
                     ORDER BY  email_id
                     
                     -- 안읽은 메일 전부 읽기
                    SELECT * FROM emails_by_folder
                     WHERE user_id = <user_id> AND folder_id = <folder_id> AND is_read = false
                     ORDER BY  email_id
                    ```
                    
                - NoSQL를 사용 시: 파티션 키, 클러스터 키에 관련한 질의만 가능
                    - 방법1: 폴더에 속한 모든 메시지를 가져 온 다음, 애플리케이션 단에서 필터링을 수행 ⇒ 대규모 서비스에는 적합X
                    - 방법2: NoSQl 데이터베이스 테이블을 비정규화
                        - emails_by_folder를 분리
                            
                            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A778a77d6-f0c3-44d8-95c8-e193be84dc66%3Aimage.png?table=block&id=327be4ca-d9d2-8033-9f92-fad371010a73&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=400&userId=&cache=v2)
                            
                            - read_emails: 읽은 상태의 이메일을 보관하는 테이블
                            - unread_emails: 읽지않은 상태의 이메일을 보관하는 테이블
                        - 읽은 메일을 unread_eamils에서 삭제 > read_emails 테이블로 옮김
                    
                    ```sql
                    -- 읽은 메일 전부 읽기
                    SELECT * FROM unread_emails
                     WHERE user_id = <user_id> AND folder_id = <folder_id>
                     ORDER BY  email_id
                     
                     -- 안읽은 메일 전부 읽기
                    SELECT * FROM read_emails
                     WHERE user_id = <user_id> AND folder_id = <folder_id>
                     ORDER BY  email_id
                    ```
                    
            - 보너스: 이메일 타래 가져오기
                - 모든 답장을 최초 메시지에 타래로 엮어 보여주는 기능
                - 전통적으로 JWZ 같은 알고리즘을 통해 구현.
                - 이메일 헤더의 필드. 이 필드들이 있으면 타래 내의 모든 메시지가 사전에 메모리로 로드되어 있는 경우 전체 대화 타래를 재구성해낼 수 있음
                    
                    ```json
                    {
                      "headers": {
                        "Message-ID": "<7BA8B43-04C2-85B6-83C5-0B234567@gmail.com>",
                        "In-Reply-To": "<CAEBAB4A-30C4-12B5-862C3501@gmail.com>",
                        "References": ["<7BA8B43-04C2-85B6-83C5-0B234567@gmail.com>"]
                      }
                    }
                    
                    ```
                    
                    | **Message-ID** | 메일의 식별자. 메시지를 보내는 클라이언트가 생성 |
                    | --- | --- |
                    | **In-Reply-To** | 이 메시지가 어느 메일에 대한 답장인지를 나타내는 식별자 |
                    | **References** | 타래에 관계된 메시지 식별자 목록. |
            - 일관성 문제
                - 높은 가용성을 달성하기 위해 다중화에 의존하는 분산 데이터 베이스 → 일관성↔가용성 사이에 타협적인 결정을 내려야함.
                - 이메일 시스템은 데이터의 정확성이 아주 중요 → 모든 메일함은 반드시 하나의 주(primary) 사본을 통해 서비스된다고 가정.
                - 장애가 발생하면 클라이언트는 주 사본이 복원될 때까지 동기화/갱신 작업 대기
    
    ### 이메일 전송 가능성(deliverability)
    
    - 스팸메일
        - 메일 가운데 50%는 스팸 메일 (스태티스타 사에서 수행한 연구)
        - 신규 구성 메일 서버의 메일은 십중팔구 스팸 폴더로 들어감
    - 이메일의 전송 가능성을 높이는 방법
        - 전용 IP: 전용 IP주소를 사용해서 전송. 이메일 사업자는 아무 이력이 없는 신규 IP 주소에서 온 이메일을 무시함
        - 범주화: 범주가 다른 이메일은 다른 IP 주소를 통해 전송(ex. 마케팅 메일 전송 IP ↔ 중요 이메일 IP)
        - 발신인 평판: 신규 이메일 서버 IP는 사용빈도를 서서히 올려 좋은 평판을 쌓을 것
            - 대략 2~6주 소요 (출처: 아마존 SES)
        - 스팸 발송자의 신속한 차단
        - 피드백 처리: ISP 측에서의 피드백을 빠르게 받아 처리하여 불만 신고 비율 🔽, 스팸계정 신속히 차단
            
            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3Ab075d7bc-a24e-4961-9ae7-fa6220883013%3Aimage.png?table=block&id=327be4ca-d9d2-8090-b41c-d196e740ef4e&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=710&userId=&cache=v2)
            
            - 경성 반송: 수신인의 이메일 주소가 올바르지 않아 ISP가 전달을 거부
            - 연성 반송: ISP 측의 이메일 처리 자원 부족 등의 이유로 일시적 전송 불가
            - 불만 신고: 수신인이 ‘스팸으로 신고’ 버튼을 누른경우
        - 이메일 인증
            
            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A156347f9-284e-4f35-a9e1-1548695d41ad%3Aimage.png?table=block&id=327be4ca-d9d2-80f8-9739-f56fc105141f&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=710&userId=&cache=v2)
            
    
    ### 검색
    
    - 기본: 이메일 제목 or 본문의 특정 키워드 검색
    - 고급: 발신인, 제목, 읽지 않음 속성 핉러링
    - 이메일시 전송,수신,삭제 될 떄마다 색인 작업 필요
    - 검색: 검색버튼을 누를 시에만 실행
    - 쓰기 > 읽기
        - 방안 1: 일래스틱서치
            
            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A5d5764de-48ff-4e9e-a054-9a360e4e322c%3Aimage.png?table=block&id=327be4ca-d9d2-80d6-955f-f34340bc8b47&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=710&userId=&cache=v2)
            
            - user_id를 파티션 키로 사용
            - 색인 작업: 후면(background) 작업으로 처리
                - 카프카를 통해 색인 작업 시작 서비스 / 실제 색인 수행 서비스 분리
            - 주 이메일 저장소와 동기화를 맞추는게 조금 까다로움
        - 방안 2: 맞춤형 검색 솔루션
            - 자체적으로 검색 솔루션을 구현하는 경우 마주하게 될 주요과제: I/O 병목 문제
                - 매일 저장소에 추가되는 메타데이터/첨부 파일의 양은 페타바이트(PB) 단위
                - 이메일 색인 서버의 주된 병목: 디스크 I/O
                - 색인 구축 프로세스는 다량의 쓰기 연산을 발생 → LSM(Log-Structured Merge) 트리를 사용하여 디스크에 저장되는 색인을 구조화
                    - 쓰기 경로는 순차적 쓰기 연산만 수행하도록 최적화
                    
                    ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A16f9b8fd-1340-4f99-9fdb-b916da236eea%3Aimage.png?table=block&id=327be4ca-d9d2-8027-a791-d93b61e46e1b&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=560&userId=&cache=v2)
                    
                    1. 새로운 메일이 도착하면 메모리 캐시로 구현되는 0번 계층에 추가
                    2. 메모리에 보관된 데이터의 양이 사전에 정의된 임계치를 넘으면 다음 계층에 병함
                    - LSM을 사용하면 자주 바뀌는 데이터(이메일 폴더) 와 그렇지 않은 데이터(이메일 데이터)를 분리 가능
        - 일래스틱서치 vs 맞춤형 검색 엔진
            
            ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A239468c1-e451-4091-9948-d65f934aef7c%3Aimage.png?table=block&id=327be4ca-d9d2-8032-a8e3-c95ae06c9dd7&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=710&userId=&cache=v2)
            
            - 소규모: 일래스틱서치
    
    ### 규모 확장성
    
    - 가용성을 향상시키기 위해 데이터를 여러 데이터센터에 다중화
        
        ![image.png](https://possible-telephone-196.notion.site/image/attachment%3A7a245c47-f7c5-43ce-940e-750b43172f6b%3Aimage.png?table=block&id=327be4ca-d9d2-80a2-b6c2-f274b8659291&spaceId=594418fe-2b8a-40ce-8d83-fb1b120f4b8f&width=780&userId=&cache=v2)
        

---

## 추가 공부

### SMTP, POP, IMAP

이메일 생명주기(송신→수신→접근)를 담당하는 **3대 프로토콜**

- **상세 비교**
    
    
    | **항목** | **SMTP (Simple Mail Transfer Protocol)** | **POP3 (Post Office Protocol v3)** | **IMAP (Internet Message Access Protocol)** |
    | --- | --- | --- | --- |
    | **역할** | **메일 전송** (보내기 전용) | **메일 다운로드** (수신) | **메일 동기화** (수신+관리) |
    | **포트** | 25(기본), 587(보안) | 110(기본), 995(보안) | 143(기본), 993(보안) |
    | **동작** | 클라이언트→서버, 서버→서버 | 서버에서 PC로 **이동+삭제** | 서버에 **원본 그대로**, 클라이언트 동기화 |
    | **보안** | STARTTLS 지원 | - | - |
    | **용량** | 제한 없음 | 로컬 저장 (용량 절약) | 서버 저장 (메타데이터만 동기화) |
- **실제 흐름**

```sql
1. 보내기: Outlook → SMTP(587) → Gmail 서버
2. 수신: Gmail 서버 → IMAP(993) → Outlook, iPhone
3. 동기화: 휴대폰 삭제 → IMAP → Gmail/PC 모두 삭제
```

- **언제 무엇?**

```sql
🏠 POP3: 1대 PC만 쓰는 개인 (메일 다운로드)
🏢 IMAP: 회사/다중기기 (모두 동기화)
📤 SMTP: 모든 클라이언트 공통 (보내기)
```

## MIME

 

**이메일/SMTP에서 멀티미디어 전송을 위한 표준**

- **핵심**
    - **문제 해결**: SMTP는 7bit ASCII만 지원 → 바이너리(이미지 등) 전송 불가
    - **해결**: Base64 인코딩으로 8bit→7bit 변환 + Content-Type 헤더
- **주요 헤더**
    
    
    | **헤더** | **역할** |
    | --- | --- |
    | `Content-Type` | `text/plain`, `image/jpeg` |
    | `Content-Transfer-Encoding` | `base64`, `quoted-printable` |
    | `Content-Disposition` | `attachment; filename="photo.jpg"` |
- **예시 (호텔 예약 확인 메일)**
    
    ```html
    Content-Type: multipart/mixed; boundary="boundary123"
    
    --boundary123
    Content-Type: text/html; charset=UTF-8
    
    <h1>예약완료</h1>
    
    --boundary123
    Content-Type: image/png
    Content-Transfer-Encoding: base64
    Content-Disposition: attachment; filename="hotel.png"
    
    iVBORw0KGgoAAAANSUhEUgAA...
    
    ```
    
    **호텔 시스템**: 예약 PDF 첨부 → MIME multipart → SMTP 전송
