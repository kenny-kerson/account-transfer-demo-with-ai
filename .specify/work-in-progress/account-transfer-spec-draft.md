# 계좌이체 서비스 Spec 작업 — 진행 중

**상태**: 진행 중 (2026-05-25). Round 6 A-1 Transfer 속성 검토 마무리 단계 — 데이터 모델 컨벤션·DB 컬럼 케이스·도메인-영속화 매핑 전략 결정 완료. 다음: A-1.4 속성 표 카멜케이스 환원 → A-2 ImmediateTransferDetail 검토.

**ADR 백필 완료** (2026-05-25): 본 작업의 모든 의사결정은 `docs/decisions/` 에 ADR-0001~0013으로 기록. 향후 결정도 신규 ADR로 추가. 인덱스: [docs/decisions/README.md](../../docs/decisions/README.md). 매핑 전략 참고 자료: [docs/references/mapping-strategies.md](../../docs/references/mapping-strategies.md).
**원본 요청**: "계좌이체 서비스를 구현하려고 해. … 질의응답을 통해 하나씩 구체화 … 애매한 부분이 있으면 자의적으로 해석하지 말고 물어봐줘."

---

## 진행 위치
- Round 1~5 (질의응답): **완료**
- Round 6 (엔터티 후보 검토): **A-1 진행 중**
  - A-1.1 구조 결정: ✅ **드라이빙(Transfer) + 드리븐(ImmediateTransferDetail·ScheduledTransferDetail) + 다건메타(BatchTransferRequest)**
  - A-1.2 다건자식 표현: ✅ A-1.1 답변에 포함됨 — ImmediateTransferDetail에 그대로 + BatchTransferRequest로 묶음
  - A-1.3 멱등성 단위: ✅ **하이브리드 — 2-Phase transferId(INITIATED→EXECUTED) + 출금계좌 락 + 요청 핑거프린트 관찰(논블로킹 경고)**. 의도된 연속 이체는 차단하지 않음.
  - A-1.4 속성 세트 검토: ✅ 1차 확정 (29개 속성, PK 명시, 카멜케이스 표기, 시스템 컬럼 6종 포함). 추가 수정 의견 발생 시 갱신
- 다음: **A-2 ImmediateTransferDetail 검토 시작**
- `before_specify` hook (`speckit.git.feature`): **미실행**
- `specs/<NNN>-account-transfer/spec.md`: **미작성**

## 재개 방법
1. 본 파일을 읽어 결정 사항·엔터티 후보 확인
2. Round 6 엔터티 검토를 **A-1 Transfer**부터 하나씩 재개 (사용자 선호: 묶음 X, 하나씩 ✅)
3. 전체 엔터티 검토 완료 후 `before_specify` hook 실행 → spec 디렉토리 생성 → `spec.md` 작성 → requirements 체크리스트 생성·검증

---

## 데이터 모델 컨벤션 (모든 엔터티에 일관 적용)

A-1.4 Transfer 검토 중 사용자가 확정한 컨벤션. 본 spec 작업의 **모든 엔터티(A~H 전체)**에 동일하게 적용.

1. **PK 명시**: 각 테이블 정의 시 PK 컬럼을 명시적으로 표시
2. **PK 구성 패턴**: `(사용자ID, 일자, 식별자)` 형태의 복합 PK 사용
   - **사용자별 조회 path 대응** + **사용자별 샤딩 or 일자별 파티션 대응**
   - 사용자 비귀속 엔터티는 별도 PK 패턴 (예: 정책 마스터는 종류·인증수준 조합 등)
3. **컬럼명에 도메인 prefix 강제**: 컬럼명만 봐도 의미가 자립하도록
   - 예: `KIND` ❌ → `TRANSFER_KIND` ✅, `MEMO` ❌ → `TRANSFER_MEMO` ✅, `STATUS` ❌ → `TRANSFER_STATUS` ✅
