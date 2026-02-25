# Toss 계좌인증 API 갭 분석 및 통합 실행 계획

## TL;DR

> **Quick Summary**: 현재 구현은 내부 4자리 코드 기반 1원 인증 흐름이며, 토스 공식 계좌인증 API는 아직 미연동 상태다. 본 계획은 `Custom -> Hybrid -> Toss 중심`으로 안전하게 전환하면서 정산 보류/재개 정책을 유지하는 것을 목표로 한다.
>
> **Deliverables**:
> - 토스 계좌인증 대비 갭 매트릭스(구현/미구현/보류)
> - 백엔드 계좌인증 Provider 전략(Custom/Toss/Hybrid)
> - API/DB/모바일 계약 확장안과 롤아웃 가드레일
>
> **Estimated Effort**: Medium
> **Parallel Execution**: YES - 4 waves
> **Critical Path**: T1 -> T3 -> T7 -> T9 -> T11 -> T12

---

## Context

### Original Request
토스페이먼츠 계좌인증 API 문서와 현재 개발 상태를 비교해 구현/미구현 항목을 식별하고, 후속 작업 계획을 세운다.

### Interview Summary
**Key Discussions**:
- 현재 계좌인증은 내부 4자리 코드 기반(`verify/request`, `verify/confirm`)으로 동작한다.
- 계좌 미인증 상태에서는 정산이 `on_hold`이며, 인증 완료 시 `pending`으로 재개된다.
- 토스 지급대행(`/v2/payouts`)은 일부 연동되어 있으나 토스 계좌인증 API 5종은 미연동이다.

**Research Findings**:
- 공식 계좌인증 API는 예금주 조회/계좌 유효성/식별자 일치/예금주 일치/복합 일치의 5개 endpoint를 제공한다.
- 지급대행은 계좌인증과 별도이며, POST 본문 보안(JWE) 등 운영 가드레일이 중요하다.

### Metis Review
**Identified Gaps (addressed in this plan)**:
- 검증 의미 불일치(계좌 유효성 vs 소유자 실명 일치) -> 검증 Tier 정의로 해결
- PII 확장 리스크 -> 최소 수집/마스킹/로그 레드액션 가드레일 정의
- 롤아웃 리스크 -> Feature Flag + 하이브리드 단계 전환 + 롤백 경로 정의

---

## Work Objectives

### Core Objective
토스 공식 계좌인증 API를 기준으로 현재 기능의 누락 지점을 정량화하고, 기존 정산 흐름을 깨지 않으면서 계좌인증 신뢰도를 높이는 통합 실행 경로를 확정한다.

### Concrete Deliverables
- `.sisyphus/evidence/toss-bank-verification-gap-matrix.md`
- 백엔드 Provider 전략 명세(Custom/Toss/Hybrid)
- API/DB/모바일 변경 TODO와 롤아웃/롤백 절차

### Definition of Done
- [ ] 토스 계좌인증 5개 API 대비 구현 상태가 엔드포인트 단위로 매핑됨
- [ ] 미구현 항목이 우선순위/의존성/검증 시나리오와 함께 TODO로 정리됨
- [ ] 정산 `on_hold -> pending` 정책이 신규 인증 전략에서도 유지됨

### Must Have
- 기존 `verify/request`, `verify/confirm` 계약의 하위호환 경로
- Provider 전환 가능 설정값(`Custom|Toss|Hybrid`)
- PII 보호 기준(저장/로그/전송)

### Must NOT Have (Guardrails)
- 토스 API 응답 원문(민감정보) 로그 저장 금지
- 사용자/운영자 수동 확인이 필요한 acceptance criteria 금지
- 계좌인증 작업 중 정산 도메인 리팩터링 범위 확장 금지

---

## Verification Strategy (MANDATORY)

> **ZERO HUMAN INTERVENTION** — 모든 검증은 agent 실행 기준으로 작성한다.

### Test Decision
- **Infrastructure exists**: Backend 부분적(NO dedicated test project), Mobile YES
- **Automated tests**: Tests-after
- **Framework**: Backend는 `curl + dotnet build` 기반 검증, Mobile은 `flutter analyze` 및 widget/integration 보강
- **If TDD**: 본 계획은 미적용

### QA Policy
- Backend API: Bash(`curl`)로 상태코드/응답필드/에러코드 검증
- Frontend/UI: Playwright로 계좌 등록/인증/가이드 노출 검증
- 배치/CLI: Bash로 background service 로그 및 DB 상태 변화를 검증
- 증적 저장: `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`

---

## Execution Strategy

### Parallel Execution Waves

Wave 1 (Start Immediately — 분석/계약/기반 설계):
- T1. 토스 계좌인증 5종 대비 갭 매트릭스 작성
- T2. 토스 계좌인증 요청/응답/에러코드 명세 잠금
- T3. 검증 Tier/상태 모델 설계(Custom/Toss/Hybrid)
- T4. 보안/PII/로그 레드액션 정책 확정

