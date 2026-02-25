# Bank Account Verification Refactor

## TL;DR

> **Quick Summary**: Replace the 2-step 1원 인증 code flow with direct Toss Payments `verify-holder-real-name` API verification. Add bank account update (change) capability. Backend + Mobile full-stack refactor.
> 
> **Deliverables**:
> - DB migration: `identity_number` column added to `bank_account` table
> - Backend: Single-step Toss API verification, new PUT update endpoint, dead code removal
> - Mobile: Simplified register-with-verify flow, new 계좌 변경 screen, removed verify code UI
> 
> **Estimated Effort**: Medium
> **Parallel Execution**: YES - 3 waves
> **Critical Path**: Task 1 (DB) → Task 2 (Toss API) → Task 3 (Service) → Task 5 (Mobile DTOs) → Task 7 (ViewModel) → Task 8 (UI)

---

## Context

### Original Request
계좌 인증 로직을 리팩토링. 토스페이먼츠 `POST /v2/bank-accounts/verify-holder-real-name` (계좌번호·예금주명·식별자 일치 확인) API를 사용하여 실시간 인증. 계좌 변경도 가능해야 함.

### Interview Summary
**Key Discussions**:
- 계좌 변경: UPDATE 방식 (기존 레코드 유지, 정보만 수정, Verified=false 초기화 후 재인증)
- 생년월일(identityNumber): DB에 저장 (추후 본인 인증 기능 확장 고려)

**Research Findings**:
- Toss `verify-holder-real-name` API: `{bankCode, accountNumber, holderName, identityNumber}` → `entityBody.isValid`
- entityBody v2 wrapper 파싱 버그 방금 수정됨 — 신규 메서드에도 동일 패턴 적용
- Settlement FK → bank_account: pending settlement 체크 필수 (UPDATE에도 적용)
- `ValidateBankAccountAsync`, `VerifyBankAccountHolderNameAsync`는 BankAccountService에서만 사용 — 안전하게 제거 가능

### Metis Review
**Identified Gaps** (addressed):
- Settlement FK integrity on update → reuse `HasPendingOrProcessingSettlementsAsync` check
- `ResumeHeldSettlementsAsync` must still be called after verification → included in new flow
- VerificationState enum simplification needed → timer/codeInput states removed
- Existing DB records → leave as-is (stop using old columns, don't drop them)
- identityNumber storage → plaintext VARCHAR(20), nullable

---

## Work Objectives

### Core Objective
Replace the 2-step "1원 인증" code verification with direct Toss Payments `verify-holder-real-name` API call, making bank account verification instant. Add the ability to update (change) an existing bank account.

### Concrete Deliverables
- SQL migration file: `database_history/migrations/` adding `identity_number` column
- Backend: Rewritten `BankAccountService`, updated controller (5 → 4 endpoints), new Toss API method
- Mobile: Updated register flow with identityNumber field, new update flow, removed verify screen

### Definition of Done
- [ ] `dotnet build` passes with 0 errors
- [ ] `flutter analyze` passes with 0 errors
- [ ] `dart run build_runner build` completes successfully
- [ ] Register with identityNumber → Toss API called → account verified immediately
- [ ] Update existing account → Toss API called → re-verified
- [ ] Old verify/request and verify/confirm endpoints return 404
- [ ] No references to `requestVerification` or `confirmVerification` in non-generated Dart files

### Must Have
- Single-step Toss API verification (no more 2-step code flow)
- Bank account UPDATE capability with pending settlement check
- `identity_number` column in DB (nullable, VARCHAR(20))
- `ResumeHeldSettlementsAsync` called after successful verification
- identityNumber (생년월일 YYMMDD) input field on mobile

### Must NOT Have (Guardrails)
- Do NOT add 본인 인증 feature — only store identityNumber for future use
- Do NOT add multiple bank accounts per user (keep 1:1)
- Do NOT drop `verification_code` or `verification_expires_at` DB columns (migration safety)
- Do NOT modify Settlement entity, FK, or processing logic
- Do NOT add client-side Toss API calls — all verification through backend
- Do NOT add unit tests (no test project exists)
- Do NOT modify `PaymentService.ResumeHeldSettlementsAsync` — only call it
- Do NOT refactor `TossPaymentsService` beyond adding new method and removing unused ones

---

## Verification Strategy

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed.

### Test Decision
- **Infrastructure exists**: NO (backend has no test project; mobile has test/ but no relevant tests)
- **Automated tests**: None
- **Framework**: n/a

### QA Policy
Every task MUST include agent-executed QA scenarios.
Evidence saved to `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`.

- **Backend API**: Use Bash (curl) — send requests, assert status + response fields
- **Mobile build**: Use Bash — flutter analyze, build_runner, grep for dead references
- **DB**: Use Bash (mysql) — verify column exists

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Foundation — DB + types + Toss API method):
├── Task 1: DB migration + entity update [quick]
├── Task 2: TossPaymentsService — add VerifyBankAccountHolderRealNameAsync [quick]
└── Task 3: Backend DTOs — new/update request/response DTOs [quick]

