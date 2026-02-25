# 정산 계좌 인증 + 정산 완료 플로우 — Backend

## TL;DR

> **Quick Summary**: 판매자 계좌 등록/1원 인증 API와 구매 확정 이후 토스페이먼츠를 통한 정산 처리 배치 잡 및 정산 조회 API를 구현한다.
> 
> **Deliverables**:
> - DB: `on_hold` 정산 상태 시드 + BankAccount 스키마 확장
> - API: 계좌 CRUD + 1원 인증 + 정산 조회
> - Service: BankAccountService (CRUD + 인증) + SettlementProcessingService (배치 잡)
> - Toss: ITossPaymentsService 정산 송금 API 확장
> - 기존 수정: ReleaseEscrowAsync에 on_hold 로직 추가
> 
> **Estimated Effort**: Large
> **Parallel Execution**: YES - 4 waves
> **Critical Path**: Task 1 (DB) → Task 3 (계좌 서비스) → Task 5 (정산 서비스 확장) → Task 7 (정산 배치) → Task 9 (통합 검증)

---

## Context

### Original Request
판매자가 구매 확정 후 정산을 받기 위한 계좌 인증 기능과, 구매 확정 이후 에스크로 해제 → 정산 생성 → 토스페이먼츠 정산 위임 → 실제 지급 완료까지의 전체 마무리 플로우 구현. **(Backend 파트)**

### Interview Summary
**Key Decisions**:
- 정산 지급 방식: 토스페이먼츠 정산 위임 (Toss가 판매자 계좌로 직접 입금)
- 계좌 인증 방식: 1원 인증 (마이크로 입금 후 입금자명에 포함된 인증 코드 확인)
- 계좌 등록 시점: 자유 등록 (프로필에서 언제든 가능; 구매확정 시 계좌 미인증이어도 허용, 정산만 보류)
- 보류 정산 자동 재개: 계좌 인증 완료 시 on_hold 상태의 정산이 pending으로 자동 전환

**Research Findings**:
- 기존 DB 모델: `BankAccount` (verified 필드 존재), `Settlement` (bank_account_id nullable), `SettlementStatus` (pending/processing/completed/failed 시드됨)
- 기존 `ReleaseEscrowAsync`: verified 계좌 없으면 경고만 로그하고 bank_account_id=null로 Settlement 생성
- `TransactionAutoConfirmService`: BackgroundService 패턴으로 10분 주기 배치 잡 (정산 배치 잡 패턴 레퍼런스)
- `TossPaymentsService`: HttpClientFactory 기반 Toss API 호출 (confirm/get/cancel). 정산 API 없음
- `TossPaymentsSettings`: SecretKey, ClientKey, ApiBaseUrl, EscrowFeePercentage 포함

### Metis Review
- 1원 인증 코드 만료/재시도 정책 → 5분 만료, 최대 3회 재시도
- 정산 실패 시 재시도 → RetryCount 활용, 최대 3회
- 계좌 삭제 시 processing 정산 있으면 거부
- BankAccount에 인증 필드 추가 필요
- 토스 정산 위임 API 미확인 → 인터페이스 추상화로 교체 가능하게 설계

### 관련 Mobile 플랜
> Mobile 파트는 `.sisyphus/plans/settlement-mobile.md`에 별도 관리됩니다.
> Backend API가 선행되어야 Mobile이 연동 가능합니다.

---

## Work Objectives

### Core Objective
판매자 계좌 등록/1원 인증 API를 구축하고, 구매 확정 시 계좌 인증 여부에 따른 정산 보류(on_hold)/처리(pending) 분기 및 배치 잡을 통한 자동 정산 처리 파이프라인을 완성한다.

### Concrete Deliverables
- `BankAccountController` + `BankAccountService` + `BankAccountRepository` (계좌 CRUD + 1원 인증)
- `SettlementProcessingService` (배치 잡 — pending 정산을 토스 API로 처리)
- `ITossPaymentsService` 확장 (송금 요청, 송금 상태 조회)
- `SettlementController` + `SettlementService` (정산 조회 API)
- `on_hold` 정산 상태 + `ReleaseEscrowAsync` 수정
- DB 마이그레이션 + DTO 정의