Wave 2 (After Wave 1 — 백엔드 설계/구현 계획 고도화):
- T5. Provider 추상화 인터페이스 및 DI 전략 수립
- T6. DB 확장안(검증수단/최종실패코드/검증시각) 정의
- T7. API 계약안(v1 호환 + v2 확장) 정의
- T8. 에러 매핑/재시도/서킷브레이커 정책 수립

Wave 3 (After Wave 2 — 정산/모바일/롤아웃):
- T9. 정산 게이트 연계 시나리오(on_hold/pending) 확정
- T10. 모바일 상태/문구/화면전환 변경안 정의
- T11. 점진 롤아웃 + 롤백 시나리오 정의
- T12. 통합 QA 시나리오 및 증적 포맷 고정

Wave 4 (After Wave 3 — 실행 검증):
- T13. 백엔드 구현 실행
- T14. 모바일 구현 실행
- T15. 통합 검증 실행
- T16. 운영 전환 리허설

### Dependency Matrix

- **T1**: Blocked By None -> Blocks T3, T7
- **T2**: Blocked By None -> Blocks T7, T8
- **T3**: Blocked By T1 -> Blocks T5, T6, T9
- **T4**: Blocked By None -> Blocks T8, T11
- **T5**: Blocked By T3 -> Blocks T13
- **T6**: Blocked By T3 -> Blocks T13, T15
- **T7**: Blocked By T1, T2 -> Blocks T10, T13, T14
- **T8**: Blocked By T2, T4 -> Blocks T13, T16
- **T9**: Blocked By T3 -> Blocks T13, T15
- **T10**: Blocked By T7 -> Blocks T14, T15
- **T11**: Blocked By T4 -> Blocks T16
- **T12**: Blocked By T6, T9, T10 -> Blocks T15
- **T13**: Blocked By T5, T6, T7, T8, T9 -> Blocks T15, T16
- **T14**: Blocked By T7, T10 -> Blocks T15
- **T15**: Blocked By T6, T9, T10, T12, T13, T14 -> Blocks T16
- **T16**: Blocked By T8, T11, T13, T15 -> Blocks Final Wave

### Agent Dispatch Summary

- Wave 1: T1 `deep`, T2 `librarian`, T3 `deep`, T4 `unspecified-high`
- Wave 2: T5 `quick`, T6 `unspecified-high`, T7 `deep`, T8 `unspecified-high`
- Wave 3: T9 `deep`, T10 `visual-engineering`, T11 `unspecified-high`, T12 `deep`
- Wave 4: T13 `unspecified-high`, T14 `visual-engineering`, T15 `deep`, T16 `unspecified-high`
- Final: F1 `oracle`, F2 `unspecified-high`, F3 `unspecified-high`, F4 `deep`

---

## TODOs

- [x] 1. 토스 계좌인증 5종 대비 갭 매트릭스 작성

  **What to do**:
  - 현재 구현 API/서비스/DB 필드와 토스 공식 5개 endpoint를 1:1 매핑한다.
  - 상태를 `구현완료/부분구현/미구현/보류`로 분류하고 근거 파일을 명시한다.

  **Must NOT do**:
  - 구현 코드 변경 없이 분석만 수행한다.

  **Recommended Agent Profile**:
  - **Category**: `deep` (교차 도메인 매핑 정확도 필요)
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `git-master` (git 작업 불필요)

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (T2, T4와 병렬)
  - **Blocks**: T3, T7
  - **Blocked By**: None

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs` - 내부 인증 흐름 근거
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/BankAccountController.cs` - 현재 공개 API 경로 근거
  - `https://docs.tosspayments.com/reference/additional#계좌인증` - 공식 비교 기준

  **Acceptance Criteria**:
  - [ ] `.sisyphus/evidence/toss-bank-verification-gap-matrix.md` 생성
  - [ ] 5개 endpoint 모두 매핑되고 근거 파일 경로 포함

  **QA Scenarios**:
  ```
  Scenario: Gap matrix completeness
    Tool: Bash
    Preconditions: 계획 브랜치 최신 상태
    Steps:
      1. Read로 `.sisyphus/evidence/toss-bank-verification-gap-matrix.md` 확인
      2. `lookup-holder-name/validate/verify-identifier/verify-holder-name/verify-holder-real-name` 문자열 존재 검증
      3. 각 항목에 구현상태 컬럼 존재 검증
    Expected Result: 5개 항목 모두 존재하고 상태/근거가 채워져 있음
    Failure Indicators: endpoint 누락, 상태 미기재, 근거 파일 미표기
    Evidence: .sisyphus/evidence/task-1-gap-matrix.txt

  Scenario: Negative - missing endpoint detection
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. 매트릭스에서 `verify-holder-real-name` 항목 유무 확인
      2. 없으면 실패 처리 로그 기록
    Expected Result: 누락 시 즉시 FAIL로 기록됨
    Evidence: .sisyphus/evidence/task-1-gap-matrix-error.txt
  ```

  **Commit**: NO

