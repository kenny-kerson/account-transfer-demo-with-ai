# ADR-0014: IdempotencyKey 별도 테이블 분리 (Transfer에서 멱등 컬럼 추출)

- Status: Accepted (Amends [ADR-0010](0010-idempotency-concurrency.md))
- Date: 2026-05-25
- Deciders: kenny.k
- Tags: data-model, idempotency, concurrency

## Context

ADR-0009의 Transfer 드라이빙 테이블 1차 확정안에는 멱등 관련 4개 컬럼이 포함되어 있었다 (#19 `idempotencyPhase`, #20 `initiatedAt`, #21 `requestedAt`, #22 `completedAt`).

A-1 속성 재검토 단계에서 사용자가 본질적 질문을 제기:
- "`idempotencyPhase`와 `transferStatus`가 중복 아닌가?"
- "`transferStatus`만으로 방어 못하는 동시성 케이스가 있는가?"

다단계 검증 결과 (전체 분석은 [매핑전략 KB와 별개로 idempotency-design-discussion KB](../references/idempotency-design-discussion.md) 참조):
1. `idempotencyPhase`는 `transferStatus`와 의미 공간이 동등하다
2. 동시성 방어는 **락·조건부 UPDATE가 본질**이고, 컬럼 분리가 강화하지 않는다
3. 차이의 본질은 "기능 방어"가 아니라 **"운영·성능·격리"**이다
4. 별도 `IdempotencyKey` 테이블 패턴은 결제 API 산업 표준(Stripe, PayPal, 토스페이먼츠 등)에 일치한다

## Decision

### 1. Transfer 테이블에서 멱등 컬럼 정리

| Transfer 컬럼 | 처리 | 사유 |
|---|---|---|
| #19 `idempotencyPhase` | **제거** | `transferStatus`로 동등 표현 가능 |
| #20 `initiatedAt` | **제거** | `createdAt`(시스템 컬럼)이 동일 시각. 도메인적 1-Phase 시각은 IdempotencyKey가 보유 |
| #21 `requestedAt` | **유지** | 비즈니스 의미 ("사용자가 이체 실행을 요청한 시점") — IdempotencyKey의 `executedAt`(멱등 관점)과 의미 차원이 다름. 두 컬럼 모두 시각은 동일하지만 도메인 관점에서 명확한 보존 가치 있음 |
| #22 `completedAt` | **유지** | 비즈니스 종료 시각 (성공/실패/취소 통합). SLA·보관기간·통계의 핵심 필드 |

### 2. IdempotencyKey 엔터티 정식화 (D-12)

별도 테이블로 분리하여 다음 책임을 담당:
- 2-Phase 단계 마커 (`INITIATED` → `EXECUTED`)
- 요청 핑거프린트 (출금계좌+수취계좌+금액+사용자+시간윈도우 해시)
- 결과 응답 캐시 (재호출 시 즉시 반환할 JSON payload + HTTP status)
- TTL 관리 (24h 등 만료 후 정리, plan 단계 확정)

제안 속성 (개념 수준):
```
IdempotencyKey
├─ transferId (Transfer와 1:1)
├─ userId
├─ keyDate
├─ phase (INITIATED / EXECUTED)
├─ requestFingerprintHash
├─ resultPayloadCache (JSON)
├─ resultHttpStatus
├─ initiatedAt
├─ executedAt
├─ expiresAt
└─ system columns 6종 (ADR-0011)
```

> 상세 PK 구성·핑거프린트 알고리즘·TTL 정책 등은 카테고리 D 검토 단계에서 별도 라운드로 확정.

### 3. 멱등 처리 로직 위치 변경

- **재호출 처리**: 우선 IdempotencyKey 단독 조회로 캐시된 응답 반환 (Transfer 접촉 X)
- **동시성 방어**: Transfer의 `transferStatus` 조건부 UPDATE (`WHERE transferStatus='INITIATED'`) + AccountLockState 다층 락. ADR-0010의 다층 락 정책은 그대로 유효
- **정리 job**: IdempotencyKey TTL 만료 처리만 수행. Transfer 본 데이터는 보관기간 10년(ADR-0008) 정책에 따라 별도 관리

## Rationale

본 결정의 가치는 **기능 방어 능력이 아니라 운영·성능·격리** 차원이다:

| 차원 | Transfer 통합 (이전) | IdempotencyKey 분리 (현재) |
|---|---|---|
| 재호출 응답 성능 | 활성 거래 테이블 락 경합 | 별도 테이블 단일 row 조회 |
| 인덱스 크기 | 만료 캐시까지 10년 누적 | 24h만 유지, 항상 작음 |
| 보관 정책 | 10년 vs 24h 충돌 | TTL 자동 정리, 정책 분리 |
| 코드 격리 | 비즈니스+멱등 결합 | 멱등 어댑터 격리 |
| 스키마 진화 | 10년 보관 테이블 마이그레이션 비용 | IdempotencyKey만 변경 |
| 산업 표준 | 비표준 | Stripe·PayPal·국내 PG 패턴 일치 |

본 프로젝트의 NFR(5,000 TPS · 99.99% — ADR-0002) + 보관 정책(10년 — ADR-0008) + 시니어 실무 시연 가치를 종합하면, 운영 가치가 명확하다.

## Alternatives Considered

- **안 1: `transferStatus`만 유지, 멱등 컬럼 일체 없음** — 거부됨. 핑거프린트·응답캐시 부재로 의심거래 감지·재호출 응답 정확성 약함
- **안 2: Transfer에 핑거프린트·응답캐시 컬럼 추가, IdempotencyKey 테이블 없음** — 거부됨. 운영·성능·격리 차원 약함. 단일 테이블 단순함이 NFR·보관 정책과 충돌
- **안 3 (채택): 별도 IdempotencyKey 테이블** — 산업 표준, 운영 가치 명확, 시연 가치 우수

## Consequences

- (+) 결제 API 산업 표준 패턴 시연 (학습 자료 가치)
- (+) Transfer 테이블이 비즈니스에 집중, 멱등 로직 격리
- (+) 재호출 응답이 IdempotencyKey 1 row 조회로 종료 → 성능 이득
- (+) TTL 정리 정책 분리 (24h vs 10년)
- (-) 테이블 1개 추가 (관리 포인트 증가)
- (-) INITIATE/EXECUTE 시 2개 테이블 동시 변경 (트랜잭션 정합성 필요)
- (-) ADR-0010의 일부가 amend됨 → 향후 ADR-0010 참조 시 본 ADR도 함께 검토 필요
- (?) D-12 IdempotencyKey 엔터티의 상세 속성·PK·핑거프린트·TTL 등은 카테고리 D 검토 단계에서 별도 라운드로 확정

## References

- [Idempotency 설계 검토 KB](../references/idempotency-design-discussion.md) — 전체 분석 흐름, 시나리오, 트레이드오프 비교 자세히
- [ADR-0010 멱등성·동시성 (초기 결정, amended)](0010-idempotency-concurrency.md)
- [ADR-0009 Transfer 엔터티 구조](0009-transfer-structure-driving-driven.md)
- [ADR-0002 프로젝트 성격 + NFR](0002-project-character-and-nfr.md)
- [ADR-0008 취소·보관·FDS 정책](0008-cancellation-retention-fds.md)
- [Stripe Idempotency 공식 문서](https://docs.stripe.com/api/idempotent_requests)