Wave 2 (Backend core — service + controller + cleanup):
├── Task 4: BankAccountService rewrite (depends: 1, 2, 3) [deep]
└── Task 5: BankAccountController update (depends: 4) [quick]

Wave 3 (Mobile — full-stack, runs after backend is stable):
├── Task 6: Mobile data layer — DTOs, datasource, repository (depends: 5) [unspecified-high]
├── Task 7: Mobile domain + viewmodel — entities, usecases, state (depends: 6) [unspecified-high]
└── Task 8: Mobile UI — register, detail, router cleanup (depends: 7) [visual-engineering]

Wave FINAL (Verification):
├── F1: Plan compliance audit [oracle]
├── F2: Code quality review [unspecified-high]
├── F3: Real manual QA [unspecified-high]
└── F4: Scope fidelity check [deep]

Critical Path: Task 1 → Task 2 → Task 4 → Task 5 → Task 6 → Task 7 → Task 8 → FINAL
Parallel Speedup: ~40% faster than sequential (Wave 1 has 3 parallel tasks)
Max Concurrent: 3 (Wave 1)
```

### Dependency Matrix

| Task | Depends On | Blocks | Wave |
|------|-----------|--------|------|
| 1 | — | 4 | 1 |
| 2 | — | 4 | 1 |
| 3 | — | 4, 6 | 1 |
| 4 | 1, 2, 3 | 5 | 2 |
| 5 | 4 | 6 | 2 |
| 6 | 3, 5 | 7 | 3 |
| 7 | 6 | 8 | 3 |
| 8 | 7 | FINAL | 3 |

### Agent Dispatch Summary

- **Wave 1**: **3** — T1 → `quick`, T2 → `quick`, T3 → `quick`
- **Wave 2**: **2** — T4 → `deep`, T5 → `quick`
- **Wave 3**: **3** — T6 → `unspecified-high`, T7 → `unspecified-high`, T8 → `visual-engineering`
- **FINAL**: **4** — F1 → `oracle`, F2 → `unspecified-high`, F3 → `unspecified-high`, F4 → `deep`

---

## TODOs

- [ ] 1. DB Migration + Entity Update

  **What to do**:
  - Create SQL migration file `TicketPlatFormServer/TicketPlatFormServer/database_history/migrations/20260225_002_add_identity_number_to_bank_account.sql`
  - SQL: `ALTER TABLE bank_account ADD COLUMN identity_number VARCHAR(20) NULL AFTER account_holder;`
  - Use idempotent pattern: check column existence before adding (follow existing migration patterns in `database_history/migrations/`)
  - Add `IdentityNumber` property to `DBModel/BankAccount.cs`: `public string? IdentityNumber { get; set; }` with XML doc `/// <summary>계좌 소유자 식별자 (생년월일 YYMMDD 또는 사업자번호)</summary>`
  - Verify `TicketContext.cs` maps `identity_number` column correctly (EF conventions should auto-map)

  **Must NOT do**:
  - Do NOT drop `verification_code` or `verification_expires_at` columns
  - Do NOT modify Settlement entity or FK
  - Do NOT change any other DB table

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 2, 3)
  - **Blocks**: Task 4
  - **Blocked By**: None

  **References**:
  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/database_history/migrations/` — existing migration files for idempotent SQL pattern
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/BankAccount.cs` — current entity with XML doc style to follow

  **Acceptance Criteria**:
  - [ ] Migration file exists and is valid SQL
  - [ ] `dotnet build TicketPlatFormServer/TicketPlatFormServer.sln` → 0 errors
  - [ ] `IdentityNumber` property exists on `BankAccount` entity

  **QA Scenarios:**
  ```
  Scenario: DB migration applies cleanly
    Tool: Bash
    Steps:
      1. Run: `cat TicketPlatFormServer/TicketPlatFormServer/database_history/migrations/20260225_002_add_identity_number_to_bank_account.sql`
      2. Assert: file contains ALTER TABLE with identity_number VARCHAR(20) NULL
      3. Run: `dotnet build TicketPlatFormServer/TicketPlatFormServer.sln`
      4. Assert: Build succeeded, 0 errors
    Expected Result: Migration file valid, build passes
    Evidence: .sisyphus/evidence/task-1-migration-applied.txt

  Scenario: Entity has new property
    Tool: Bash (grep)
    Steps:
      1. Run: `grep -n 'IdentityNumber' TicketPlatFormServer/TicketPlatFormServer/DBModel/BankAccount.cs`
      2. Assert: Property found with `string?` type
    Expected Result: IdentityNumber property exists
    Evidence: .sisyphus/evidence/task-1-entity-property.txt
  ```

  **Commit**: YES (groups with Wave 1)
  - Message: `feat(bank-account): add identity_number column and Toss verify-holder-real-name API`
  - Files: `database_history/migrations/*.sql`, `DBModel/BankAccount.cs`

