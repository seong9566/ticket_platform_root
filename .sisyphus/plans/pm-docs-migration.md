# PM 문서 이동 + v3.0 최신화

## TL;DR

> **Quick Summary**: 루트에 흩어진 3개 PM 문서(IMPLEMENTATION_STATUS.md, ROADMAP.md, PM_README.md)를 `docs/pm/status/`로 이동하고, 2026-02-23 기준 실제 코드베이스 상태에 맞춰 v3.0으로 최신화. 6개 미반영 완성 기능 추가, D+3→D+1 수정, Phase 진행률 갱신, 상호 참조 경로 업데이트.
> 
> **Deliverables**:
> - `docs/pm/status/` 디렉토리 신설 + 3개 파일 이동 (git mv)
> - `docs/pm/status/IMPLEMENTATION_STATUS.md` 최신화 (v3.0)
> - `docs/pm/status/ROADMAP.md` 최신화 (v3.0)
> - `docs/pm/status/PM_README.md` 최신화 (v3.0)
> - 외부 참조 경로 수정 (4개 파일)
> - `docs/pm/README.md` 업데이트 (status/ 폴더 설명 추가)
> - 3개 문서 간 교차 검증 완료
> 
> **Estimated Effort**: Short (1-2시간)
> **Parallel Execution**: YES - 4 waves
> **Critical Path**: Task 0 → Task 1~3 (병렬) → Task 4 → Task 5 → F1~F3

---

## Context

### Original Request
사용자가 PM 문서를 `docs/pm/` 하위로 정리하고, 동시에 v3.0 최신화를 요청. 모든 설명은 한글로 작성.

### Interview Summary
**Key Discussions**:
- 이동 경로: 루트 → `docs/pm/status/` (하위 폴더 신설)
- 이동 + 최신화 동시 진행 (권장 옵션 선택)
- WORK_MANAGEMENT.md는 루트에 유지 (이동 안 함)
- 정산 정책: D+1 유지 (코드가 정확, 문서의 D+3 수정)
- Apple 소셜 로그인: Phase 4 (출시 전 앱스토어 심사용)
- 관리자 대시보드 기술: TBD
- 문서 버전: 모두 v3.0 통일

**Research Findings (이전 세션에서 분석 완료)**:
- 서버: 12 컨트롤러, 40+ 서비스, 66 DB 엔티티, 1 SignalR Hub
- 모바일: 13 피처 모듈
- 6개 완성 기능이 문서에 미반영: 신고/분쟁, 알림/FCM, 소셜 로그인, BackgroundServices, 비밀번호 변경, AES-256-GCM
- Phase 진행률: Phase 1 100%, Phase 2 30%, Phase 3 35%, Phase 4 5%, 전체 65%

**경로 참조 맵 (grep 분석 완료)**:

| 참조하는 파일 | 참조 대상 | 라인 | 타입 |
|--------------|----------|------|------|
| `IMPLEMENTATION_STATUS.md` | `./PM_README.md`, `./ROADMAP.md`, `./WORK_MANAGEMENT.md` | 554-556 | 마크다운 링크 |
| `IMPLEMENTATION_STATUS.md` | `./TicketPlatFormServer/*`, `./ticket_platform_mobile/*` | 559-568 | 마크다운 링크 |
| `PM_README.md` | `./ROADMAP.md` | 223 | 마크다운 링크 |
| `PM_README.md` | `./TicketPlatFormServer/*`, `./ticket_platform_mobile/*` | 343-346 | 마크다운 링크 |
| `ROADMAP.md` | `./구매_판매_내역_개발_계획서.md`, `./ticket_platform_mobile/*` | 87, 106-107 | 마크다운 링크 |
| `docs/README.md` | `../PM_README.md`, `../ROADMAP.md` | 205-206 | 마크다운 링크 |
| `docs/shared/README.md` | `../../PM_README.md`, `../../ROADMAP.md`, `../../IMPLEMENTATION_STATUS.md` | 65-68 | 마크다운 링크 |
| `docs/pm/handoff/README.md` | PM 문서명 | 132-134 | 텍스트 참조 |
| `docs/pm/handoff/2026-02-10_MVP_작업전달_요약.md` | PM 문서명 | 272-273 | 텍스트 참조 |
| `SETUP_COMPLETE.md` | PM 문서명 | 11-12, 58, 64-65, 107, 111, 185 | 텍스트 참조 |

### Metis Review
**자체 갭 분석 결과 (addressed)**:
- 문서 내부 상호 참조 경로가 이동 후 깨짐 → Task별 경로 수정 포함
- 외부 문서(docs/README.md 등)의 참조도 깨짐 → Task 4에서 일괄 수정
- docs/pm/README.md에 status/ 폴더 설명 누락 → Task 4에서 추가
- 기존 `.sisyphus/plans/roadmap-revision.md` 정리 필요 → Task 0에서 삭제
- SETUP_COMPLETE.md, handoff 문서의 PM 문서 참조는 텍스트 멘션(링크 아님) → 경로 변경 불필요, 현행 유지

---

## Work Objectives

### Core Objective
3개 PM 문서를 `docs/pm/status/`로 이동하고 v3.0으로 최신화하여, 프로젝트 현황을 정확히 반영하는 단일 진실 공급원(Single Source of Truth)으로 만든다.

