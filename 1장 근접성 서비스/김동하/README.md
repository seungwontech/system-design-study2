# 1. 근접성 서비스란?

- 위치(공간) 데이터를 기준으로 대상(편의시설 등)을 순위화하여 제공해주는 서비스

---

# 2. 기능적 요구사항

## 1. 사용자

### 1. 위치

- 사용자의 현재 위치에 대한 정보
- 좌표로 표현 - (x, y)
    - latitude : 위도 - decimal
    - longitude 경도 - decimal

### 2. 검색 범위

- 사용자로부터 얼마나 먼 곳에 있는 편의시설을 검색할 것인지에 대한 정보
- 반경(0.5km, 1km, 2km, 5km, 10km, 20km …)
    - radius : 반경 - int

## 2. 사업주

- 사업장에 대한 상세 정보
- 추가 / 삭제 / 갱신 기능
    - business_id : pk - int
    - address : 상세주소 - string
    - city : 주소 - string || int
    - state : 영업장 상태 - string
    - country : 국가 - string || int
    - latitude : 위도 - decimal
    - longitude : 경도 - decimal
    - content : 사업장 상세정보 - string
    - snapshot : 사업장 배너 이미지 정보 - string
    - image : 사업장 이미지 정보 - string

## 3. 비기능적 요구사항

### 1. 정보 갱신

- 사업장에 대한 정보갱신을 실시간일 필요 없음

### 2. 낮은 응답 지연

### 3. 데이터 보호

- 사용자의 위치 정보는 민감성 정보이기 때문에 아래와 같은 보호 법안을 준수하여 설계
    - GDOR(General Data Protection Regulation)
    - CCPA(California Consumer Privacy Act)

### 4. 고가용성

- 인구가 밀집되어 있는 지역이나 사용자가 많이 접속하는 시간대의 트래픽을 견딜 수 있도록 설계

---

# 3. 규모 추정

- 일간 능동 사용자 : 1억
    - 하루 기준 사용자 한 사람 당 평균 검색 시도 수 : 5회
- 등록된 사업장 수 : 2억
- QPS  = 1억 * 5 / 100,000
    - 1일 = 24*60*60 = 86,400 := 100,000

---

# 4. API 설계

## 1. 검색 기준 사업장 목록 반환

### 1️⃣ GET /v1/search/nearby

- 페이지네이션 사용
- requestDto 예시
    - latitude : 위도 - decimal
    - longitude : 경도 - decimal
    - radius : 반경 - int
- responseDto 예시
- total : 검색된 리스트 수
- businesses : 사업장 리스트
    - business_id : pk - int
    - address : 상세주소 - string
    - city : 주소 - string || int
    - state : 영업장 상태 - string
    - country : 국가 - string || int
    - snapshot : 사업장 배너 이미지 정보 - string

## 2. 사업장 정보 관련

### 1️⃣ GET /v1/business/:id

- 사업장 상세 정보 반환
- 사업장에 대한 설명, 이미지 등

### 2️⃣ POST /v1/businesses

- 신규 사업장 정보 추가

### 3️⃣ PUT /v1/businesses/:id

- 사업장 상세정보 갱신

### 4️⃣ DELETE /v1/businesses/:id

- 사업장 상세정보 삭제

---

# ❗근접성 서비스의 핵심질문

근접성 서비스의 본질은 “가까운 걸 찾아주는 기능”이 아니라, **수많은 사업장 데이터(예: 2억 개)를 ‘위치’라는 기준으로 얼마나 빠르게 좁혀서 후보를 만들 수 있느냐**에 있다.

사용자가 검색할 때마다 전체 데이터를 전수 조사하면(naive) 정확하긴 해도 절대 운영이 불가능하다. 그래서 이 장은 “정확한 거리 계산”을 잘하는 법이 아니라, **정확한 계산을 하기 전에 후보를 줄이는 공간 인덱스(Spatial Index)를 어떻게 설계할지**를 다룬다.

즉, 이 장의 흐름은 아래 한 줄로 정리된다.

> 공간 인덱싱으로 후보를 빠르게 줄이고 → 마지막에만 정확한 거리 계산으로 재정렬한다.
> 

---

# 5. 데이터 모델

## 1️⃣ 근접성 서비스 (Search 중심)

