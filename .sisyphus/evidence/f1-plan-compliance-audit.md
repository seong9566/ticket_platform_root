# F1 Plan Compliance Audit
**플랜:** `toss-bank-account-gap-analysis`  
**감사일:** 2026-02-25  
**감사자:** oracle (F1 Final Wave)  
**Branch:** feat/notification-fcm

---

## 요약

| 범주 | 결과 |
|------|------|
| Must Have | **10/10** |
| Must NOT Have | **6/6** |
| **VERDICT** | ✅ **PASS** |

---

## Must Have 항목 감사 (10/10)

### MH-1. Provider 추상화 인터페이스 (`IBankAccountVerificationProvider`) 존재
**결과: ✅ 충족**

- **파일:** `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/IBankAccountVerificationProvider.cs`
- **근거:**
  - Line 58: `public interface IBankAccountVerificationProvider`
  - Line 61: `string Name { get; }` — Provider 식별자 프로퍼티
  - Line 68: `Task<VerificationRequestResult> RequestAsync(VerificationRequestInput input, CancellationToken ct = default);`
  - Line 75: `Task<VerificationConfirmResult> ConfirmAsync(VerificationConfirmInput input, CancellationToken ct = default);`
  - 인터페이스 계약 완전: 요청 입력/결과 레코드(`VerificationRequestInput`, `VerificationRequestResult`, `VerificationConfirmInput`, `VerificationConfirmResult`) 동일 파일에 정의됨.

---

### MH-2. Custom / Toss / Hybrid 3개 Provider 구현체 존재
**결과: ✅ 충족**

| Provider | 파일 | Name 속성 위치 |
|----------|------|---------------|
| Custom | `Services/BankAccount/CustomBankVerificationProvider.cs` | Line 15: `public string Name => "CUSTOM";` |
| Toss | `Services/BankAccount/TossBankVerificationProvider.cs` | Line 15: `public string Name => "TOSS";` |
| Hybrid | `Services/BankAccount/HybridBankVerificationProvider.cs` | Line 15: `public string Name => "HYBRID";` |

- **CustomBankVerificationProvider:** 4자리 랜덤 코드 생성 + IMemoryCache 기반 시도 횟수 관리.
- **TossBankVerificationProvider:** `ITossPaymentsService.ValidateBankAccountAsync` + `VerifyBankAccountHolderNameAsync` 2단계 사전 검증.
- **HybridBankVerificationProvider:** Toss 우선 → 실패 시 Custom fallback (`BankVerificationFallbackEnabled` 조건).

---

### MH-3. Provider Factory 패턴 구현 존재
**결과: ✅ 충족**

- **파일:** `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountVerificationProviderFactory.cs`
- **근거:**
  - Line 6-13: `IBankAccountVerificationProviderFactory` 인터페이스 — `Resolve(string? providerSetting)` 계약
  - Line 18-44: `BankAccountVerificationProviderFactory` 구현체
  - Lines 30-35: switch 분기
    ```csharp
    "custom"  => customProvider,
    "toss"    => tossProvider,
    "hybrid"  => hybridProvider,
    _         => FallbackToCustom(providerSetting)
    ```
  - Line 38-44: Unknown 설정 값은 `LogWarning` 후 Custom fallback — 안전한 기본 동작 보장.

---

### MH-4. DB 마이그레이션 SQL 파일 존재 (idempotent)
**결과: ✅ 충족**

- **파일:** `TicketPlatFormServer/TicketPlatFormServer/database_history/migrations/20260225_003_add_verification_provider_fields.sql`
- **근거:**
  - Lines 1-24: 5개 컬럼 추가 (`verification_provider`, `verification_tier`, `verification_status`, `last_verification_failure_code`, `last_verification_at`)
  - **Idempotent 패턴:** 각 컬럼별 `information_schema.columns` 존재 여부 확인 → `IF(@c = 0, 'ALTER TABLE ... ADD COLUMN ...', 'SELECT 1')` 패턴 적용
  - Lines 27-32: Backfill 로직 — 기존 `verified=true` 레코드 보존 (`COALESCE` + `CASE WHEN verified = 1`)
  - **파괴적 변경 없음:** `DROP COLUMN`, `DELETE`, `TRUNCATE` 문 없음.

---

### MH-5. `BankAccountService`가 Provider Factory를 통해 라우팅
**결과: ✅ 충족**