- [x] 2. 토스 계좌인증 요청/응답/에러코드 계약 잠금

  **What to do**:
  - 토스 문서 기준 파라미터 제한(은행코드, 계좌번호 길이, 식별자 포맷)을 계약 표로 고정한다.
  - 재시도 가능/불가 에러코드를 분류한다.

  **Must NOT do**:
  - 문서에 없는 임의 필드를 계약에 추가하지 않는다.

  **Recommended Agent Profile**:
  - **Category**: `writing` (명세 정리 중심)
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `frontend-ui-ux` (UI 작업 아님)

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (T1, T4와 병렬)
  - **Blocks**: T7, T8
  - **Blocked By**: None

  **References**:
  - `https://docs.tosspayments.com/reference/additional#계좌인증` - API 파라미터/응답 기준
  - `https://docs.tosspayments.com/reference/error-codes#계좌인증-에러-목록` - 에러코드 분류 기준

  **Acceptance Criteria**:
  - [ ] `.sisyphus/evidence/toss-bank-api-contract.md` 생성
  - [ ] 요청/응답 필드, 제약, 에러 분류(재시도 가능 여부) 포함

  **QA Scenarios**:
  ```
  Scenario: Contract document validity
    Tool: Bash
    Preconditions: 문서 생성 완료
    Steps:
      1. 계약 문서에서 5개 endpoint 섹션 확인
      2. 각 섹션에 request fields/response fields/error codes 확인
      3. retryable/non-retryable 분류 존재 확인
    Expected Result: 5개 모두 동일 포맷으로 정리됨
    Failure Indicators: endpoint별 형식 불일치, 에러 분류 누락
    Evidence: .sisyphus/evidence/task-2-contract-check.txt

  Scenario: Negative - undocumented field guard
    Tool: Bash
    Preconditions: 문서 생성 완료
    Steps:
      1. 계약 문서에서 `customCode` 같은 비공식 필드 포함 여부 검색
      2. 발견 시 실패 기록
    Expected Result: 비공식 필드 없음
    Evidence: .sisyphus/evidence/task-2-contract-check-error.txt
  ```

  **Commit**: NO

- [x] 3. 검증 Tier/상태 모델 설계(Custom/Toss/Hybrid)

  **What to do**:
  - `Verified` 단일 boolean을 `verificationProvider`, `verificationMethod`, `verificationStatus`로 확장하는 설계를 작성한다.
  - 정산 정책(`on_hold`, `pending`)과 각 Tier 간 관계를 표로 명시한다.

  **Must NOT do**:
  - 기존 정산 상태코드 체계를 임의 변경하지 않는다.

  **Recommended Agent Profile**:
  - **Category**: `deep`
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `playwright` (브라우저 검증 불필요)

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential after T1
  - **Blocks**: T5, T6, T9
  - **Blocked By**: T1

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/BankAccount.cs` - 현재 계좌 인증 필드 구조
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs` - 보류/재개 로직 연계점
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/SettlementStatus.cs` - 상태코드 기준

  **Acceptance Criteria**:
  - [ ] Tier 정의 문서에 `Custom`, `Toss`, `Hybrid` 상태 전이도 포함
  - [ ] 각 Tier별 정산 허용 조건이 명시됨

  **QA Scenarios**:
  ```
  Scenario: Tier-state consistency
    Tool: Bash
    Preconditions: 설계 문서 생성 완료
    Steps:
      1. 문서에서 3개 Tier 정의 존재 확인
      2. 각 Tier에 허용 액션(정산 보류/재개) 명시 확인
      3. 상태 전이 표에 terminal state 포함 확인
    Expected Result: 모호한 상태 없이 전이 정의 완료
    Failure Indicators: Tier 누락, 전이 조건 불명확
    Evidence: .sisyphus/evidence/task-3-tier-model.txt

  Scenario: Negative - invalid transition
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. `UNVERIFIED -> COMPLETED` 직접 전이 허용 여부 확인
      2. 허용 시 실패 처리
    Expected Result: 직접 전이 금지
    Evidence: .sisyphus/evidence/task-3-tier-model-error.txt
  ```

  **Commit**: NO

- [x] 4. 보안/PII/로그 레드액션 정책 확정

  **What to do**:
  - 식별자/계좌번호/예금주명에 대해 저장/전송/로그 정책을 확정한다.
  - 실패 로그에는 reason code만 남기고 원문 payload를 저장하지 않도록 규칙화한다.

  **Must NOT do**:
  - 개인정보 원문을 evidence 또는 로그 예시에 넣지 않는다.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `frontend-ui-ux` (보안 정책 문서 작업)

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (T1, T2와 병렬)
  - **Blocks**: T8, T11
  - **Blocked By**: None

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/TossPaymentsService.cs` - 현재 raw response 처리 위치
  - `TicketPlatFormServer/TicketPlatFormServer/Common/Exception/GlobalExceptionMiddleware.cs` - 에러 응답 로깅 경계

  **Acceptance Criteria**:
  - [ ] 레드액션 정책 표(필드별 마스킹 규칙) 작성
  - [ ] 로그 금지 필드 목록과 허용 필드 목록 분리

  **QA Scenarios**:
  ```
  Scenario: Redaction policy enforceability
    Tool: Bash
    Preconditions: 정책 문서 생성 완료
    Steps:
      1. 금지 필드(accountNumber, holderName, identityNumber) 목록 확인
      2. 허용 로그 필드(reasonCode, status, requestId) 확인
      3. 샘플 로그가 마스킹 규칙 준수하는지 확인
    Expected Result: 민감 필드 원문 노출 규칙 없음
    Failure Indicators: 원문 출력 허용 또는 마스킹 누락
    Evidence: .sisyphus/evidence/task-4-redaction-policy.txt

  Scenario: Negative - raw payload logging
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. 정책 문서에서 `RawResponse` 전체 저장 허용 여부 검색
      2. 발견 시 실패
    Expected Result: 전체 payload 저장 금지
    Evidence: .sisyphus/evidence/task-4-redaction-policy-error.txt
  ```

  **Commit**: NO

