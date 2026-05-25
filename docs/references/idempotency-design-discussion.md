# Idempotency 설계 검토 — 본 KB

> 본 문서는 2026-05-25 A-1 Transfer 속성 재검토 과정에서 진행된 **멱등성 관련 컬럼·테이블 분리 논의의 전체 흐름**을 향후 학습·재검토 가능하도록 원문에 가깝게 보존한 KB이다. ADR-0010(초기 결정)·ADR-0014(개정 결정)의 배경 지식.
>
> **연관 ADR**: [ADR-0010 멱등성·동시성](../decisions/0010-idempotency-concurrency.md), [ADR-0014 IdempotencyKey 별도 테이블 분리](../decisions/0014-idempotency-key-extraction.md)
> **연관 작업 산출물**: `.specify/work-in-progress/account-transfer-spec-draft.md` (A-1 Transfer, D-12 IdempotencyKey)

---

## 1. 논의의 배경

A-1 Transfer 드라이빙 테이블 1차 확정안에는 다음 4개 멱등 관련 컬럼이 포함되어 있었다:

| # | 컬럼 | 의미 |
|---|---|---|
| 19 | `idempotencyPhase` | `INITIATED` (transferId 발급만 됨) → `EXECUTED` (실행 요청 받음) |
| 20 | `initiatedAt` | 1-Phase 시각 (transferId 발급 = INSERT 시점) |
| 21 | `requestedAt` | 2-Phase 시각 (사용자가 실제 "이체 실행" 누른 시점) |
| 22 | `completedAt` | 종료(성공/실패/취소) 시각 |

사용자의 질문: **"`idempotencyPhase`와 `transferStatus`가 중복 아닌가? `transferStatus`만 있으면 어떤 동시성 케이스를 방어하지 못하는가?"**

이 질문은 매우 본질적이었고, 무비판적으로 "결제 API 2-Phase 패턴은 멱등 단계 마커가 필요하다"는 일반론을 적용한 초기 설계의 정당성을 재검증하는 계기가 되었다.

---

## 2. 두 컬럼의 정의 비교 (1차 분석)

| 측면 | `transferStatus` | `idempotencyPhase` |
|---|---|---|
| 값 셋 (제안) | `INITIATED` → `PROCESSING` → (TransferStateHistory가 세부 단계 추적) → `COMPLETED` / `FAILED` / `CANCEL_REQUESTED` / `CANCELED` | `INITIATED` → `EXECUTED` |
| 전이 횟수 | 여러 번 (생애 5~10회) | 1회 (생애 1번) |
| 단위 | 비즈니스 상태머신 | 멱등 단계 마커 |
| 갱신 시점 | 모든 도메인 이벤트 | 2-Phase EXECUTE 수신 시점 1회 |

표면적으로는 다른 차원이지만, 다음 두 등식이 성립한다:
- `transferStatus == INITIATED` ⟺ `idempotencyPhase == INITIATED`
- `transferStatus != INITIATED` ⟺ `idempotencyPhase == EXECUTED`

→ 의미 공간이 동등하다. 다른 이름으로 같은 정보를 중복 표현하는 것에 가깝다.

---

## 3. 시나리오 검증 (시간순 Top-Down)

각 시나리오에서 두 컬럼이 어떻게 채워지는지, 그리고 **"`transferStatus`만 있어도 같은 결정이 가능한가?"** 분석.

### 시나리오 A: 정상 흐름

| t | 이벤트 | `transferStatus` | `idempotencyPhase` | 서버 결정 | `idempotencyPhase` 단독 기여 |
|---|---|---|---|---|---|
| t1 | INITIATE (transferId 발급) | `INITIATED` | `INITIATED` | INSERT | (없음) |
| t2 | EXECUTE 수신 | `PROCESSING` | `EXECUTED` | UPDATE, 외부 호출 시작 | (없음 — `transferStatus != INITIATED` 체크로 충분) |
| t3~t6 | 외부 호출 진행 | (TransferStateHistory가 세부 추적) | `EXECUTED` (불변) | UPDATE, 다음 단계 진행 | (없음 — 이미 EXECUTED, 무의미) |
| t7 | 완료 | `COMPLETED` | `EXECUTED` (불변) | 최종 UPDATE | (없음) |