### Concrete Deliverables
- `docs/pm/status/IMPLEMENTATION_STATUS.md` (v3.0)
- `docs/pm/status/ROADMAP.md` (v3.0)
- `docs/pm/status/PM_README.md` (v3.0)
- `docs/pm/README.md` (status/ 폴더 설명 추가)
- `docs/README.md` (참조 경로 수정)
- `docs/shared/README.md` (참조 경로 수정)

### Definition of Done
- [ ] 루트에 3개 PM 문서 없음 (`ls *.md` 시 PM 문서 미포함)
- [ ] `docs/pm/status/`에 3개 파일 존재
- [ ] 3개 문서 모두 `2026-02-23` 날짜, `v3.0` 버전
- [ ] D+3 참조 전부 D+1로 변경 (`grep "D+3"` → 0건)
- [ ] 6개 신규 완성 기능이 모든 문서에 반영
- [ ] Phase별 진행률 정확 반영 (Phase 1: 100%, Phase 2: 30%, Phase 3: 35%, Phase 4: 5%)
- [ ] 모든 마크다운 링크 정상 동작 (깨진 링크 0건)
- [ ] 3개 문서 간 교차 모순 없음

### Must Have
- 모든 새 콘텐츠는 한국어로 작성 (기술 용어만 영문)
- 기존 문서의 이모지, 테이블, 체크박스 스타일 유지
- D+3 → D+1 전체 변경
- 컨트롤러 수: 10 → 12, 피처 모듈 수: 11 → 13
- Apple 로그인은 Phase 4에 기재
- git mv 사용 (git 히스토리 보존)

### Must NOT Have (Guardrails)
- **기존 문서 구조 변경 금지** — 제목 계층, 섹션 순서를 변경하지 않음
- **이미 정확한 섹션 수정 금지** — 변경이 필요한 부분만 수정
- **미래 기능 추가 금지** — 인터뷰에서 논의된 것만 반영
- **날짜 임의 설정 금지** — 미래 마일스톤은 TBD
- **영문 산문 작성 금지** — 기술 용어만 영문
- **리스크 섹션 변경 금지** — ROADMAP의 "리스크 및 대응 방안" 수정 금지
- **scope 외 문서 수정 금지** — WORK_MANAGEMENT.md, AGENTS.md, SETUP_COMPLETE.md 등 수정 금지
- **SETUP_COMPLETE.md, handoff 문서 수정 금지** — 텍스트 멘션만 있으므로 경로 변경 불필요

---

## Verification Strategy

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed. No exceptions.

### Test Decision
- **Infrastructure exists**: NO
- **Automated tests**: None (문서 작업)
- **Framework**: N/A

### QA Policy
모든 Task는 `grep` + `ls` 기반 검증으로 문서 내용과 파일 위치 정확성을 확인.
Evidence: `.sisyphus/evidence/task-{N}-{scenario-slug}.txt`

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Start Immediately — 파일 이동):
└── Task 0: docs/pm/status/ 생성 + git mv + 기존 계획 정리 [quick]

Wave 2 (After Wave 1 — 3개 문서 독립 최신화, MAX PARALLEL):
├── Task 1: IMPLEMENTATION_STATUS.md v3.0 최신화 + 내부 경로 수정 [unspecified-high]
├── Task 2: ROADMAP.md v3.0 최신화 + 내부 경로 수정 [unspecified-high]
└── Task 3: PM_README.md v3.0 최신화 + 내부 경로 수정 [unspecified-high]

Wave 3 (After Wave 2 — 외부 참조 수정):
└── Task 4: 외부 참조 경로 수정 + docs/pm/README.md 업데이트 [quick]

Wave 4 (After Wave 3 — 교차 검증):
└── Task 5: 3개 문서 교차 검증 + 링크 검증 [deep]

Wave FINAL (After ALL tasks — 독립 검증, 3 parallel):
├── Task F1: Plan compliance audit (oracle)
├── Task F2: Scope fidelity check (deep)
└── Task F3: 문서 grep 기반 자동 검증 (unspecified-high)