- **파일:** `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs`
- **근거:**
  - Line 14: 생성자 파라미터 `IBankAccountVerificationProviderFactory providerFactory`
  - **RequestVerificationAsync** (Line 76-114): Line 84 — `var provider = providerFactory.Resolve(settings.BankVerificationProvider);`
  - **ConfirmVerificationAsync** (Line 116-186): Line 138 — `var provider = providerFactory.Resolve(settings.BankVerificationProvider);`
  - 양쪽 메서드 모두 Factory를 통해 런타임에 Provider를 결정함. 하드코딩된 Provider 참조 없음.

---

### MH-6. 모바일 `tossVerified` 상태 존재
**결과: ✅ 충족**

- **파일:** `ticket_platform_mobile/lib/features/bank_account/presentation/viewmodels/bank_account_viewmodel.dart`
- **근거:**
  - Lines 12-19: `VerificationState` enum에 `tossVerified` 포함:
    ```dart
    enum VerificationState { idle, requesting, codeInput, tossVerified, verified, error }
    ```
  - Line 169-171 (`_confirmTossVerification`): `VerificationState.tossVerified` 상태로 전환
  - View 파일(`bank_account_verify_view.dart`) Line 39-43: `tossVerified` 상태 감지 시 자동 화면 이동 처리.

---

### MH-7. 모바일 Provider-aware UI (Toss=즉시완료, Custom=코드입력)
**결과: ✅ 충족**

- **파일:** `ticket_platform_mobile/lib/features/bank_account/presentation/viewmodels/bank_account_viewmodel.dart`
- **근거 (분기 로직):**
  - Lines 122-126 (`requestVerification`):
    ```dart
    if (result.provider == 'TOSS' ||
        (result.provider == 'HYBRID' && result.expiresAt == null)) {
      await _confirmTossVerification(current, result.verificationTier);
      return;
    }
    ```
  - Toss/Hybrid(코드 없음): `_confirmTossVerification` → `tossVerified` 상태로 즉시 완료
  - Custom/Hybrid(코드 있음): Lines 129-142 — `codeInput` 상태로 전환, 코드 입력 화면 표시
- **파일:** `ticket_platform_mobile/lib/features/bank_account/presentation/views/bank_account_verify_view.dart`
- **근거 (UI 분기):**
  - Lines 53-74: `VerificationState.tossVerified` → 체크 아이콘 + "계좌 인증이 완료되었습니다." + "확인" 버튼
  - Lines 76-142: Custom 경로 → "1원 인증 요청" 버튼 + 4자리 코드 입력 필드

---

### MH-8. `reasonCode` 기반 메시지 매핑 존재
**결과: ✅ 충족**

- **파일:** `ticket_platform_mobile/lib/features/bank_account/presentation/viewmodels/bank_account_viewmodel.dart`
- **근거:**
  - Lines 255-264: `mapReasonCodeToMessage(String? reasonCode, String fallback)` static 메서드:
    ```dart
    'INVALID_INPUT'       => '입력 정보를 다시 확인해주세요.',
    'ACCOUNT_MISMATCH'    => '계좌/예금주 정보가 일치하지 않습니다.',
    'IDENTITY_MISMATCH'   => '본인(사업자) 정보가 일치하지 않습니다.',
    'UPSTREAM_TEMPORARY'  => '은행 점검 중입니다. 잠시 후 다시 시도해주세요.',
    'PROVIDER_AUTH_ERROR' => '인증 서비스 설정 문제입니다. 잠시 후 다시 시도해주세요.',
    _                     => fallback
    ```
  - 총 5개 reasonCode 매핑 + fallback 처리
- **파일:** `bank_account_verify_view.dart` Lines 120-123:
  - `BankAccountViewModel.mapReasonCodeToMessage(state?.reasonCode, state!.error!)` 호출하여 에러 메시지 노출

---

### MH-9. 운영 전환 go/no-go 문서 존재
**결과: ✅ 충족**

- **파일:** `.sisyphus/evidence/task-16-go-nogo.md`
- **근거:**
  - Phase 1 (Custom → Hybrid) GO 조건 6개 및 NO-GO 조건 5개 명시
  - Phase 3 (Hybrid → Toss Full) GO 조건 3개 및 NO-GO 조건 5개 명시
  - 모니터링 지표 테이블: 응답시간/성공률/fallback비율/on_hold증가율/5xx오류율
  - Line 125: `현재 판정: Custom Provider 빌드 상태 GO ✓`
  - SQL 확인 쿼리(전환 전/후 기준) 포함

---

### MH-10. 롤백 runbook 존재 (10분 이내 목표)
**결과: ✅ 충족**