---

- [ ] 2. TossPaymentsService — Add VerifyBankAccountHolderRealNameAsync

  **What to do**:
  - Add new method to `ITossPaymentsService.cs`: `Task<bool> VerifyBankAccountHolderRealNameAsync(string bankCode, string accountNumber, string holderName, string identityNumber);`
  - Implement in `TossPaymentsService.cs` following the exact pattern of existing `ValidateBankAccountAsync` (lines 319-358):
    - Endpoint: `POST /v2/bank-accounts/verify-holder-real-name`
    - Payload: `{ bankCode, accountNumber, holderName, identityNumber }`
    - Parse response: `entityBody.isValid` (v2 wrapper — use the fixed parsing pattern)
    - Log request/response at Information level, failures at Warning level
  - Remove `ValidateBankAccountAsync` and `VerifyBankAccountHolderNameAsync` from both interface and implementation (confirmed unused outside BankAccountService)

  **Must NOT do**:
  - Do NOT modify any other TossPaymentsService methods (payment confirm, cancel, transfer, etc.)
  - Do NOT change `TossPaymentsSettings` here (Task 4 handles config cleanup)

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 3)
  - **Blocks**: Task 4
  - **Blocked By**: None

  **References**:
  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/TossPaymentsService.cs:319-358` — exact pattern for Toss v2 API call with entityBody parsing
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/ITossPaymentsService.cs:34-36` — existing method signatures to replace

  **API/Type References**:
  - Toss API docs: `POST /v2/bank-accounts/verify-holder-real-name` — params: bankCode (필수 string), accountNumber (필수 string), holderName (필수 string), identityNumber (필수 string, YYMMDD or 사업자번호)
  - Response: `{ "entityBody": { "isValid": true/false } }` — v2 wrapper

  **Acceptance Criteria**:
  - [ ] New method exists on interface and implementation
  - [ ] Old methods (`ValidateBankAccountAsync`, `VerifyBankAccountHolderNameAsync`) removed
  - [ ] `dotnet build` → 0 errors

  **QA Scenarios:**
  ```
  Scenario: New method compiles and old methods removed
    Tool: Bash
    Steps:
      1. Run: `grep -n 'VerifyBankAccountHolderRealNameAsync' TicketPlatFormServer/TicketPlatFormServer/Services/Payment/TossPaymentsService.cs`
      2. Assert: Method found
      3. Run: `grep -n 'ValidateBankAccountAsync' TicketPlatFormServer/TicketPlatFormServer/Services/Payment/ITossPaymentsService.cs`
      4. Assert: No matches (removed)
      5. Run: `grep -n 'VerifyBankAccountHolderNameAsync' TicketPlatFormServer/TicketPlatFormServer/Services/Payment/ITossPaymentsService.cs`
      6. Assert: No matches (removed)
      7. Run: `dotnet build TicketPlatFormServer/TicketPlatFormServer.sln`
      8. Assert: 0 errors
    Expected Result: New method present, old methods gone, build clean
    Evidence: .sisyphus/evidence/task-2-toss-api-method.txt

  Scenario: New method uses verify-holder-real-name endpoint
    Tool: Bash (grep)
    Steps:
      1. Run: `grep 'verify-holder-real-name' TicketPlatFormServer/TicketPlatFormServer/Services/Payment/TossPaymentsService.cs`
      2. Assert: Endpoint string found
      3. Run: `grep 'identityNumber' TicketPlatFormServer/TicketPlatFormServer/Services/Payment/TossPaymentsService.cs`
      4. Assert: Parameter used in payload
    Expected Result: Correct Toss endpoint and all 4 params in payload
    Evidence: .sisyphus/evidence/task-2-endpoint-verify.txt
  ```

  **Commit**: YES (groups with Wave 1)
  - Message: `feat(bank-account): add identity_number column and Toss verify-holder-real-name API`
  - Files: `ITossPaymentsService.cs`, `TossPaymentsService.cs`

---