### Definition of Done
- [ ] 계좌 등록 API (POST /api/bank-account) 동작
- [ ] 1원 인증 요청/확인 API 동작
- [ ] 구매 확정 시 인증 계좌 있으면 Settlement status=pending
- [ ] 구매 확정 시 인증 계좌 없으면 Settlement status=on_hold
- [ ] 계좌 인증 완료 시 on_hold → pending 자동 전환
- [ ] 배치 잡이 pending 정산을 처리
- [ ] 정산 조회 API (GET /api/settlement) 동작
- [ ] `dotnet build` 0 errors, 0 warnings

### Must Have
- 1원 인증을 통한 계좌 소유 확인
- 계좌 미인증 시 정산 보류 (on_hold)
- 인증 완료 시 보류 정산 자동 재개
- 정산 처리 배치 잡 (pending → processing → completed/failed)
- 정산 실패 시 자동 재시도 (최대 3회)

### Must NOT Have (Guardrails)
- 계좌 인증 없이 정산 처리되는 경로 금지
- 직접 DB 접근으로 정산 상태 변경 금지 (반드시 서비스 레이어 경유)
- Controller에 비즈니스 로직 금지
- Repository에서 DTO 반환 금지
- `Task.WhenAll`로 Repository 병렬 호출 금지
- 과도한 추상화/유틸리티 추출 금지 — 기존 패턴 따르기
- 하드코딩된 은행 코드 — 반드시 설정 또는 코드 테이블 사용

---

## Verification Strategy (MANDATORY)

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed.

### Test Decision
- **Infrastructure exists**: NO (backend test project 없음)
- **Automated tests**: NO
- **Primary verification**: Agent-Executed QA Scenarios per task

### QA Policy
- **Backend API**: Bash (curl) — 요청/응답 검증
- **Backend Build**: Bash (dotnet build) — 0 errors
- **Batch Jobs**: Bash (dotnet run + DB 조회) — 상태 변경 확인

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Foundation — DB + Types + Config):
├── Task 1: DB 마이그레이션 (on_hold 상태 + BankAccount 스키마 확장) [quick]
├── Task 2: Backend DTO 정의 (계좌 + 정산 API 계약) [quick]
└── Task 3-prep: TossPaymentsSettings 확장 [quick]

Wave 2 (Core Services):
├── Task 3: BankAccountRepository + BankAccountService [unspecified-high]
├── Task 4: BankAccountController [quick]
├── Task 5: ReleaseEscrowAsync 수정 + 정산 보류/재개 로직 [deep]
└── Task 6: ITossPaymentsService 정산 API 확장 [unspecified-high]

Wave 3 (Integration):
├── Task 7: SettlementProcessingService 배치 잡 [deep]
└── Task 8: 정산 조회 API [quick]

Wave 4 (Verification):
└── Task 9: Backend 통합 빌드 + API 스모크 테스트 [unspecified-high]

Wave FINAL (Review — 4 parallel):
├── Task F1: Plan compliance audit (oracle)
├── Task F2: Code quality review (unspecified-high)
├── Task F3: API QA (unspecified-high)
└── Task F4: Scope fidelity check (deep)