- [x] 5. Provider 추상화 인터페이스 및 DI 전략 수립

  **What to do**:
  - `IBankAccountVerificationProvider` 계약과 `CustomProvider`, `TossProvider`, `HybridProvider` 라우팅 규칙을 정의한다.
  - `TossPaymentsSettings`에 Provider 선택 설정을 추가한다.

  **Must NOT do**:
  - 기존 `BankAccountService` public API 시그니처를 파괴적으로 변경하지 않는다.

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `artistry` (창의적 탐색 불필요)

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential after T3
  - **Blocks**: T13
  - **Blocked By**: T3

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/IBankAccountService.cs` - 서비스 계약 경계
  - `TicketPlatFormServer/TicketPlatFormServer/Program.cs` - DI 등록 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/Config/TossPaymentsSettings.cs` - 설정 확장 위치

  **Acceptance Criteria**:
  - [ ] Provider 인터페이스/DI 결정 문서 작성
  - [ ] `Custom|Toss|Hybrid` 설정별 라우팅 규칙 명시

  **QA Scenarios**:
  ```
  Scenario: Provider routing matrix
    Tool: Bash
    Preconditions: 설계 문서 작성 완료
    Steps:
      1. 설정값별 호출 Provider 표 확인
      2. verify/request, verify/confirm에서 분기 경로 확인
      3. fallback 조건(토스 장애 시) 확인
    Expected Result: 모든 설정값에 결정적 분기 존재
    Failure Indicators: 분기 누락 또는 중복 경로
    Evidence: .sisyphus/evidence/task-5-provider-routing.txt

  Scenario: Negative - undefined provider
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. `Unknown` 설정 입력 시 처리 정책 확인
      2. 정책 미정의 시 실패
    Expected Result: Unknown 입력은 안전한 기본값/예외로 처리
    Evidence: .sisyphus/evidence/task-5-provider-routing-error.txt
  ```

  **Commit**: NO

- [x] 6. DB 확장안 정의(검증수단/실패코드/검증시각)

  **What to do**:
  - `bank_account`에 provider/method/last_failure_code 등 최소 필드 추가안을 설계한다.
  - 기존 verified 데이터의 backward compatibility 마이그레이션 규칙을 정한다.

  **Must NOT do**:
  - 기존 컬럼 의미를 덮어써서 이력 추적 불가능하게 만들지 않는다.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `frontend-ui-ux`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (T7, T8와 병렬)
  - **Blocks**: T13, T15
  - **Blocked By**: T3

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/BankAccount.cs`
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/TicketContext.cs`
  - `TicketPlatFormServer/TicketPlatFormServer/database_history/migrations/20260225_001_add_on_hold_and_bank_account_verification.sql`

  **Acceptance Criteria**:
  - [ ] 마이그레이션 초안 SQL 작성
  - [ ] 기존 데이터 보존 규칙(기본값/백필) 포함

  **QA Scenarios**:
  ```
  Scenario: Migration backward compatibility
    Tool: Bash
    Preconditions: SQL 초안 존재
    Steps:
      1. 신규 컬럼이 nullable 또는 안전 기본값인지 확인
      2. 기존 `verified=true` 레코드 유지 규칙 확인
      3. 다운그레이드 없이 롤백 가능한지 확인
    Expected Result: 기존 서비스 중단 없이 적용 가능
    Failure Indicators: NOT NULL 강제/기존 데이터 손실 위험
    Evidence: .sisyphus/evidence/task-6-db-migration.txt

  Scenario: Negative - destructive migration
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. `DROP COLUMN` 또는 데이터 삭제 문 포함 여부 확인
      2. 발견 시 실패
    Expected Result: 파괴적 변경 없음
    Evidence: .sisyphus/evidence/task-6-db-migration-error.txt
  ```

  **Commit**: NO