- [ ] 3. Backend DTOs — New/Updated Request and Response DTOs

  **What to do**:
  - Add `IdentityNumber` field to `RegisterBankAccountRequestDto.cs`: `public string IdentityNumber { get; set; } = null!;`
  - Create new `UpdateBankAccountRequestDto.cs` in `DTO/BankAccount/` with fields: BankName, BankCode, AccountNumber, AccountHolder, IdentityNumber (all required strings, same as RegisterBankAccountRequestDto)
  - Simplify `BankAccountResponseDto.cs`: keep all existing fields, add `string? IdentityNumber` (masked or null for non-owner views)
  - Delete `RequestVerificationResponseDto.cs` (1원 인증 응답 — no longer needed)
  - Delete `VerifyAccountRequestDto.cs` (코드 입력 요청 — no longer needed)
  - Delete `VerifyAccountResponseDto.cs` (코드 검증 응답 — no longer needed)

  **Must NOT do**:
  - Do NOT change `ApiResponse<T>` base class
  - Do NOT change DTO naming convention

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2)
  - **Blocks**: Tasks 4, 6
  - **Blocked By**: None

  **References**:
  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/BankAccount/RegisterBankAccountRequestDto.cs` — current DTO structure to extend
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/BankAccount/BankAccountResponseDto.cs` — response shape to update

  **Acceptance Criteria**:
  - [ ] `RegisterBankAccountRequestDto` has `IdentityNumber` property
  - [ ] `UpdateBankAccountRequestDto.cs` exists with all required fields
  - [ ] 3 old verification DTOs deleted
  - [ ] `dotnet build` → 0 errors (will fail until Task 4 cleans up references — this is expected)

  **QA Scenarios:**
  ```
  Scenario: New DTOs exist with correct fields
    Tool: Bash
    Steps:
      1. Run: `grep -n 'IdentityNumber' TicketPlatFormServer/TicketPlatFormServer/DTO/BankAccount/RegisterBankAccountRequestDto.cs`
      2. Assert: Found
      3. Run: `test -f TicketPlatFormServer/TicketPlatFormServer/DTO/BankAccount/UpdateBankAccountRequestDto.cs && echo EXISTS`
      4. Assert: EXISTS
      5. Run: `test ! -f TicketPlatFormServer/TicketPlatFormServer/DTO/BankAccount/RequestVerificationResponseDto.cs && echo DELETED`
      6. Assert: DELETED
    Expected Result: New DTOs present, old DTOs removed
    Evidence: .sisyphus/evidence/task-3-dtos.txt
  ```

  **Commit**: YES (groups with Wave 1)
  - Message: `feat(bank-account): add identity_number column and Toss verify-holder-real-name API`
  - Files: `DTO/BankAccount/*.cs`

---