Critical Path: Task 1 → Task 3 → Task 5 → Task 7 → Task 9 → F1-F4
Max Concurrent: 4 (Wave 2)
```

### Dependency Matrix

| Task | Depends On | Blocks | Wave |
|------|-----------|--------|------|
| 1 | — | 3, 5, 6, 7 | 1 |
| 2 | — | 3, 4, 5, 6, 8 | 1 |
| 3-prep | — | 6 | 1 |
| 3 | 1, 2 | 4, 5, 7 | 2 |
| 4 | 2, 3 | 9 | 2 |
| 5 | 1, 3 | 7, 9 | 2 |
| 6 | 1, 2, 3-prep | 7 | 2 |
| 7 | 5, 6 | 9 | 3 |
| 8 | 2, 5 | 9 | 3 |
| 9 | 4, 5, 7, 8 | F1-F4 | 4 |
| F1-F4 | 9 | — | FINAL |

### Agent Dispatch Summary

- **Wave 1**: **3 tasks** — T1 → `quick`, T2 → `quick`, T3-prep → `quick`
- **Wave 2**: **4 tasks** — T3 → `unspecified-high`, T4 → `quick`, T5 → `deep`, T6 → `unspecified-high`
- **Wave 3**: **2 tasks** — T7 → `deep`, T8 → `quick`
- **Wave 4**: **1 task** — T9 → `unspecified-high`
- **FINAL**: **4 tasks** — F1 → `oracle`, F2 → `unspecified-high`, F3 → `unspecified-high`, F4 → `deep`

---

## TODOs

- [ ] 1. DB 마이그레이션 — on_hold 상태 시드 + BankAccount 스키마 확장

  **What to do**:
  - `settlement_statuses` 테이블에 `on_hold` 상태 추가 (id=1005, code='on_hold', name_ko='정산 보류', sort_order=5)
  - `bank_accounts` 테이블에 인증 관련 컬럼 추가:
    - `verification_code` VARCHAR(10) NULL — 1원 인증 코드
    - `verification_expires_at` DATETIME NULL — 인증 코드 만료 시각
    - `verified_at` DATETIME NULL — 인증 완료 시각
    - `bank_code` VARCHAR(10) NULL — 은행 코드 (토스 API용, 예: '088' = 신한)
  - `BankAccount.cs` DBModel에 새 필드 추가
  - `TicketContext.cs`에서 BankAccount 엔티티 매핑 확인/업데이트
  - 기존 시드 파일 패턴(`20260212_002_seed_settlement_statuses.sql`) 따를 것

  **Must NOT do**:
  - 기존 settlement_statuses의 id/code 변경 금지
  - bank_accounts 기존 컬럼 타입 변경 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: SQL 시드 파일 1개 + C# 모델 필드 추가 — 단순 반복 작업
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 2, 3-prep)
  - **Blocks**: Tasks 3, 5, 6, 7
  - **Blocked By**: None (can start immediately)

  **References**:

  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/database_history/migrations/20260212_002_seed_settlement_statuses.sql` — 기존 정산 상태 시드 패턴 (idempotent INSERT ... WHERE NOT EXISTS)
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/BankAccount.cs` — 현재 BankAccount 엔티티 (확장할 대상)
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/SettlementStatus.cs` — SettlementStatus 엔티티 구조

  **API/Type References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/TicketContext.cs:370` — BankAccount EF 매핑 위치

  **WHY Each Reference Matters**:
  - 시드 SQL은 반드시 idempotent 패턴(`WHERE NOT EXISTS`)을 따라야 함
  - BankAccount.cs는 nullable 타입 컨벤션(`bool?`, `string?`)을 따라야 함

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: on_hold 상태 시드 적용 확인
    Tool: Bash (mysql)
    Steps:
      1. 마이그레이션 SQL 실행
      2. SELECT * FROM settlement_statuses WHERE code = 'on_hold'
      3. Assert: id=1005, name_ko='정산 보류', is_active=1, sort_order=5
    Expected Result: on_hold 행이 정확히 1건 존재
    Failure Indicators: 행 0건 또는 중복
    Evidence: .sisyphus/evidence/task-1-on-hold-seed.txt

  Scenario: BankAccount 스키마 확장
    Tool: Bash (mysql)
    Steps:
      1. DESCRIBE bank_accounts
      2. Assert: verification_code, verification_expires_at, verified_at, bank_code 컬럼 존재 (모두 NULL 허용)
    Evidence: .sisyphus/evidence/task-1-bank-account-schema.txt

  Scenario: dotnet build 성공
    Tool: Bash
    Steps: dotnet build --project TicketPlatFormServer/TicketPlatFormServer.sln → 0 error(s)
    Evidence: .sisyphus/evidence/task-1-dotnet-build.txt
  ```

  **Commit**: YES
  - Message: `feat(settlement): add on_hold status seed + bank account schema migration`
  - Files: `database_history/migrations/`, `DBModel/BankAccount.cs`, `Repository/TicketContext.cs`
  - Pre-commit: `dotnet build --project TicketPlatFormServer/TicketPlatFormServer.sln`

