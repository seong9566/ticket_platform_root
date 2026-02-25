# F3 Real QA Replay
Date: 2026-02-25
Plan: toss-bank-account-gap-analysis
Branch: feat/notification-fcm

---

## QA Scenarios [10/10]

| # | 시나리오 | 결과 | 근거 |
|---|---------|------|------|
| 1 | Custom Provider: `RequestAsync` 4자리 코드 생성 | ✅ | `CustomBankVerificationProvider.cs:20` — `Random.Shared.Next(0,10000).ToString("D4")`, `ExpiresAt=DateTime.UtcNow.AddMinutes(...)` (line 22). 증적: `task-15-provider-routing.txt` PASS |
| 2 | Toss Provider: `RequestAsync` Toss API 호출 로직 | ✅ | `TossBankVerificationProvider.cs:27-39` — `ValidateBankAccountAsync` + `VerifyBankAccountHolderNameAsync` 2단계 호출. `ExpiresAt: null` 반환 (line 53) |
| 3 | Hybrid Provider: Toss 우선 + Custom fallback | ✅ | `HybridBankVerificationProvider.cs:20-35` — `BankVerificationFallbackEnabled=true` 시 try Toss → catch → custom fallback. disabled 시 직접 Toss 호출 |
| 4 | Toss 경로: `expiresAt=null` → `tossVerified` 상태 → 즉시 완료 | ✅ | `bank_account_viewmodel.dart:122-126` — `result.expiresAt == null` 분기에서 `_confirmTossVerification` 즉시 호출. 결과: `VerificationState.tossVerified` (line 170). 증적: `task-15-mobile-provider.txt` PASS |
| 5 | Custom 경로: `expiresAt!=null` → 코드 입력 화면 | ✅ | `bank_account_viewmodel.dart:129-142` — expiresAt 존재 시 `codeInput` 상태 전환. `bank_account_verify_view.dart:75-142` — 4자리 코드 입력 UI 분기 |
| 6 | reasonCode 메시지 매핑: 5개 코드 + fallback | ✅ | `bank_account_viewmodel.dart:255-264` — `mapReasonCodeToMessage`: INVALID_INPUT / ACCOUNT_MISMATCH / IDENTITY_MISMATCH / UPSTREAM_TEMPORARY / PROVIDER_AUTH_ERROR + `_ => fallback`. View에서 호출: `bank_account_verify_view.dart:120-123` |
| 7 | 정산 연계: `ConfirmVerificationAsync` → `ResumeHeldSettlementsAsync` | ✅ | `BankAccountService.cs:165-175` — Verified=true, VerificationStatus="VERIFIED", LastVerificationAt=UtcNow 설정 후 `paymentService.ResumeHeldSettlementsAsync(userId, bankAccount.Id)` 호출. 증적: `task-15-settlement-gate.txt` PASS |
| 8 | 마이그레이션 SQL idempotent 확인 | ✅ | 증적 `task-15-migration-sql.txt`: 5개 컬럼(`verification_provider`, `verification_tier`, `verification_status`, `last_verification_failure_code`, `last_verification_at`) 각각 `information_schema.columns` 존재 확인 후 `IF(@c=0, ALTER TABLE ..., SELECT 1)` 패턴. DROP COLUMN/DELETE 없음 |
| 9 | go/no-go 문서 존재 및 내용 확인 | ✅ | `task-16-go-nogo.md`: Phase 1 GO 조건 6개 + NO-GO 조건 5개, Phase 3 GO 조건 3개 + NO-GO 조건 5개 명시. 모니터링 지표 테이블 포함. 현재 판정: `Custom Provider 빌드 상태 GO ✓` |
| 10 | 롤백 runbook 8분 목표 확인 | ✅ | `task-16-rollback-runbook.md:70-77` — 즉시 롤백 4단계: 설정변경(1분)+재시작(2분)+Smoke test(2분)+무결성확인(3분) = **합계 8분** (10분 목표 달성, 2분 여유) |

---

## Evidence Files [12/12]

| 파일 | 존재 | 내용 확인 |
|------|------|---------|
| `task-15-backend-build.txt` | ✅ | `dotnet build` 결과: 경고 0개, 오류 0개, 경과 0.87초 |
| `task-15-migration-sql.txt` | ✅ | 5개 컬럼 추가 idempotent SQL 검증 결과 PASS, DROP/DELETE 없음 |
| `task-15-mobile-analyze.txt` | ✅ | `flutter analyze` 결과: error 0, 23 issues (기존 전부, 신규 없음) |
| `task-15-mobile-provider.txt` | ✅ | tossVerified 상태, `_confirmTossVerification`, reasonCode 5개 매핑 확인 PASS |
| `task-15-provider-routing.txt` | ✅ | Factory 3분기(custom/toss/hybrid) + Unknown fallback + LogWarning 확인 PASS |
| `task-15-settlement-gate.txt` | ✅ | `ResumeHeldSettlementsAsync` 호출(line 175), VERIFIED 상태 설정(line 169), LastVerificationAt(line 171) 확인 PASS |
| `task-15-summary.md` | ✅ | T15 6개 시나리오 전체 PASS, 최종 판정 PASS |
| `task-16-cutover-rehearsal.txt` | ✅ | 설정 3파일 확인, Factory 3분기 + DI 등록 + dotnet build PASS, Smoke Test GO ✓ |
| `task-16-go-nogo.md` | ✅ | Phase 1/3 GO/NO-GO 기준, 모니터링 지표 테이블, SQL 확인 쿼리, 현재 GO 판정 |
| `task-16-rollback-runbook.md` | ✅ | 4단계 롤백 절차 + 8분 소요표 + 환경별 재시작 명령 + 데이터 무결성 SQL |
| `f1-plan-compliance-audit.md` | ✅ | Must Have 10/10, Must NOT Have 6/6, VERDICT: PASS |
| `f2-code-quality-review.md` | ✅ | Build PASS(0/0), Analyze PASS(error 0), PII 로그 CLEAN, VERDICT: PASS |

