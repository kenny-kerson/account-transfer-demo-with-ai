# ADR-0013: 도메인-영속화 계층 분리 + 톰 홈버그 매핑 전략 혼합

- Status: Accepted
- Date: 2026-05-25
- Deciders: kenny.k
- Tags: architecture, mapping, cqrs

## Context

도메인 객체와 영속화 엔터티의 관계(같은 객체 vs 분리)를 결정. 검토 옵션:
1. 완전 분리 (Hexagonal)
2. 완전 통합 (JPA 엔터티 직접 노출)
3. **부분 분리 (CQRS 영향)** — 쓰기 코어만 분리, 단순 조회는 엔터티 직접
4. 통합 + 도메인 용어 통일 (케이스만 변환)

추가로 톰 홈버그(*Get Your Hands Dirty on Clean Architecture*, 2019)의 4가지 매핑 전략을 검토:
- ① No Mapping
- ② Two-Way Mapping
- ③ One-Way Mapping
- ④ Full Mapping

(상세는 [매핑 전략 KB](../references/mapping-strategies.md) 참조)

## Decision

### 계층 분리 정책: **옵션 3 (부분 분리, CQRS 영향)**
- 쓰기 코어와 단순 조회를 분리하되 모든 곳에 같은 전략을 강제하지 않음
- 컨텍스트별로 톰 홈버그 4가지 전략 혼합

### 매핑 전략 혼합

| 컨텍스트 | 전략 | 근거 |
|---|---|---|
| **이체 코어** (실행·상태머신·보상·멱등) | **③ One-Way Mapping** | Transfer/Compensation 등 풍부한 도메인 행위 필요. ②보다 보일러플레이트 적고 ①처럼 오염되지 않음 |
| **백오피스 단순 조회·대시보드** | **① No Mapping** + Projection | 단순 노출이면 충분. 매퍼 비용 회피 |
| **명령 유스케이스·복잡한 조회** (다건이체 실행, 한도 사용량 + 잔여한도 등) | **④ Full Mapping** (Command / Query Result) | 입출력 격리·검증·권한·감사 명확화 |

### 강제 분리 (조회 path에서 우발적 쓰기 방지) — A+B+C+D **모두 적용**

| 층 | 방법 |
|---|---|
| A | Repository 인터페이스 분리 (`TransferCommandRepository` vs `TransferQueryRepository`) |
| B | Read-only Projection 반환 (JPA Projection / 별도 ReadModel) |
| C | `@Transactional(readOnly = true)` 강제 |
| D | 엔터티 캡슐화 (public setter 제거, `Transfer.cancel()` 등 의미 메서드만) |

### Spec 문서 표기
- 모든 엔터티 속성은 도메인 카멜케이스로 표기 (`transferId`, `fromBankCode` 등)
- DB 컬럼은 ADR-0012에 따라 snake_case (`transfer_id`, `from_bank_code`) — JPA `SpringPhysicalNamingStrategyStandardImpl` 자동 변환

## Rationale

- 외부 위임 시스템이 많아 코어 도메인이 작지만 그 안의 *상태머신·2-Phase 멱등·다층 락·보상 트랜잭션·다건 진행률* 복잡도는 높음 → 코어는 ③, 외곽은 ①+④ 혼합이 비용/가치 균형점
- 사용자는 ③ One-Way 매핑을 한 번도 안 써본 상태에서 의도적으로 선택 — 실무 트레이드오프 직접 관찰 학습 가치
- 강제분리 A+B+C+D 모두 적용 = 코드 리뷰·런타임·트랜잭션 경계·도메인 행위 4중 방어 → 옵션 3의 약점 보완

## Alternatives Considered

- **보수적 술계 (코어=② Two-Way, 외곽=① No Mapping)** — 거부됨. 매퍼 비용 ↑, 학습 가치 (③ 미체험)
- **소극적 술계 (전체 ① No Mapping = 옵션 2 회귀)** — 거부됨. 옵션 3 선택과 충돌·대형 NFR 부적합
- **강제분리 중 일부만 적용 (예: A+C만)** — 거부됨. 데모 시연 가치를 위해 4중 방어 패턴을 모두 보여주는 게 의도적

## Consequences

- (+) 시니어 패턴 시연 풍부 (4가지 매핑 전략이 한 시스템에 공존하는 모습)
- (+) 케이스 자동 변환 (ADR-0012)으로 spec 표기 ↔ DB 컬럼 매핑 보일러플레이트 0
- (+) 4중 강제분리로 옵션 3 약점 보완
- (-) ③ One-Way의 공통 인터페이스 추상화 학습 곡선
- (-) 혼합 전략 = "어디서 어떤 전략을 쓰는지" 가이드라인 명문화 필요 (plan/구현 단계에서 작성)
- (-) Repository 분리·Projection·readOnly·도메인 캡슐화 4중 설정 = 초기 보일러플레이트 (단, ORM·코드 생성으로 일부 완화 가능)
- (?) 사용자가 ③ One-Way를 처음 적용 → 진행 중 실무 트레이드오프 발견 시 본 ADR 갱신 가능 (Status: Superseded by ...)

## References

- [매핑 전략 KB](../references/mapping-strategies.md) — 톰 홈버그 4가지 전략 상세 + 출처·검증 상태
- Hombergs, Tom. *Get Your Hands Dirty on Clean Architecture*. Packt
  - 1판 (2019) — 8장 "Mapping Between Boundaries"
  - 2판 (2023) — 9장 "Mapping Between Boundaries"
- [thombergs/buckpal](https://github.com/thombergs/buckpal) — 책 예제 (참고로 buckpal 자체는 ② Two-Way를 채택)
- ADR-0011 (데이터 모델 컨벤션)
- ADR-0012 (DB 컬럼 케이스 — snake_case 자동 변환)

## Validation Note

본 ADR의 매핑 전략 정보는 2026-05-25에 일부 검증됨 (KB 출처·검증 상태 섹션 참조). 4가지 전략의 명명·핵심 골자는 공식 자료에서 확인되었고, 세부 책 원문 표현은 향후 사용자가 책 직접 검증 시 본 ADR과 KB를 갱신할 수 있음. 결정 자체(혼합 전략 + 4중 강제분리)는 검증 결과에 영향받지 않음.
