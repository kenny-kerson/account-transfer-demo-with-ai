# 매핑 전략 — 톰 홈버그 4가지 전략

> 본 문서는 Tom Hombergs의 **"Get Your Hands Dirty on Clean Architecture"** (Packt, 2019, 국문: 만들면서 배우는 클린 아키텍처)에서 다룬 4가지 매핑 전략을 정리한 참고 지식이다. ADR-0013의 결정 근거.

## ⚠️ 출처·검증 상태 (2026-05-25 일부 검증 완료)

본 문서는 작성 시점에 AI 어시스턴트의 **학습 데이터(트레이닝 데이터)에 의존하여 재구성**되었고, 이후 일부 항목을 공식 자료로 검증했다.

### ✅ 공식 자료로 검증된 사실

| 항목 | 검증 출처 |
|---|---|
| 챕터 제목 = "Mapping Between Boundaries" | reflectoring.io/book/ TOC, buckpal README |
| 챕터 번호 = **1판 8장 / 2판 9장** | Packt URL(`ch08`)이 1판 (ISBN 9781839211966), reflectoring/buckpal README는 2판(ISBN 9781805128373) 9장 |
| 4가지 전략 명명 = "No Mapping" / "Two-Way" / "One-Way" / "Full" Mapping Strategy | WebSearch 결과 본문 인용 (Packt 공식 페이지 발췌) |
| Full Mapping 핵심 정의: "각 operation마다 별도 입력·출력 모델 도입. web layer가 입력을 application layer의 command 객체로 매핑" | WebSearch 결과 본문 인용 (Packt 공식 페이지 발췌) |
| Buckpal 예제 프로젝트의 실제 매핑 전략 = **② Two-Way Mapping** | `buckpal/src/.../adapter/out/persistence/AccountMapper.java` 직접 분석: `mapToDomainEntity()`(영속→도메인) + `mapToJpaEntity()`(도메인→영속) 두 메서드 존재 |

### ❌ 미검증 (학습 데이터 기반 재구성)

- 각 전략의 책 원문 정확한 표현·코드 예제·페이지 인용
- "공통 State 인터페이스" 같은 One-Way Mapping의 구체 구현 패턴 (책 본문 인용 미확보)
- 각 전략의 정확한 "적합/부적합" 가이드 표현

### 검증 권장 방법

