# 정산 계좌 인증 + 정산 상태 표시 — Mobile

## TL;DR

> **Quick Summary**: 판매자 계좌 등록/1원 인증 화면과 판매 대시보드 정산 상태 표시(on_hold 포함)를 Flutter 앱에 구현한다.
> 
> **Deliverables**:
> - 계좌 등록/인증/상세 화면 (bank_account 피처)
> - 판매 대시보드 정산 상태 개선 (on_hold 상태 추가)
> - 프로필 메뉴 계좌 인증 연결
> 
> **Estimated Effort**: Medium
> **Parallel Execution**: YES - 3 waves
> **Critical Path**: M1 (모델) → M2 (데이터) → M3 (ViewModel) → M4 (UI) → M6 (프로필 연결) → Task 10 (통합 검증)

---

## Context

### Original Request
판매자 계좌 등록/1원 인증 UI와 구매 확정 후 정산 상태가 판매 대시보드에 올바르게 표시되는 Mobile 파트 구현.

### 관련 Backend 플랜
> Backend API는 `.sisyphus/plans/settlement-backend.md`에서 관리됩니다.
> Mobile은 Backend API가 구현된 후에 실제 연동 테스트가 가능합니다.
> 단, Mobile 코드 자체의 빌드/분석은 Backend 없이 독립적으로 가능합니다.

### Interview Summary
**Key Decisions**:
- 계좌 등록 시점: 자유 등록 (프로필에서 언제든 가능)
- 인증 방식: 1원 인증 (코드 입력, 5분 만료)
- 프로필 메뉴에서 계좌 상태에 따른 동적 라우팅

**Research Findings**:
- Mobile 프로필: `계좌 인증` 메뉴 아이템 존재하나 onTap 핸들러 없음 (플레이스홀더)
- Sales Dashboard: feature-first Clean MVVM 구조로 이미 정산 상태 표시 존재
- Freezed + Riverpod codegen 패턴 사용 중
- Dio 기반 네트워크 스택 (`lib/core/network/`)

---

## Work Objectives

### Core Objective
판매자가 모바일 앱에서 계좌를 등록하고 1원 인증을 완료할 수 있으며, 판매 대시보드에서 정산 보류(on_hold) 상태를 포함한 정산 진행 상황을 확인할 수 있게 한다.

### Concrete Deliverables
- `lib/features/bank_account/` 피처 디렉터리 (data/domain/presentation)
- 계좌 등록 화면, 인증 화면, 상세 화면
- 판매 대시보드 on_hold 상태 표시 + 인증 유도 안내
- 프로필 메뉴 계좌 인증 onTap 연결

### Definition of Done
- [ ] 계좌 등록 화면에서 은행/계좌번호/예금주 입력 가능
- [ ] 1원 인증 화면에서 인증 코드 입력 + 타이머(5분) 표시
- [ ] 인증 완료/미인증 상태가 프로필에 표시
- [ ] 판매 대시보드에 on_hold 상태가 "정산 보류"로 표시
- [ ] `flutter analyze` no issues
- [ ] `build_runner build` 성공

### Must Have
- 계좌 등록 폼 (은행 선택, 계좌번호, 예금주)
- 1원 인증 요청/확인 화면
- 프로필 메뉴에서 계좌 상태별 라우팅
- 판매 대시보드 on_hold 상태 표시

### Must NOT Have (Guardrails)
- 생성 파일 (*.g.dart, *.freezed.dart) 직접 수정 금지
- 하드코딩 색상/스타일 금지 — AppColors, 테마 토큰 사용
- feature 코드에서 ad-hoc HTTP 클라이언트 금지 — core/network/ 사용
- shared widgets에 feature 비즈니스 로직 금지

---

## Verification Strategy (MANDATORY)

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed.

### Test Decision
- **Infrastructure exists**: Flutter test 디렉터리 존재
- **Automated tests**: NO (최소)
- **Primary verification**: flutter analyze + build_runner build