### 시나리오 B: 동일 transferId로 EXECUTE 재호출 (네트워크 재시도)

| t | 이벤트 | `transferStatus` | `idempotencyPhase` | 서버 결정 |
|---|---|---|---|---|
| t1 | INITIATE | `INITIATED` | `INITIATED` | INSERT |
| t2 | EXECUTE | `PROCESSING` | `EXECUTED` | 정상 진행 |
| **t2'** | **EXECUTE 재호출 (수ms 뒤)** | `PROCESSING` | `EXECUTED` | **캐시된 결과 반환, 추가 처리 X** |

서버의 판단 로직 비교:
- `idempotencyPhase` 기준: `if (idempotencyPhase == EXECUTED) return cached_result`
- `transferStatus` 단독 기준: `if (transferStatus != INITIATED) return cached_result`
- → 동일 결과. `idempotencyPhase` 단독 기여 없음.

### 시나리오 C: 동시 EXECUTE 호출 (진짜 race condition)

두 클라이언트(앱·재시도)가 **수 밀리초 차이로 동시에** EXECUTE 호출. 두 서버 워커가 동시에 처리.

| t | 워커1 | 워커2 | `transferStatus` | `idempotencyPhase` |
|---|---|---|---|---|
| t2.000 | SELECT → `INITIATED` 확인 | | `INITIATED` | `INITIATED` |
| t2.001 | | SELECT → `INITIATED` 확인 | `INITIATED` | `INITIATED` |
| t2.002 | UPDATE 시도 | UPDATE 시도 | **여기가 동시성 위험 지점** | |

이 시점에 두 워커가 모두 "phase=INITIATED 확인"한 상태. `idempotencyPhase`가 있든 없든 **두 워커 모두 다음 단계로 진입하려 함** → 이중 출금 가능.

**진짜 방어는 락 또는 조건부 UPDATE**:
```sql
-- transferStatus 단독으로도 가능
UPDATE transfer SET transfer_status='PROCESSING' 
 WHERE transfer_id=? AND transfer_status='INITIATED';
-- → 한 워커만 1 row 영향, 다른 워커는 0 row → 후자가 캐시 반환
```

→ `idempotencyPhase` **단독으로는 race를 막지 못함**. 락·DB 조건부 UPDATE가 필수이고, 그 조건절은 `transferStatus`만으로 충분.

### 시나리오 D: 진행 중 사용자 취소

| t | 이벤트 | `transferStatus` | `idempotencyPhase` |
|---|---|---|---|
| t1, t2 | INITIATE, EXECUTE | (정상) | (정상) |
| t5 | 사용자 취소 요청 | `CANCEL_REQUESTED` | `EXECUTED` (불변) |
| t5+α | 보상 처리 완료 | `CANCELED` | `EXECUTED` (불변) |

→ `idempotencyPhase`는 전혀 변하지 않고 의사결정에 사용되지 않음.

### 시나리오 E: 1-Phase 후 이탈 (EXECUTE 호출 없음)

| t | 이벤트 | `transferStatus` | `idempotencyPhase` |
|---|---|---|---|
| t1 | INITIATE | `INITIATED` | `INITIATED` |
| t1+24h | 정리 job 실행 | `INITIATED` (또는 `EXPIRED`) | `INITIATED` (또는 별도) |

정리 job 조건: `WHERE transferStatus='INITIATED' AND initiatedAt < now() - 24h` — `idempotencyPhase` 불필요.

---

## 4. 1차 결론

`idempotencyPhase`는 본 테이블에서 **제거 가능하다** — `transferStatus`로 동등하게 표현되고, race condition 방어는 락 메커니즘이 본질이므로 컬럼 분리가 강화하지 않는다.

다음 질문이 자연스럽게 이어졌다: **그렇다면 멱등성을 어디서·어떻게 관리할 것인가?**

---

## 5. D-12 IdempotencyKey 별도 엔터티 분석 (2차 분석)

### 정체

결제 API 산업 표준(Stripe, PayPal, 국내 PG 등)의 핵심 패턴인 **별도 멱등 추적 테이블**. 메인 거래 테이블(Transfer)과 분리하여 **멱등 처리 로직만 격리해서 담당**.

### 책임 분담