### 역할

- 사용자 위치 + 반경 입력
- **수많은 사업장 중 ‘후보’를 빠르게 좁힘**
- 순위화(거리 기준)
- 페이지네이션

👉 핵심은 **속도와 스케일**, 정확성은 “거리 계산” 정도만 보장하면 됨

### 데이터 성격

- 자주 조회됨
- 짧은 필드
- 정형 구조
- 인덱스 중심

### 데이터 모델

- `cell_key (geohash / S2)`
- `business_id`
- `lat / lon`
- `snapshot`, `state`, `city` 정도

👉 **RDB + 인덱스** or **Redis / KV**

- WHERE IN
- ORDER BY
- LIMIT / OFFSET
- 캐시 친화적

### 설계 포인트

- 공간 인덱싱 전략(Uniform Grid → Geohash → S2)
- 핫셀 대응
- 캐시 전략
- 후보 → 정확 거리 → 정렬 파이프라인

## 2️⃣ 편의시설 상세 서비스 (Detail / Content 중심)

### 역할

- 특정 `business_id`에 대한 상세 정보 제공
- 이미지, 설명, 부가 정보
- 검색 로직 없음

👉 핵심은 **유연성, 확장성, 스키마 자유도**

### 데이터 성격

- 단건 조회
- 필드 많음
- 구조가 자주 바뀔 수 있음
- JSON / 문서형 적합

### 데이터 모델

- `business_id`
- `content`
- `images`
- `metadata`
- 확장 필드

👉 **NoSQL(Document)**

- 스키마 변경 쉬움
- 부분 업데이트 간단
- CDN/캐시 연계 쉬움

---

# 6. 사업장 검색 알고리즘

- 공통 패턴은 **(1) 후보를 줄이는 인덱싱 + (2) 정확한 거리로 재정렬**이다.
- 차이는 “**후보를 어떻게, 얼마나 효율적으로 줄이느냐**”에 있다.

## 1️⃣ 2차원 검색 (Naive / Basic)

!https://machine-learning-tutorial-abi.readthedocs.io/en/latest/_images/knn.png

### 1. 개념

- 모든 사업장에 대해 사용자 위치와의 **정확한 거리**를 계산
- 계산 결과를 기준으로 정렬하여 가까운 순으로 반환

### 2. 왜 이 방식부터 생각하는가?

- 가장 직관적이고 수학적으로 정확한 방식
- “정답이 무엇인지”를 정의하는 **기준선(baseline)** 역할

### 3. 구조적 특징

- 인덱스 없음
- 모든 데이터에 대해 반복 계산

```
for business in all_businesses:
    distance = calc(user, business)
sort by distance
```

---

### 4. 장점

- 구현이 가장 단순
- 거리 계산 정확도 최상
- 경계/누락 문제 없음

### 5. 치명적 단점

- 사업장 수 N → **O(N)** 거리 계산
- 사업장 2억이면 요청 1번에 2억 번 계산 → 현실적으로 불가능

### 6. 이 방식의 의미

> “정확한 거리 계산은 절대 처음에 하면 안 된다”
> 
> 
> → **항상 마지막 단계에서만 사용한다**
> 

## 2️⃣ 균등 격자 (Uniform Grid)