### QA Policy
- **Mobile Build**: Bash (flutter analyze, build_runner) — 0 errors
- **Mobile UI**: Playwright 또는 Flutter integration test (backend 연동 시)

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Foundation — Models + Types):
└── Task M1: Mobile 계좌 모델/DTO + Repository 인터페이스 [quick]

Wave 2 (Data + Logic):
├── Task M2: Mobile 계좌 Remote Data Source + Repository Impl [unspecified-high]
└── Task M3: Mobile 계좌 UseCase + ViewModel [unspecified-high]

Wave 3 (UI + Integration):
├── Task M4: Mobile 계좌 등록/인증 UI 화면 [visual-engineering]
├── Task M5: Mobile 판매 대시보드 정산 상태 개선 [visual-engineering]
└── Task M6: Mobile 프로필 메뉴 계좌 인증 연결 [quick]

Wave 4 (Verification):
└── Task 10: Mobile 통합 빌드 + 분석 [unspecified-high]

Wave FINAL (Review — 4 parallel):
├── Task F1: Plan compliance audit (oracle)
├── Task F2: Code quality review (unspecified-high)
├── Task F3: UI QA (unspecified-high + frontend-ui-ux)
└── Task F4: Scope fidelity check (deep)

Critical Path: M1 → M2 → M3 → M4 → M6 → Task 10 → F1-F4
Max Concurrent: 3 (Wave 3)
```

### Dependency Matrix

| Task | Depends On | Blocks | Wave |
|------|-----------|--------|------|
| M1 | — | M2, M3 | 1 |
| M2 | M1 | M3 | 2 |
| M3 | M2 | M4 | 2 |
| M4 | M3 | M6, 10 | 3 |
| M5 | M1 | 10 | 3 |
| M6 | M4 | 10 | 3 |
| 10 | M4, M5, M6 | F1-F4 | 4 |
| F1-F4 | 10 | — | FINAL |

### Agent Dispatch Summary

- **Wave 1**: **1 task** — M1 → `quick`
- **Wave 2**: **2 tasks** — M2 → `unspecified-high`, M3 → `unspecified-high`
- **Wave 3**: **3 tasks** — M4 → `visual-engineering`, M5 → `visual-engineering`, M6 → `quick`
- **Wave 4**: **1 task** — T10 → `unspecified-high`
- **FINAL**: **4 tasks** — F1 → `oracle`, F2 → `unspecified-high`, F3 → `unspecified-high`, F4 → `deep`

---

## TODOs

- [ ] M1. Mobile 계좌 모델/DTO + Repository 인터페이스

  **What to do**:
  - `lib/features/bank_account/` 피처 디렉터리 생성:
    - `data/dto/request/register_bank_account_req_dto.dart` — bankName, accountNumber, accountHolder, bankCode
    - `data/dto/request/verify_account_req_dto.dart` — code
    - `data/dto/response/bank_account_resp_dto.dart` — id, bankName, accountNumber, accountHolder, verified, verifiedAt
    - `data/dto/response/verify_account_resp_dto.dart` — verified, message
    - `data/dto/response/request_verification_resp_dto.dart` — expiresAt, message
    - `domain/entities/bank_account_entity.dart` — Freezed entity
    - `domain/repositories/bank_account_repository.dart` — 추상 인터페이스
  - Freezed 어노테이션 사용 (기존 패턴)
  - `dart run build_runner build --delete-conflicting-outputs` 실행

  **Must NOT do**: 생성 파일 직접 수정 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 모델/DTO 파일 생성 + build_runner — 패턴 따라하기
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES (독립적으로 시작 가능)
  - **Parallel Group**: Wave 1
  - **Blocks**: M2, M3
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/features/sales_dashboard/data/dto/response/sales_dashboard_resp_dto.dart` — Freezed DTO 패턴 (json_serializable, @freezed)
  - `ticket_platform_mobile/lib/features/sales_dashboard/domain/entities/event_group_entity.dart` — Entity 패턴
  - `ticket_platform_mobile/lib/features/sales_dashboard/domain/repositories/sales_dashboard_repository.dart` — Repository 인터페이스 패턴

  **WHY Each Reference Matters**:
  - DTO는 @freezed + @JsonSerializable 패턴을 정확히 따라야 build_runner가 코드를 생성함
  - Entity는 UI 레이어에서 사용하므로 DTO와 분리하되 같은 Freezed 패턴 사용

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: build_runner 성공
    Tool: Bash
    Preconditions: bank_account 피처 파일 작성 완료
    Steps:
      1. cd ticket_platform_mobile && dart run build_runner build --delete-conflicting-outputs
      2. Assert: 생성 파일 (*.g.dart, *.freezed.dart) 존재
      3. flutter analyze → bank_account 관련 에러 0건
    Expected Result: 빌드 성공, 생성 파일 정상
    Failure Indicators: build_runner 에러 또는 analyze 에러
    Evidence: .sisyphus/evidence/task-m1-build-runner.txt
  ```

  **Commit**: YES (groups with M2-M6)
  - Message: `feat(mobile/bank-account): add models, DTOs, and repository interface`
  - Files: `lib/features/bank_account/data/`, `lib/features/bank_account/domain/`
  - Pre-commit: `flutter analyze`

- [ ] M2. Mobile 계좌 Remote Data Source + Repository Impl

  **What to do**:
  - `data/datasources/bank_account_remote_data_source.dart`:
    - `registerBankAccount(RegisterBankAccountReqDto)` → BankAccountRespDto
    - `getMyBankAccount()` → BankAccountRespDto?
    - `deleteBankAccount()` → void
    - `requestVerification()` → RequestVerificationRespDto
    - `confirmVerification(VerifyAccountReqDto)` → VerifyAccountRespDto
  - `data/repositories/bank_account_repository_impl.dart` — DataSource → Entity 변환
  - 기존 Dio 프로바이더 패턴 사용 (core/network/)

  **Must NOT do**: ad-hoc HTTP 클라이언트 금지

  **Recommended Agent Profile**: `unspecified-high` | **Skills**: []
  **Parallelization**: Wave 2 | **Blocks**: M3 | **Blocked By**: M1

  **References**:
  - `ticket_platform_mobile/lib/features/sales_dashboard/data/datasources/sales_dashboard_remote_data_source.dart` — Remote 데이터 소스 패턴 (Dio 주입, API 호출)
  - `ticket_platform_mobile/lib/features/sales_dashboard/data/repositories/sales_dashboard_repository_impl.dart` — Repository 구현 패턴 (DataSource → Entity 변환)
  - `ticket_platform_mobile/lib/core/network/` — Dio 프로바이더, 인터셉터, 토큰 관리

  **WHY Each Reference Matters**:
  - sales_dashboard의 데이터 소스와 리포지토리 구현이 이 프로젝트에서 따라야 할 정확한 패턴

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: 빌드 + 분석
    Tool: Bash
    Steps: flutter analyze → bank_account 관련 에러 0건
    Evidence: .sisyphus/evidence/task-m2-analyze.txt
  ```
  **Commit**: YES (groups with M1-M6)

