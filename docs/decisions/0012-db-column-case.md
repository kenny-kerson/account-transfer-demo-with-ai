# ADR-0012: DB 컬럼명 케이스 — 소문자 스네이크

- Status: Accepted
- Date: 2026-05-25
- Deciders: kenny.k
- Tags: data-model, conventions

## Context

DB 컬럼 케이스를 결정하는 단계에서 회사 관례(대문자 스네이크 — Oracle/은행권 전통)와 MySQL/PostgreSQL 표준 관례(소문자 스네이크) 사이 트레이드오프를 검토.

검토 시 다음 사실을 확인:
- MySQL/PostgreSQL: 소문자 스네이크가 압도적 관례 (PostgreSQL은 unquoted identifier 소문자 강제, MySQL은 ORM 기본값 snake_case)
- Oracle/은행권: 대문자 스네이크 (unquoted identifier 대문자 변환 + COBOL 유산)

## Decision

**DB 컬럼명은 `소문자 스네이크 케이스` 사용** (`transfer_id`, `transfer_kind`, `from_bank_code` 등).

- 본 프로젝트는 MySQL 주력
- MySQL + JPA 생태계 관례에 일치
- Spec 문서 표기는 도메인 카멜케이스 (`transferId`), 컬럼명은 자동 변환 (`transfer_id`)
- 은행/COBOL 관습 의도적으로 따르지 않음

## Rationale

- MySQL/PostgreSQL 생태계의 도구·ORM·교재 모두 snake_case 전제 → 학습·운영 마찰 0
- JPA `SpringPhysicalNamingStrategyStandardImpl` (snake_case 변환) 기본값 사용 가능 → 보일러플레이트 0
- 회사 관례는 Oracle/은행권 전통이지 SQL 표준이 아님. 본 데모는 MySQL 기반이라 표준 관례 채택이 더 적합
- Spec 문서 표기는 도메인 용어 카멜케이스로 유지 (옵션 4·도메인-영속화 분리 참고)

## Alternatives Considered

- **대문자 스네이크 `TRANSFER_ID`** (회사 관례) — 거부됨. MySQL 표준이 아니고 학습 자료로서도 비표준
- **카멜케이스 `transferId` 컬럼** — 거부됨. MySQL 대소문자 식별자 처리 OS별 차이로 운영 리스크 (lower_case_table_names)
- **결정 보류, 구현 단계에서 결정** — 거부됨. 데이터 모델 컨벤션과 함께 정해야 일관성

## Consequences

- (+) MySQL + JPA 자연 흐름 → 매핑 보일러플레이트 0
- (+) 표준 관례 → 학습 자료 가치 ↑
- (-) 회사 관례에 익숙한 합류자는 초기 어색함 (단 표준 학습 기회)

## References

- ADR-0002 (NFR)
- ADR-0011 (데이터 모델 컨벤션)
- ADR-0013 (도메인-영속화 매핑 — 케이스 자동 변환의 근거)
- 사용자 답변 인용: "mysql과 jpa 생태계의 관례가 스네이크 케이스라서 그렇게 결정할께. 은행과 코볼의 관습을 따를 필요는 없어"
