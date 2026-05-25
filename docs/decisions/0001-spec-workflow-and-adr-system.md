# ADR-0001: Spec 작업 진행 스타일 + ADR 체계

- Status: Accepted
- Date: 2026-05-23 (워크플로 스타일), 2026-05-25 (ADR 체계 형식 확정)
- Deciders: kenny.k
- Tags: meta, process

## Context

`/speckit-specify` 워크플로우에서 사용자는 다음을 명시적으로 요구했다:
- "질의응답을 통해 하나씩 구체화"
- "애매한 부분이 있으면 자의적으로 해석하지 말고 물어봐줘"
- "하나씩 검토하자" (엔터티 검토 시)
- "과거부터 지금, 그리고 앞으로 결정되는 모든 의사결정기록들은 문서로 남겨줘"

기본 `/speckit-specify`는 AI가 합리적 기본값으로 추정하고 최대 3개의 NEEDS CLARIFICATION만 마크하는 동작이지만, 본 사용자는 모든 미정 사항을 질의응답으로 명확히 한 후 spec을 작성하길 원한다.

## Decision

### 진행 스타일
1. **AI 자의 해석 금지** — 모든 미정 사항은 질의로 전환. 기본 자동 추정 비활성화
2. **검토는 1건씩** — 다수 검토 대상(엔터티·결정)은 묶음 X, "어디부터 할까요?" 먼저 묻고 1건씩 진행
3. **AskUserQuestion 묶음 허용 범위** — "선택지 기반의 짧은 정책 결정" 라운드 한정. 본격적 설계 검토는 1건씩
4. **결정 누적 보존** — 작업 산출물(`.specify/work-in-progress/`)에 진행 상태 + memory에 메타 컨벤션 동시 보존

### ADR 체계
- 위치: `docs/decisions/NNNN-kebab-case-title.md`
- 양식: MADR 0.x 간이형 (Status / Date / Deciders / Tags / Context / Decision / Rationale / Alternatives Considered / Consequences / References)
- 인덱스: `docs/decisions/README.md` 표 자동 유지
- 결정 근거가 된 외부 지식(이론·패턴 등)은 `docs/references/` 에 별도 KB로 보관하고 ADR References에서 링크

## Rationale

- 시니어 엔지니어 수준의 정밀한 결정을 사용자가 직접 내리길 원함 → AI는 분석·옵션 제시 역할로 한정
- 묶음 검토는 의사결정 품질 저하 (한 옵션이 다른 선택을 강요), 1건씩이 의도된 비용 ↔ 명확성 트레이드오프
- MADR은 공개 표준이라 외부 도구·신규 참여자 학습 비용 0

## Alternatives Considered

- **표준 `/speckit-specify` 동작 유지** — 거부됨. 사용자 요구와 불일치
- **묶음 검토** — 거부됨. 검토 품질·인지 부하 문제
- **단일 `docs/decisions.md` 파일 누적** — 거부됨. 검색·참조·갱신·번복 추적 약함
- **Spec Kit 내부(`.specify/decisions/`) 보관** — 거부됨. ADR은 시스템 전반 결정이라 spec 디렉토리에 종속시키지 않음

## Consequences

- (+) 모든 결정에 사용자가 직접 책임. 자의적 결정으로 인한 재작업 0
- (+) 미래 합류자가 결정 히스토리를 단일 위치에서 파악 가능
- (-) 결정 1개당 라운드 트립 비용. 진행 속도 감소 (대신 품질 ↑)
- (-) ADR 백필 비용 (Round 1~6 결정 13건 소급 작성 필요)

## References

- [MADR (Markdown Any Decision Records)](https://adr.github.io/madr/)
- [Michael Nygard, "Documenting Architecture Decisions"](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
- 작업 산출물: `.specify/work-in-progress/account-transfer-spec-draft.md`
- 사용자 작업 스타일 memory: `~/.claude/projects/.../memory/feedback_spec_qa_style.md`