- [ ] M3. Mobile 계좌 UseCase + ViewModel

  **What to do**:
  - `domain/usecases/`:
    - `register_bank_account_usecase.dart`
    - `get_bank_account_usecase.dart`
    - `delete_bank_account_usecase.dart`
    - `request_verification_usecase.dart`
    - `confirm_verification_usecase.dart`
  - `presentation/viewmodels/bank_account_viewmodel.dart`:
    - 상태: loading, bankAccount, verificationState (idle/requesting/codeInput/verified/error), error
    - 액션: register, getMyAccount, requestVerification, confirmVerification, delete
  - `presentation/providers/bank_account_providers_di.dart` — Riverpod DI
  - Riverpod codegen 패턴 (`@riverpod`) 사용
  - `dart run build_runner build` 실행

  **Must NOT do**: 생성 파일 수정 금지

  **Recommended Agent Profile**: `unspecified-high` | **Skills**: []
  **Parallelization**: Wave 2 | **Blocks**: M4 | **Blocked By**: M2

  **References**:
  - `ticket_platform_mobile/lib/features/sales_dashboard/presentation/viewmodels/sales_dashboard_viewmodel.dart` — ViewModel 패턴 (@riverpod, state management)
  - `ticket_platform_mobile/lib/features/sales_dashboard/presentation/providers/sales_dashboard_providers_di.dart` — Riverpod DI 패턴
  - `ticket_platform_mobile/lib/features/sales_dashboard/domain/usecases/get_sales_dashboard_usecase.dart` — UseCase 패턴

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: build_runner + analyze
    Tool: Bash
    Steps:
      1. dart run build_runner build
      2. flutter analyze → 0 errors
    Evidence: .sisyphus/evidence/task-m3-analyze.txt
  ```
  **Commit**: YES (groups with M1-M6)

- [ ] M4. Mobile 계좌 등록/인증 UI 화면

  **What to do**:
  - `presentation/views/bank_account_register_view.dart`:
    - 은행 선택 (드롭다운), 계좌번호 입력, 예금주 입력
    - 유효성 검사 (빈값, 계좌번호 형식)
    - 등록 버튼 → API 호출 → 성공 시 인증 화면으로 이동
  - `presentation/views/bank_account_verify_view.dart`:
    - "1원 인증 요청" 버튼 → 인증 코드 입력 필드 표시
    - 인증 코드 4자리 입력 → 확인 → 성공/실패 상태 표시
    - 만료 시간 타이머 표시 (5분)
  - `presentation/views/bank_account_detail_view.dart`:
    - 등록된 계좌 정보 표시 (마스킹 계좌번호)
    - 인증 상태 표시 (인증 완료/미인증)
    - 계좌 삭제 버튼
  - 라우팅: `AppRouter`에 새 라우트 추가
  - AppColors, 기존 위젯 사용

  **Must NOT do**: 하드코딩 색상, 생성파일 수정 금지

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
    - Reason: UI 화면 3개 + 폼 + 타이머 — UI/UX 전문성 필요
  - **Skills**: [`frontend-ui-ux`]
    - `frontend-ui-ux`: 디자인 목업 없이도 좋은 UI를 만들어야 함

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (with M5, M6)
  - **Blocks**: M6, Task 10
  - **Blocked By**: M3

  **References**:
  - `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart:154-170` — 계좌 인증 메뉴 아이템 (연결 대상)
  - `ticket_platform_mobile/lib/shared/widgets/` — 공용 위젯 (폼 입력, 버튼 등)
  - `ticket_platform_mobile/lib/core/theme/` — AppColors, 텍스트 스타일
  - `ticket_platform_mobile/lib/features/sell/` — 멀티스텝 폼 UI 참고

  **WHY Each Reference Matters**:
  - profile_view.dart의 메뉴 아이템이 이 화면들의 진입점
  - shared/widgets와 core/theme은 일관된 UI를 위해 반드시 참조

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: 계좌 등록 화면 빌드
    Tool: Bash
    Steps:
      1. flutter analyze → 0 errors
      2. grep 'AppRouterPath.bankAccount' 관련 라우트 등록 확인
    Expected Result: UI 파일 존재, 라우트 등록, 분석 통과
    Failure Indicators: analyze 에러 또는 라우트 미등록
    Evidence: .sisyphus/evidence/task-m4-ui-analyze.txt

  Scenario: 인증 코드 입력 위젯 확인
    Tool: Bash (grep)
    Steps:
      1. grep 'Timer\|countdown\|expire' bank_account_verify_view.dart
      2. Assert: 타이머 관련 코드 존재
    Expected Result: 만료 타이머 구현 존재
    Evidence: .sisyphus/evidence/task-m4-timer-check.txt
  ```

  **Commit**: YES (groups with M1-M6)
  - Message: `feat(mobile/bank-account): add registration, verification, and detail views`
  - Files: `lib/features/bank_account/presentation/views/`