- [ ] 4. BankAccountService Rewrite

  **What to do**:
  - **Rewrite `RegisterBankAccountAsync`**: Accept `RegisterBankAccountRequestDto` (now with IdentityNumber). After creating the DB record, call `tossPaymentsService.VerifyBankAccountHolderRealNameAsync(bankCode, accountNumber, accountHolder, identityNumber)`. If `isValid=true`, set `Verified=true`, `VerifiedAt=DateTime.UtcNow`, save, call `paymentService.ResumeHeldSettlementsAsync`. If `isValid=false`, keep `Verified=false`, return response indicating failure. Store `IdentityNumber` on entity.
  - **Add `UpdateBankAccountAsync`**: Accept `UpdateBankAccountRequestDto` + userId. Check account exists. Check `HasPendingOrProcessingSettlementsAsync` — block if true. Update all fields (BankName, BankCode, AccountNumber, AccountHolder, IdentityNumber). Reset `Verified=false`, `VerifiedAt=null`. Call Toss `VerifyBankAccountHolderRealNameAsync`. If valid, set `Verified=true`. Call `ResumeHeldSettlementsAsync` on success.
  - **Remove `RequestVerificationAsync`**: Delete entirely (1원 인증 코드 발급)
  - **Remove `ConfirmVerificationAsync`**: Delete entirely (코드 검증)
  - **Remove `IMemoryCache` dependency**: No longer needed (verification attempt tracking removed)
  - **Remove provider resolution logic**: Delete `ResolveProvider()` method and all hybrid/toss/custom branching
  - **Update `IBankAccountService`**: Remove `RequestVerificationAsync`, `ConfirmVerificationAsync`. Add `UpdateBankAccountAsync`.
  - **Clean `TossPaymentsSettings`**: Remove `BankVerificationProvider`, `BankVerificationFallbackEnabled`, `VerificationCodeExpiryMinutes`, `MaxVerificationAttempts` properties. Keep payment/settlement settings.

  **Must NOT do**:
  - Do NOT modify `DeleteBankAccountAsync` logic (settlement check must remain)
  - Do NOT modify `PaymentService.ResumeHeldSettlementsAsync`
  - Do NOT change `GetMyBankAccountAsync`
  - Do NOT add retry logic to Toss API call (Polly handles at HTTP client level)

  **Recommended Agent Profile**:
  - **Category**: `deep`
    - Reason: Complex rewrite touching service, interface, config. Must preserve settlement integration. High-risk file.
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 2 (sequential after Wave 1)
  - **Blocks**: Task 5
  - **Blocked By**: Tasks 1, 2, 3

  **References**:
  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs:58-73` — DELETE flow with settlement check (reuse pattern for UPDATE)
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs:19-50` — RegisterBankAccountAsync (base to rewrite)
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs:170-227` — ConfirmVerificationAsync (ResumeHeldSettlementsAsync call pattern to preserve)

  **API/Type References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/IBankAccountService.cs` — interface to update
  - `TicketPlatFormServer/TicketPlatFormServer/Config/TossPaymentsSettings.cs` — config properties to remove (lines 56-64)
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/BankAccount/IBankAccountRepository.cs:18` — `HasPendingOrProcessingSettlementsAsync` to call in update flow

  **Acceptance Criteria**:
  - [ ] `RegisterBankAccountAsync` calls Toss verify API and sets Verified based on result
  - [ ] `UpdateBankAccountAsync` exists with settlement check + Toss verification
  - [ ] `RequestVerificationAsync` and `ConfirmVerificationAsync` removed from interface and implementation
  - [ ] `IMemoryCache` removed from constructor
  - [ ] `ResolveProvider` method removed
  - [ ] Dead config properties removed from `TossPaymentsSettings`
  - [ ] `dotnet build` → 0 errors

  **QA Scenarios:**
  ```
  Scenario: Service compiles without old verification methods
    Tool: Bash
    Steps:
      1. Run: `grep -c 'RequestVerificationAsync\|ConfirmVerificationAsync' TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs`
      2. Assert: 0 (removed)
      3. Run: `grep -c 'IMemoryCache' TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs`
      4. Assert: 0 (removed)
      5. Run: `grep -c 'ResolveProvider' TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs`
      6. Assert: 0 (removed)
      7. Run: `grep 'UpdateBankAccountAsync' TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/IBankAccountService.cs`
      8. Assert: Method found
      9. Run: `dotnet build TicketPlatFormServer/TicketPlatFormServer.sln`
      10. Assert: 0 errors
    Expected Result: Clean rewrite, all dead code removed, build passes
    Evidence: .sisyphus/evidence/task-4-service-rewrite.txt

  Scenario: ResumeHeldSettlements still called after verification
    Tool: Bash (grep)
    Steps:
      1. Run: `grep -c 'ResumeHeldSettlementsAsync' TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs`
      2. Assert: >= 1 (still called in register or update flow)
    Expected Result: Settlement resume preserved in new flow
    Evidence: .sisyphus/evidence/task-4-settlement-resume.txt
  ```

  **Commit**: YES (Wave 2 commit)
  - Message: `refactor(bank-account): replace 1-won verification with Toss direct API`
  - Files: `BankAccountService.cs`, `IBankAccountService.cs`, `TossPaymentsSettings.cs`
  - Pre-commit: `dotnet build TicketPlatFormServer/TicketPlatFormServer.sln`

---

- [ ] 5. BankAccountController Update

  **What to do**:
  - **Remove** `POST verify/request` endpoint (`RequestVerification` action)
  - **Remove** `POST verify/confirm` endpoint (`ConfirmVerification` action)
  - **Add** `PUT /api/bank-account` endpoint: calls `bankAccountService.UpdateBankAccountAsync`, accepts `UpdateBankAccountRequestDto`, returns `ApiResponse<BankAccountResponseDto>`
  - **Update** `RegisterBankAccount` action if DTO shape changed (should auto-resolve since DTO updated in Task 3)
  - Update using statements to remove deleted DTO references

  **Must NOT do**:
  - Do NOT change `GET me`, `DELETE` endpoints
  - Do NOT add business logic in controller

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 2 (after Task 4)
  - **Blocks**: Task 6
  - **Blocked By**: Task 4

  **References**:
  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/BankAccountController.cs:16-27` — POST register pattern (thin controller, delegate to service)
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/BankAccountController.cs:42-53` — DELETE pattern to follow for PUT

  **Acceptance Criteria**:
  - [ ] PUT endpoint exists at `/api/bank-account`
  - [ ] Old verify/request and verify/confirm endpoints removed
  - [ ] `dotnet build` → 0 errors

  **QA Scenarios:**
  ```
  Scenario: Controller has correct endpoints
    Tool: Bash
    Steps:
      1. Run: `grep -c 'verify/request\|verify/confirm' TicketPlatFormServer/TicketPlatFormServer/Controllers/BankAccountController.cs`
      2. Assert: 0 (removed)
      3. Run: `grep 'HttpPut' TicketPlatFormServer/TicketPlatFormServer/Controllers/BankAccountController.cs`
      4. Assert: Found (new PUT endpoint)
      5. Run: `dotnet build TicketPlatFormServer/TicketPlatFormServer.sln`
      6. Assert: 0 errors
    Expected Result: New PUT endpoint, old verify endpoints gone
    Evidence: .sisyphus/evidence/task-5-controller.txt
  ```

  **Commit**: YES (groups with Wave 2)
  - Message: `refactor(bank-account): replace 1-won verification with Toss direct API`
  - Files: `BankAccountController.cs`

---

- [ ] 6. Mobile Data Layer — DTOs, DataSource, Repository

  **What to do**:
  - **Update `RegisterBankAccountReqDto`**: Add `identityNumber` field (Freezed)
  - **Create `UpdateBankAccountReqDto`**: New Freezed DTO with bankName, bankCode, accountNumber, accountHolder, identityNumber
  - **Update `BankAccountRespDto`**: Add optional `identityNumber` field
  - **Delete** `RequestVerificationRespDto` (+ .g.dart + .freezed.dart)
  - **Delete** `VerifyAccountReqDto` (+ .g.dart + .freezed.dart)
  - **Delete** `VerifyAccountRespDto` (+ .g.dart + .freezed.dart)
  - **Update `BankAccountRemoteDataSource`**: Remove `requestVerification()` and `confirmVerification()`. Add `updateBankAccount(UpdateBankAccountReqDto)` using PUT to `ApiEndpoint.bankAccount`
  - **Update `BankAccountRepository` interface**: Remove `requestVerification()`, `confirmVerification()`. Add `updateBankAccount()`
  - **Update `BankAccountRepositoryImpl`**: Match interface changes
  - **Update `ApiEndpoint`**: No new constant needed (PUT on existing `/api/bank-account`)
  - Run `dart run build_runner build --delete-conflicting-outputs` after all changes

  **Must NOT do**:
  - Do NOT edit generated files directly
  - Do NOT change `safeApiCall` or Dio provider

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Multiple files across data layer, Freezed codegen, careful deletion of generated files
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES (within Wave 3, parallel with nothing — starts Wave 3)
  - **Parallel Group**: Wave 3
  - **Blocks**: Task 7
  - **Blocked By**: Tasks 3, 5

  **References**:
  **Pattern References**:
  - `ticket_platform_mobile/lib/features/bank_account/data/dto/request/register_bank_account_req_dto.dart` — Freezed DTO pattern to follow
  - `ticket_platform_mobile/lib/features/bank_account/data/datasources/bank_account_remote_data_source.dart` — datasource pattern with `safeApiCall`
  - `ticket_platform_mobile/lib/features/bank_account/data/repositories/bank_account_repository_impl.dart` — repository impl pattern

  **Acceptance Criteria**:
  - [ ] `RegisterBankAccountReqDto` has `identityNumber` field
  - [ ] `UpdateBankAccountReqDto` exists (Freezed)
  - [ ] Old verification DTOs/datasource methods removed
  - [ ] `dart run build_runner build --delete-conflicting-outputs` succeeds

  **QA Scenarios:**
  ```
  Scenario: Data layer updated and builds
    Tool: Bash
    Steps:
      1. Run: `grep -r 'requestVerification\|confirmVerification' ticket_platform_mobile/lib/features/bank_account/data/ --include='*.dart' | grep -v '.g.dart' | grep -v '.freezed.dart'`
      2. Assert: No matches
      3. Run: `cd ticket_platform_mobile && dart run build_runner build --delete-conflicting-outputs`
      4. Assert: Exit code 0
    Expected Result: Clean data layer, build_runner passes
    Evidence: .sisyphus/evidence/task-6-mobile-data.txt

  Scenario: Old generated files cleaned up
    Tool: Bash
    Steps:
      1. Run: `test ! -f ticket_platform_mobile/lib/features/bank_account/data/dto/response/request_verification_resp_dto.dart && echo DELETED`
      2. Assert: DELETED
      3. Run: `test ! -f ticket_platform_mobile/lib/features/bank_account/data/dto/response/verify_account_resp_dto.dart && echo DELETED`
      4. Assert: DELETED
    Expected Result: All old DTO files and their generated counterparts removed
    Evidence: .sisyphus/evidence/task-6-deleted-files.txt
  ```

  **Commit**: YES (Wave 3 commit)
  - Message: `refactor(bank-account): simplify mobile verification flow and add account update`
  - Files: All `data/` layer files in bank_account feature

---

- [ ] 7. Mobile Domain + ViewModel — Entities, UseCases, State

  **What to do**:
  - **Update `BankAccountEntity`**: Add `String? identityNumber` field (Freezed)
  - **Delete** `request_verification_usecase.dart` and `confirm_verification_usecase.dart`
  - **Create** `update_bank_account_usecase.dart`: single `call()` method, delegates to repository
  - **Update `BankAccountRepository` (domain interface)**: Remove `requestVerification()`, `confirmVerification()`. Add `updateBankAccount()`
  - **Simplify `BankAccountViewModel`**:
    - Remove `Timer? _timer` and `_startTimer()` method
    - Remove `requestVerification()` and `confirmVerification()` methods
    - Simplify `VerificationState` enum: `{ idle, verifying, verified, error }` (remove `requesting`, `codeInput`)
    - Simplify `BankAccountState`: remove `expiresAt` and `remainingSeconds`
    - Update `register()` method: after register call, if bankAccount.verified == true → state = verified
    - Add `updateAccount()` method: calls update usecase, updates state
  - **Update `bank_account_providers_di.dart`**: Remove `requestVerificationUsecaseProvider`, `confirmVerificationUsecaseProvider`. Add `updateBankAccountUsecaseProvider`
  - Run `dart run build_runner build --delete-conflicting-outputs`

  **Must NOT do**:
  - Do NOT change `GetBankAccountUsecase` or `DeleteBankAccountUsecase`
  - Do NOT change `RegisterBankAccountUsecase` signature (update input params to include identityNumber)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Multiple domain files + viewmodel state machine simplification. Must preserve Riverpod codegen patterns.
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (depends on Task 6)
  - **Parallel Group**: Wave 3 (sequential after Task 6)
  - **Blocks**: Task 8
  - **Blocked By**: Task 6

  **References**:
  **Pattern References**:
  - `ticket_platform_mobile/lib/features/bank_account/presentation/viewmodels/bank_account_viewmodel.dart` — current viewmodel to simplify
  - `ticket_platform_mobile/lib/features/bank_account/domain/usecases/register_bank_account_usecase.dart` — usecase pattern to follow for update
  - `ticket_platform_mobile/lib/features/bank_account/presentation/providers/bank_account_providers_di.dart` — DI wiring pattern

  **Acceptance Criteria**:
  - [ ] `BankAccountEntity` has `identityNumber` field
  - [ ] Old usecases deleted
  - [ ] New `UpdateBankAccountUsecase` exists
  - [ ] `VerificationState` enum simplified (no `requesting` or `codeInput`)
  - [ ] Timer logic removed from viewmodel
  - [ ] `dart run build_runner build` succeeds

  **QA Scenarios:**
  ```
  Scenario: ViewModel simplified and builds
    Tool: Bash
    Steps:
      1. Run: `grep -c 'Timer' ticket_platform_mobile/lib/features/bank_account/presentation/viewmodels/bank_account_viewmodel.dart`
      2. Assert: 0 (timer removed)
      3. Run: `grep -c 'codeInput\|requesting' ticket_platform_mobile/lib/features/bank_account/presentation/viewmodels/bank_account_viewmodel.dart`
      4. Assert: 0 (old states removed)
      5. Run: `test ! -f ticket_platform_mobile/lib/features/bank_account/domain/usecases/request_verification_usecase.dart && echo DELETED`
      6. Assert: DELETED
      7. Run: `cd ticket_platform_mobile && dart run build_runner build --delete-conflicting-outputs`
      8. Assert: Exit code 0
    Expected Result: Simplified viewmodel, old usecases gone, codegen passes
    Evidence: .sisyphus/evidence/task-7-domain-viewmodel.txt
  ```

  **Commit**: YES (groups with Wave 3)
  - Message: `refactor(bank-account): simplify mobile verification flow and add account update`
  - Files: All `domain/` and `presentation/viewmodels/` files in bank_account feature

---

- [ ] 8. Mobile UI — Register, Detail, Router Cleanup

  **What to do**:
  - **Update `BankAccountRegisterView`**:
    - Add 생년월일 input field: `AppTextField(label: '생년월일', hintText: 'YYMMDD (6자리)', keyboardType: TextInputType.number, inputFormatters: [FilteringTextInputFormatter.digitsOnly, LengthLimitingTextInputFormatter(6)])`
    - Update `_submit()`: pass `identityNumber` to `register()` call
    - After register, if verified → navigate to `bankAccountDetail` (not verify page)
    - Support "edit mode": accept optional existing account data, pre-fill fields for update flow
  - **Update `BankAccountDetailView`**:
    - Add "계좌 변경" button (alongside existing "계좌 삭제")
    - "계좌 변경" navigates to RegisterView in edit mode with pre-filled data
    - Show identityNumber (masked: `**0101`) if available
  - **Delete `BankAccountVerifyView`** (entire file)
  - **Update `app_router.dart`**: Remove `bankAccountVerify` route. Update register route to accept optional `extra` param for edit mode
  - **Update `app_router_path.dart`**: Remove `bankAccountVerify` path enum value
  - **Run `flutter analyze`** to verify no errors

  **Must NOT do**:
  - Do NOT change `BankSelectionBottomSheet` (reuse as-is)
  - Do NOT change shared widgets (`AppButton`, `AppTextField`)
  - Do NOT hardcode colors or spacing (use AppColors/AppSpacing tokens)

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
    - Reason: UI changes, form layout, navigation flow. Must match existing design system.
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (depends on Task 7)
  - **Parallel Group**: Wave 3 (final task in wave)
  - **Blocks**: FINAL
  - **Blocked By**: Task 7

  **References**:
  **Pattern References**:
  - `ticket_platform_mobile/lib/features/bank_account/presentation/views/bank_account_register_view.dart` — current register form layout to extend
  - `ticket_platform_mobile/lib/features/bank_account/presentation/views/bank_account_detail_view.dart` — detail view to add update button
  - `ticket_platform_mobile/lib/core/router/app_router.dart` — route definition pattern
  - `ticket_platform_mobile/lib/core/router/app_router_path.dart` — path enum to update

  **External References**:
  - `ticket_platform_mobile/lib/shared/widgets/app_text_field.dart` — reuse for identityNumber field
  - `ticket_platform_mobile/lib/core/enums/bank_type.dart` — bank selection enum (unchanged)

  **Acceptance Criteria**:
  - [ ] RegisterView has 생년월일 input field
  - [ ] DetailView has 계좌 변경 button
  - [ ] VerifyView deleted
  - [ ] bankAccountVerify route removed from router
  - [ ] `flutter analyze` → no issues

  **QA Scenarios:**
  ```
  Scenario: Register view has identityNumber field and verify view removed
    Tool: Bash
    Steps:
      1. Run: `grep -c 'identityNumber\|생년월일\|YYMMDD' ticket_platform_mobile/lib/features/bank_account/presentation/views/bank_account_register_view.dart`
      2. Assert: >= 1 (field present)
      3. Run: `test ! -f ticket_platform_mobile/lib/features/bank_account/presentation/views/bank_account_verify_view.dart && echo DELETED`
      4. Assert: DELETED
      5. Run: `grep -c 'bankAccountVerify' ticket_platform_mobile/lib/core/router/app_router.dart`
      6. Assert: 0 (route removed)
      7. Run: `cd ticket_platform_mobile && flutter analyze`
      8. Assert: No issues found
    Expected Result: New field present, old view gone, analyze clean
    Evidence: .sisyphus/evidence/task-8-mobile-ui.txt

  Scenario: Detail view has update button
    Tool: Bash
    Steps:
      1. Run: `grep -c '계좌 변경' ticket_platform_mobile/lib/features/bank_account/presentation/views/bank_account_detail_view.dart`
      2. Assert: >= 1 (button present)
    Expected Result: Update button exists in detail view
    Evidence: .sisyphus/evidence/task-8-detail-update.txt
  ```

  **Commit**: YES (Wave 3 commit)
  - Message: `refactor(bank-account): simplify mobile verification flow and add account update`
  - Files: RegisterView, DetailView, VerifyView (deleted), app_router.dart, app_router_path.dart

---

## Final Verification Wave (MANDATORY — after ALL implementation tasks)

> 4 review agents run in PARALLEL. ALL must APPROVE. Rejection → fix → re-run.

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Read the plan end-to-end. For each "Must Have": verify implementation exists (read file, curl endpoint, run command). For each "Must NOT Have": search codebase for forbidden patterns — reject with file:line if found. Check evidence files exist in .sisyphus/evidence/. Compare deliverables against plan.
  Output: `Must Have [N/N] | Must NOT Have [N/N] | Tasks [N/N] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  Run `dotnet build` + `flutter analyze`. Review all changed files for: `as any`/`@ts-ignore`, empty catches, console.log in prod, commented-out code, unused imports. Check AI slop: excessive comments, over-abstraction, generic names. Verify Korean comment style preserved in backend.
  Output: `Build [PASS/FAIL] | Analyze [PASS/FAIL] | Files [N clean/N issues] | VERDICT`

