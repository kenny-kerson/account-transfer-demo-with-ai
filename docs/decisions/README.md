# Architecture Decision Records (ADR)

이 디렉토리는 본 프로젝트의 모든 의사결정을 기록한다. 양식은 [MADR (Markdown Any Decision Records)](https://adr.github.io/madr/) 0.x 간이형을 따른다.

## 규칙

- **파일명**: `NNNN-kebab-case-title.md` (4자리 일련번호)
- **양식**: Status / Date / Deciders / Tags / Context / Decision / Rationale / Alternatives Considered / Consequences / References
- **번호**: 신규 ADR은 항상 다음 일련번호 사용. 결정이 번복되면 새 ADR을 작성하고 기존 ADR의 Status를 `Superseded by ADR-NNNN`으로 갱신
- **참고 지식**: 결정 근거가 된 외부 정보·이론·패턴 자료는 `docs/references/` 에 별도 보관하고 ADR의 References에서 링크

## 인덱스

| # | 파일 (영문) | 한글 제목 (한눈에 보기) | 일자 | Status | 카테고리 |
|---|---|---|---|---|---|
| [0001](0001-spec-workflow-and-adr-system.md) | spec-workflow-and-adr-system | **Spec 작업 진행 스타일 + 의사결정 기록(ADR) 체계** | 2026-05-23 | Accepted | Meta |
| [0002](0002-project-character-and-nfr.md) | project-character-and-nfr | **프로젝트 성격: 시니어 실무 수준 데모 + 비기능 목표(NFR)** | 2026-05-23 | Accepted | Architecture |
| [0003](0003-external-delegated-systems.md) | external-delegated-systems | **외부 단위 시스템 위임 구성 (인증·계정계·수수료·타행망·알림·FDS)** | 2026-05-23 | Accepted | Architecture |
| [0004](0004-transfer-scope-users-auth.md) | transfer-scope-users-auth | **이체 종류·사용자 범위·인증 수준 (Round 1)** | 2026-05-23 | Accepted | Scope |
| [0005](0005-processing-mechanisms.md) | processing-mechanisms | **이체 처리 메커니즘: 타행 Mock·다건 부분성공·예약 즉시예치·백오피스 범위 (Round 2)** | 2026-05-23 | Accepted | Behavior |
| [0006](0006-limit-fee-availability.md) | limit-fee-availability | **이체 한도·수수료·계정계 위임·24/7 가용시간 (Round 3)** | 2026-05-23 | Accepted | Policy |
| [0007](0007-completion-payee-notify-batch.md) | completion-payee-notify-batch | **완료응답 동기 모델·수취인 사전검증 필수·알림 위임·다건 1만건+ 상한 (Round 4)** | 2026-05-23 | Accepted | Behavior |
| [0008](0008-cancellation-retention-fds.md) | cancellation-retention-fds | **사용자 취소 정책·금융권 표준 보관기간·FDS v1 제외 (Round 5 운영 부분)** | 2026-05-23 | Accepted | Policy |
| [0009](0009-transfer-structure-driving-driven.md) | transfer-structure-driving-driven | **Transfer 엔터티 구조: 드라이빙 + 드리븐 + 다건 시도 메타** | 2026-05-25 | Accepted | Data Model |
| [0010](0010-idempotency-concurrency.md) | idempotency-concurrency | **멱등성·동시성 제어: 서버 GUID 2-Phase + 다층 락 + 핑거프린트 관찰** | 2026-05-25 | Accepted | Data Model |
| [0011](0011-data-model-conventions.md) | data-model-conventions | **데이터 모델 컨벤션: PK 명시·복합PK(사용자ID/일자/식별자)·도메인 prefix·은행계좌 분리·시스템 컬럼 6종** | 2026-05-25 | Accepted | Data Model |
| [0012](0012-db-column-case.md) | db-column-case | **DB 컬럼명 케이스: 소문자 스네이크 (MySQL+JPA 관례)** | 2026-05-25 | Accepted | Data Model |
| [0013](0013-domain-persistence-mapping.md) | domain-persistence-mapping | **도메인-영속화 계층 분리: 옵션 3 부분분리 + 톰 홈버그 매핑전략 혼합(③ One-Way / ① No / ④ Full) + 4중 강제분리** | 2026-05-25 | Accepted | Architecture |

## 참고 지식 (Knowledge Base)

- [매핑 전략 — 톰 홈버그 4가지 전략](../references/mapping-strategies.md) — ADR-0013 근거