- [ ] M5. Mobile 판매 대시보드 정산 상태 개선

  **What to do**:
  - 상태 매핑에 `settlement_on_hold` → "정산 보류" 추가 (색상: 주황색/경고)
  - 기존 `settlement_completed`, `payment_completed` 등과 일관성 유지
  - 대시보드 UI에 "계좌 인증 필요" 안내 메시지 + 인증 화면 이동 링크 (on_hold 시)
  - `SellService.ResolveEventTicketDetailStatus` 백엔드 매핑과 일치시킬 것

  **Must NOT do**: 생성파일 수정 금지

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
    - Reason: 상태 표시 UI 수정 + 안내 메시지
  - **Skills**: [`frontend-ui-ux`]

  **Parallelization**: Wave 3 | **Blocks**: Task 10 | **Blocked By**: M1

  **References**:
  - `ticket_platform_mobile/lib/features/sales_dashboard/presentation/ui_models/event_ticket_ui_model.dart` — 상태 매핑 모델
  - `ticket_platform_mobile/lib/features/sales_dashboard/presentation/widgets/event_ticket_item.dart` — 티켓 아이템 UI
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Sell/SellService.cs:563+` — 백엔드 상태 매핑 (on_hold 포함)

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: on_hold 상태 매핑 존재 + 빌드
    Tool: Bash
    Steps:
      1. grep 'on_hold\|settlement_on_hold' 관련 파일 → 매핑 존재
      2. flutter analyze → 0 errors
    Evidence: .sisyphus/evidence/task-m5-status-mapping.txt
  ```

  **Commit**: YES (groups with M1-M6)
  - Message: `feat(mobile/sales-dashboard): add on_hold settlement status display`