- [ ] 2. Backend DTO 정의 — 계좌 + 정산 API 계약

  **What to do**:
  - `DTO/BankAccount/` 디렉터리 생성:
    - `RegisterBankAccountRequestDto` — bankName, accountNumber, accountHolder, bankCode
    - `BankAccountResponseDto` — id, bankName, accountNumber(마스킹), accountHolder, verified, verifiedAt
    - `VerifyAccountRequestDto` — code (1원 인증 코드)
    - `VerifyAccountResponseDto` — verified, message
    - `RequestVerificationResponseDto` — expiresAt, message
  - `DTO/Settlement/` 디렉터리 생성:
    - `SettlementResponseDto` — id, transactionId, amount, fee, netAmount, statusCode, statusName, scheduledAt, processedAt, failureReason
    - `SettlementListResponseDto` — settlements, totalCount, summary
  - 기존 DTO 패턴 따를 것 (record 또는 class, XML 한글 주석)

  **Must NOT do**:
  - 계좌번호 평문 노출 금지 — 마스킹 필수
  - 인증 코드(verification_code)를 Response에 포함 금지

  **Recommended Agent Profile**: `quick` | **Skills**: []
  **Parallelization**: Wave 1 | **Blocks**: Tasks 3, 4, 5, 6, 8 | **Blocked By**: None

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Payment/PaymentRequestDto.cs` — 기존 DTO record 패턴 + XML 주석
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Payment/TossPaymentResponseDto.cs` — Response DTO 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/BankAccount.cs` — Entity 필드 참조
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/Settlement.cs` — Settlement Entity 필드 참조

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: DTO 파일 생성 및 빌드
    Tool: Bash
    Steps:
      1. ls DTO/BankAccount/ → 5개 파일
      2. ls DTO/Settlement/ → 2개 파일
      3. dotnet build → 0 errors
    Evidence: .sisyphus/evidence/task-2-dto-build.txt

  Scenario: 인증 코드 미노출 확인
    Tool: Bash (grep)
    Steps: grep 'verification_code\|VerificationCode' DTO/BankAccount/BankAccountResponseDto.cs → 0줄
    Evidence: .sisyphus/evidence/task-2-no-code-leak.txt
  ```

  **Commit**: YES
  - Message: `feat(settlement): add bank account and settlement DTOs`
  - Files: `DTO/BankAccount/`, `DTO/Settlement/`

- [ ] 3-prep. TossPaymentsSettings 확장 — 정산 관련 설정 추가

  **What to do**:
  - `Config/TossPaymentsSettings.cs`에 정산 관련 설정 필드 추가:
    - `SettlementApiBaseUrl` (string) — 정산 API URL
    - `SettlementCallbackUrl` (string) — 정산 웹훅 URL
    - `MaxSettlementRetryCount` (int, default=3)
    - `SettlementProcessingIntervalMinutes` (int, default=5)
    - `VerificationCodeExpiryMinutes` (int, default=5)
    - `MaxVerificationAttempts` (int, default=3)
  - `appsettings.json`, `appsettings.Development.json` 설정 섹션 추가

  **Recommended Agent Profile**: `quick` | **Skills**: []
  **Parallelization**: Wave 1 | **Blocks**: Task 6 | **Blocked By**: None

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Config/TossPaymentsSettings.cs` — 확장 대상
  - `TicketPlatFormServer/TicketPlatFormServer/Program.cs:170-172` — 설정 바인딩

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: 설정 필드 존재 + 빌드
    Tool: Bash
    Steps: grep 'MaxSettlementRetryCount' Config/TossPaymentsSettings.cs → 1건, dotnet build → 0 errors
    Evidence: .sisyphus/evidence/task-3prep-settings.txt
  ```
  **Commit**: YES | Message: `chore(config): extend TossPaymentsSettings for settlement`

- [ ] 3. BankAccountRepository + BankAccountService — 계좌 CRUD + 1원 인증

  **What to do**:
  - `Repository/BankAccount/IBankAccountRepository.cs` 인터페이스:
    - `GetBankAccountByUserIdAsync(long userId)` → BankAccount?
    - `GetVerifiedBankAccountByUserIdAsync(long userId)` → BankAccount?
    - `CreateBankAccountAsync(BankAccount)` → BankAccount
    - `UpdateBankAccountAsync(BankAccount)` → Task
    - `DeleteBankAccountAsync(long id)` → Task
    - `HasPendingOrProcessingSettlementsAsync(long bankAccountId)` → bool
  - `Repository/BankAccount/BankAccountRepository.cs` — EF Core 구현
  - `Services/BankAccount/IBankAccountService.cs` 인터페이스:
    - `RegisterBankAccountAsync(RegisterBankAccountRequestDto, long userId)` → BankAccountResponseDto
    - `GetMyBankAccountAsync(long userId)` → BankAccountResponseDto?
    - `DeleteBankAccountAsync(long userId)` → Task
    - `RequestVerificationAsync(long userId)` → RequestVerificationResponseDto
    - `ConfirmVerificationAsync(VerifyAccountRequestDto, long userId)` → VerifyAccountResponseDto
  - `Services/BankAccount/BankAccountService.cs` 구현:
    - 등록: 유저당 1개 계좌, 기존 있으면 AppException
    - 1원 인증 요청: 4자리 랜덤 코드 → DB 저장 → (추후) 토스 API 1원 송금
    - 인증 확인: 코드 일치 + 만료 확인 → verified=true
    - **인증 완료 시**: on_hold 정산들 → pending 전환 + bank_account_id 설정
    - 삭제: processing 정산 있으면 AppException
    - 계좌번호 마스킹 (DTO 매핑 시)
  - `Program.cs` DI 등록

  **Must NOT do**: Repository에 비즈니스 로직 금지, DTO 반환 금지, Task.WhenAll 금지

  **Recommended Agent Profile**: `unspecified-high` | **Skills**: []
  **Parallelization**: Wave 2 | **Blocks**: Tasks 4, 5, 7 | **Blocked By**: Tasks 1, 2

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Payment/PaymentRepository.cs` — EF Core 리포지토리 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Payment/IPaymentRepository.cs` — 인터페이스 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs:527-561` — 기존 Settlement 생성 + 계좌 조회
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/BankAccount.cs` — 엔티티
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/Settlement.cs` — Settlement 엔티티
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/BankAccount/` — Task 2 DTO들

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: 계좌 등록 후 조회
    Tool: Bash (curl)
    Steps:
      1. POST /api/bank-account {"bankName":"신한","accountNumber":"110123456789","accountHolder":"홍길동","bankCode":"088"}
      2. Assert: 201, verified == false
      3. GET /api/bank-account/me → 200, accountNumber 마스킹
    Evidence: .sisyphus/evidence/task-3-register-query.txt

  Scenario: 중복 등록 거부
    Tool: Bash (curl)
    Steps: POST /api/bank-account (동일 유저 또 등록) → 409 Conflict
    Evidence: .sisyphus/evidence/task-3-duplicate-reject.txt

  Scenario: 1원 인증 요청 + 확인
    Tool: Bash (curl + mysql)
    Steps:
      1. POST /api/bank-account/verify/request → 200, expiresAt 존재
      2. DB에서 verification_code 조회
      3. POST /api/bank-account/verify/confirm {"code":"{code}"} → 200, verified==true
    Evidence: .sisyphus/evidence/task-3-verification-flow.txt

  Scenario: 만료 코드 거부
    Tool: Bash (curl + mysql)
    Steps:
      1. DB에서 verification_expires_at 과거로 수정
      2. POST /api/bank-account/verify/confirm → 400 (만료 메시지)
    Evidence: .sisyphus/evidence/task-3-expired-code.txt
  ```
  **Commit**: YES (groups with Task 4) | Message: `feat(bank-account): add CRUD + 1원 verification`