1. 책 원본 1판 8장 / 2판 9장 "Mapping Between Boundaries" 직접 참조
2. 저자 GitHub 예제 코드 ([thombergs/buckpal](https://github.com/thombergs/buckpal)) — Buckpal은 ② Two-Way를 채택하므로 ① ③ ④는 코드로 직접 확인되지 않음
3. Packt 공식 페이지: `subscription.packtpub.com/book/programming/9781839211966/...`

### 본 프로젝트 결정에 미치는 영향

- 4가지 전략의 이름·핵심 구조·트레이드오프 골자는 검증됨 → **ADR-0013의 매핑 전략 혼합 결정은 유효**
- 본 프로젝트가 ③ One-Way를 선택한 것은 Buckpal 예제와 다른 전략을 의도적으로 경험하려는 사용자 결정 (ADR-0013 참고)
- 만약 책 원문에서 ③/④의 명명·구조가 본 KB와 다르게 발견되면 본 KB·ADR-0013을 갱신

## 배경

헥사고날 아키텍처에서 Web 계층, 도메인 계층, 영속화 계층 간 데이터를 어떻게 주고받을지(=객체를 공유할지, 매퍼로 변환할지)는 핵심 설계 결정이다. 톰 홈버그는 4가지 전략을 제시하고, **상황에 따라 혼합 사용**할 것을 권한다.

## 4가지 전략

### ① No Mapping Strategy (매핑 없음)

**구성**: Web/UseCase/Persistence 계층이 동일한 객체를 공유. JPA `@Entity`가 도메인이자 응답 모델.

```
Controller ──▶ JPA Entity ──▶ Repository
              (도메인 메서드도 같은 객체 안에)
```

- **장점**: 보일러플레이트 0. 빠른 개발. 단순 CRUD에 최적
- **단점**: JPA 어노테이션이 도메인 오염. 영속화 변경이 도메인 전체에 파급. 풍부한 도메인 행위 표현에 제약 (기본 생성자·세터 강제 등)
- **적합**: CRUD 위주 단순 시스템, 단순 조회 API, 프로토타입

### ② Two-Way Mapping Strategy (양방향 매핑)

**구성**: 각 계층마다 자체 모델 (Web Model, Domain Model, Persistence Entity). 양방향 매퍼 2개 (Web↔Domain, Domain↔Persistence).

```
Controller ──▶ WebRequest ─[매퍼]─▶ Domain ─[매퍼]─▶ Entity ──▶ Repository
                            ◀───            ◀───
```

- **장점**: 도메인이 100% 순수 (JPA·Jackson 어노테이션 침투 X). 각 계층 독립 진화
- **단점**: 매퍼 보일러플레이트 많음 (필드 30개면 매퍼도 라인 수 비례 증가). 같은 정보가 3벌
- **적합**: 핵심 도메인이 풍부한 행위·규칙을 보유하고 장기간 진화하는 컨텍스트

### ③ One-Way Mapping Strategy (단방향 매핑)

**구성**: 모든 계층의 모델이 공통 `State` 인터페이스 구현. 영속화 엔터티가 도메인 인터페이스를 구현하므로, 영속화에서 가져온 객체를 도메인 객체로 그대로 취급 가능. 도메인 → 영속화 방향에만 매핑 필요.

```
              ┌─ 도메인 인터페이스 (Transfer)
              │
   WebModel ──┤  implements
   Domain  ───┤
   Entity  ───┘
   
Controller ──▶ WebModel(도메인 인터페이스) ──▶ UseCase가 Domain으로 캐스팅·활용 ──▶ Entity 매핑(쓰기 시점만)
```

- **장점**: ②보다 보일러플레이트 절반. 도메인 객체 풍부함 유지
- **단점**: 공통 인터페이스 도입에 따른 추상화 학습 곡선. State 인터페이스가 비대해질 수 있음
- **적합**: 도메인이 풍부하지만 ②의 매퍼 비용이 부담스러운 컨텍스트, 도메인 행위가 안정화된 시스템

### ④ Full Mapping Strategy (완전 매핑)

**구성**: **각 유스케이스마다** 입력 Command 객체와 출력 Query Result 객체를 별도로 정의. UseCase는 Command를 받아 도메인 호출 → 결과를 Result로 변환하여 반환.

```
Controller ──▶ TransferCommand ──▶ UseCase ──▶ Domain ──▶ Entity
                                                  │
                                                  ▼
Controller ◀── TransferResult  ◀── UseCase ◀────┘
```

- **장점**: 각 유스케이스가 자체 모델로 격리. 입력 검증·권한·감사가 Command에 명확히 표현. 외부 API 변경이 도메인에 파급 X
- **단점**: 가장 많은 매퍼 클래스. 작은 필드 추가에도 여러 곳 수정 필요
- **적합**: 복잡한 유스케이스 (다건이체 실행, 백오피스 수동 조정 등), 입력·출력 규약이 도메인과 명확히 다른 컨텍스트, 명령/조회의 책임이 다양

## 책의 권고: 케이스별 혼합

> "한 가지 전략을 모든 곳에 강제할 필요는 없다. 컨텍스트마다 가장 적합한 전략을 선택하라."

| 컨텍스트 | 권장 전략 |
|---|---|
| 단순 CRUD, 단순 조회 | ① No Mapping |
| 핵심 도메인, 풍부한 행위 | ② Two-Way 또는 ③ One-Way |
| 복잡한 유스케이스, 다양한 입력 검증 | ④ Full Mapping |

## 본 프로젝트 적용 (ADR-0013)

- **이체 코어** (실행·상태머신·보상·멱등): ③ One-Way Mapping
- **단순 조회·백오피스 대시보드**: ① No Mapping
- **명령 유스케이스·복잡한 조회**: ④ Full Mapping

본 프로젝트는 외부 위임 시스템이 많아 **이체 코어 도메인이 비교적 작지만**, 그 안의 *상태머신·2-Phase 멱등·다층 락·보상 트랜잭션·다건 진행률* 등 복잡도는 높다. 따라서 코어는 ③, 외곽은 ① + ④의 혼합이 트레이드오프 균형점.

## 강제 분리 (조회 path에서 우발적 쓰기 방지)

옵션 3(부분 분리) + ③ One-Way의 약점은 "조회에서 같은 객체를 쓰므로 우발적 변경이 가능하다"는 점. 4중 방어:

| 층 | 방법 | 효과 |
|---|---|---|
| A | Repository 인터페이스 분리 (`TransferCommandRepository` / `TransferQueryRepository`) | 컴파일 타임 차단 |
| B | Read-only Projection 반환 (JPA Projection / 별도 ReadModel) | 엔터티 직접 노출 X |
| C | `@Transactional(readOnly = true)` 강제 | JPA flush 비활성화 |
| D | 엔터티 캡슐화 (public setter 제거, `Transfer.cancel()` 등 의미 메서드만) | 우발 변경 작성 자체가 어려움 |

본 프로젝트는 A+B+C+D **모두 적용** (ADR-0013).

## 참조

- Hombergs, Tom. *Get Your Hands Dirty on Clean Architecture*. Packt Publishing
  - 1판 (2019, ISBN 9781839211966) — 8장 "Mapping Between Boundaries"
  - 2판 (2023, ISBN 9781805128373) — 9장 "Mapping Between Boundaries"
- [thombergs/buckpal](https://github.com/thombergs/buckpal) — 책 예제 프로젝트 (실제 채택: ② Two-Way Mapping)
- [reflectoring.io 저자 페이지](https://reflectoring.io/authors/tom/)
- Cockburn, Alistair. *Hexagonal Architecture* (2005)
- Vernon, Vaughn. *Implementing Domain-Driven Design*. Addison-Wesley, 2013