| 책임 | 담당 |
|---|---|
| **비즈니스 상태머신** (PROCESSING, COMPLETED, FAILED, CANCELED 등) | **Transfer** |
| **2-Phase 단계 마커** (INITIATED → EXECUTED) | **IdempotencyKey** |
| **요청 핑거프린트** (출금계좌+수취계좌+금액+사용자+시간윈도우 해시) | **IdempotencyKey** |
| **결과 응답 캐시** (재호출 시 즉시 반환할 JSON payload) | **IdempotencyKey** |
| **TTL 관리** (24h 등 만료 후 정리) | **IdempotencyKey** |
| **거래의 모든 도메인 정보** (금액·계좌·메모·관계 등) | **Transfer** |

### 제안 속성 (개념 수준)

```
IdempotencyKey
├─ transferId (PK3, Transfer와 1:1)
├─ userId (PK1)  
├─ keyDate (PK2)
├─ phase (INITIATED / EXECUTED)
├─ requestFingerprintHash (출금계좌+수취계좌+금액+사용자+시간윈도우 해시)
├─ resultPayloadCache (JSON, EXECUTED 후 응답 본문 저장)
├─ resultHttpStatus (응답 HTTP 상태)
├─ initiatedAt
├─ executedAt (EXECUTED 전이 시각)
├─ expiresAt (TTL 만료 시각, 예: initiatedAt + 24h)
└─ system columns 6종 (createdAt/createdById/...)
```

### 동작 시나리오 (Transfer + IdempotencyKey 협업)

#### 시나리오 A: 정상

| t | API 호출 | Transfer | IdempotencyKey |
|---|---|---|---|
| t1 | INITIATE | INSERT (transferStatus=INITIATED) | INSERT (phase=INITIATED, fingerprint=h1, payload=null) |
| t2 | EXECUTE | UPDATE (transferStatus=PROCESSING) | UPDATE (phase=EXECUTED, executedAt=t2) |
| t2.1 | (응답 직전) | — | UPDATE (resultPayloadCache=응답JSON) |
| t3~t7 | 외부 호출 진행, 완료 | UPDATE (PROCESSING→COMPLETED) | (불변) |

#### 시나리오 B: 동일 transferId 재호출 (네트워크 재시도)

| t | API 호출 | 서버 로직 | Transfer 조회 여부 |
|---|---|---|---|
| t2 | EXECUTE | (정상 흐름) | YES |
| t2' | EXECUTE 재호출 | **IdempotencyKey 조회 → phase=EXECUTED & payload 존재 → 캐시 즉시 반환** | **NO (Transfer 안 건드림)** |

→ 성능 이득: 재호출은 IdempotencyKey 1 row 조회만으로 종료. Transfer는 변경 0.

#### 시나리오 C: 핑거프린트 불일치 (의도된 연속 이체 vs 의심 케이스)

| t | API 호출 | 핑거프린트 | 서버 판단 |
|---|---|---|---|
| t1, t2 | A씨가 B씨에게 10만원 송금 | h1 | 정상 처리 |
| t10 | A씨가 B씨에게 10만원 송금 (또 다른 transferId 발급) | h1 | **로그 경고만** (논블로킹, ADR-0010), 진행 |

→ 핑거프린트는 의심 모니터링용. 강제 차단 X.

#### 시나리오 E: 1-Phase 후 이탈

| t | 이벤트 | Transfer | IdempotencyKey |
|---|---|---|---|
| t1 | INITIATE | INSERT (INITIATED) | INSERT (INITIATED, expiresAt=t1+24h) |
| t1+24h | TTL 만료 정리 job | (선택: soft delete or 유지) | DELETE |

→ 정리 job은 IdempotencyKey만 청소. Transfer 본 데이터는 보관기간 10년(ADR-0008) 정책에 따라 별도 관리.

### 결제 API 산업 표준과의 비교

| 시스템 | 패턴 |
|---|---|
| **Stripe** | `idempotency_keys` 별도 테이블, 메인 거래 테이블과 분리. phase·result·expires 관리 |
| **PayPal** | 별도 멱등 캐시 (Idempotency-Key 헤더 기반) |
| **국내 PG (토스페이먼츠 등)** | 거래 키와 멱등 키 분리. 멱등 캐시 별도 |
| **본 제안 D-12** | 동일 패턴 (Stripe와 거의 1:1) |