- **파일:** `.sisyphus/evidence/task-16-rollback-runbook.md`
- **근거:**
  - Lines 70-77: 즉시 롤백 소요 시간표:
    ```
    Step 1: 설정 변경  → 1분
    Step 2: 서버 재시작 → 2분
    Step 3: Smoke test → 2분
    Step 4: 데이터 무결성 → 3분
    합계: 8분
    ```
  - **10분 이내 목표 달성:** 8분 (목표 10분 대비 2분 여유)
  - 환경별 재시작 명령어 제공(systemd, Docker, dotnet run 직접)
  - 데이터 무결성 체크 SQL 3개 포함
  - 부분 롤백(Toss → Hybrid) 시나리오 추가 제공

---

## Must NOT Have 항목 감사 (6/6)

### MNH-1. Controller에 비즈니스 로직 없음
**결과: ✅ 위반 없음**

- **파일:** `TicketPlatFormServer/TicketPlatFormServer/Controllers/BankAccountController.cs`
- **근거:**
  - 총 80줄, 5개 액션 메서드 — 각 메서드는 7-10줄의 박리 구조
  - 패턴: `userId 추출 → null 체크(AppException) → 서비스 위임 → ApiResponse 래핑 반환`
  - 비즈니스 유효성 검사, DB 접근, 복잡한 조건 분기 없음
  - Line 14: 생성자 파라미터 `IBankAccountService bankAccountService` 단 하나
  - 모든 업무 로직은 `BankAccountService`에 위임됨 (Lines 25, 38, 51, 64, 77)

---

### MNH-2. Repository 병렬 호출 없음 (Task.WhenAll 금지)
**결과: ✅ 위반 없음**

- **대상 파일:** `Services/BankAccount/` 디렉토리 전체
- **근거:** `Task.WhenAll` 패턴 검색 결과 — **0건** (grep 매칭 없음)
- **BankAccountService.cs** 확인:
  - `GetBankAccountByUserIdAsync` → `CreateBankAccountAsync` → 순차 호출 (RegisterBankAccountAsync)
  - `GetBankAccountByUserIdAsync` → `UpdateBankAccountAsync` → 순차 호출 (RequestVerificationAsync)
  - `GetBankAccountByUserIdAsync` → `UpdateBankAccountAsync` → `paymentService.ResumeHeldSettlementsAsync` → 순차 호출 (ConfirmVerificationAsync)
  - 모든 레포지토리 호출이 순차적 `await` 패턴을 따름.

---

### MNH-3. DTO가 Repository 계층에 도달하지 않음
**결과: ✅ 위반 없음**

- **파일:** `TicketPlatFormServer/TicketPlatFormServer/Repository/BankAccount/IBankAccountRepository.cs`
- **근거:**
  - 모든 메서드 시그니처가 `TicketPlatFormServer.DBModel.BankAccount` 엔티티만 사용:
    - Line 8: `Task<BankAccount?> GetBankAccountByUserIdAsync(long userId)` — primitive 파라미터
    - Line 12: `Task<BankAccount> CreateBankAccountAsync(BankAccount bankAccount)` — entity 파라미터
    - Line 14: `Task UpdateBankAccountAsync(BankAccount bankAccount)` — entity 파라미터
    - Line 16: `Task DeleteBankAccountAsync(long id)` — primitive 파라미터
  - DTO 네임스페이스(`TicketPlatFormServer.DTO`) import 없음
  - `BankAccountService.cs`에서 DTO를 엔티티로 변환 후 Repository에 전달 (Service 계층에서 변환 책임)

---

### MNH-4. 민감정보(계좌번호, 예금주명) 로그 원문 출력 없음
**결과: ✅ 위반 없음**

- **BankAccount 서비스 파일 전체 로그 검사:**

| 파일 | 로그 위치 | 로그 내용 |
|------|-----------|-----------|
| `CustomBankVerificationProvider.cs` | Line 27 | `UserId={UserId}` — PII 없음 |
| `CustomBankVerificationProvider.cs` | Line 52 | `UserId={UserId}` — PII 없음 |
| `CustomBankVerificationProvider.cs` | Line 60 | `UserId={UserId}` — PII 없음 |
| `TossBankVerificationProvider.cs` | Line 46 | `UserId={UserId}` — PII 없음 |
| `TossBankVerificationProvider.cs` | Line 60 | `UserId={UserId}` — PII 없음 |
| `HybridBankVerificationProvider.cs` | Line 28-30 | Exception 정보 + `UserId={UserId}` — 계좌 정보 없음 |
| `HybridBankVerificationProvider.cs` | Lines 44, 48 | `UserId={UserId}` — PII 없음 |
| `BankAccountService.cs` | Line 158 | `UserId={UserId}` — PII 없음 |