4. **은행계좌는 은행코드 + 계좌번호로 분리**: `fromAccount` ❌ → `FROM_BANK_CODE` + `FROM_ACCOUNT_NUMBER` ✅
5. **시스템 컬럼 세트 (모든 테이블 공통)** — 모니터링·데이터 변경 탐지용 (Spec 문서는 카멜케이스, DB 컬럼은 snake_case로 변환):
   - `createdAt` (최초 INSERT 일시)
   - `createdById` (최초 INSERT actor의 비즈니스 ID — 사용자/시스템/배치)
   - `createdByGuid` (최초 INSERT 호출 추적 GUID — request trace ID)
   - `updatedAt` (최종 UPDATE 일시)
   - `updatedById` (최종 UPDATE actor ID)
   - `updatedByGuid` (최종 UPDATE 호출 추적 GUID)

### 케이스 컨벤션 (ADR-0012, ADR-0013)
- **Spec 문서·도메인 코드**: 카멜케이스 (`transferId`, `fromBankCode`)
- **DB 컬럼**: 소문자 스네이크 (`transfer_id`, `from_bank_code`) — MySQL+JPA `SpringPhysicalNamingStrategyStandardImpl` 자동 변환
- **도메인-영속화 계층 분리**: 옵션 3(부분 분리, CQRS 영향) + 톰 홈버그 매핑 전략 혼합 — 코어 ③ One-Way / 단순조회 ① No Mapping / 명령·복잡조회 ④ Full Mapping
- **강제 분리**: Repository 인터페이스 분리(A) + Read-only Projection(B) + `@Transactional(readOnly = true)`(C) + 엔터티 캡슐화(D) **모두 적용**

> ✅ Transfer 속성 표 카멜케이스 환원 완료 (2026-05-25).

---

## Round 1~5 확정 사항

### Round 1 — 서비스 성격·범위·사용자·인증
- **성격**: 은행/금융사 자체 인터넷·모바일 뱅킹 서비스. **데모지만 시니어 엔지니어 실무 수준 구현** (대용량 트래픽·동시성·트랜잭션·모니터링 포함)
- **이체 종류 v1**: 당행이체 / 타행이체 / 예약이체(당·타행) / 다건이체. **향후 확장**: QR이체, SMS이체
- **사용자**: 개인 고객(B2C) + 백오피스(기획자·상담직원)
- **인증**: PIN(간편) 기반, 향후 생체인증 확대. **인증 시스템은 별도 단위 시스템**. 이체 서비스는 인증·전자서명 요청/처리의 **클라이언트(위임 호출자)** 측을 구현

### Round 2 — 처리 메커니즘
- **타행이체 외부 금융망**: 전금망/오픈뱅킹을 모사하는 **가상 외부 시스템(Mock)을 별도 서비스로 구현** (실제 미들웨어처럼 호출, 타임아웃/재시도/대사조정 등 실무 이슈 재현)
- **다건이체 일부 실패**: **부분 성공 허용** (성공한 건만 완료, 실패 건은 별도 상태)
- **예약이체 자금 처리**: **예약 시점에 즉시 출금·예치** (확정성 우선)
- **백오피스 범위**: 이체 내역 검색·조회 대시보드 + 실시간 서비스 모니터링(메트릭/알림) + 이체 수동 취소·조정(상담용). **이상거래/한도 조정은 범위 외**

### Round 3 — 정책·NFR
- **이체 한도**: **이체 종류·인증수준별 차등 한도**
- **수수료**: 정책은 포함하되 **수수료 정책·수수료 출금은 외부 시스템에 위임** (가정)
- **계정계(원장 입출금)**: **외부 단위 시스템에 위임** (이체 서비스는 오케스트레이션 + 위임 클라이언트)
- **가용 시간**: **24/7 실시간 (오픈뱅킹 모델)**
- **NFR**: **대형 은행 수준 — 피크 5,000 TPS / 응답 P99 300ms / 99.99% 가용성**