- [ ] M6. Mobile 프로필 메뉴 계좌 인증 연결

  **What to do**:
  - `profile_view.dart`의 "계좌 인증" ProfileMenuTile에 `onTap` 핸들러 추가:
    - 계좌 미등록 → 계좌 등록 화면으로 이동
    - 계좌 등록 + 미인증 → 인증 화면으로 이동
    - 계좌 인증 완료 → 계좌 상세 화면으로 이동
  - trailingText 동적 변경: "인증하기"/"인증 완료"/"등록하기"
  - 기존 `const` 제거 (동적 데이터 필요)

  **Must NOT do**: profile_view.dart의 다른 메뉴 아이템 변경 금지

  **Recommended Agent Profile**: `quick` | **Skills**: []
  **Parallelization**: Wave 3 | **Blocks**: Task 10 | **Blocked By**: M4

  **References**:
  - `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart:154-170` — 수정 대상 (현재 const + 핸들러 없음)

  **WHY Reference Matters**:
  - 이 코드 블록이 정확한 수정 대상 — `const` 키워드 제거 필요, onTap 추가 필요

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: 프로필 메뉴 onTap 연결 + 빌드
    Tool: Bash
    Steps:
      1. flutter analyze → 0 errors
      2. grep 'onTap' profile_view.dart의 계좌 인증 영역 → 존재
      3. const 키워드가 해당 ProfileMenuTile에서 제거됨 확인
    Expected Result: onTap 핸들러 존재, 분석 통과
    Evidence: .sisyphus/evidence/task-m6-profile-connect.txt
  ```

  **Commit**: YES (groups with M1-M6)
  - Message: `feat(mobile/profile): connect bank account verification menu`

- [ ] 10. Mobile 통합 빌드 + 분석

  **What to do**:
  - `dart run build_runner build --delete-conflicting-outputs` → 생성 성공
  - `flutter analyze` → no issues
  - 라우팅 확인: 모든 새 화면이 AppRouter에 등록됨
  - 프로필 → 계좌 등록 → 인증 → 상세 네비게이션 경로 확인

  **Recommended Agent Profile**: `unspecified-high` | **Skills**: []
  **Parallelization**: Wave 4 | **Blocks**: F1-F4 | **Blocked By**: M4, M5, M6

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: 전체 빌드 + 분석
    Tool: Bash
    Steps:
      1. cd ticket_platform_mobile
      2. dart run build_runner build --delete-conflicting-outputs → 성공
      3. flutter analyze → No issues found
      4. grep 'bankAccount' 관련 라우트 등록 확인
    Expected Result: 빌드 성공, 분석 통과, 라우트 등록
    Failure Indicators: build_runner 에러, analyze 에러
    Evidence: .sisyphus/evidence/task-10-mobile-build.txt
  ```
  **Commit**: NO (검증만)