- [x] 7. API 계약안(v1 호환 + v2 확장) 정의

  **What to do**:
  - 기존 `/api/bank-account/verify/request|confirm` 유지 전략과 v2 응답 확장(`provider`, `verificationTier`, `reasonCode`)을 설계한다.
  - 모바일 영향도(필수/선택 필드)와 호환 기간을 명시한다.

  **Must NOT do**:
  - 기존 클라이언트가 파싱 실패하는 필드 타입 변경 금지.

  **Recommended Agent Profile**:
  - **Category**: `deep`
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `git-master`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (T6, T8와 병렬)
  - **Blocks**: T10, T13, T14
  - **Blocked By**: T1, T2

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/BankAccountController.cs`
  - `ticket_platform_mobile/lib/core/network/base_response.dart`
  - `ticket_platform_mobile/lib/features/bank_account/data/dto/response/verify_account_resp_dto.dart`

  **Acceptance Criteria**:
  - [ ] v1/v2 응답 예시(JSON) 문서 포함
  - [ ] 모바일 하위호환 판단표 포함

  **QA Scenarios**:
  ```
  Scenario: Backward compatibility response check
    Tool: Bash (curl)
    Preconditions: API 계약 문서 작성
    Steps:
      1. v1 응답 필드 목록 확인(기존 필드 유지)
      2. v2 확장 필드가 optional로 정의되었는지 확인
      3. 기존 앱 파서 영향도 표 확인
    Expected Result: 기존 앱은 영향 없이 동작 가능
    Failure Indicators: 기존 필수 필드 제거/타입 변경
    Evidence: .sisyphus/evidence/task-7-api-contract.txt

  Scenario: Negative - breaking change
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. `verified` 타입 변경 여부 확인
      2. 변경 시 실패
    Expected Result: 핵심 필드 타입 유지
    Evidence: .sisyphus/evidence/task-7-api-contract-error.txt
  ```

  **Commit**: NO

- [x] 8. 에러 매핑/재시도/서킷브레이커 정책 수립

  **What to do**:
  - 토스 계좌인증 실패를 `retryable`/`terminal`로 분류한다.
  - timeout, 429, 5xx 처리와 fallback(Custom) 조건을 확정한다.

  **Must NOT do**:
  - terminal 오류를 무한 재시도하도록 설계하지 않는다.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `playwright`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (T6, T7와 병렬)
  - **Blocks**: T13, T16
  - **Blocked By**: T2, T4

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/TossPaymentsService.cs`
  - `TicketPlatFormServer/TicketPlatFormServer/Config/TossPaymentsSettings.cs`
  - `https://docs.tosspayments.com/reference/error-codes#계좌인증-에러-목록`

  **Acceptance Criteria**:
  - [ ] 에러코드별 처리표(재시도 횟수/지연/사용자 메시지) 작성
  - [ ] fallback 기준 명시

  **QA Scenarios**:
  ```
  Scenario: Retry policy determinism
    Tool: Bash
    Preconditions: 정책 문서 작성
    Steps:
      1. 429/5xx/timeout 케이스 처리 규칙 확인
      2. max retry 횟수 확인
      3. fallback 전환 조건 확인
    Expected Result: 상태별 행동이 단일하게 정의됨
    Failure Indicators: 케이스 누락/중복 정책
    Evidence: .sisyphus/evidence/task-8-retry-policy.txt

  Scenario: Negative - infinite retry risk
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. retry max 값 미설정 여부 확인
      2. 미설정이면 실패
    Expected Result: 재시도 상한 존재
    Evidence: .sisyphus/evidence/task-8-retry-policy-error.txt
  ```

  **Commit**: NO

- [x] 9. 정산 게이트 연계 시나리오 확정(on_hold/pending)

  **What to do**:
  - 검증 Tier별로 `ReleaseEscrowAsync`와 `ResumeHeldSettlementsAsync`의 동작 매트릭스를 확정한다.
  - 인증 실패/만료/재검증 상황에서 정산 상태 전이를 정의한다.

  **Must NOT do**:
  - `on_hold` 상태를 우회하는 임시 예외를 만들지 않는다.

  **Recommended Agent Profile**:
  - **Category**: `deep`
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `writing`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential after T3
  - **Blocks**: T13, T15
  - **Blocked By**: T3

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs`
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BackgroundServices/SettlementProcessingService.cs`

  **Acceptance Criteria**:
  - [ ] Tier별 정산상태 전이표 작성
  - [ ] 실패/만료/재시도 시나리오 포함

  **QA Scenarios**:
  ```
  Scenario: Hold to pending transition rule
    Tool: Bash (curl)
    Preconditions: 시나리오 문서 작성
    Steps:
      1. 미인증 계좌로 구매확정 시 `on_hold` 전이 규칙 확인
      2. 인증 성공 시 `pending` 전이 규칙 확인
      3. 배치 처리 대상 조건 확인
    Expected Result: 상태 전이가 일관됨
    Failure Indicators: 전이 조건 모호/누락
    Evidence: .sisyphus/evidence/task-9-settlement-gate.txt

  Scenario: Negative - bypass hold
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. 미인증 상태에서 `completed` 직접 전이 허용 여부 확인
      2. 허용 시 실패
    Expected Result: 직접 전이 금지
    Evidence: .sisyphus/evidence/task-9-settlement-gate-error.txt
  ```

  **Commit**: NO

