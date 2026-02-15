# 5장 딥다이브: 풀/푸시 모델 심층 분석 & 시계열 DB 비교

> 본 문서는 5장 발표자료(README.md)의 보충 자료입니다.  
> 책 내용을 넘어서는 외부 리서치를 별도 파일로 분리하였습니다.

---

## 목차

1. [풀 vs 푸시 모델 딥다이브](#1-풀-vs-푸시-모델-딥다이브)
2. [시계열 DB 비교 분석](#2-시계열-db-비교-분석)
3. [참고 자료](#3-참고-자료)

---

## 1. 풀 vs 푸시 모델 딥다이브

책에서는 풀/푸시의 장단점을 비교표로 정리했다. 여기서는 각 모델의 **내부 동작 원리**와 **실무 선택 기준**을 더 깊이 파고든다.

### 1.1 풀 모델 내부 아키텍처 (Prometheus 기준)

Prometheus의 풀 모델은 4단계 사이클로 동작한다:

```
서비스 디스커버리 → 타겟 목록 갱신 → HTTP GET /metrics → TSDB 기록
```

**① 서비스 디스커버리 (Target Discovery)**

Prometheus는 수집 대상을 자동으로 찾기 위해 다양한 디스커버리 메커니즘을 지원한다:

| 방식 | 동작 | 사용 환경 |
|------|------|----------|
| `static_configs` | YAML에 IP:포트 직접 기재 | 소규모, 고정 인프라 |
| `kubernetes_sd_configs` | K8s API로 Pod/Service/Endpoint 자동 탐색 | 쿠버네티스 |
| `consul_sd_configs` | Consul 서비스 카탈로그 조회 | HashiCorp 스택 |
| `dns_sd_configs` | DNS SRV 레코드 조회 | 전통적 인프라 |
| `file_sd_configs` | 외부 도구가 갱신하는 JSON/YAML 파일 읽기 | 커스텀 통합 |

쿠버네티스 환경에서는 약 30초 간격으로 API 서버에 질의하여 새로 생성되거나 삭제된 Pod를 감지한다.

**② 스크레이프 주기와 타임아웃**

```yaml
global:
  scrape_interval: 30s    # 기본 수집 주기
  scrape_timeout: 10s     # 응답 대기 시간 (반드시 interval보다 짧아야 함)

scrape_configs:
  - job_name: 'api-server'
    scrape_interval: 15s   # job별 오버라이드 가능
    scrape_timeout: 5s
```

- 스크레이프 주기(interval)마다 각 타겟에 `GET /metrics` 요청을 보낸다
- 타임아웃 내에 응답이 없으면 해당 타겟을 `down`으로 표시한다
- `up{job="api-server", instance="10.0.1.5:8080"} = 0` 지표가 자동 생성된다

**③ Staleness(부실 데이터) 처리**

Prometheus는 타겟이 사라졌을 때 "stale marker"를 기록한다:
- 마지막 스크레이프 이후 5분(기본값)이 지나면 시계열을 부실(stale)로 표시
- PromQL 질의 시 부실 데이터는 자동으로 제외된다
- 이 메커니즘 덕분에 Pod가 삭제되어도 대시보드에 유령 시계열이 남지 않는다

**④ 수집기 간 중복 방지 — 안정 해시 링**

책에서 설명한 안정 해시 링이 실제로 어떻게 적용되는지:

```
수집기 A: 해시 범위 [0, 33%) → 타겟 1, 4, 7 담당
수집기 B: 해시 범위 [33%, 66%) → 타겟 2, 5, 8 담당  
수집기 C: 해시 범위 [66%, 100%) → 타겟 3, 6, 9 담당
```

수집기 B가 장애를 겪으면, B의 범위가 A와 C에 재분배된다. 전체 재배치 없이 인접 구간만 조정하므로 영향 범위가 최소화된다.

### 1.2 푸시 모델 내부 아키텍처

푸시 모델은 에이전트-수집기-백엔드 3계층으로 동작한다:

```
에이전트(수집) → 배치 버퍼링 → 게이트웨이/수집기 전송 → 백엔드 저장
```

**에이전트의 배치 전송 로직**

에이전트는 지표를 즉시 보내지 않고, 내부 버퍼에 모았다가 일정 조건이 충족되면 한꺼번에 전송한다:

| 전송 트리거 | 설명 |
|------------|------|
| 시간 기반 | 10초마다 버퍼 내용 전송 (flush interval) |
| 크기 기반 | 버퍼가 1,000개 지표에 도달하면 즉시 전송 |
| 종료 시 | 프로세스 종료 직전 남은 버퍼 전송 (graceful shutdown) |

**UDP vs TCP 트레이드오프**

| 비교 축 | UDP (StatsD 방식) | TCP (CloudWatch/Datadog 방식) |
|---------|-------------------|-------------------------------|
| 전송 보장 | ❌ 보장 없음 (fire-and-forget) | ✅ 수신 확인 (ACK) |
| 오버헤드 | 패킷당 ~8바이트 | 연결 유지 + 핸드셰이크 비용 |
| 처리량 | 초당 10만+ 패킷 가능 | 연결 수에 따라 제한 |
| 데이터 손실 | 네트워크 혼잡 시 패킷 유실 | 백프레셔로 속도 조절 |
| 적합 환경 | 고빈도·저정밀 지표 (카운터 등) | 중요 지표·정확성 필수 환경 |

> StatsD는 "초당 요청 수" 같은 고빈도 카운터에 UDP를 사용한다. 패킷 몇 개가 유실되어도 전체 추세에는 영향이 미미하기 때문이다. 반면 CloudWatch Agent는 TCP로 전송하여 과금 지표 등 정확성이 중요한 데이터의 손실을 방지한다.

**백프레셔(Backpressure)와 데이터 손실 위험**

푸시 모델의 가장 큰 위험은 백엔드가 과부하일 때 발생한다:

```
에이전트 → [버퍼 가득 참] → 수집기 → [처리 지연] → 백엔드 (과부하)
                ↓
        오래된 데이터 폐기 (drop oldest)
        또는 새 데이터 거부 (drop newest)
```

완화 전략:
1. **로컬 디스크 버퍼링**: 수집기 장애 시 디스크에 임시 저장
2. **우선순위 큐**: 중요 지표(경보용)를 먼저 전송
3. **샘플링**: 과부하 시 일부 지표만 선택적 전송
4. **데드 레터 큐**: 전송 실패 건을 별도 저장 후 재처리

### 1.3 하이브리드 접근법 — 현대적 트렌드

실무에서는 풀과 푸시를 결합하는 하이브리드 패턴이 주류가 되고 있다.

**패턴 1: Prometheus remote_write (로컬 풀 → 원격 푸시)**

```
[K8s 클러스터 A]                    [중앙 백엔드]
Prometheus ──pull──→ Pod들           
    │                                Thanos / Cortex / 
    └──remote_write(push)──────────→ VictoriaMetrics
                                         ↑
[K8s 클러스터 B]                         │
Prometheus ──pull──→ Pod들               │
    └──remote_write(push)────────────────┘
```

각 클러스터 내부에서는 풀 모델로 수집하고, 중앙 저장소로는 푸시 모델로 전송한다. 이렇게 하면:
- 클러스터 내부: 풀 모델의 디버깅 용이성 확보
- 클러스터 간: 푸시 모델의 네트워크 유연성 확보
- 중앙 저장소: 글로벌 뷰 + 장기 보관

**패턴 2: OpenTelemetry Collector (프로토콜 브릿지)**

OpenTelemetry Collector는 풀과 푸시를 모두 수신하여 통합 파이프라인으로 처리한다:

```
[풀 소스] Prometheus 엔드포인트 ──scrape──→ ┐
                                            │
[푸시 소스] OTLP gRPC/HTTP ──push──────────→ ├→ OTel Collector → 백엔드
                                            │
[푸시 소스] StatsD UDP ──push──────────────→ ┘
```

**패턴 3: Grafana Agent/Alloy (풀→푸시 변환기)**

Grafana Agent는 Prometheus처럼 타겟을 스크레이프하되, 로컬 TSDB 없이 즉시 원격 백엔드로 푸시한다. Prometheus의 "가벼운 버전"으로, 로컬 저장소가 필요 없는 환경에 적합하다.

### 1.4 선택 기준 프레임워크

**규모별 권장 모델**

| 규모 | 타겟 수 | 권장 모델 | 이유 |
|------|---------|----------|------|
| 소규모 | ~100 | 풀 (Prometheus 단독) | 단순, 디버깅 용이 |
| 중규모 | ~10,000 | 풀 + remote_write | 클러스터별 풀, 중앙 푸시 |
| 대규모 | 100,000+ | 하이브리드 (OTel Collector) | 프로토콜 통합, 계층적 수집 |

**네트워크 토폴로지별 권장 모델**

| 토폴로지 | 권장 모델 | 이유 |
|----------|----------|------|
| 동일 VPC 내 | 풀 | 낮은 지연, 직접 접근 가능 |
| 크로스 리전 | 하이브리드 | 로컬 풀 + 리전 간 푸시 |
| 엣지/IoT | 푸시 | 불안정한 연결, 방화벽 제약 |
| 서버리스 | 푸시 | 에이전트 설치 불가, 짧은 수명 |

**의사결정 플로우차트**

```
타겟이 장기 실행 서비스인가?
├── Yes → 네트워크 접근이 가능한가?
│         ├── Yes → 풀 모델 (Prometheus)
│         └── No  → 하이브리드 (로컬 풀 + remote_write)
└── No  → 서버리스/배치 작업인가?
          ├── Yes → 푸시 모델 (OTel/CloudWatch)
          └── No  → 하이브리드 (Pushgateway + 풀)
```

### 1.5 실제 기업 사례

| 기업 | 아키텍처 | 핵심 교훈 |
|------|---------|----------|
| **Uber** | 인프라=풀, 앱=푸시 하이브리드 | 10만+ 마이크로서비스에서 "인프라는 풀, 앱은 푸시" 패턴 정착 |
| **Netflix** | Prometheus 풀 코어 + 엣지 푸시 | "단순한 아키텍처가 더 잘 확장된다" — 리전 간 Federation 활용 |
| **Kubernetes 생태계** | Prometheus 풀 모델이 사실상 표준 | Pod의 동적 IP를 K8s SD가 자동 추적 → 풀 모델의 최대 강점 발휘 |

> **왜 쿠버네티스에서 풀 모델이 이겼는가?**  
> K8s에서는 Pod가 수시로 생성·삭제되며 IP가 바뀐다. 풀 모델은 서비스 디스커버리와 결합하여 이 변화를 자동 추적한다. 반면 푸시 모델은 각 Pod에 에이전트를 설치하고 수집기 주소를 설정해야 하므로, 동적 환경에서 관리 부담이 크다.

---

## 2. 시계열 DB 비교 분석

책에서는 InfluxDB를 시계열 DB의 대표 사례로 소개했다. 여기서는 주요 시계열 DB 5종의 **저장 엔진 아키텍처**, **성능 특성**, **운영 트레이드오프**를 비교한다.

### 2.1 InfluxDB — 책의 선택

**저장 엔진: TSM Tree (Time-Structured Merge Tree)**

LSM 트리에서 영감을 받았지만 시계열에 최적화된 구조다:

```
쓰기 요청 → WAL(Write-Ahead Log) → 메모리 캐시 → TSM 파일(디스크)
                                                      ↓
                                              컴팩션(병합·압축)
```

| 파일 유형 | 역할 |
|----------|------|
| `.wal` | 쓰기 선행 로그 — 장애 복구용 |
| `.tsm` | 압축된 시계열 데이터 (컬럼 단위 저장) |
| `.tsi` | TSI(Time Series Index) — 태그 기반 역인덱스, Robin Hood 해싱 사용 |

**질의어: InfluxQL + Flux**

- **InfluxQL**: SQL과 유사한 문법. 진입 장벽이 낮다
- **Flux**: 함수형 데이터 스크립팅 언어. 복잡한 변환·조인이 가능하지만 학습 곡선이 가파르다

```flux
// Flux 예시: 최근 1시간 CPU 지표의 5분 이동 평균
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu" and r._field == "usage_idle")
  |> movingAverage(n: 5)
```

**클러스터링 & 고가용성**

| 에디션 | 클러스터링 | 비용 |
|--------|----------|------|
| OSS (오픈소스) | ❌ 단일 노드만 | 무료 |
| Enterprise | ✅ 샤딩 + 복제 + HA | 유료 |
| InfluxDB Cloud | ✅ 관리형 클러스터 | 종량제 |

> ⚠️ **핵심 트레이드오프**: 오픈소스 버전에서는 클러스터링이 불가능하다. 고가용성이 필요하면 유료 버전을 쓰거나, Kubernetes 위에서 외부 오케스트레이션으로 HA를 구성해야 한다.

**압축**: 델타 인코딩 + 런렝스 인코딩(RLE). 원본 대비 3~7배 압축.

**대표 사용 기업**: Tesla(차량 텔레메트리), PayPal(금융 지표), Cisco(네트워크 모니터링)

---

### 2.2 Prometheus TSDB — 쿠버네티스의 사실상 표준

**저장 엔진: 3계층 블록 구조**

```
스크레이프 데이터 → Head Block(메모리 + WAL) → 2시간마다 디스크 블록으로 전환 → 컴팩션
```

| 계층 | 위치 | 역할 |
|------|------|------|
| Head Block | 메모리 + WAL | 최근 2시간 데이터. 빠른 쓰기 |
| Disk Block | 디스크 | 불변(immutable) 2시간 단위 블록 |
| Compacted Block | 디스크 | 여러 블록을 병합·다운샘플링한 장기 블록 |

**질의어: PromQL**

모니터링 업계의 사실상 표준 질의어. 벡터 연산에 특화되어 있다:

```promql
# 최근 5분간 HTTP 요청 에러율 (%)
sum(rate(http_requests_total{status=~"5.."}[5m]))
/ sum(rate(http_requests_total[5m])) * 100
```

**클러스터링 & 장기 저장소**

Prometheus 자체는 **단일 노드** 설계다. 기본 보관 기간은 15일. 장기 저장이 필요하면 외부 솔루션을 붙여야 한다:

| 솔루션 | 아키텍처 | 특징 |
|--------|---------|------|
| **Thanos** | 사이드카 패턴 + 오브젝트 스토리지(S3 등) | 글로벌 뷰, 다운샘플링, 무제한 보관 |
| **Cortex** | 마이크로서비스 (Distributor→Ingester→Querier) | 멀티테넌트, 수평 확장 |
| **Grafana Mimir** | Cortex 후속작 | 더 단순한 운영, 높은 성능 |

> **핵심 한계**: 단일 노드이므로 메모리가 부족하면 OOM(Out of Memory)으로 죽는다. 대규모 환경에서는 반드시 샤딩(여러 Prometheus 인스턴스로 분할) + 장기 저장소 조합이 필요하다.

**대표 사용 기업**: Kubernetes 생태계 전체, GitLab, Spotify, Red Hat(OpenShift)

---

### 2.3 TimescaleDB — PostgreSQL 위의 시계열 확장

**저장 엔진: 하이퍼테이블(Hypertable) + 청크(Chunk)**

TimescaleDB는 독립 DB가 아니라 **PostgreSQL 확장(extension)**이다:

```
일반 테이블 → CREATE_HYPERTABLE() → 시간 기반 자동 파티셔닝(청크)
```

| 개념 | 설명 |
|------|------|
| 하이퍼테이블 | 사용자가 보는 가상 테이블. 내부적으로 여러 청크로 분할 |
| 청크 | 시간 범위별 실제 테이블 (기본 7일 단위) |
| 압축 | 청크 단위로 컬럼 압축 적용. 4~8배 압축 |

**최대 강점: 완전한 SQL 호환**

```sql
-- 표준 SQL로 시계열 질의 가능
SELECT time_bucket('5 minutes', time) AS bucket,
       avg(cpu_usage) AS avg_cpu
FROM metrics
WHERE time > NOW() - INTERVAL '1 hour'
  AND host = 'web-01'
GROUP BY bucket
ORDER BY bucket;
```

기존 PostgreSQL 팀이라면 새로운 질의어를 배울 필요 없이 바로 사용할 수 있다. JOIN, 서브쿼리, 윈도우 함수 등 관계형 DB의 모든 기능을 시계열 데이터에 그대로 적용 가능하다.

**클러스터링**: PostgreSQL의 스트리밍 복제 + Patroni(자동 페일오버) 활용. 전용 TSDB보다 운영 복잡도가 높지만, 이미 PostgreSQL을 운영하는 팀에게는 추가 인프라 없이 시계열 기능을 얻을 수 있다.

**대표 사용 기업**: Bloomberg(금융 데이터), BMW(텔레매틱스), Siemens(산업 IoT)

---

### 2.4 VictoriaMetrics — Prometheus의 장기 저장소 강자

**저장 엔진: 메모리 매핑 + 시간 기반 파티셔닝**

```
수신 데이터 → vminsert(라우팅) → vmstorage(저장) → vmselect(질의)
```

| 배포 모드 | 구성 | 적합 환경 |
|----------|------|----------|
| 단일 노드 | 바이너리 하나로 실행 | 중소규모 (초당 100만 샘플 이하) |
| 클러스터 | vminsert + vmstorage + vmselect 분리 | 대규모 (초당 1,000만+ 샘플) |

**왜 주목받는가?**

1. **압축률**: Prometheus 대비 **10배** 더 효율적인 저장. 동일 데이터를 1/10 디스크로 보관
2. **PromQL 100% 호환**: Prometheus의 드롭인 대체재. 기존 대시보드·경보 규칙을 그대로 사용
3. **MetricsQL 확장**: PromQL의 상위 호환 질의어. `default`, `keep_last_value` 등 실무에 유용한 함수 추가
4. **운영 단순성**: 단일 바이너리 배포. 외부 의존성 없음

**Prometheus → VictoriaMetrics 마이그레이션 패턴**

```yaml
# Prometheus의 remote_write로 VictoriaMetrics에 데이터 전송
remote_write:
  - url: "http://victoriametrics:8428/api/v1/write"
```

이 한 줄 설정으로 Prometheus의 장기 저장소를 VictoriaMetrics로 교체할 수 있다.

**대표 사용 기업**: Cloudflare(CDN 지표), Booking.com, Robinhood(금융), Xiaomi(IoT)

---

### 2.5 QuestDB — 초고속 수집 특화

**저장 엔진: 3계층 컬럼 스토리지**

```
Tier 1: WAL (핫 수집) → Tier 2: 바이너리 컬럼 (실시간 SQL) → Tier 3: Apache Parquet (콜드 저장)
```

| 특징 | 설명 |
|------|------|
| 컬럼 지향 저장 | 각 컬럼을 별도 파일로 저장. 분석 질의에 최적 |
| Zero-GC Java | 가비지 컬렉션 없는 Java 구현. GC 일시 정지(pause) 없음 |
| 메모리 매핑 파일 | OS 페이지 캐시를 활용한 고속 I/O |
| Parquet 통합 | 콜드 데이터를 오픈 포맷으로 저장. 벤더 종속 없음 |

**최대 강점: 수집 성능**

단일 머신에서 **초당 100만+ 행** 수집이 가능하다고 주장한다. 금융 시장 데이터처럼 초당 수백만 이벤트가 발생하는 환경에 적합하다.

**질의어: SQL + 시계열 확장**

```sql
-- QuestDB 전용 시계열 SQL 확장
SELECT timestamp, symbol, price,
       avg(price) OVER (PARTITION BY symbol ORDER BY timestamp 
                        ROWS BETWEEN 10 PRECEDING AND CURRENT ROW) AS ma_10
FROM trades
WHERE timestamp IN '2024-01-01'
SAMPLE BY 1m;
```

`SAMPLE BY`, `LATEST ON`, `ASOF JOIN` 등 시계열 전용 SQL 확장을 제공한다.

**클러스터링**: 단일 노드 중심 설계. Kubernetes Operator로 HA 구성 가능하지만, 네이티브 클러스터링은 아직 성숙하지 않다.

**대표 사용 기업**: JPMorgan(시장 데이터), NASA(우주 미션 데이터), Ericsson(네트워크 텔레메트리)

---

### 2.6 종합 비교표

#### 아키텍처 비교

| DB | 저장 엔진 | 인덱싱 | 압축 방식 | 파티셔닝 |
|----|----------|--------|----------|---------|
| **InfluxDB** | TSM Tree + TSI | 태그 역인덱스 | 델타 + RLE | 시간 기반 |
| **Prometheus** | Head Block + Disk Block | 메모리 시리즈 메타데이터 | 델타 + Tombstone | 2시간 블록 |
| **TimescaleDB** | PostgreSQL 하이퍼테이블 | B-tree + 시간 최적화 | TOAST + 컬럼 압축 | 시간 청크 |
| **VictoriaMetrics** | 메모리 매핑 | 레이블 해싱 | 델타 + 딕셔너리 | 시간 기반 |
| **QuestDB** | 3계층 (WAL→바이너리→Parquet) | 컬럼 메타데이터 | 딕셔너리 + RLE | 시간 파티션 |

#### 성능 비교

| DB | 쓰기 처리량 | 질의 성능 | 저장 효율 | 메모리 사용 |
|----|-----------|----------|----------|-----------|
| **InfluxDB** | 높음 (25만/초, 8코어 기준) | 중간 | 중간 (3~7x 압축) | 중간 |
| **Prometheus** | 중간 (10~50만/초) | 높음 (PromQL 최적화) | 낮음 | 높음 |
| **TimescaleDB** | 중~높음 (10~50만/초) | 중간 | 중~높음 (4~8x) | 높음 |
| **VictoriaMetrics** | 매우 높음 (100만+/초) | 높음 | 매우 높음 (10x) | 중간 |
| **QuestDB** | 매우 높음 (100만+/초) | 매우 높음 | 높음 (5~10x) | 중간 |

#### 운영 비교

| DB | 네이티브 클러스터링 | 질의어 | SQL 지원 | 학습 곡선 |
|----|-------------------|--------|---------|----------|
| **InfluxDB** | Enterprise만 | InfluxQL + Flux | 부분적 | 중간 (Flux) |
| **Prometheus** | ❌ (외부 필요) | PromQL | ❌ | 낮음 |
| **TimescaleDB** | PostgreSQL 복제 | SQL + 확장 | ✅ 완전 | 낮음 (SQL 안다면) |
| **VictoriaMetrics** | ✅ (클러스터 모드) | PromQL + MetricsQL | 부분적 | 낮음 (PromQL 안다면) |
| **QuestDB** | ❌ (K8s Operator) | SQL + 확장 | ✅ 완전 | 낮음 (SQL 안다면) |

### 2.7 시스템 설계 면접에서의 선택 기준

**워크로드별 추천**

| 워크로드 | 1순위 | 2순위 | 이유 |
|----------|------|------|------|
| K8s 인프라 모니터링 | Prometheus | VictoriaMetrics | PromQL 생태계, K8s 네이티브 통합 |
| 대용량 수집 (100만+/초) | VictoriaMetrics | QuestDB | 압축률·처리량 최고 |
| 기존 PostgreSQL 팀 | TimescaleDB | — | SQL 호환, 추가 인프라 불필요 |
| IoT / 센서 데이터 | InfluxDB | QuestDB | 사용 편의성, 내장 다운샘플링 |
| 금융 시장 데이터 | QuestDB | TimescaleDB | 초고속 수집, SQL 분석 |
| 장기 보관 (1년+) | VictoriaMetrics | Thanos(Prometheus) | 10x 압축, 저비용 저장 |

**면접 팁**: "왜 InfluxDB를 선택했는가?"라는 질문에는 다음과 같이 답할 수 있다:
> "InfluxDB는 시계열 데이터에 최적화된 TSM 엔진으로 단일 서버에서 초당 25만 쓰기를 처리하며, 내장 다운샘플링과 보관 정책으로 운영 부담이 적습니다. 다만 OSS 버전은 클러스터링이 불가하므로, 대규모 환경에서는 VictoriaMetrics나 Thanos를 장기 저장소로 병행하는 것을 고려하겠습니다."

---

## 3. 참고 자료

### 풀/푸시 모델
- [Prometheus - Pull doesn't scale, or does it?](https://prometheus.io/blog/2016/07/23/pull-does-not-scale-or-does-it/)
- [Push vs Pull in Monitoring Systems (Giedrius Blog)](https://giedrius.blog/2019/05/11/push-vs-pull-in-monitoring-systems/)
- [OpenTelemetry Collector Architecture](https://opentelemetry.io/docs/collector/architecture/)
- [Grafana Agent Documentation](https://grafana.com/docs/agent/latest/)

### 시계열 DB
- [InfluxDB TSM Engine Internals](https://docs.influxdata.com/influxdb/v2/reference/internals/storage-engine/)
- [Prometheus TSDB Design](https://ganeshvernekar.com/blog/prometheus-tsdb-the-head-block/)
- [TimescaleDB Architecture](https://docs.timescale.com/about/latest/timescaledb-editions/)
- [VictoriaMetrics: How it works](https://docs.victoriametrics.com/single-server-victoriametrics/)
- [QuestDB Storage Model](https://questdb.io/docs/concept/storage-model/)

### 벤치마크
- [InfluxDB Benchmarks (공식)](https://www.influxdata.com/blog/influxdb-benchmarks/)
- [VictoriaMetrics vs Prometheus Compression](https://valyala.medium.com/prometheus-vs-victoriametrics-benchmark-on-node-exporter-metrics-4ca29c75590f)
- [QuestDB vs InfluxDB Ingestion Benchmark](https://questdb.io/blog/questdb-versus-influxdb/)

---

*Last Updated: 2026-02-12*