- [ ] 4. BankAccountController — API 엔드포인트

  **What to do**:
  - `Controllers/BankAccountController.cs`:
    - `POST /api/bank-account` — 계좌 등록
    - `GET /api/bank-account/me` — 내 계좌 조회
    - `DELETE /api/bank-account` — 계좌 삭제
    - `POST /api/bank-account/verify/request` — 1원 인증 요청
    - `POST /api/bank-account/verify/confirm` — 인증 코드 확인
  - `[Authorize]` 적용, JWT에서 userId 추출
  - Swagger XML 주석 (한글)
  - `ApiResponse<T>` 통일 응답 형식

  **Must NOT do**: Controller에 비즈니스 로직 금지

  **Recommended Agent Profile**: `quick` | **Skills**: []
  **Parallelization**: Wave 2 | **Blocks**: Task 9 | **Blocked By**: Tasks 2, 3

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/PaymentController.cs` — 컨트롤러 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/IBankAccountService.cs` — Task 3 인터페이스

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: 전체 엔드포인트 스모크
    Tool: Bash (curl)
    Steps: POST → 201, GET → 200, verify/request → 200, verify/confirm → 200, DELETE → 200
    Evidence: .sisyphus/evidence/task-4-endpoints-smoke.txt

  Scenario: 미인증 요청 시 401
    Tool: Bash (curl)
    Steps: GET /api/bank-account/me (Authorization 없이) → 401
    Evidence: .sisyphus/evidence/task-4-unauthorized.txt
  ```
  **Commit**: YES (groups with Task 3) | Message: `feat(bank-account): add BankAccountController`

- [ ] 5. ReleaseEscrowAsync 수정 — 정산 보류/재개 로직

  **What to do**:
  - `PaymentService.ReleaseEscrowAsync` (lines 527-561) 수정:
    - 기존: `defaultBankAccount == null`이면 경고 로그 + status=pending
    - **변경**: `defaultBankAccount == null` → status=**on_hold** (bank_account_id=null)
    - 인증 계좌 있으면: status=pending + bank_account_id 설정 (기존 동작 유지)
  - `IPaymentService`에 새 메서드 추가:
    - `ResumeHeldSettlementsAsync(long sellerId, long bankAccountId)` — on_hold → pending
  - `PaymentService`에 구현:
    - seller의 on_hold Settlement 모두 조회
    - StatusId = pending, BankAccountId = bankAccountId, ScheduledAt = 현재+1일
  - **중요**: BankAccountService.ConfirmVerificationAsync에서 이 메서드 호출 (Task 3과 연계)

  **Must NOT do**:
  - ReleaseEscrowAsync의 기존 정상 흐름 (에스크로 해제, 트랜잭션 확정, 티켓 이전) 변경 금지
  - on_hold 정산을 직접 처리하는 코드 금지 (재개 시에만 pending으로 전환)

  **Recommended Agent Profile**: `deep`
    - Reason: 기존 복잡한 트랜잭션 로직 수정 — 세심한 컨텍스트 이해 필요
  - **Skills**: []

  **Parallelization**: Wave 2 | **Blocks**: Tasks 7, 9 | **Blocked By**: Tasks 1, 3

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs:377-575` — ReleaseEscrowAsync 전체
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs:527-561` — Settlement 생성 부분 (핵심 수정점)
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/IPaymentService.cs` — 인터페이스 확장 대상

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: 미인증 판매자 구매확정 → on_hold
    Tool: Bash (curl + mysql)
    Preconditions: 판매자 계좌 미등록, 결제 완료 거래
    Steps:
      1. POST /api/chat/confirm-purchase (구매자 JWT)
      2. DB: SELECT s.*, ss.code FROM settlements s JOIN settlement_statuses ss ON s.status_id=ss.id WHERE s.transaction_id={tid}
      3. Assert: ss.code = 'on_hold', s.bank_account_id IS NULL
    Evidence: .sisyphus/evidence/task-5-on-hold-creation.txt

  Scenario: 인증 판매자 구매확정 → pending
    Tool: Bash (curl + mysql)
    Preconditions: 판매자 계좌 인증 완료
    Steps:
      1. POST /api/chat/confirm-purchase
      2. DB → ss.code = 'pending', s.bank_account_id IS NOT NULL
    Evidence: .sisyphus/evidence/task-5-pending-with-account.txt

  Scenario: 계좌 인증 후 on_hold 자동 재개
    Tool: Bash (curl + mysql)
    Preconditions: on_hold 정산 존재, 계좌 미인증
    Steps:
      1. 계좌 인증 완료 (Task 3 인증 플로우 실행)
      2. DB → 기존 on_hold 정산이 pending으로 변경, bank_account_id 설정
    Evidence: .sisyphus/evidence/task-5-auto-resume.txt
  ```
  **Commit**: YES | Message: `feat(settlement): modify ReleaseEscrowAsync for on_hold + auto-resume`

- [ ] 6. ITossPaymentsService 정산 API 확장 — 송금/조회

  **What to do**:
  - `ITossPaymentsService.cs`에 새 메서드:
    - `RequestTransferAsync(TransferRequestDto)` → TransferResponseDto
    - `GetTransferStatusAsync(string transferId)` → TransferStatusDto
  - `TossPaymentsService.cs` 구현: 기존 HTTP 클라이언트 패턴 재사용
  - 새 DTO: `TransferRequestDto`, `TransferResponseDto`, `TransferStatusDto` in `DTO/Payment/`
  - **NOTE**: 인터페이스 추상화로 향후 다른 송금 서비스 교체 가능하게

  **Must NOT do**: 기존 Confirm/Cancel/Get 메서드 변경 금지

  **Recommended Agent Profile**: `unspecified-high` | **Skills**: []
  **Parallelization**: Wave 2 | **Blocks**: Task 7 | **Blocked By**: Tasks 1, 2, 3-prep

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/ITossPaymentsService.cs` — 현재 인터페이스
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/TossPaymentsService.cs` — HTTP 패턴, Basic Auth
  - `TicketPlatFormServer/TicketPlatFormServer/Config/TossPaymentsSettings.cs` — 설정

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: 인터페이스 + 빌드
    Tool: Bash
    Steps: grep 'RequestTransferAsync' ITossPaymentsService.cs + dotnet build → 0 errors
    Evidence: .sisyphus/evidence/task-6-toss-extension.txt
  ```
  **Commit**: YES | Message: `feat(toss): extend ITossPaymentsService with transfer API`