- [ ] F3. **Real Manual QA** — `unspecified-high`
  Start from clean state. Execute EVERY QA scenario from EVERY task — follow exact steps, capture evidence. Test cross-task integration (register → verify → update → re-verify → detail view). Test edge cases: empty identityNumber, wrong format, pending settlement blocking update. Save to `.sisyphus/evidence/final-qa/`.
  Output: `Scenarios [N/N pass] | Integration [N/N] | Edge Cases [N tested] | VERDICT`

- [ ] F4. **Scope Fidelity Check** — `deep`
  For each task: read "What to do", read actual diff (git log/diff). Verify 1:1 — everything in spec was built (no missing), nothing beyond spec was built (no creep). Check "Must NOT do" compliance. Detect cross-task contamination. Flag unaccounted changes.
  Output: `Tasks [N/N compliant] | Contamination [CLEAN/N issues] | Unaccounted [CLEAN/N files] | VERDICT`

---

## Commit Strategy

- **Wave 1**: `feat(bank-account): add identity_number column and Toss verify-holder-real-name API` — migration file, DBModel, TossPaymentsService, DTOs
- **Wave 2**: `refactor(bank-account): replace 1-won verification with Toss direct API` — BankAccountService, Controller, config cleanup
- **Wave 3**: `refactor(bank-account): simplify mobile verification flow and add account update` — all mobile changes

---

## Success Criteria

### Verification Commands
```bash
# Backend builds
dotnet build TicketPlatFormServer/TicketPlatFormServer.sln  # Expected: 0 errors, 0 warnings

# Mobile analyzes
cd ticket_platform_mobile && flutter analyze  # Expected: No issues found

# Build runner succeeds
cd ticket_platform_mobile && dart run build_runner build --delete-conflicting-outputs  # Expected: exit 0

# Old endpoints gone
grep -r "verify/request\|verify/confirm" TicketPlatFormServer/TicketPlatFormServer/Controllers/  # Expected: no matches

# Old mobile code gone
grep -r "requestVerification\|confirmVerification\|BankAccountVerifyView" ticket_platform_mobile/lib/ --include="*.dart" | grep -v ".g.dart" | grep -v ".freezed.dart"  # Expected: no matches
```

### Final Checklist
- [ ] All "Must Have" present
- [ ] All "Must NOT Have" absent
- [ ] Backend builds clean
- [ ] Mobile analyzes clean
- [ ] build_runner completes
