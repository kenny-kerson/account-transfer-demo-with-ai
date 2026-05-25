# ADR-0011: 데이터 모델 컨벤션 (PK·prefix·시스템 컬럼)

- Status: Accepted
- Date: 2026-05-25
- Deciders: kenny.k
- Tags: data-model, conventions

## Context

A-1 Transfer 속성 검토 단계에서 사용자가 모든 엔터티에 일관 적용할 5가지 컨벤션을 일괄 지정.

## Decision

### 1. PK 명시
각 테이블 정의 시 PK 컬럼을 별도로 표기 (PK1/PK2/PK3 등 위치도 표시).

### 2. 복합 PK `(사용자ID, 일자, 식별자)`
- **사용자별 조회 path 대응** + **사용자별 샤딩 또는 일자별 파티션 대응**
- 사용자 비귀속 엔터티(예: 정책 마스터)는 도메인에 맞는 다른 PK 패턴

### 3. 컬럼명에 도메인 prefix 강제
컬럼명만 봐도 의미가 자립하도록.
- `kind` → `transferKind`
- `memo` → `transferMemo`
- `status` → `transferStatus`
- `amount` → `transferAmount`

### 4. 은행계좌는 분리 컬럼
하나의 "계좌"는 항상 `*BankCode` + `*AccountNumber` 두 컬럼으로 분리.
- `fromAccount` → `fromBankCode` + `fromAccountNumber`
- `toAccount` → `toBankCode` + `toAccountNumber` (+ 필요 시 `toPayeeNameInput`)

### 5. 시스템 컬럼 6종 (모든 테이블 공통)
모니터링·데이터 변경 탐지용. 6개 컬럼 빠짐없이:
- `createdAt` — 최초 INSERT 일시
- `createdById` — 최초 INSERT actor의 비즈니스 ID (사용자/시스템/배치)
- `createdByGuid` — 최초 INSERT 호출 추적 GUID (request trace ID)
- `updatedAt` — 최종 UPDATE 일시
- `updatedById` — 최종 UPDATE actor ID
- `updatedByGuid` — 최종 UPDATE 호출 추적 GUID

> 표기는 spec 문서 기준 카멜케이스. DB 컬럼 케이스는 ADR-0012 (소문자 스네이크).

## Rationale

- 대형 은행 NFR (5,000 TPS·99.99% — ADR-0002)을 위한 샤딩/파티셔닝 대응을 PK 차원에서 보장
- 컬럼명 prefix → SQL 작성·로그 분석·운영 트러블슈팅 시 어떤 컨텍스트 컬럼인지 즉시 식별
- 계좌 분리 → 은행코드 단독 조회·인덱싱·외부 시스템(타행망) 매핑 효율
- 시스템 컬럼 6종 → 데이터 변경 추적·CDC 트레이싱·감사·디버깅 일관 지원

## Alternatives Considered

- **단일 PK (`transferId` 단독)** — 거부됨. 사용자별 조회·샤딩·파티셔닝 적용이 보조 인덱스로만 가능
- **시스템 컬럼 4종 (`createdAt`, `createdBy`, `updatedAt`, `updatedBy`)** — 거부됨. GUID 분리 추적이 운영·디버깅에 더 정확
- **컬럼명 단순화 (prefix X)** — 거부됨. 다중 도메인 조인 쿼리에서 컬럼 출처 모호

## Consequences

- (+) 모든 엔터티 정의가 일관됨 → 신규 합류자 학습 비용 ↓
- (+) 샤딩·파티셔닝 적용 시 PK가 이미 대응 형태
- (+) 시스템 컬럼 6종으로 CDC·감사 추적 정확
- (-) 모든 테이블에 6개 시스템 컬럼 추가 = 스토리지 오버헤드 (수용)
- (-) 사용자 비귀속 엔터티(정책 마스터 등)는 별도 PK 패턴 필요

## References

- ADR-0002 (NFR — 샤딩·파티셔닝 정당화)
- ADR-0009 (Transfer 엔터티 구조 — 첫 적용)
- ADR-0012 (DB 컬럼 케이스 — 표기 변환)
- 메모리: `project_data_model_conventions.md`