- [ ] 7. SettlementProcessingService — 정산 배치 잡

  **What to do**:
  - `Services/BackgroundServices/SettlementProcessingService.cs` — BackgroundService 상속:
    - `TransactionAutoConfirmService` 패턴 복사: CreateScope, foreach, try/catch per item
    - 주기: `TossPaymentsSettings.SettlementProcessingIntervalMinutes` (5분)
    - `ProcessPendingSettlementsAsync`:
      1. ScheduledAt <= now + status=pending인 Settlement 조회
      2. status → processing
      3. `RequestTransferAsync` 호출
      4. 성공 → completed+ProcessedAt | 실패 → RetryCount++, max 넘으면 failed+FailureReason
  - `Program.cs`에 `AddHostedService<SettlementProcessingService>()`

  **Must NOT do**: on_hold/completed/failed 정산 처리 금지, Task.WhenAll 금지

  **Recommended Agent Profile**: `deep` | **Skills**: []
  **Parallelization**: Wave 3 | **Blocks**: Task 9 | **Blocked By**: Tasks 5, 6

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BackgroundServices/TransactionAutoConfirmService.cs` — **핵심 패턴** (BackgroundService, CreateScope, 로깅)
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/Settlement.cs` — RetryCount, FailureReason, ProcessedAt
  - `TicketPlatFormServer/TicketPlatFormServer/Config/TossPaymentsSettings.cs` — MaxSettlementRetryCount

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: 배치 등록 + 빌드
    Tool: Bash
    Steps: grep 'AddHostedService<SettlementProcessingService>' Program.cs + dotnet build → 0 errors
    Evidence: .sisyphus/evidence/task-7-batch-build.txt

  Scenario: pending 정산 처리
    Tool: Bash (mysql + app logs)
    Preconditions: pending+scheduledAt 과거인 Settlement 1건
    Steps: 앱 실행 후 로그에서 처리 확인, DB → status 변경
    Evidence: .sisyphus/evidence/task-7-batch-processing.txt

  Scenario: 실패 재시도
    Tool: Bash (mysql)
    Steps: RetryCount 증가 확인, max 넘으면 status=failed+FailureReason
    Evidence: .sisyphus/evidence/task-7-retry-logic.txt
  ```
  **Commit**: YES | Message: `feat(settlement): add SettlementProcessingService batch job`

- [ ] 8. 정산 조회 API — 판매자용

  **What to do**:
  - `Services/Settlement/ISettlementService.cs` + `SettlementService.cs`:
    - `GetMySettlementsAsync(long sellerId)` → SettlementListResponseDto
    - `GetSettlementByIdAsync(long id, long sellerId)` → SettlementResponseDto?
  - `Controllers/SettlementController.cs`:
    - `GET /api/settlement` — 목록
    - `GET /api/settlement/{id}` — 상세
  - `Program.cs` DI 등록

  **Recommended Agent Profile**: `quick` | **Skills**: []
  **Parallelization**: Wave 3 | **Blocks**: Task 9 | **Blocked By**: Tasks 2, 5

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Sell/SellService.cs:563+` — 정산 상태 매핑
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/PaymentController.cs` — Controller 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Settlement/` — Task 2 DTO

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: 정산 목록 조회
    Tool: Bash (curl)
    Steps: GET /api/settlement (판매자 JWT) → 200, settlements[] 존재
    Evidence: .sisyphus/evidence/task-8-settlement-list.txt

  Scenario: 타인 정산 접근 불가
    Tool: Bash (curl)
    Steps: GET /api/settlement/{other} (다른 판매자 JWT) → 404/403
    Evidence: .sisyphus/evidence/task-8-access-denied.txt
  ```
  **Commit**: YES | Message: `feat(settlement): add settlement query API`

- [ ] 9. Backend 통합 빌드 + API 스모크 테스트

  **What to do**:
  - `dotnet build --project TicketPlatFormServer/TicketPlatFormServer.sln` → 0 errors, 0 warnings
  - `dotnet run` 후 전체 API 스모크 테스트:
    - POST /api/bank-account → 201
    - GET /api/bank-account/me → 200
    - POST /api/bank-account/verify/request → 200
    - POST /api/bank-account/verify/confirm → 200
    - GET /api/settlement → 200
    - Swagger UI에서 새 엔드포인트 확인

  **Recommended Agent Profile**: `unspecified-high` | **Skills**: []
  **Parallelization**: Wave 4 | **Blocks**: F1-F4 | **Blocked By**: Tasks 4, 5, 7, 8

  **QA Scenarios (MANDATORY):**
  ```
  Scenario: 전체 빌드 + 엔드포인트 스모크
    Tool: Bash (dotnet + curl)
    Steps: dotnet build → 0 errors, dotnet run + curl 전체 엔드포인트
    Evidence: .sisyphus/evidence/task-9-integration-smoke.txt
  ```
  **Commit**: NO (검증만)

---

## Final Verification Wave (MANDATORY — after ALL implementation tasks)

> 4 review agents run in PARALLEL. ALL must APPROVE. Rejection → fix → re-run.

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Read the plan end-to-end. For each "Must Have": verify implementation exists. For each "Must NOT Have": search codebase for forbidden patterns. Check evidence files.
  Output: `Must Have [N/N] | Must NOT Have [N/N] | Tasks [N/N] | VERDICT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  Run `dotnet build`. Review changed files for: empty catches, Console.Write in prod, commented-out code, unused imports. Check AI slop.
  Output: `Build [PASS/FAIL] | Files [N clean/N issues] | VERDICT`