- [x] 10. 모바일 상태/문구/화면전환 변경안 정의

  **What to do**:
  - `verificationTier`, `reasonCode`, `nextAction` 기반 UI 상태머신을 정의한다.
  - 프로필/판매대시보드 경고 문구를 provider-aware 메시지로 정리한다.

  **Must NOT do**:
  - 기존 라우트 경로를 불필요하게 변경하지 않는다.

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
  - **Skills**: [`frontend-ui-ux`]
  - **Skills Evaluated but Omitted**: `playwright` (설계 단계)

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (T11과 병렬)
  - **Blocks**: T14, T15
  - **Blocked By**: T7

  **References**:
  - `ticket_platform_mobile/lib/features/bank_account/presentation/viewmodels/bank_account_viewmodel.dart`
  - `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart`
  - `ticket_platform_mobile/lib/features/sales_dashboard/presentation/ui_models/event_ticket_ui_model.dart`

  **Acceptance Criteria**:
  - [ ] 상태머신 다이어그램(미인증/검증중/검증성공/실패) 작성
  - [ ] 문구 맵(사유코드 -> 사용자 메시지) 작성

  **QA Scenarios**:
  ```
  Scenario: UI state map coverage
    Tool: Bash
    Preconditions: 모바일 설계 문서 작성
    Steps:
      1. 각 상태별 화면/버튼/문구 확인
      2. reasonCode별 안내문구 매핑 확인
      3. 계좌삭제/재인증 분기 확인
    Expected Result: 상태별 UX가 비어있지 않음
    Failure Indicators: 상태 누락 또는 동일 문구 남발
    Evidence: .sisyphus/evidence/task-10-mobile-state.txt

  Scenario: Negative - dead-end flow
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. 실패 상태에서 재시도 경로 존재 여부 확인
      2. 없으면 실패
    Expected Result: 실패 상태에서도 next action 제공
    Evidence: .sisyphus/evidence/task-10-mobile-state-error.txt
  ```

  **Commit**: NO

- [x] 11. 점진 롤아웃 + 롤백 시나리오 정의

  **What to do**:
  - 신규계좌 우선 적용, 기존계좌 유지, 강제 재검증 시점 등 단계별 rollout을 정의한다.
  - `Hybrid -> Custom` 즉시 롤백 절차(설정 전환, 데이터 무결성 체크)를 명시한다.

  **Must NOT do**:
  - DB 롤백을 전제한 운영 절차를 기본 경로로 두지 않는다.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `git-master`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (T10과 병렬)
  - **Blocks**: T16
  - **Blocked By**: T4

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/appsettings.json`
  - `TicketPlatFormServer/TicketPlatFormServer/appsettings.TossPayments.json`
  - `.sisyphus/plans/settlement-backend.md`

  **Acceptance Criteria**:
  - [ ] 단계별 전환 기준(트래픽/실패율/장애시간) 명시
  - [ ] 10분 내 롤백 runbook 작성

  **QA Scenarios**:
  ```
  Scenario: Rollback runbook executable
    Tool: Bash
    Preconditions: runbook 작성 완료
    Steps:
      1. 설정값 전환 단계 확인
      2. 전환 후 확인 커맨드(dotnet build/health endpoint) 확인
      3. 데이터 무결성 체크 쿼리 확인
    Expected Result: 운영자가 즉시 실행 가능한 절차
    Failure Indicators: 확인 커맨드/검증 쿼리 누락
    Evidence: .sisyphus/evidence/task-11-rollout-runbook.txt

  Scenario: Negative - rollback ambiguity
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. 장애 시 first action 미정의 여부 확인
      2. 미정의 시 실패
    Expected Result: 장애시 첫 조치가 명확
    Evidence: .sisyphus/evidence/task-11-rollout-runbook-error.txt
  ```

  **Commit**: NO

- [x] 12. 통합 QA 시나리오 및 증적 포맷 고정

  **What to do**:
  - backend/mobile/e2e를 묶은 시나리오 세트를 정의한다.
  - 증적 파일 네이밍과 PASS/FAIL 판정 기준을 고정한다.

  **Must NOT do**:
  - 사람의 주관적 판정("잘 동작해 보임")을 허용하지 않는다.

  **Recommended Agent Profile**:
  - **Category**: `deep`
  - **Skills**: [`playwright`]
  - **Skills Evaluated but Omitted**: `frontend-ui-ux`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential after T6, T9, T10
  - **Blocks**: T15
  - **Blocked By**: T6, T9, T10

  **References**:
  - `.sisyphus/evidence/`
  - `ticket_platform_mobile/lib/features/bank_account/presentation/views/bank_account_verify_view.dart`
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/BankAccountController.cs`

  **Acceptance Criteria**:
  - [ ] 최소 6개 시나리오(행복경로 3, 실패경로 3) 정의
  - [ ] 증적 저장 규칙(`task-{N}-{slug}`) 명시

  **QA Scenarios**:
  ```
  Scenario: Scenario catalog completeness
    Tool: Bash
    Preconditions: QA 카탈로그 작성
    Steps:
      1. happy path 3개 이상 확인
      2. error path 3개 이상 확인
      3. 각 시나리오 evidence path 존재 확인
    Expected Result: 시나리오 최소 개수 및 포맷 충족
    Failure Indicators: error 시나리오 부족, evidence path 누락
    Evidence: .sisyphus/evidence/task-12-qa-catalog.txt

  Scenario: Negative - unverifiable criteria
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. "수동 확인" 문구 포함 여부 검색
      2. 발견 시 실패
    Expected Result: 수동 확인 의존 없음
    Evidence: .sisyphus/evidence/task-12-qa-catalog-error.txt
  ```

  **Commit**: NO

