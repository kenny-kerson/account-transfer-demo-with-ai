# ADR-0010: 멱등성·동시성 — 2-Phase + 다층 락 + 핑거프린트

- Status: Accepted
- Date: 2026-05-25 (Round 5 멱등성 결정 2026-05-23 + A-1.3 2-Phase 결정 2026-05-25 통합)
- Deciders: kenny.k
- Tags: data-model, concurrency, idempotency

## Context

클라이언트의 중복 요청(수ms 내 동일 이체)을 어떻게 방지할지 + 동일 출금계좌의 동시 이체 시 정합성을 어떻게 보장할지.

## Decision

### 멱등키 발급 정책
- **서버 생성 GUID** (클라이언트가 멱등키를 생성하지 않음)
- 클라이언트는 수ms 내 중복 요청 가능 가정 → 서버가 모든 방어 책임

### 멱등 단위 (하이브리드)
1. **2-Phase transferId 발급**:
   - 1-Phase (INITIATE): 클라이언트 → 서버에 이체 시작 요청 → 서버가 transferId 생성·반환
   - 2-Phase (EXECUTE): 동일 transferId로 이체 실행 요청. 재사용 시 캐시된 결과 반환
2. **출금계좌 단위 락**: 비관적 락 / 낙관적 락 / 분산락 (Redisson 등) / ShedLock / ReentrantLock — plan 단계에서 조합 결정
3. **요청 핑거프린트 관찰**: 출금계좌+수취계좌+금액+사용자+시간윈도우 해시. **논블로킹 경고만 발생** (의도된 연속 이체는 차단하지 않음)

### 보관·캐시
- IdempotencyKey 엔터티 (ADR-0009 이후 카테고리 D)
- 결과 캐시 TTL (예: 24h) — plan 단계에서 확정
- AccountLockState 엔터티로 락 상태 운영 모니터링

## Rationale

- 클라이언트 신뢰 X → 서버 단독 방어 = 일관된 정책 적용 가능
- 2-Phase: 결제 API의 일반 패턴. 명확한 거래 식별자 발급·재사용 시점 분리
- 다층 락: 단일 메커니즘으로 모든 경합 시나리오를 막을 수 없음 (단일 인스턴스 vs 분산, 짧은 락 vs 긴 트랜잭션)
- 핑거프린트 차단이 아닌 관찰: 의도된 연속 이체(예: 동일 금액 2회 송금)를 막으면 UX 손상

## Alternatives Considered

- **클라이언트 발급 Idempotency-Key 헤더 (Stripe 패턴)** — 거부됨. 모바일 클라이언트 신뢰 가정이 약함, 서버 방어가 더 일관됨
- **서버 측 해시 자동 중복 감지 (단독)** — 거부됨. 의도된 연속 이체 차단 위험
- **2-Phase 없이 단일 요청 + 핑거프린트** — 거부됨. transferId 명시 발급의 추적성 손실

## Consequences

- (+) 시니어 동시성 패턴 시연 풍부 (비관·낙관·분산락·ShedLock·ReentrantLock 트레이드오프 분석 가능)
- (+) 2-Phase = 결제 API 산업 표준과 일치
- (+) 핑거프린트 관찰 → 잠재 중복·이상 패턴 모니터링 자료 축적
- (-) 락 메커니즘 선택·튜닝 복잡도 (plan 단계 결정 필요)
- (-) 2-Phase = 클라이언트 호출 2회 + 1-Phase 정리 정책(미사용 transferId 만료) 필요
- (-) 핑거프린트 윈도우 길이 조정에 따른 false positive 위험

## References

- ADR-0002 (NFR — 5,000 TPS, 99.99%)
- ADR-0009 (Transfer 엔터티 구조 — IDEMPOTENCY_PHASE 컬럼 추가)
- ADR-0011 (데이터 모델 컨벤션 — 시스템 컬럼)
- Stripe Idempotency-Key 문서 (참고): https://docs.stripe.com/api/idempotent_requests