---

## 6. "`transferStatus`만으로 방어 못하는 케이스" 검증 (3차 분석)

사용자의 가장 본질적 질문이었다. 정직한 답은 **"기능적 방어는 거의 다 가능하다"**이다.

| 케이스 | `transferStatus` 단독 + 락 | `transferStatus` + Transfer에 추가 컬럼 | 별도 IdempotencyKey 테이블 |
|---|---|---|---|
| **(1)** 동일 transferId 재호출 → 캐시 응답 | ✅ 가능 (status 조회 + 응답 재구성) | ✅ (응답 캐시 컬럼 추가) | ✅ |
| **(2)** 동시 EXECUTE race condition | ✅ 가능 (`WHERE status='INITIATED'` 조건부 UPDATE / 락) | ✅ | ✅ |
| **(3)** 동일 출금계좌 동시 이체 | ✅ 가능 (계좌 락 — AccountLockState) | ✅ | ✅ |
| **(4)** 다른 transferId지만 동일 의미 거래 중복 감지 (핑거프린트) | ❌ (transferId 키만으로는 못 봄) | ✅ (핑거프린트 컬럼 추가) | ✅ |
| **(5)** 1-Phase 발급 후 미실행 정리 | ✅ 가능 (`WHERE status='INITIATED' AND initiatedAt < ...`) | ✅ | ✅ |
| **(6)** 응답 페이로드 비결정성 보장 (외부 거래 ID 같은 값) | ⚠️ 어려움 (재구성 시 외부 거래 ID 재조회 필요) | ✅ (응답 캐시 컬럼) | ✅ |

→ **거의 모든 방어는 컬럼만 추가하면 Transfer 단독으로도 가능하다.** "방어 가능 여부"는 차이의 본질이 아니다.

### 별도 IdempotencyKey 테이블의 진짜 가치

**기능이 아니라 성능·운영·격리** 차원이다.

| 차원 | Transfer에 통합 | IdempotencyKey 분리 |
|---|---|---|
| **재호출 응답 성능** | 활성 거래 테이블 락 경합 (예: 진행 중인 거래 UPDATE와 멱등 조회 SELECT 경합) | 별도 테이블 단일 row 조회 → 락 분리 |
| **인덱스 크기** | 만료된 멱등 캐시(24h)도 Transfer 인덱스에 영향. 10년 보관 데이터에 멱등용 컬럼이 NULL로 누적 | 24h만 유지, 만료 후 DELETE → 인덱스 항상 작음 |
| **보관 정책 충돌** | Transfer는 10년 보관(ADR-0008), 멱등 캐시는 24h만 필요 → 정책 분리 어려움 | TTL 자동 정리, 정책 명확 분리 |
| **코드 격리** | 비즈니스 로직과 멱등 로직이 같은 테이블에 묶임 | 멱등 어댑터가 IdempotencyKey만 다룸. Transfer는 도메인에 집중 |
| **스키마 진화** | 멱등 로직 변경 시 Transfer 스키마 영향 → 마이그레이션 비용 큼 (10년 보관 테이블) | IdempotencyKey만 변경 |
| **샤딩·파티션 영향** | 멱등용 컬럼이 Transfer 파티션 전략에 결합 | 독립 파티션·TTL 전략 가능 |

→ **방어 능력이 아니라 "대형 은행 NFR(5,000 TPS, 99.99%)에서 운영하기 좋은가"**가 차이의 본질.

### "`transferStatus` 단독으로 정말 못 막는 케이스"가 있나?

**거의 없다.** 굳이 꼽으면:

- **케이스 (4)** 핑거프린트 기반 의심 거래 감지 → `transferStatus` 자체와 무관한 별도 컬럼 추가가 필요 (위치는 Transfer든 IdempotencyKey든 무관)
- **케이스 (6)** 응답 비결정성 → 응답 캐시 컬럼이 필요 (위치 무관)

즉 진정한 의미의 "방어 능력 차이"는 **`transferStatus` 단독 vs (`transferStatus` + 핑거프린트·캐시 컬럼)**의 차이이고, **그 컬럼들을 어디에 두느냐**(Transfer vs IdempotencyKey)는 운영·성능 트레이드오프 결정이다.