- **계좌번호 마스킹:** `BankAccountService.cs` Lines 202-211 `MaskAccountNumber()` — 응답 DTO에서 계좌번호 마지막 4자리만 노출(`****1234`)
- `AccountNumber`, `AccountHolder`, `BankCode`는 Toss API 호출에 직접 전달되지만 로그에 출력되지 않음.

---

### MNH-5. `*.g.dart`, `*.freezed.dart` 직접 수정 없음
**결과: ✅ 위반 없음**

- **확인한 생성 파일:**
  - `request_verification_resp_dto.g.dart` — Line 1: `// GENERATED CODE - DO NOT MODIFY BY HAND`
  - `bank_account_viewmodel.g.dart` — Line 1: `// GENERATED CODE - DO NOT MODIFY BY HAND`
- **근거:**
  - 모든 `.g.dart`, `.freezed.dart` 파일의 첫 줄에 표준 `// GENERATED CODE - DO NOT MODIFY BY HAND` 헤더 존재
  - `request_verification_resp_dto.g.dart` 내용이 소스 파일(`request_verification_resp_dto.dart`)의 `@freezed` 클래스 정의와 완전히 일치
  - 소스 파일은 `part '*.freezed.dart'` / `part '*.g.dart'` 선언으로 생성 파일 참조 (역방향 아님)
  - 수동으로 편집된 흔적(커스텀 메서드, 추가 import, 비표준 구조) 없음

---

### MNH-6. Settlement 도메인 코드 수정 없음
**결과: ✅ 위반 없음**

- **확인 대상:**
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Settlement/SettlementService.cs`
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Settlement/ISettlementService.cs`
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BackgroundServices/SettlementProcessingService.cs`
- **근거:**
  - Settlement 도메인 파일 내 `VerificationProvider`, `VerificationStatus`, `VerificationTier`, `tossVerified` 등 신규 은행인증 필드 참조 **0건** (grep 매칭 없음)
  - `SettlementService.cs`는 기존 정산 조회/요약 기능만 포함 (Lines 1-55, bank account 관련 로직 없음)
  - `SettlementProcessingService.cs`는 기존 `BankAccountId` (기존 FK) 참조만 존재 (Line 70: `settlement.BankAccountId == null`)
  - `ResumeHeldSettlementsAsync`는 **Payment 도메인** (`PaymentService.cs` Line 577)에 위치 — Settlement 도메인 코드 수정 없음
  - 정산 해제 흐름: `BankAccountService` → `IPaymentService.ResumeHeldSettlementsAsync` — Settlement Service 직접 수정 없이 기존 계층 구조 활용

---

## 세부 증거 파일 참조

| 증적 | 경로 | 주요 확인 내용 |
|------|------|---------------|
| T15 통합 검증 | `.sisyphus/evidence/task-15-summary.md` | 빌드 PASS, analyze PASS, 6개 시나리오 전체 PASS |
| T16 Go/No-Go | `.sisyphus/evidence/task-16-go-nogo.md` | Phase 1/3 전환 기준, 모니터링 지표 |
| T16 Rollback | `.sisyphus/evidence/task-16-rollback-runbook.md` | 8분 롤백 절차 |
| T16 Smoke Test | `.sisyphus/evidence/task-16-cutover-rehearsal.txt` | Factory 3분기 확인, DI 등록 확인, `dotnet build` PASS |

---

## VERDICT

```
Must Have  [10/10] ✅
Must NOT Have [6/6] ✅

VERDICT: PASS ✅
```

**근거 요약:**
- 10개 Must Have 항목 모두 파일/라인 번호로 실증됨.
- 6개 Must NOT Have 항목 모두 위반 없음 확인됨.
- 백엔드: `IBankAccountVerificationProvider` 추상화 + 3개 구현체 + Factory 패턴 + idempotent 마이그레이션 SQL + Service 라우팅 완전 구현.
- 모바일: `tossVerified` enum 상태 + Provider-aware UI 분기 + `mapReasonCodeToMessage` 5개 코드 매핑.
- 운영: Go/No-Go 기준 문서 + 8분 롤백 런북 (10분 목표 달성).
- 위반: Controller 비즈니스 로직 없음, Task.WhenAll 없음, DTO-Repository 경계 준수, PII 로그 없음, 생성 파일 직접 수정 없음, Settlement 도메인 수정 없음.