---

## 코드 리뷰 세부 결과

### T13: 백엔드 Provider 구현 검증

**CustomBankVerificationProvider.cs**
- `Name => "CUSTOM"` (line 15)
- `RequestAsync`: `Random.Shared.Next(0,10000).ToString("D4")` → 4자리 코드 (line 20)
- `ExpiresAt = DateTime.UtcNow.AddMinutes(expiryMinutes)` (line 22)
- `IMemoryCache` 기반 시도 횟수 관리 (line 25/42)
- `ConfirmAsync`: 코드 일치 검사 + 최대 시도 횟수 초과 처리 (lines 44-63)

**TossBankVerificationProvider.cs**
- `Name => "TOSS"` (line 15)
- `RequestAsync`: `ValidateBankAccountAsync` → `VerifyBankAccountHolderNameAsync` 2단계 Toss API 호출 (lines 27-44)
- `ExpiresAt: null` 반환 (line 53) — 즉시 완료 신호
- `ConfirmAsync`: 사전 검증 완료 → 항상 `true` 반환 (line 61)

**HybridBankVerificationProvider.cs**
- `Name => "HYBRID"` (line 15)
- `RequestAsync`: `BankVerificationFallbackEnabled=true` → Toss 시도 후 실패 시 Custom fallback (lines 20-33)
- `ConfirmAsync`: `VerificationCode` 없으면 Toss 경로, 있으면 Custom 경로 (lines 42-50)

### T14: 모바일 Provider-aware UI 검증

**bank_account_viewmodel.dart**
- `VerificationState.tossVerified` enum 값 존재 (line 16)
- `requestVerification()`: `result.expiresAt == null` → `_confirmTossVerification` 즉시 호출 (lines 122-126)
- `_confirmTossVerification()`: `confirmVerification(code: '')` 호출 → `tossVerified` 상태 전환 (lines 156-189)
- `mapReasonCodeToMessage()`: 5개 코드 + fallback switch (lines 255-264)

**bank_account_verify_view.dart**
- `VerificationState.tossVerified` 감지 시 `bankAccountDetail` 자동 이동 (lines 39-43)
- Toss 완료 UI: 체크 아이콘 + "계좌 인증이 완료되었습니다." (lines 53-74)
- Custom 경로 UI: 4자리 코드 입력 필드 + 타이머 표시 (lines 76-142)
- 에러 시 `mapReasonCodeToMessage` 호출하여 사용자 친화적 메시지 출력 (lines 119-125)

### T15: 통합 검증

**BankAccountService.cs**
- `RequestVerificationAsync`: Factory 통해 Provider 결정 (line 84), DB 업데이트 (lines 94-103)
- `ConfirmVerificationAsync`: Factory 통해 Provider 결정 (line 138), 인증 성공 시 VERIFIED 상태 저장 + `ResumeHeldSettlementsAsync` 호출 (lines 165-175)
- 계좌번호 마스킹: `MaskAccountNumber()` (lines 202-211)

---

## VERDICT: PASS

### 근거
1. **QA 시나리오 10/10 통과**: 모든 시나리오가 실제 구현 코드(파일:라인)로 직접 검증됨
2. **증적 파일 12/12 존재 및 내용 확인**: 각 파일이 실질적인 검증 내용을 포함하며 PASS 판정
3. **코드-증적 일치**: 증적 파일의 라인 번호와 실제 구현 파일 내용이 일치함
   - `task-15-provider-routing.txt`: "3분기 확인" → `HybridBankVerificationProvider.cs` 코드와 일치
   - `task-15-settlement-gate.txt`: "line 175" → `BankAccountService.cs` `ResumeHeldSettlementsAsync` 호출과 일치
   - `task-15-mobile-provider.txt`: "line 257~261" → `mapReasonCodeToMessage` 5개 매핑과 일치
4. **빌드/분석 통과**: Backend 0 errors 0 warnings (증적: `task-15-backend-build.txt`), Flutter error 0 (증적: `task-15-mobile-analyze.txt`)
5. **운영 전환 준비 완료**: 8분 롤백 런북(10분 목표 달성), Phase 1/3 Go/No-Go 기준 문서화