- [x] 13. 백엔드 구현 실행

  **What to do**:
  - Provider 추상화, 토스 계좌인증 클라이언트, 서비스 라우팅, DB 반영을 구현한다.
  - 기존 v1 API를 유지하고 확장 필드는 선택값으로 추가한다.

  **Must NOT do**:
  - Controller에 비즈니스 로직 추가 금지.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `frontend-ui-ux`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential after T5,T6,T7,T8,T9
  - **Blocks**: T15, T16
  - **Blocked By**: T5, T6, T7, T8, T9

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs`
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/TossPaymentsService.cs`
  - `TicketPlatFormServer/TicketPlatFormServer/Program.cs`

  **Acceptance Criteria**:
  - [ ] `dotnet build --project TicketPlatFormServer/TicketPlatFormServer.sln` PASS
  - [ ] 계좌인증 API가 provider 설정에 따라 분기 동작

  **QA Scenarios**:
  ```
  Scenario: Provider branch runtime
    Tool: Bash (curl)
    Preconditions: 서버 기동, Provider=Hybrid
    Steps:
      1. verify/request 호출
      2. 응답에 provider/tier 필드 확인
      3. 설정값 변경 후 재호출하여 분기 변화 확인
    Expected Result: 설정별 분기 동작 확인
    Failure Indicators: 설정 변경에도 동일 경로 고정
    Evidence: .sisyphus/evidence/task-13-backend-provider.txt

  Scenario: Negative - toss timeout fallback
    Tool: Bash (curl)
    Preconditions: 토스 호출 실패를 모의(타임아웃)
    Steps:
      1. verify/request 호출
      2. fallback 경로 응답/에러코드 확인
    Expected Result: 서비스 장애 없이 정의된 fallback 수행
    Evidence: .sisyphus/evidence/task-13-backend-provider-error.txt
  ```

  **Commit**: YES
  - Message: `feat(bank-account): add provider-based verification strategy`

- [x] 14. 모바일 구현 실행

  **What to do**:
  - DTO/상태/UI를 v2 확장 필드에 맞춰 업데이트한다.
  - 기존 화면 흐름을 유지하면서 provider-aware 안내를 반영한다.

  **Must NOT do**:
  - 생성 파일(`*.g.dart`, `*.freezed.dart`) 직접 수정 금지.

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
  - **Skills**: [`frontend-ui-ux`]
  - **Skills Evaluated but Omitted**: `git-master`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential after T7,T10
  - **Blocks**: T15
  - **Blocked By**: T7, T10

  **References**:
  - `ticket_platform_mobile/lib/features/bank_account/presentation/viewmodels/bank_account_viewmodel.dart`
  - `ticket_platform_mobile/lib/features/bank_account/data/dto/response/verify_account_resp_dto.dart`
  - `ticket_platform_mobile/lib/features/sales_dashboard/presentation/widgets/event_ticket_item.dart`

  **Acceptance Criteria**:
  - [ ] `flutter analyze` 신규 error 0
  - [ ] 인증실패 사유코드 기반 문구 노출

  **QA Scenarios**:
  ```
  Scenario: Mobile verify flow with reason code
    Tool: Playwright
    Preconditions: 앱 실행, 미인증 계정
    Steps:
      1. 프로필 > 계좌인증 진입
      2. 인증 요청 후 실패 응답 모의
      3. 사유코드별 안내문구/재시도 버튼 노출 확인
    Expected Result: 사용자 행동 가능한 안내 제공
    Failure Indicators: generic 오류만 표시, 재시도 경로 없음
    Evidence: .sisyphus/evidence/task-14-mobile-reason-code.png

  Scenario: Negative - broken old flow
    Tool: Playwright
    Preconditions: 기존 verified 계정
    Steps:
      1. 계좌 상세 화면 진입
      2. 기존 정보 조회/삭제 동작 확인
    Expected Result: 기존 기능 회귀 없음
    Evidence: .sisyphus/evidence/task-14-mobile-reason-code-error.png
  ```

  **Commit**: YES
  - Message: `feat(mobile): support provider-aware account verification UX`