---

## 7. 3가지 안 비교 및 최종 결정

| 안 | 설명 | 적합 컨텍스트 |
|---|---|---|
| **안 1** `transferStatus`만, 멱등 컬럼 일체 없음 | 가장 단순. 핑거프린트·캐시 X. 동일 transferId 재호출은 status 조회 + 응답 재구성 | 데모를 아주 단순하게. 운영 NFR 무시 |
| **안 2** `transferStatus` + Transfer에 핑거프린트·응답캐시 컬럼 추가 | 모든 방어 가능. 단일 테이블 단순함 | 중소 규모 시스템. 10년 보관과 24h 캐시 정책 충돌 수용 |
| **안 3** (산업 표준) Transfer는 비즈니스만, **별도 IdempotencyKey** 테이블 | 운영·성능·격리 우수. 코드 분리 | 대형 NFR + 시니어 실무 시연 |

### 본 프로젝트 컨텍스트에 비춘 의견

- **ADR-0002 (NFR 5,000 TPS · 99.99%)**: 안 3이 자연스러움
- **ADR-0008 (보관 10년)**: 멱등 캐시 24h와 분리 필요 → 안 3 정당화
- **시니어 실무 시연 가치**: 결제 API 산업 표준 패턴(Stripe 류)을 코드로 보이는 게 학습 자료로 우수 → 안 3

### 사용자 최종 결정: 안 3

ADR-0014로 정식 채택. ADR-0010의 일부(idempotencyPhase 컬럼 위치)는 본 결정으로 amend.

---

## 8. 향후 참고용 시사점

### A. 결정 자체에 미치는 시사

- Transfer 테이블 컬럼 정리 (#19 idempotencyPhase 제거, #20 initiatedAt 제거 — IdempotencyKey가 보유, createdAt과 중복)
- #21 requestedAt, #22 completedAt은 Transfer에 유지 (비즈니스 의미)
- D-12 IdempotencyKey 엔터티 상세 검토는 카테고리 D 진행 시 별도 라운드 (PK 구성·핑거프린트 알고리즘·TTL 정책 등)

### B. 일반적인 학습 가치

- "이름이 다르면 의미가 다르다"고 가정하지 말 것. 두 컬럼이 동등한 정보 공간을 다른 이름으로 표현하는지 의심하라.
- "필수냐 아니냐"의 질문보다 "어디에 두는 게 운영 가치가 큰가"의 질문이 본질일 때가 많다.
- 동시성 방어는 **컬럼 분리가 아니라 락·조건부 UPDATE**가 본질이다.
- 결제 API 산업 표준(Stripe 등)을 무비판적으로 적용하지 말고, **본 프로젝트의 NFR·보관 정책·시연 가치**에 비추어 재해석하라.

### C. 의사결정 패턴 (방법론으로서)

본 논의는 다음 단계를 거쳤다 — 향후 유사 결정에 재사용 가능한 패턴:

1. **사용자가 의심을 제기** ("두 컬럼이 중복 아닌가?")
2. **시간순 시나리오 표로 동작 시각화** (5~6개 케이스)
3. **각 컬럼이 단독으로 못 막는 케이스를 찾기**
4. **차이의 본질이 "기능 방어"인지 "운영·성능·격리"인지 분리**
5. **3가지 안으로 정리하여 트레이드오프 명시**
6. **본 프로젝트 컨텍스트(NFR·보관·시연 가치)에 비추어 권고**
7. **사용자 결정 + KB 보존 + ADR로 정식 기록**

---

## 9. 참조

- [ADR-0010 멱등성·동시성 (초기 결정)](../decisions/0010-idempotency-concurrency.md) — Amended by ADR-0014
- [ADR-0014 IdempotencyKey 별도 테이블 분리](../decisions/0014-idempotency-key-extraction.md)
- [ADR-0009 Transfer 엔터티 구조](../decisions/0009-transfer-structure-driving-driven.md)
- [ADR-0002 프로젝트 성격 + NFR](../decisions/0002-project-character-and-nfr.md)
- [ADR-0008 취소·보관·FDS 정책](../decisions/0008-cancellation-retention-fds.md)
- Stripe Idempotency 공식 문서: https://docs.stripe.com/api/idempotent_requests