[https://www.researchgate.net/publication/316186904/figure/fig2/AS%3A484353701617665%401492490329760/The-spatial-map-uniform-grid-division-the-distribution-of-trajectory-locations-The.png](https://images.openai.com/thumbnails/url/x2ocw3icu5mZUVJSUGylr5-al1xUWVCSmqJbkpRnoJdeXJJYkpmsl5yfq5-Zm5ieWmxfaAuUsXL0S7F0Tw7xt3SMTwtzTqqKT_GqSjd19cgst9RNSymPd3I0CXQpyo_0r_J1CQg0Niu0jPAK9i1XKwYAVhIlwA)

### 1. 개념

- 전체 지도를 **같은 크기의 격자(cell)** 로 분할
- 각 셀에 포함된 business 목록을 저장
- 검색 시 사용자 위치가 속한 셀 + 주변 셀만 조회

### 2. 왜 이 방식을 쓰는가?

- Naive 방식의 “전수 탐색”을 피하기 위한 **첫 번째 인덱싱 시도**
- 공간을 “주소(Key)”처럼 다루기 시작한 시점

### 3. 구조적 특징

- 셀 크기 고정 (예: 200m x 200m)
- Key: `cell_id`
- Value: 해당 셀의 business 리스트

### 4. 근접 검색 방식

1. 사용자 위치 → cell_id 계산
2. 반경에 따라 주변 셀 목록 계산
3. 해당 셀들의 business를 후보로 수집
4. 정확한 거리 계산 후 필터링/정렬

### 5. 장점

- 구현이 쉽고 직관적
- Key-Value / Redis / NoSQL과 궁합이 좋음
- 단순 캐시 전략 가능

### 6. 단점

1. 핫셀 문제
- 도심 셀 하나에 business가 과도하게 몰림
- 특정 key에 트래픽/데이터 집중
1. 셀 크기 튜닝의 어려움
- 너무 작으면 → 셀 조회 개수 폭증
- 너무 크면 → 후보가 많아져 Naive에 가까워짐

### 7. 의미

> 균등 격자는
> 
> 
> **“공간을 나누는 것만으로는 부족하다”** 는 사실을 알려준다.
> 

## 3. 지오해시 (Geohash)

!https://substackcdn.com/image/fetch/%24s_%21_FWk%21%2Cf_auto%2Cq_auto%3Agood%2Cfl_progressive%3Asteep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F4f50c60b-732a-4f24-ab77-574d4c40f5ac_2210x858.png

### 1. 개념

- 위도/경도를 **문자열로 인코딩**한 공간 키
- 문자열의 **접두어(prefix)** 가 같을수록 지리적으로 가깝다
- prefix 길이가 곧 **공간 해상도**

### 2. 왜 지오해시를 쓰는가?

- 균등 격자의 “고정 크기” 문제를 해결
- **동적 해상도 조절** 가능
- 문자열 prefix 기반 → 분산/샤딩/캐시에 매우 유리

### 3. 구조적 특징

- Key: `geohash_prefix`
- prefix 길이 ↑ → 셀 크기 ↓
- 동일 prefix = 동일 셀

### 4. 검색 방식 (전형적 흐름)

1. 사용자 위치 → geohash 변환
2. radius에 맞는 prefix 길이 선택
3. 현재 셀 + 인접 8개 셀 prefix 계산
4. 해당 prefix들로 후보 조회
5. 정확 거리 계산 후 필터링/정렬

### 5. 장점

- 구현 난이도 낮음
- 샤딩/캐시 친화적
- 실무에서 가장 널리 사용됨

### 6. 단점

1.  경계 문제
    - 셀 경계에 있으면 인접 셀을 조회하지 않으면 누락 발생
2. 셀 면적 불균등
    - 고위도로 갈수록 셀 실제 면적이 줄어듦

### 7. 의미

> 지오해시는
> 
> 
> **쿼드 트리의 “동적 분할 개념”을
> 문자열 키 기반으로 현실화한 방식**이다.
> 

## 4️⃣ 쿼드 트리 (Quadtree)

[https://www.gitta.info/SpatPartitio/en/image/quad_tree.gif](https://images.openai.com/thumbnails/url/S8V14Hicu5mZUVJSUGylr5-al1xUWVCSmqJbkpRnoJdeXJJYkpmsl5yfq5-Zm5ieWmxfaAuUsXL0S7F0Tw4JSAlyd45Pcsmt8Eo3q8pJsTD0NIjPMPdyKc1MMzN2cTZ3qijLyHQ1KAh3S7asME5WKwYATyoliw)

### 1. 개념

- 2차원 공간을 **4개의 사각 영역**으로 재귀적으로 분할하는 공간 인덱싱 구조
- 각 노드는 하나의 사각 영역을 나타내며,
    - 해당 영역에 포함된 객체 수가 **임계값(threshold)** 을 넘으면
    - 그 영역을 다시 4등분하여 자식 노드를 생성

👉 핵심은

**“객체가 몰린 곳만 더 잘게 쪼개고, 한산한 곳은 크게 유지한다”**

### 2. 왜 쿼드 트리를 쓰는가?

균등 격자의 근본적 문제는 이거였지:

- 도심: 객체 과밀 → 한 셀에 너무 많이 몰림
- 시골: 객체 희소 → 셀이 낭비됨

쿼드 트리는 이걸 이렇게 해결한다:

| 지역 | 균등 격자 | 쿼드 트리 |
| --- | --- | --- |
| 도심 | 셀 하나에 과밀 | **계속 분할 → 밀도 균형** |
| 시골 | 셀 낭비 | **분할 안 함** |

즉, **데이터 분포에 따라 공간 해상도가 자동으로 조절**된다.

### 3. 구조적 특징

- 루트 노드: 전체 지도
- 내부 노드: 4개의 자식(NW, NE, SW, SE)
- 리프 노드:
    - business 개수가 threshold 이하
    - 또는 최소 셀 크기에 도달

```
Root
 ├─ NW
 │   ├─ NW
 │   ├─ NE
 │   ├─ SW
 │   └─ SE
 ├─ NE
 ├─ SW
 └─ SE

```

### 4. 근접성 검색에서의 사용 방식

### (1) 인덱스 구축

- business 위치를 루트 노드에 삽입
- 노드 내 객체 수가 threshold 초과 → 분할
- business를 자식 노드로 재배치

### (2) 반경 검색

1. 사용자 위치 + 반경(radius)으로 **검색 원(circular range)** 생성
2. 쿼드 트리를 내려가며:
    - 원과 **겹치지 않는 노드**는 즉시 제외
    - 겹치는 노드만 탐색
3. 리프 노드의 business들을 후보로 수집
4. 최종적으로 정확한 거리 계산 후 필터링/정렬

👉 이 단계에서 이미 후보 수가 매우 줄어 있음

### 5. 장점

- **데이터 밀집도에 강함** (핫셀 문제 완화)
- 불필요한 공간 탐색이 줄어듦
- 반경 검색, 영역 검색(range query)에 적합
- 이론적으로는 매우 “정석적인” 공간 분할 구조

### 6. 단점 (실무에서 중요한 부분)

이게 중요하다.

쿼드 트리가 “교과서적으로 멋있지만”, 대규모 서비스에서 바로 쓰기 어려운 이유다.

1.  분산 시스템에 부적합
    - 트리 구조 자체가 **포인터 기반**
    - 노드가 어느 서버에 있는지 추적이 복잡
    - 샤딩 키로 쓰기 애매함
2. 업데이트 비용
    - business 이동/추가 시
        - 트리 재조정
        - 분할/병합 발생 가능
    - 쓰기 트래픽이 많아지면 관리 난이도 급증
3. 캐시 친화적이지 않음
    - prefix 기반 키 접근(지오해시, S2)에 비해
    - 접근 패턴이 불규칙
    

## 5️⃣ S2

!https://s2geometry.io/devguide/img/s2curve-small.gif

### 1. 개념

- 지구를 **구(Sphere)** 로 보고, 이를 **정육면체에 투영**
- 각 면을 재귀적으로 분할해 셀 생성
- 셀 ID를 기반으로 공간 인덱싱

### 2. 왜 S2가 나왔는가?

- 지오해시의 **구면 왜곡 문제** 개선
- “지구는 평면이 아니다”를 전제로 설계

### 3. 구조적 특징

- Cell ID = 계층적 공간 키
- Level에 따라 해상도 조절
- 반경 검색 시 **covering** 으로 필요한 셀 집합 계산

### 4. 장점

- 구면 기준 공간 분할 → 정확성 높음
- 다양한 해상도에서 안정적인 반경 검색
- 대규모 글로벌 서비스에 적합

### 5. 단점

- 개념/구현 난이도 높음
- 라이브러리 의존도 큼
- 직접 구현은 비현실적

## 📌 쿼드 트리 vs 지오해시 vs S2

| 항목 | 쿼드 트리 | 지오해시 | S2 |
| --- | --- | --- | --- |
| 분할 기준 | 데이터 밀도 | 문자열 prefix | 구면 셀 |
| 해상도 | 동적 | prefix 길이 | level |
| 구현 난이도 | 중 | 낮음 | 높음 |
| 분산/샤딩 | ❌ 어려움 | ✅ 쉬움 | ✅ 쉬움 |
| 실무 사용 | 개념/로컬 | 매우 많음 | 대규모 서비스 |

---