Critical Path: Task 0 → Task 1~3 → Task 4 → Task 5 → F1~F3
Parallel Speedup: ~50% faster than sequential
Max Concurrent: 3 (Wave 2)
```

### Dependency Matrix

| Task | Depends On | Blocks | Wave |
|------|-----------|--------|------|
| 0 | — | 1, 2, 3 | 1 |
| 1 | 0 | 4, 5 | 2 |
| 2 | 0 | 4, 5 | 2 |
| 3 | 0 | 4, 5 | 2 |
| 4 | 1, 2, 3 | 5 | 3 |
| 5 | 4 | F1-F3 | 4 |

### Agent Dispatch Summary

- **Wave 1**: **1** — T0 → `quick`
- **Wave 2**: **3** — T1 → `unspecified-high`, T2 → `unspecified-high`, T3 → `unspecified-high`
- **Wave 3**: **1** — T4 → `quick`
- **Wave 4**: **1** — T5 → `deep`
- **FINAL**: **3** — F1 → `oracle`, F2 → `deep`, F3 → `unspecified-high`

---

## TODOs

> Implementation + verification = ONE Task.
> EVERY task MUST have: Recommended Agent Profile + Parallelization info + QA Scenarios.


- [ ] 0. docs/pm/status/ 생성 + git mv + 기존 계획 정리

  **What to do**:
  - `docs/pm/status/` 디렉토리 생성 (`mkdir -p docs/pm/status/`)
  - `git mv IMPLEMENTATION_STATUS.md docs/pm/status/IMPLEMENTATION_STATUS.md`
  - `git mv ROADMAP.md docs/pm/status/ROADMAP.md`
  - `git mv PM_README.md docs/pm/status/PM_README.md`
  - 기존 미실행 계획 파일 삭제: `rm .sisyphus/plans/roadmap-revision.md`
  - 기존 드래프트 파일 삭제: `rm .sisyphus/drafts/roadmap-revision.md`

  **Must NOT do**:
  - WORK_MANAGEMENT.md 이동 금지
  - 문서 내용 수정 금지 (이동만)
  - git commit은 포함하되, push는 하지 않음

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 단순 파일 이동 + 삭제 작업, 로직 복잡도 없음
  - **Skills**: [`git-master`]
    - `git-master`: git mv로 히스토리 보존 이동에 필요

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 1 (Sequential — 첫 번째)
  - **Blocks**: [Task 1, Task 2, Task 3]
  - **Blocked By**: None (can start immediately)

  **References**:
  - 루트 디렉토리: `IMPLEMENTATION_STATUS.md` (574줄), `ROADMAP.md` (684줄), `PM_README.md` (382줄)
  - 기존 계획: `.sisyphus/plans/roadmap-revision.md` (삭제 대상)
  - 기존 드래프트: `.sisyphus/drafts/roadmap-revision.md` (삭제 대상)

  **Acceptance Criteria**:
  - [ ] `ls docs/pm/status/IMPLEMENTATION_STATUS.md` → 파일 존재
  - [ ] `ls docs/pm/status/ROADMAP.md` → 파일 존재
  - [ ] `ls docs/pm/status/PM_README.md` → 파일 존재
  - [ ] `ls IMPLEMENTATION_STATUS.md 2>/dev/null` → 파일 없음
  - [ ] `ls ROADMAP.md 2>/dev/null` → 파일 없음
  - [ ] `ls PM_README.md 2>/dev/null` → 파일 없음
  - [ ] `ls .sisyphus/plans/roadmap-revision.md 2>/dev/null` → 파일 없음

  **QA Scenarios**:
  ```
  Scenario: 파일 이동 성공 확인
    Tool: Bash (ls)
    Steps:
      1. ls docs/pm/status/IMPLEMENTATION_STATUS.md docs/pm/status/ROADMAP.md docs/pm/status/PM_README.md
      2. 3개 파일 모두 존재하는지 확인
      3. ls IMPLEMENTATION_STATUS.md ROADMAP.md PM_README.md 2>/dev/null
      4. 루트에 파일이 없는지 확인
    Expected Result: docs/pm/status/에 3개 파일 존재, 루트에 0개
    Evidence: .sisyphus/evidence/task-0-file-move.txt

  Scenario: git 히스토리 보존 확인
    Tool: Bash (git log)
    Steps:
      1. git log --follow --oneline -5 docs/pm/status/IMPLEMENTATION_STATUS.md
      2. 이전 커밋 히스토리가 보존되는지 확인
    Expected Result: 이전 커밋 이력이 표시됨
    Evidence: .sisyphus/evidence/task-0-git-history.txt
  ```

  **Commit**: YES
  - Message: `docs(pm): move PM docs to docs/pm/status/ and cleanup old plan`
  - Files: `docs/pm/status/IMPLEMENTATION_STATUS.md`, `docs/pm/status/ROADMAP.md`, `docs/pm/status/PM_README.md` (이동), `.sisyphus/plans/roadmap-revision.md` (삭제), `.sisyphus/drafts/roadmap-revision.md` (삭제)

- [ ] 1. IMPLEMENTATION_STATUS.md v3.0 최신화 + 내부 경로 수정

  **What to do**:
  - `최종 업데이트` 날짜를 `2026-02-23`으로, `문서 버전`을 `3.0`으로 변경
  - `진행률`을 `~85% (MVP 거의 완료)`에서 `~65% (Phase 1 완료, Phase 2-3 진행 중)`으로 변경
  - `전체 구현 현황` 테이블에 아래 항목 추가:
    - `신고/분쟁 시스템 | ✅ | ✅ | 완료 | DisputeController 5개 엔드포인트`
    - `알림 시스템(FCM) | ✅ | ✅ | 완료 | NotificationController 6개 엔드포인트`
    - `소셜 로그인 | ✅ | ✅ | 완료 | Google, Kakao (Apple은 Phase 4)`
    - `비밀번호 변경 | ✅ | ✅ | 완료 | PUT /api/users/password`
  - `프로필 업데이트` 행: 모바일 `🚧` → `✅`, 상태 `진행중` → `완료`
  - `상세 구현 현황` > `백엔드 API` 섹션에 신규 컨트롤러 2개 추가:
    - `1.11 NotificationController` — POST /api/notifications/token, DELETE /api/notifications/token, GET /api/notifications, PUT /api/notifications/{id}/read, PUT /api/notifications/read-all, GET /api/notifications/unread-count
    - `1.12 DisputeController` — POST /api/disputes, GET /api/disputes, GET /api/disputes/{id}, POST /api/disputes/{id}/evidence, PUT /api/disputes/{id}/cancel
  - `모바일 앱` 섹션에 2개 피처 추가:
    - `2.10 Notification (알림)` — notification_list_view, FCM 토큰 관리, 뱃지
    - `2.11 Dispute (신고)` — create_dispute_view, dispute_list_view, dispute_detail_view, 증거 업로드
  - `인증 시스템`(1.1)에 소셜 로그인(Google, Kakao) + 비밀번호 변경 추가
  - `프로젝트 구조 요약` 컨트롤러 `10개` → `12개`, 피처 `11개` → `13개`
  - 프로젝트 구조 트리에 `NotificationController`, `DisputeController`, `notification/`, `dispute/` 추가
  - `진행률 요약` 섹션: Phase 1 85%→100%, Phase 2 0%→30%, Phase 3 15%→35%, Phase 4 0%→5%, 전체 50%→65%
  - `진행 중인 작업` 섹션: 프로필 이미지 업로드 제거(완료), Phase 2 진행 중 언급
  - `에스크로 정산 시스템`(3.1) D+3 → D+1
  - `알림 시스템`(3.3)을 ✅ 완료로 변경 (Phase 2에서도 신고/분쟁 완료 표시)
  - `보안 강화`(4.2)에 AES-256-GCM 부분 완료 언급
  - `최근 완료된 작업` 섹션에 2026-02-23 기준 항목 추가
  - `다음 단계` 섹션을 현재 우선순위로 업데이트
  - **내부 참조 경로 수정** (이동 후 새 상대 경로):
    - `[PM_README.md](./PM_README.md)` → `[PM_README.md](./PM_README.md)` (같은 폴더 — 유지)
    - `[ROADMAP.md](./ROADMAP.md)` → `[ROADMAP.md](./ROADMAP.md)` (같은 폴더 — 유지)
    - `[WORK_MANAGEMENT.md](./WORK_MANAGEMENT.md)` → `[WORK_MANAGEMENT.md](../../../WORK_MANAGEMENT.md)` (루트로 올라가야 함)
    - `./TicketPlatFormServer/*` → `../../../TicketPlatFormServer/*`
    - `./ticket_platform_mobile/*` → `../../../ticket_platform_mobile/*`
    - `./구매_판매_내역_개발_계획서.md` → `../../../구매_판매_내역_개발_계획서.md`

  **Must NOT do**:
  - 문서 구조(제목 계층, 섹션 순서) 변경 금지
  - 이미 정확한 섹션(채팅, 결제, 찜 등) 내용 수정 금지
  - 영문 산문 작성 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: 대규모 문서(574줄) 수정, 다수 섹션 동시 변경 + 경로 수정
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 2, 3)
  - **Blocks**: [Task 4, Task 5]
  - **Blocked By**: [Task 0]

  **References**:

  **Pattern References**:
  - `docs/pm/status/IMPLEMENTATION_STATUS.md` — 전체 문서 (574줄). 기존 이모지, 테이블, 체크박스 스타일 따를 것

  **API/Type References** (신규 컨트롤러 엔드포인트 확인용):
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/NotificationController.cs` — 6개 엔드포인트. route, method, 설명 확인
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/DisputeController.cs` — 5개 엔드포인트. route, method, 설명 확인
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/AuthController.cs:71-90` — 소셜 로그인 엔드포인트
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/UserController.cs:125-140` — 비밀번호 변경 엔드포인트
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Notification/FcmService.cs` — 227줄, FCM 구현 상세
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Dispute/DisputeService.cs` — 419줄, 신고 로직 상세
  - `ticket_platform_mobile/lib/features/notification/` — 41 dart 파일, 모바일 알림 구현
  - `ticket_platform_mobile/lib/features/dispute/` — 47 dart 파일, 모바일 신고 구현

  **Acceptance Criteria**:
  - [ ] `grep "2026-02-23" docs/pm/status/IMPLEMENTATION_STATUS.md` → 날짜 확인
  - [ ] `grep "3.0" docs/pm/status/IMPLEMENTATION_STATUS.md` → 버전 확인
  - [ ] `grep "D+3" docs/pm/status/IMPLEMENTATION_STATUS.md` → 0건
  - [ ] `grep -c "분쟁" docs/pm/status/IMPLEMENTATION_STATUS.md` → 1건 이상
  - [ ] `grep -c "FCM\|NotificationController" docs/pm/status/IMPLEMENTATION_STATUS.md` → 1건 이상
  - [ ] `grep "12개 컨트롤러" docs/pm/status/IMPLEMENTATION_STATUS.md` → match
  - [ ] `grep "13개 피처" docs/pm/status/IMPLEMENTATION_STATUS.md` → match
  - [ ] `grep "100%" docs/pm/status/IMPLEMENTATION_STATUS.md` → Phase 1 100% 확인

  **QA Scenarios**:
  ```
  Scenario: D+3 완전 제거 확인
    Tool: Bash (grep)
    Steps:
      1. grep "D+3" docs/pm/status/IMPLEMENTATION_STATUS.md
      2. 결과가 0건인지 확인
    Expected Result: 0 matches
    Evidence: .sisyphus/evidence/task-1-d3-removal.txt

  Scenario: 신규 기능 반영 확인
    Tool: Bash (grep)
    Steps:
      1. grep -c "분쟁\|FCM\|소셜.*로그인\|비밀번호.*변경" docs/pm/status/IMPLEMENTATION_STATUS.md
      2. 결과가 4 이상인지 확인
    Expected Result: count ≥ 4
    Evidence: .sisyphus/evidence/task-1-new-features.txt

  Scenario: 내부 참조 경로 유효성 확인
    Tool: Bash (grep)
    Steps:
      1. grep -oP '\]\(\K[^)]+' docs/pm/status/IMPLEMENTATION_STATUS.md | head -20
      2. 상대 경로가 유효한지 확인 (../../../ 패턴)
    Expected Result: 모든 링크가 유효한 상대 경로
    Evidence: .sisyphus/evidence/task-1-link-check.txt
  ```

  **Commit**: YES
  - Message: `docs(pm): update IMPLEMENTATION_STATUS.md to v3.0 — sync with codebase 2026-02-23`
  - Files: `docs/pm/status/IMPLEMENTATION_STATUS.md`
  - Pre-commit: `grep "D+3" docs/pm/status/IMPLEMENTATION_STATUS.md` (should be empty)


- [ ] 2. ROADMAP.md v3.0 최신화 + 내부 경로 수정

  **What to do**:
  - `문서 버전`을 `3.0`으로, `최종 수정일`을 `2026-02-23`으로 변경
  - Phase 1 섹션:
    - `1.4 거래 내역 시스템` 모바일 체크박스 `[ ]` → `[x]` (이미 완료됨, IMPLEMENTATION_STATUS와 모순 해결)
    - `1.5 프로필 업데이트` 체크박스 모두 `[x]`로, 상태를 `✅ 완료`로 변경
    - `Phase 1 완료 기준` 체크리스트에서 '남은 작업' 항목 제거 (완료됨)
    - `1.1 인증 시스템`에 소셜 로그인(Google, Kakao) 항목 추가 `[x]`
  - Phase 2 섹션:
    - `2.3 신고/분쟁 시스템`을 `✅ 완료`로 변경, 하위 체크박스 `[x]`로
    - 결과물에 `DisputeController`, `DisputeService`, Mobile `dispute` feature 명시
  - Phase 3 섹션:
    - `3.1 에스크로 정산 시스템` 정산 주기 `D+3` → `D+1`
    - `3.3 알림 시스템`을 `✅ 완료`로 변경, 하위 체크박스 `[x]`로
    - 결과물에 `NotificationController`, `FcmService`, Mobile `notification` feature 명시
  - Phase 4 섹션:
    - `4.2 보안 강화`에 `AES-256-GCM 암호화 구현 완료` 추가 (부분 완료)
    - `4.4 앱스토어 심사 준비`에 `Apple 소셜 로그인 구현` 항목 추가
  - `우선순위 요약` 섹션:
    - 완료된 항목(프로필, 찜, 신고/분쟁, 알림)에 완료 노트 추가
  - `마일스톤` 테이블:
    - Phase 1: `2026-02-23 (완료)` | `✅ 100% 완료`
    - Phase 2 상태: `🚧 30% 진행 중`
    - Phase 2~Launch 날짜: `TBD`
  - **내부 참조 경로 수정**:
    - `./구매_판매_내역_개발_계획서.md` → `../../../구매_판매_내역_개발_계획서.md`
    - `./ticket_platform_mobile/*` → `../../../ticket_platform_mobile/*`

  **Must NOT do**:
  - 문서 구조 변경 금지
  - `리스크 및 대응 방안` 섹션 수정 금지
  - 영문 산문 작성 금지
  - 미래 마일스톤에 날짜 임의 설정 금지 (TBD만)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: 대규모 문서(684줄) 수정, 다수 Phase 섹션 동시 변경 + 경로 수정
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 1, 3)
  - **Blocks**: [Task 4, Task 5]
  - **Blocked By**: [Task 0]

  **References**:

  **Pattern References**:
  - `docs/pm/status/ROADMAP.md` — 전체 문서 (684줄). Phase 구조, `[x]`/`[ ]` 패턴, 이모지 스타일 따를 것

  **API/Type References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/DisputeController.cs` — 신고 시스템 완성 확인
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Notification/INotificationService.cs` — 알림 시스템 완성 확인
  - `ticket_platform_mobile/lib/features/dispute/` — 47 dart 파일, 모바일 신고 기능
  - `ticket_platform_mobile/lib/features/notification/` — 41 dart 파일, 모바일 알림 기능

  **Acceptance Criteria**:
  - [ ] `grep "D+3" docs/pm/status/ROADMAP.md` → 0건
  - [ ] `grep "2026-02-23" docs/pm/status/ROADMAP.md` → 날짜 확인
  - [ ] `grep "3.0" docs/pm/status/ROADMAP.md` → 버전 확인
  - [ ] `grep "TBD" docs/pm/status/ROADMAP.md` → 미래 마일스톤 TBD 확인
  - [ ] `grep -A2 "Apple" docs/pm/status/ROADMAP.md` → Phase 4 컨텍스트 확인

  **QA Scenarios**:
  ```
  Scenario: D+3 완전 제거 확인
    Tool: Bash (grep)
    Steps:
      1. grep "D+3" docs/pm/status/ROADMAP.md
      2. 결과가 0건인지 확인
    Expected Result: 0 matches
    Evidence: .sisyphus/evidence/task-2-d3-removal.txt

  Scenario: 마일스톤 TBD 반영 확인
    Tool: Bash (grep)
    Steps:
      1. grep "TBD" docs/pm/status/ROADMAP.md
      2. 미래 마일스톤에 TBD가 있는지 확인
    Expected Result: ≥2 TBD matches
    Evidence: .sisyphus/evidence/task-2-milestones.txt
  ```

  **Commit**: YES
  - Message: `docs(pm): update ROADMAP.md to v3.0 — sync with codebase 2026-02-23`
  - Files: `docs/pm/status/ROADMAP.md`
  - Pre-commit: `grep "D+3" docs/pm/status/ROADMAP.md` (should be empty)

- [ ] 3. PM_README.md v3.0 최신화 + 내부 경로 수정

  **What to do**:
  - `문서 버전`을 `3.0`으로, `최종 수정일`을 `2026-02-23`으로 변경
  - `현재 상태` > `완료된 기능` 섹션:
    - `진행률` `~85%` → `100% (Phase 1 완료)`로 변경 (Phase 1 기준)
    - 신규 완성 기능 추가:
      - `10. 신고/분쟁 시스템 ✅` (DisputeController + 모바일 dispute)
      - `11. 알림 시스템 (FCM) ✅` (NotificationController + 모바일 notification)
      - `12. 소셜 로그인 (Google, Kakao) ✅`
      - `13. 비밀번호 변경 ✅`
    - 프로필 업데이트를 `진행 중`에서 `완료`로 이동
  - `진행 중인 작업` 섹션: 프로필 이미지 업로드 제거 (완료됨)
  - `미완료 기능` 섹션:
    - Phase 2에서 `신고/분쟁 시스템` 제거 (완료됨)
    - Phase 3에서 `알림 시스템 (FCM)` 제거 (완료됨)
    - Phase 4에 `Apple 소셜 로그인` 항목 추가
  - `개발 로드맵` 섹션:
    - 완료된 항목(티켓 상세/구매, 판매 등록, 결제, 찜) 체크박스 `[x]`로 변경
    - 에스크로 정산 D+3 → D+1 변경
  - `다음 단계` 섹션 전면 재작성:
    - `즉시 처리 필요`: Phase 2 계속 — 티켓 검증 시스템(QR), 본인 인증
    - `단기 목표 (1-2주)`: 정산 스케줄러(D+1), 평판 시스템
    - `중기 목표 (1개월)`: 관리자 대시보드, Apple 소셜 로그인, 성능 최적화
  - `외부 서비스` 섹션:
    - `Payment: Toss Payments (연동 예정)` → `Payment: Toss Payments (연동 완료)`
  - `프로젝트 구조` 트리에 `dispute/`, `notification/` 피처 추가
  - `핵심 기능` 목록에 알림 시스템(FCM) 항목 추가
  - `비즈니스 모델` 섹션:
    - `에스크로 보관금 정책: D+3` → `D+1`
  - **내부 참조 경로 수정**:
    - `[ROADMAP.md](./ROADMAP.md)` → `[ROADMAP.md](./ROADMAP.md)` (같은 폴더 — 유지)
    - `./TicketPlatFormServer/*` → `../../../TicketPlatFormServer/*`
    - `./ticket_platform_mobile/*` → `../../../ticket_platform_mobile/*`

  **Must NOT do**:
  - 문서 구조 변경 금지
  - 이미 정확한 섹션 수정 금지
  - 영문 산문 작성 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: 다수 섹션 변경 + '다음 단계' 전면 재작성 + 경로 수정
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 1, 2)
  - **Blocks**: [Task 4, Task 5]
  - **Blocked By**: [Task 0]

  **References**:

  **Pattern References**:
  - `docs/pm/status/PM_README.md` — 전체 문서 (382줄). 이모지, 테이블, 체크박스 스타일 따를 것

  **Acceptance Criteria**:
  - [ ] `grep "2026-02-23" docs/pm/status/PM_README.md` → 날짜 확인
  - [ ] `grep "3.0" docs/pm/status/PM_README.md` → 버전 확인
  - [ ] `grep "D+3" docs/pm/status/PM_README.md` → 0건
  - [ ] `grep "연동 완료" docs/pm/status/PM_README.md` → Toss Payments 상태 확인
  - [ ] `grep -c "티켓 검증\|본인 인증\|정산 스케줄러" docs/pm/status/PM_README.md` → 다음 단계 업데이트 확인

  **QA Scenarios**:
  ```
  Scenario: '다음 단계' 섹션 최신화 확인
    Tool: Bash (grep)
    Steps:
      1. grep -A10 "다음 단계" docs/pm/status/PM_README.md
      2. '티켓 검증', '본인 인증', '정산 스케줄러' 등 실제 남은 작업이 언급되는지 확인
    Expected Result: 현재 우선순위 내용 포함
    Evidence: .sisyphus/evidence/task-3-next-steps.txt

  Scenario: D+3 완전 제거 확인
    Tool: Bash (grep)
    Steps:
      1. grep "D+3" docs/pm/status/PM_README.md
      2. 결과가 0건인지 확인
    Expected Result: 0 matches
    Evidence: .sisyphus/evidence/task-3-d3-removal.txt
  ```

  **Commit**: YES
  - Message: `docs(pm): update PM_README.md to v3.0 — sync with codebase 2026-02-23`
  - Files: `docs/pm/status/PM_README.md`
  - Pre-commit: `grep "D+3" docs/pm/status/PM_README.md` (should be empty)

- [ ] 4. 외부 참조 경로 수정 + docs/pm/README.md 업데이트

  **What to do**:
  - `docs/README.md` 라인 205-206 수정:
    - `[PM_README.md](../PM_README.md)` → `[PM_README.md](pm/status/PM_README.md)`
    - `[ROADMAP.md](../ROADMAP.md)` → `[ROADMAP.md](pm/status/ROADMAP.md)`
  - `docs/shared/README.md` 라인 65-68 수정:
    - `[PM_README.md](../../PM_README.md)` → `[PM_README.md](../../docs/pm/status/PM_README.md)`
    - `[ROADMAP.md](../../ROADMAP.md)` → `[ROADMAP.md](../../docs/pm/status/ROADMAP.md)`
    - `[IMPLEMENTATION_STATUS.md](../../IMPLEMENTATION_STATUS.md)` → `[IMPLEMENTATION_STATUS.md](../../docs/pm/status/IMPLEMENTATION_STATUS.md)`
  - `docs/pm/README.md`에 `status/` 폴더 설명 추가:
    - `### 📊 status/ — 프로젝트 현황 문서`
    - 포함 내용: IMPLEMENTATION_STATUS.md, ROADMAP.md, PM_README.md
    - 기존 README의 이모지/마크다운 스타일에 맞춤

  **Must NOT do**:
  - scope 외 문서(SETUP_COMPLETE.md, handoff 문서) 수정 금지 (텍스트 멘션만 있으므로)
  - 기존 docs/pm/README.md 구조 변경 금지 (섹션 추가만)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 소규모 경로 변경 3건 + README 섹션 추가 1건
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 3 (After Wave 2)
  - **Blocks**: [Task 5]
  - **Blocked By**: [Task 1, Task 2, Task 3]

  **References**:
  - `docs/README.md:205-206` — PM 문서 참조 링크 2개
  - `docs/shared/README.md:65-68` — PM 문서 참조 링크 3개
  - `docs/pm/README.md` — status/ 폴더 설명 추가 대상 (77줄)

  **Acceptance Criteria**:
  - [ ] `grep "pm/status/PM_README.md" docs/README.md` → 새 경로 확인
  - [ ] `grep "pm/status/ROADMAP.md" docs/README.md` → 새 경로 확인
  - [ ] `grep "pm/status" docs/shared/README.md` → 새 경로 확인
  - [ ] `grep "status/" docs/pm/README.md` → 폴더 설명 추가 확인

  **QA Scenarios**:
  ```
  Scenario: 외부 참조 경로 수정 확인
    Tool: Bash (grep)
    Steps:
      1. grep -n "PM_README\|ROADMAP\|IMPLEMENTATION_STATUS" docs/README.md
      2. 경로가 pm/status/ 를 포함하는지 확인
      3. grep -n "PM_README\|ROADMAP\|IMPLEMENTATION_STATUS" docs/shared/README.md
      4. 경로가 pm/status/ 를 포함하는지 확인
    Expected Result: 모든 참조가 새 경로 사용
    Evidence: .sisyphus/evidence/task-4-external-refs.txt

  Scenario: docs/pm/README.md에 status/ 설명 추가 확인
    Tool: Bash (grep)
    Steps:
      1. grep "status" docs/pm/README.md
      2. status 폴더 설명이 존재하는지 확인
    Expected Result: status 폴더 설명 포함
    Evidence: .sisyphus/evidence/task-4-pm-readme.txt
  ```

  **Commit**: YES
  - Message: `docs: update cross-references for moved PM docs`
  - Files: `docs/README.md`, `docs/shared/README.md`, `docs/pm/README.md`

- [ ] 5. 3개 문서 교차 검증 + 링크 검증

  **What to do**:
  - 3개 문서를 모두 읽고 다음 항목 교차 확인:
    1. 완료로 표시된 기능이 3개 문서 모두에서 일관되는지
    2. Phase별 진행률이 동일한지
    3. D+3 참조가 전부 제거되었는지
    4. 날짜/버전이 모두 `2026-02-23`/`v3.0`인지
    5. Apple 로그인이 Phase 4로 기재되었는지
    6. 모든 마크다운 링크가 실제 파일을 가리키는지 (상대 경로 확인)
  - 모순 발견 시 해당 문서를 직접 수정하여 해결
  - 외부 참조 파일(docs/README.md, docs/shared/README.md)의 링크도 유효한지 확인
  - 최종 결과를 evidence 파일로 저장

  **Must NOT do**:
  - 기능 상태 판단 임의 변경 금지
  - scope 외 문서 수정 금지

  **Recommended Agent Profile**:
  - **Category**: `deep`
    - Reason: 3개 문서 + 2개 외부 참조 파일 교차 분석, 깊은 이해력 필요
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 4 (Sequential after Wave 3)
  - **Blocks**: [F1, F2, F3]
  - **Blocked By**: [Task 4]

  **References**:
  - `docs/pm/status/IMPLEMENTATION_STATUS.md` — Task 1에서 수정된 문서
  - `docs/pm/status/ROADMAP.md` — Task 2에서 수정된 문서
  - `docs/pm/status/PM_README.md` — Task 3에서 수정된 문서
  - `docs/README.md` — Task 4에서 수정된 문서
  - `docs/shared/README.md` — Task 4에서 수정된 문서

  **Acceptance Criteria**:
  - [ ] `grep "D+3" docs/pm/status/*.md` → 0건 (3개 문서 전체)
  - [ ] 3개 문서의 Phase별 진행률이 동일한지 확인
  - [ ] 완료된 기능 목록이 3개 문서에서 일관되는지 확인
  - [ ] 모든 마크다운 링크가 유효한 파일을 가리키는지 확인
  - [ ] 모순 발견 0건 또는 모두 해결

  **QA Scenarios**:
  ```
  Scenario: 전체 D+3 제거 확인
    Tool: Bash (grep)
    Steps:
      1. grep "D+3" docs/pm/status/IMPLEMENTATION_STATUS.md docs/pm/status/ROADMAP.md docs/pm/status/PM_README.md
      2. 결과가 0건인지 확인
    Expected Result: 0 matches across all 3 files
    Evidence: .sisyphus/evidence/task-5-cross-d3.txt

  Scenario: 날짜/버전 일관성 확인
    Tool: Bash (grep)
    Steps:
      1. grep "2026-02-23" docs/pm/status/*.md
      2. 모든 문서에 2026-02-23이 있는지 확인
      3. grep "3.0" docs/pm/status/*.md
      4. 모든 문서에 3.0이 있는지 확인
    Expected Result: All 3 files contain both 2026-02-23 and 3.0
    Evidence: .sisyphus/evidence/task-5-cross-version.txt

  Scenario: 마크다운 링크 유효성 확인
    Tool: Bash (grep + ls)
    Steps:
      1. 각 문서에서 마크다운 링크 추출
      2. 상대 경로를 절대 경로로 변환하여 파일 존재 확인
    Expected Result: 모든 링크가 유효한 파일을 가리킴
    Evidence: .sisyphus/evidence/task-5-link-validation.txt
  ```

  **Commit**: NO (verification only)

---

## Final Verification Wave (MANDATORY — after ALL implementation tasks)

> 3 review agents run in PARALLEL. ALL must APPROVE. Rejection → fix → re-run.

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Read the plan end-to-end. For each "Must Have": verify implementation exists (grep, ls). For each "Must NOT Have": search for forbidden patterns — reject with file:line if found. Check evidence files exist in .sisyphus/evidence/. Compare deliverables against plan.
  Output: `Must Have [N/N] | Must NOT Have [N/N] | Tasks [N/N] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Scope Fidelity Check** — `deep`
  For each task: read "What to do", read actual diff (git log/diff). Verify 1:1 — everything in spec was done (no missing), nothing beyond spec was done (no creep). Check "Must NOT do" compliance. Flag unaccounted changes.
  Output: `Tasks [N/N compliant] | Unaccounted [CLEAN/N files] | VERDICT`

- [ ] F3. **문서 grep 자동 검증** — `unspecified-high`
  Execute ALL acceptance criteria grep commands across all tasks. Verify exact expected results. Cross-check all 3 documents for mutual consistency. Verify all markdown links point to existing files.
  Output: `Grep checks [N/N pass] | Cross-doc [N/N consistent] | Links [N/N valid] | VERDICT`

---

## Commit Strategy

- **0**: `docs(pm): move PM docs to docs/pm/status/ and cleanup old plan` — git mv + 디렉토리 생성
- **1**: `docs(pm): update IMPLEMENTATION_STATUS.md to v3.0` — docs/pm/status/IMPLEMENTATION_STATUS.md
- **2**: `docs(pm): update ROADMAP.md to v3.0` — docs/pm/status/ROADMAP.md
- **3**: `docs(pm): update PM_README.md to v3.0` — docs/pm/status/PM_README.md
- **4**: `docs: update cross-references for moved PM docs` — docs/README.md, docs/shared/README.md, docs/pm/README.md
- **5**: (no commit — verification only)

---

## Success Criteria

### Verification Commands
```bash
# 파일 이동 확인
ls docs/pm/status/IMPLEMENTATION_STATUS.md docs/pm/status/ROADMAP.md docs/pm/status/PM_README.md  # Expected: all exist
ls IMPLEMENTATION_STATUS.md ROADMAP.md PM_README.md 2>/dev/null  # Expected: no such file

# D+3 제거 확인
grep "D+3" docs/pm/status/*.md  # Expected: no matches

# 날짜 업데이트 확인
grep "2026-02-23" docs/pm/status/*.md  # Expected: matches in all 3

# 버전 확인
grep "3.0" docs/pm/status/*.md  # Expected: matches in all 3

# 6개 신규 기능 반영 확인
grep -c "분쟁\|FCM\|소셜.*로그인\|BackgroundService\|비밀번호.*변경\|AES" docs/pm/status/IMPLEMENTATION_STATUS.md  # Expected: ≥ 6

# 컨트롤러 수 확인
grep "12개 컨트롤러" docs/pm/status/IMPLEMENTATION_STATUS.md  # Expected: match

# 깨진 링크 없는지 확인 (마크다운 링크 추출 → 파일 존재 확인)
grep -oP '\]\(\K[^)]+' docs/pm/status/*.md | head -20  # Manual review
```

### Final Checklist
- [ ] 루트에 PM 문서 없음
- [ ] docs/pm/status/에 3개 문서 존재
- [ ] All "Must Have" present
- [ ] All "Must NOT Have" absent
- [ ] 3개 문서 교차 모순 없음
- [ ] 모든 마크다운 링크 유효
- [ ] 모든 D+3 → D+1 변경 완료