### Round 4 — 운영 디테일
- **타행이체 완료 응답 시점**: **동기 — 수신은행 입금 확인 후 완료 응답** (타임아웃·재시도·SLA 설계 핵심)
- **수취인 사전검증(PreflightCheck)**: **필수** — 예금주명 + 계좌 상태(정상/지급정지/해지) + 잔액 수취가능 여부 등 선검증 일체화
- **알림**: **외부 알림 시스템에 위임** (이체 서비스는 클라이언트 구현만)
- **다건이체 상한**: **대규모 배치, 최대 약 10,000건 이상** → 비동기 처리·진행률 추적·부분 재실행 필수

### Round 5 — 트랜잭션 안전성·운영
- **멱등성**: **서버 생성 GUID** 방식(클라이언트 멱등키 X). 클라이언트는 수ms 내 중복 요청 가능 가정 → **서버에서 비관적락·낙관적락·Shedlock·분산락·ReentrantLock 등 다층 방어 검토**
- **취소 정책**: **완료 전 경과 상태(이체중 등)에서 사용자 취소 가능** (보상 트랜잭션·경합 처리 필요). 백오피스는 완료 후 수동 조정
- **보관 기간**: 국내 금융권 표준 — **거래내역 10년 / 감사로그 5년 / 운영로그 1년**
- **FDS(이상거래 탐지)**: **v1 제외, 향후 구현 대상으로 식별**

---

## 외부 위임 시스템 목록 (이체 서비스는 클라이언트 측만 구현)
1. **인증·전자서명 시스템** — PIN 검증, 전자서명 발급/검증
2. **계정계** — 출금·입금 원장 처리, 잔액 관리
3. **수수료 시스템** — 수수료 정책 평가, 수수료 출금
4. **타행 외부 금융망(전금망/오픈뱅킹) Mock** — 별도 서비스로 자체 구현
5. **알림 시스템** — SMS/Push/이메일 발송
6. **(향후) FDS 시스템** — 이상거래 심사

---

## Round 6 엔터티 후보 (검토 대기)

### A. 핵심 거래 (필수) — A-1 검토 결과 반영

#### A-1. Transfer (드라이빙 테이블) — 1차 확정안

> 표기 규칙: Spec 문서·도메인 코드는 **카멜케이스**, DB 컬럼은 **소문자 스네이크**로 자동 변환 (ADR-0012). 예: `transferId` ↔ `transfer_id`.

**PK** (도메인 표기): `(requesterUserId, transferDate, transferId)` → DB: `(requester_user_id, transfer_date, transfer_id)`

| # | 속성 (도메인) | 의미 | 비고 |
|---|---|---|---|
| 1 | `requesterUserId` | 요청자 (개인 고객 ID) | **PK1** — 사용자별 조회 path / 샤딩 |
| 2 | `transferDate` | 거래 발생 일자 (KST 기준, `requestedAt`의 날짜 부분) | **PK2** — 일자별 파티션 |
| 3 | `transferId` | 서버 생성 GUID, 외부 식별자·1-Phase 멱등키 겸용 | **PK3** |
| 4 | `transferKind` | `IMMEDIATE_INTRABANK` / `IMMEDIATE_INTERBANK` / `SCHEDULED` / `BATCH_CHILD` | 드리븐 라우팅 |
| 5 | `fromBankCode` | 출금 은행 코드 (자행) | |
| 6 | `fromAccountNumber` | 출금 계좌번호 | |
| 7 | `toBankCode` | 수취 은행 코드 | |
| 8 | `toAccountNumber` | 수취 계좌번호 | |
| 9 | `toPayeeNameInput` | 사용자가 입력한 수취 예금주명 (검증은 PayeeVerificationSnapshot) | |
| 10 | `transferAmount` | 이체 금액 | |
| 11 | `transferCurrency` | 통화 (v1 KRW, ISO 4217) | |
| 12 | `transferMemo` | 사용자 입력 적요 | |
| 13 | `transferStatus` | 전체 상태 (TransferStateHistory 최신과 동기화) | |
| 14 | `failureReasonCode` | 실패 사유 코드 | 성공 시 NULL |
| 15 | `failureReasonMessage` | 실패 사유 메시지 | 성공 시 NULL |
| 16 | `parentBatchRequestId` | `BATCH_CHILD`일 때 BatchTransferRequest 참조 | nullable |
| 17 | `authenticationRefId` | 인증·전자서명 토큰 참조 (외부 인증시스템) | |
| 18 | `payeeVerificationSnapshotId` | PayeeVerificationSnapshot 참조 | |
| 19 | `idempotencyPhase` | `INITIATED` / `EXECUTED` (2-Phase 추적) | A-1.3 반영 |
| 20 | `initiatedAt` | 1-Phase 시각 (transferId 발급) | |
| 21 | `requestedAt` | 2-Phase 시각 (이체 실행 요청) | |
| 22 | `completedAt` | 종료 시각 (성공/실패/취소) | nullable |
| 23 | `userContextSnapshotId` | UserContextSnapshot 참조 (G-19) | |
| 24 | `createdAt` | 시스템: 최초 INSERT 시각 | |
| 25 | `createdById` | 시스템: 최초 INSERT actor ID | |
| 26 | `createdByGuid` | 시스템: 최초 INSERT 호출 추적 GUID | |
| 27 | `updatedAt` | 시스템: 최종 UPDATE 시각 | |
| 28 | `updatedById` | 시스템: 최종 UPDATE actor ID | |
| 29 | `updatedByGuid` | 시스템: 최종 UPDATE 호출 추적 GUID | |

