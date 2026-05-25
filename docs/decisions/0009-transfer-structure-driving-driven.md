# ADR-0009: Transfer 엔터티 구조 — 드라이빙 + 드리븐 + 다건 메타

- Status: Accepted
- Date: 2026-05-25
- Deciders: kenny.k
- Tags: data-model, architecture

## Context

이체 종류 4종(당행 / 타행 / 예약 / 다건 자식)을 데이터 모델로 어떻게 표현할지 결정. 후보:
- 단일 Transfer + `kind` 필드 (단일 테이블 상속)
- 종류별 독립 테이블
- 공통 Transfer + 종류별 확장 테이블 (1:1 확장)

## Decision

**드라이빙 + 드리븐 + 다건 메타 분리 구조**:

| 테이블 | 역할 |
|---|---|
| **Transfer (드라이빙)** | 모든 이체의 공통 기본정보 (요청자·금액·통화·전체 상태·시각 등). `kind` 필드로 드리븐 라우팅 |
| **ImmediateTransferDetail (드리븐)** | 당행·타행 즉시이체 공통 특수정보 |
| **ScheduledTransferDetail (드리븐)** | 예약이체 전용 (예약일시·예치자금 식별자) |
| **BatchTransferRequest (다건 메타)** | 다건이체 1회 시도의 메타. 자식은 ImmediateTransferDetail에 그대로 들어가고 `parentBatchRequestId`로 연결 |

## Rationale

- 당행/타행은 본질적으로 동일한 즉시이체 흐름 (인증 → 사전검증 → 출금 → 입금/타행망)이라 하나의 드리븐으로 묶는 게 자연스러움
- 예약이체는 예약일시·예치자금 등 별개 속성이 명확히 분리되므로 독립 드리븐
- 다건이체는 본질적으로 "여러 즉시이체의 묶음"이므로 자식 거래는 ImmediateTransferDetail로 흘리고 묶음 메타(BatchTransferRequest)만 별도 — 진행률·청크·재실행 패턴이 메타에 집중됨
- 1:1 확장 테이블 패턴은 조회 시 조인 비용이 있지만 도메인 표현력·정합성이 우수

## Alternatives Considered

- **단일 테이블 상속 (`Transfer` 하나 + nullable 종류별 컬럼)** — 거부됨. 컬럼 nullable 폭증·정합성 약함
- **종류별 완전 분리 (4개 독립 테이블)** — 거부됨. 공통 조회·이력 통합 어려움·중복 컬럼 다수

## Consequences

- (+) 공통 조회 (사용자별 전체 이체 내역)는 Transfer 드라이빙 단독 조회로 효율적
- (+) 종류별 특수 처리는 드리븐 조인으로 명확
- (+) 다건이체 묶음 메타가 명시적 — 진행률·청크·재실행 시연 가치
- (-) 1:1 조인 오버헤드 (단, 사용자별 샤딩·일자별 파티션과 결합하면 일반적으로 수용 가능)
- (-) 다건 자식 거래가 ImmediateTransferDetail에 섞이므로 단순 통계에서 분리 조건 필요 (`parentBatchRequestId IS NOT NULL`)

## References

- ADR-0004 (이체 종류 v1 4종)
- ADR-0005 (다건 부분성공)
- ADR-0007 (다건 10,000건+ 규모)
- ADR-0011 (데이터 모델 컨벤션) — PK·시스템 컬럼 적용
- 작업 산출물 `.specify/work-in-progress/account-transfer-spec-draft.md` 카테고리 A