- [ ] F3. **API QA** — `unspecified-high`
  Start from clean state. Execute EVERY QA scenario. Test edge cases: empty state, invalid input. Save to `.sisyphus/evidence/final-qa/`.
  Output: `Scenarios [N/N pass] | Edge Cases [N tested] | VERDICT`

- [ ] F4. **Scope Fidelity Check** — `deep`
  For each task: read spec, read actual diff. Verify 1:1. Check "Must NOT do" compliance. Flag unaccounted changes.
  Output: `Tasks [N/N compliant] | Unaccounted [CLEAN/N files] | VERDICT`

---

## Commit Strategy

| Tasks | Commit Message | Files |
|-------|---------------|-------|
| 1 | `feat(settlement): add on_hold status seed + bank account schema migration` | `database_history/migrations/`, `DBModel/BankAccount.cs`, `Repository/TicketContext.cs` |
| 2 | `feat(settlement): add bank account and settlement DTOs` | `DTO/BankAccount/`, `DTO/Settlement/` |
| 3-prep | `chore(config): extend TossPaymentsSettings for settlement` | `Config/TossPaymentsSettings.cs`, `appsettings*.json` |
| 3, 4 | `feat(bank-account): add CRUD + 1원 verification API` | `Repository/BankAccount/`, `Services/BankAccount/`, `Controllers/BankAccountController.cs`, `Program.cs` |
| 5 | `feat(settlement): modify ReleaseEscrowAsync for on_hold + auto-resume` | `Services/Payment/PaymentService.cs`, `Services/Payment/IPaymentService.cs` |
| 6 | `feat(toss): extend ITossPaymentsService with transfer API` | `Services/Payment/TossPaymentsService.cs`, `Services/Payment/ITossPaymentsService.cs`, `DTO/Payment/Transfer*.cs` |
| 7 | `feat(settlement): add SettlementProcessingService batch job` | `Services/BackgroundServices/SettlementProcessingService.cs`, `Program.cs` |
| 8 | `feat(settlement): add settlement query API` | `Controllers/SettlementController.cs`, `Services/Settlement/` |

---

## Success Criteria

### Verification Commands
```bash
dotnet build --project TicketPlatFormServer/TicketPlatFormServer.sln  # Expected: 0 errors, 0 warnings
curl -X POST http://localhost:5224/api/bank-account -H "Authorization: Bearer {token}" -d '...' # Expected: 201
curl -X POST http://localhost:5224/api/bank-account/verify/request -H "Authorization: Bearer {token}" # Expected: 200
curl -X GET http://localhost:5224/api/settlement -H "Authorization: Bearer {token}" # Expected: 200
```

### Final Checklist
- [ ] All "Must Have" present (5/5)
- [ ] All "Must NOT Have" absent (7/7)
- [ ] `dotnet build` passes
- [ ] on_hold → pending auto-transition works
- [ ] Settlement batch job processes pending settlements
- [ ] All API endpoints respond correctly