- [x] 15. 통합 검증 실행

  **What to do**:
  - API + 모바일 + 정산 상태전이를 end-to-end로 검증한다.
  - 증적을 task 단위로 수집하고 실패 케이스를 재현 가능하게 기록한다.

  **Must NOT do**:
  - 증적 없이 "통과" 처리 금지.

  **Recommended Agent Profile**:
  - **Category**: `deep`
  - **Skills**: [`playwright`]
  - **Skills Evaluated but Omitted**: `writing`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential after T6,T9,T10,T12,T13,T14
  - **Blocks**: T16
  - **Blocked By**: T6, T9, T10, T12, T13, T14

  **References**:
  - `.sisyphus/evidence/task-12-qa-catalog.txt`
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs`
  - `ticket_platform_mobile/lib/features/sales_dashboard/presentation/ui_models/event_ticket_ui_model.dart`

  **Acceptance Criteria**:
  - [ ] 6개 이상 시나리오 증적 확보
  - [ ] 정산 상태 전이(`on_hold -> pending`) 실증

  **QA Scenarios**:
  ```
  Scenario: End-to-end verification to settlement release
    Tool: Bash + Playwright
    Preconditions: 서버/앱 실행, 테스트 계정 준비
    Steps:
      1. 계좌 미인증 상태로 구매확정 실행
      2. 정산 상태 `on_hold` 확인
      3. 계좌 인증 성공 후 상태 `pending` 전환 확인
    Expected Result: 정책대로 상태 전이
    Failure Indicators: 상태 고착 또는 잘못된 완료 처리
    Evidence: .sisyphus/evidence/task-15-e2e-settlement.txt

  Scenario: Negative - verification failure retains hold
    Tool: Bash
    Preconditions: 인증 실패 입력
    Steps:
      1. 잘못된 코드/식별자 입력
      2. 정산 상태가 여전히 `on_hold`인지 확인
    Expected Result: 실패 시 정산 해제 금지
    Evidence: .sisyphus/evidence/task-15-e2e-settlement-error.txt
  ```

  **Commit**: NO

- [x] 16. 운영 전환 리허설

  **What to do**:
  - 스테이징에서 Provider 전환, 장애 모의, 롤백까지 리허설한다.
  - 운영 체크리스트와 커뮤니케이션 템플릿을 확정한다.

  **Must NOT do**:
  - 실제 운영 키로 무통제 호출 금지.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**: `frontend-ui-ux`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential after T8,T11,T13,T15
  - **Blocks**: Final Wave
  - **Blocked By**: T8, T11, T13, T15

  **References**:
  - `.sisyphus/plans/toss-bank-account-gap-analysis.md`
  - `TicketPlatFormServer/TicketPlatFormServer/appsettings.TossPayments.json`

  **Acceptance Criteria**:
  - [ ] 리허설 결과 리포트와 go/no-go 기준 문서화
  - [ ] 롤백 소요시간 10분 이내 확인

  **QA Scenarios**:
  ```
  Scenario: Cutover rehearsal success
    Tool: Bash
    Preconditions: 스테이징 환경 준비
    Steps:
      1. Provider=Hybrid 적용 후 smoke test 실행
      2. Provider=Toss 전환 후 smoke test 실행
      3. Provider=Custom 롤백 후 smoke test 실행
    Expected Result: 각 단계 서비스 정상
    Failure Indicators: 전환 후 API 실패율 급증
    Evidence: .sisyphus/evidence/task-16-cutover-rehearsal.txt

  Scenario: Negative - rollback failure
    Tool: Bash
    Preconditions: 동일
    Steps:
      1. 롤백 절차 수행
      2. health check 실패 시 즉시 원인 기록
    Expected Result: 실패 원인/복구 절차 문서화
    Evidence: .sisyphus/evidence/task-16-cutover-rehearsal-error.txt
  ```

  **Commit**: NO

---

## Final Verification Wave (MANDATORY)

- [x] F1. **Plan Compliance Audit** — `oracle`
  - Must Have/Must NOT Have 충족 여부를 파일/응답/증적 기준으로 감사
  - Output: `Must Have [N/N] | Must NOT Have [N/N] | VERDICT`

- [x] F2. **Code Quality Review** — `unspecified-high`
  - `dotnet build`, `flutter analyze`, lint 결과와 민감정보 로그 노출 여부 점검
  - Output: `Build [PASS/FAIL] | Analyze [PASS/FAIL] | VERDICT`

- [x] F3. **Real QA Replay** — `unspecified-high` (+ `playwright`)
  - 각 Task QA 시나리오를 재실행하고 증적 경로 존재 여부 검증
  - Output: `Scenarios [N/N] | Evidence [N/N] | VERDICT`

- [x] F4. **Scope Fidelity Check** — `deep`
  - 계획 범위 외 변경(스코프 크립)과 누락 여부 검토
  - Output: `Scope [PASS/FAIL] | Missing [N] | VERDICT`

---

## Commit Strategy

- 1: `docs(plan): add toss bank-account gap integration plan` — `.sisyphus/plans/toss-bank-account-gap-analysis.md`

---

## Success Criteria

### Verification Commands
```bash
dotnet build --project TicketPlatFormServer/TicketPlatFormServer.sln
flutter analyze
```

### Final Checklist
- [ ] 토스 계좌인증 5개 endpoint 대비 구현/미구현 매핑 완료
- [ ] Provider 전환 전략(Custom/Toss/Hybrid) 문서화 완료
- [ ] 정산 `on_hold -> pending` 연계 시나리오 검증 완료
- [ ] 롤백 절차(설정 전환 + 데이터 무결성) 확인 완료