#### A 카테고리 잔여 엔터티 (A-2 이하, 동일 컨벤션으로 후속 검토)
- **A-2. ImmediateTransferDetail (드리븐)** — 당행/타행 즉시이체 공통 특수정보 (신규, A-1.1에서 도출)
- **A-3. ScheduledTransferDetail (드리븐)** — 예약이체 전용 (예약일시·예치자금 식별자) [구 `ScheduledTransfer`]
- **A-4. BatchTransferRequest (다건 시도 메타)** — 다건이체 1회 시도의 메타. 자식은 ImmediateTransferDetail에 + `PARENT_BATCH_REQUEST_ID`로 연결 [구 `TransferBatch`]
- **A-5. TransferStateHistory** — 상태 전이 트레일 (이전 #2)
- **A-6. PayeeVerificationSnapshot** — 수취인 사전검증 결과 스냅샷 (이전 #5)

### B. 한도 관리
6. **TransferLimitPolicy** — 이체 종류·인증수준별 한도 정의(시스템)
7. **UserTransferLimit** — 사용자 개인 한도 (시스템 한도 이하)
8. **TransferLimitUsage** — 일·월 누적 사용량

### C. 외부 시스템 위임 추적
9. **ExternalCallLog** — 인증/계정계/타행망/수수료/알림 호출 요청·응답·지연·재시도
10. **ExternalNetworkTransaction** — 타행망 전문 송수신·대사조정
11. **CompensationLog** — 출금 후 실패 시 보상 처리 이력

### D. 동시성·멱등성
12. **IdempotencyKey** — 서버 생성 GUID·요청 핑거프린트·결과 캐시·TTL
13. **AccountLockState** — 동시 출금 직렬화용 락 상태(운영 모니터링용)

### E. 백오피스·감사
14. **TransferAdjustment** — 백오피스 수동 취소·조정 기록
15. **BackofficeAuditLog** — 백오피스 사용자 활동 감사 (5년)

### F. 알림·이벤트
16. **NotificationDispatchLog** — 외부 알림 시스템 위임 호출 이력
17. **DomainEvent** — 도메인 이벤트 발행 트레일 (Outbox/CDC 패턴)

### G. 운영·디버깅
18. **DeadLetterMessage** — 비동기 처리 실패 메시지 보관·재처리
19. **UserContextSnapshot** — 이체 시점 회원·디바이스·세션·IP (감사·향후 FDS 대비)

### H. 향후 (v1 제외, 식별만)
- **FraudReviewCase** — FDS 심사 케이스
- **QRTransferRequest / SMSTransferRequest** — QR·SMS 이체 확장