---

## Final Verification Wave (MANDATORY — after ALL implementation tasks)

> 4 review agents run in PARALLEL. ALL must APPROVE. Rejection → fix → re-run.

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Read the plan end-to-end. For each "Must Have": verify implementation exists. For each "Must NOT Have": search codebase for forbidden patterns. Check evidence files.
  Output: `Must Have [N/N] | Must NOT Have [N/N] | Tasks [N/N] | VERDICT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  Run `flutter analyze`. Review changed files for: hardcoded colors, ad-hoc HTTP, commented-out code. Check AI slop.
  Output: `Analyze [PASS/FAIL] | Files [N clean/N issues] | VERDICT`

- [ ] F3. **UI QA** — `unspecified-high` (+ `frontend-ui-ux` skill)
  Review all new views for: consistent design tokens, proper error states, loading states, empty states. Verify navigation paths. Check responsive layout.
  Output: `Views [N/N reviewed] | Issues [N found] | VERDICT`

- [ ] F4. **Scope Fidelity Check** — `deep`
  For each task: read spec, read actual diff. Verify 1:1. Check "Must NOT do" compliance. Flag unaccounted changes.
  Output: `Tasks [N/N compliant] | Unaccounted [CLEAN/N files] | VERDICT`

---

## Commit Strategy

| Tasks | Commit Message | Files |
|-------|---------------|-------|
| M1 | `feat(mobile/bank-account): add models, DTOs, and repository interface` | `lib/features/bank_account/data/dto/`, `lib/features/bank_account/domain/` |
| M2 | `feat(mobile/bank-account): add remote data source and repository impl` | `lib/features/bank_account/data/datasources/`, `lib/features/bank_account/data/repositories/` |
| M3 | `feat(mobile/bank-account): add usecases and viewmodel` | `lib/features/bank_account/domain/usecases/`, `lib/features/bank_account/presentation/` |
| M4 | `feat(mobile/bank-account): add registration, verification, and detail views` | `lib/features/bank_account/presentation/views/` |
| M5 | `feat(mobile/sales-dashboard): add on_hold settlement status display` | `lib/features/sales_dashboard/` |
| M6 | `feat(mobile/profile): connect bank account verification menu` | `lib/features/profile/presentation/views/profile_view.dart` |

---

## Success Criteria

### Verification Commands
```bash
cd ticket_platform_mobile && dart run build_runner build --delete-conflicting-outputs  # Expected: success
cd ticket_platform_mobile && flutter analyze  # Expected: No issues found
```

### Final Checklist
- [ ] All "Must Have" present (4/4)
- [ ] All "Must NOT Have" absent (4/4)
- [ ] `flutter analyze` passes
- [ ] `build_runner build` passes
- [ ] 계좌 등록/인증/상세 화면 존재
- [ ] 프로필 메뉴 연결 동작
- [ ] 판매 대시보드 on_hold 상태 표시
