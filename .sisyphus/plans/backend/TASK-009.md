# TASK-009 정산 스케줄러 — Backend 계획

## TL;DR

> **요약**: D+1 정산 스케줄러를 구현하여 구매확정 익일(공휴일 건너뜀) 판매자에게 Toss Payments 지급대행 API로 자동 송금한다.
> 실패 시 최대 5회 고정 간격(1h) 재시도하며, 정산 완료 시 FCM 발송한다.
>
> **산출물**:
> - `SettlementProcessingService` — BackgroundService (1시간 주기 실행)
> - `SettlementRepository` — 대기 정산 조회, 상태 변경, 재시도 카운트 관리
> - `SettlementService` — 정산 비즈니스 로직 (Toss API 호출, 공휴일 판단, 상태 전이)
> - `TossPayoutService` — Toss Payments 지급대행 API 클라이언트
> - `GET /api/settlements` — 판매자 정산 내역 조회 API
> - `GET /api/settlements/{id}` — 정산 상세 조회 API
> - DB 마이그레이션 (공휴일 테이블, notification_types 시드)
> - `PaymentService.ReleaseEscrowAsync()` — ScheduledAt 계산 로직 개선
>
> **예상 공수**: High
> **병렬 실행**: YES — 3 Wave

---

## Context

### ⚠️ 작업 기준 디렉토리
**모든 소스 파일은 `TicketPlatFormServer/TicketPlatFormServer/` 아래에 위치합니다.**
- 예: Controller → `TicketPlatFormServer/TicketPlatFormServer/Controllers/`
- 예: DB 덤프 → `TicketPlatFormServer/TicketPlatFormServer/database_history/`
- 빌드 명령: `dotnet build TicketPlatFormServer` (솔루션 파일 기준, 루트에서 실행)

### 기획 확정 사항
| 항목 | 결정 |
|------|------|
| 정산 지연 일수 | D+1 기본, 공휴일이면 건너뛰고 바로 정산 |
| 은행 송금 연동 | Toss Payments 지급대행 API 연동 (실제 송금) |
| 모바일 UI 범위 | 중간: 프로필에서 '정산 내역' 메뉴 → 정산 리스트 화면 |
| 재시도 정책 | 최대 5회, 고정 간격 1시간 |
| 정산 완료 알림 | FCM만 (채팅방 메시지 없음) |

### 현재 구현 상태
- `Settlement` 테이블 이미 존재 (Id, TransactionId, SellerId, Amount, Fee, NetAmount, BankAccountId, StatusId, ScheduledAt, ProcessedAt, FailureReason, RetryCount)
- `SettlementStatus` 시드: pending, processing, completed, failed
- `BankAccount` 테이블 이미 존재 (UserId, BankName, AccountNumber, AccountHolder, Verified)
- `PaymentService.ReleaseEscrowAsync()` (line 458-489)에서 Settlement 레코드 생성 중:
  - `ScheduledAt = DateTime.UtcNow.AddDays(1)` — ⚠️ 공휴일 미고려
  - `BankAccountId = defaultBankAccount?.Id ?? 0` — ⚠️ 0 fallback 문제
- Toss Payments 설정 존재: `appsettings.TossPayments.json` (SecretKey, ApiBaseUrl)
- BackgroundService 패턴: `TransactionReservationCleanupService.cs` (1시간 주기, scope 생성, 개별 에러 catch)

### 참조 패턴
- `TicketPlatFormServer/TicketPlatFormServer/Services/BackgroundServices/TransactionReservationCleanupService.cs` — BackgroundService 구현 패턴
- `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs` — Toss API HttpClient 호출 패턴, EF 트랜잭션 패턴
- `TicketPlatFormServer/TicketPlatFormServer/Repository/Payment/PaymentRepository.cs` — Repository 구현 패턴 (Dapper + EF Core, IMemoryCache)
- `TicketPlatFormServer/TicketPlatFormServer/Repository/Dispute/DisputeRepository.cs` — Repository 인터페이스/구현체 패턴

---

## Work Objectives

### 핵심 목표
`ScheduledAt <= NOW`이고 `status = pending`인 Settlement 레코드를 주기적으로 조회하여,
Toss Payments 지급대행 API로 판매자 계좌에 실제 송금하고, 상태를 completed로 갱신한다.
실패 시 최대 5회 재시도하며, 완료 시 FCM을 발송한다.

### Must Have
- 1시간 주기 BackgroundService (`TransactionReservationCleanupService` 패턴)
- `ScheduledAt <= DateTime.UtcNow && StatusId == pending` 조건으로 대기 정산 배치 조회
- 멱등성: 동일 Settlement 중복 처리 방지 (status 체크 + 낙관적 잠금)
- Toss Payments 지급대행 API 호출 (POST, SecretKey 인증)
- 실패 시 `RetryCount++`, `FailureReason` 기록, 다음 주기에 재시도
- `RetryCount >= 5` 도달 시 status → `failed`, 더 이상 재시도하지 않음
- 공휴일 테이블: `korean_holidays(date DATE PRIMARY KEY, name VARCHAR(50))`
- `ScheduledAt` 계산: D+1 기준, 공휴일이면 당일 즉시 정산 (건너뛰기)
- `PaymentService.ReleaseEscrowAsync()` 내 `ScheduledAt` 계산 로직 개선
 `BankAccountId = 0` 유지 → 인증 계좌 없으면 Settlement은 생성하되, 스케줄러 처리 시 `BankAccountId == 0`이면 실패 처리 (`"인증된 정산 계좌가 없습니다"`) + 로그 경고. 판매자가 이후 계좌 등록하면 재시도 시 성공 가능.
- 정산 완료 시 판매자에게 FCM 발송 (`SETTLEMENT_COMPLETED` 타입)
- `GET /api/settlements` — 판매자 본인의 정산 내역 목록 (페이지네이션)
- `GET /api/settlements/{id}` — 정산 상세 조회

### Must NOT Have (가드레일)
- 수동 정산 트리거 API (어드민 기능 별도)
- 정산 취소/환불 API (별도 Task)
- 실시간 정산 (WebSocket/SignalR 연동) — FCM만
- 공휴일 자동 크롤링/API 연동 — 수동 시드 데이터로 관리
- Toss 지급대행 Webhook 수신 (이번 스코프에서는 동기 API 결과로만 판단)
- 복수 계좌 선택 UI (기본 인증 계좌 1개만 사용)

---

## Verification Strategy

### 테스트 인프라
- **자동 테스트**: 없음 (프로젝트에 테스트 프로젝트 미존재)
- **검증 방법**: Agent-Executed QA (curl 기반 API 테스트)

### QA 정책
모든 Task의 QA 시나리오는 curl로 실행 가능해야 하며, 응답 body를 `.sisyphus/evidence/` 경로에 저장한다.

---

## Execution Strategy

### 병렬 실행 Wave

```
Wave 1 (즉시 시작 — DB + DTO + Config 기반 작업):
├── Task 1: DB 마이그레이션 (공휴일 테이블, notification_types 시드)
├── Task 2: DTO 클래스 생성 (Settlement 관련 Request/Response)
└── Task 3: TossPayments 설정 확장 + TossPayoutService 클라이언트

Wave 2 (Wave 1 완료 후 — 핵심 로직):
├── Task 4: SettlementRepository 구현 (depends: 1)
├── Task 5: PaymentService.ReleaseEscrowAsync() ScheduledAt 개선 (depends: 4)
└── Task 6: SettlementService 구현 (depends: 2, 3, 4)

Wave 3 (Wave 2 완료 후 — 통합):
├── Task 7: SettlementProcessingService (BackgroundService) 구현 (depends: 6)
├── Task 8: SettlementController API 엔드포인트 구현 (depends: 6)
└── Task 9: FCM 알림 연동 (depends: 6)

Critical Path: Task 1 → Task 4 → Task 6 → Task 7
```

### Agent Dispatch
- **Wave 1**: Task 1 → `quick`, Task 2 → `quick`, Task 3 → `unspecified-high`
- **Wave 2**: Task 4 → `unspecified-high`, Task 5 → `quick`, Task 6 → `unspecified-high`
- **Wave 3**: Task 7 → `unspecified-high`, Task 8 → `unspecified-high`, Task 9 → `quick`

---

## TODOs

---

- [ ] 1. DB 마이그레이션 — 공휴일 테이블 + notification_types 시드

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/database_history/TASK-009-migration.sql` 파일 생성:
    ```sql
    -- 1. 공휴일 테이블 생성
    CREATE TABLE IF NOT EXISTS korean_holidays (
        date DATE NOT NULL PRIMARY KEY,
        name VARCHAR(50) NOT NULL
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

    -- 2. 2025~2026년 공휴일 시드 데이터
    INSERT INTO korean_holidays (date, name) VALUES
    -- 2025년
    ('2025-01-01', '신정'),
    ('2025-01-28', '설날 전날'),
    ('2025-01-29', '설날'),
    ('2025-01-30', '설날 다음날'),
    ('2025-03-01', '삼일절'),
    ('2025-05-05', '어린이날'),
    ('2025-05-06', '대체공휴일(어린이날)'),
    ('2025-06-06', '현충일'),
    ('2025-08-15', '광복절'),
    ('2025-10-03', '개천절'),
    ('2025-10-05', '추석 전날'),
    ('2025-10-06', '추석'),
    ('2025-10-07', '추석 다음날'),
    ('2025-10-08', '대체공휴일(추석)'),
    ('2025-10-09', '한글날'),
    ('2025-12-25', '성탄절'),
    -- 2026년
    ('2026-01-01', '신정'),
    ('2026-02-16', '설날 전날'),
    ('2026-02-17', '설날'),
    ('2026-02-18', '설날 다음날'),
    ('2026-03-01', '삼일절'),
    ('2026-03-02', '대체공휴일(삼일절)'),
    ('2026-05-05', '어린이날'),
    ('2026-05-24', '석가탄신일'),
    ('2026-06-06', '현충일'),
    ('2026-08-15', '광복절'),
    ('2026-08-17', '대체공휴일(광복절)'),
    ('2026-09-24', '추석 전날'),
    ('2026-09-25', '추석'),
    ('2026-09-26', '추석 다음날'),
    ('2026-10-03', '개천절'),
    ('2026-10-05', '대체공휴일(개천절)'),
    ('2026-10-09', '한글날'),
    ('2026-12-25', '성탄절')
    ON DUPLICATE KEY UPDATE name = VALUES(name);

    -- 3. notification_types 시드 (정산 완료)
    INSERT INTO notification_types (code, name_ko, is_active)
    VALUES ('SETTLEMENT_COMPLETED', '정산 완료', 1)
    ON DUPLICATE KEY UPDATE is_active = 1;

    -- 4. notification_types 시드 (정산 실패)
    INSERT INTO notification_types (code, name_ko, is_active)
    VALUES ('SETTLEMENT_FAILED', '정산 실패', 1)
    ON DUPLICATE KEY UPDATE is_active = 1;
    ```
  - EF Core DBModel에 `KoreanHoliday` 엔티티 추가:
    ```csharp
    // TicketPlatFormServer/TicketPlatFormServer/DBModel/KoreanHoliday.cs
    public partial class KoreanHoliday
    {
        public DateOnly Date { get; set; }
        public string Name { get; set; } = null!;
    }
    ```
  - `TicketContext.cs`에 `DbSet<KoreanHoliday>` 추가 및 `OnModelCreating`에 테이블 매핑:
    ```csharp
    public virtual DbSet<KoreanHoliday> KoreanHolidays { get; set; }
    // OnModelCreating:
    modelBuilder.Entity<KoreanHoliday>(entity =>
    {
        entity.ToTable("korean_holidays");
        entity.HasKey(e => e.Date);
        entity.Property(e => e.Date).HasColumnName("date");
        entity.Property(e => e.Name).HasColumnName("name").HasMaxLength(50);
    });
    ```

  **Must NOT do**:
  - EF Core 마이그레이션 파일 생성 금지 (프로젝트는 수동 SQL 방식 사용)
  - 기존 Settlement/SettlementStatus 테이블 구조 변경 금지 (이미 충분)
  - 기존 데이터 삭제 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (Task 2, 3과 동시)
  - **Blocks**: Task 4, 5, 6
  - **Blocked By**: 없음

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/database_history/` — 기존 덤프 파일 구조 참조
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/Settlement.cs` — Settlement 엔티티 확인
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/SettlementStatus.cs` — 상태 코드 확인
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/TicketContext.cs` — DbSet 추가 위치

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: DB 마이그레이션 적용 확인
    Tool: Bash (mysql)
    Steps:
      1. mysql -u root -p tickethub -e "SELECT COUNT(*) FROM korean_holidays;"
      2. mysql -u root -p tickethub -e "SELECT * FROM korean_holidays WHERE date = '2026-02-17';"
      3. mysql -u root -p tickethub -e "SELECT * FROM notification_types WHERE code IN ('SETTLEMENT_COMPLETED', 'SETTLEMENT_FAILED');"
    Expected Result:
      1. 30+ 행 반환
      2. name='설날' 레코드 존재
      3. SETTLEMENT_COMPLETED, SETTLEMENT_FAILED 레코드 2건
    Evidence: .sisyphus/evidence/task-009-1-db-migration.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-1-*.txt`
  **Commit**: YES (groups with Task 2)
  - Message: `chore(db): add korean_holidays table and settlement notification seeds`
  - Files: `TicketPlatFormServer/TicketPlatFormServer/database_history/TASK-009-migration.sql`, `TicketPlatFormServer/TicketPlatFormServer/DBModel/KoreanHoliday.cs`, `TicketPlatFormServer/TicketPlatFormServer/Repository/TicketContext.cs`

---

- [ ] 2. DTO 클래스 생성 — Settlement 관련

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Settlement/` 디렉토리 생성
  - `SettlementRespDto.cs` 작성 (정산 내역 리스트 아이템):
    ```csharp
    public record SettlementRespDto(
        long Id,
        long TransactionId,
        string EventTitle,           // 거래 대상 이벤트명 (JOIN 필요)
        int Amount,                  // 총 금액
        int Fee,                     // 수수료
        int NetAmount,               // 순 정산 금액
        string Status,               // pending, processing, completed, failed
        string? StatusNameKo,        // 대기, 처리중, 완료, 실패
        DateTime ScheduledAt,        // 정산 예정일
        DateTime? ProcessedAt,       // 정산 완료 시각
        string? FailureReason,       // 실패 사유
        int RetryCount,              // 재시도 횟수
        DateTime CreatedAt           // 생성 시각
    );
    ```
  - `SettlementListRespDto.cs` 작성 (페이지네이션):
    ```csharp
    public record SettlementListRespDto(
        IEnumerable<SettlementRespDto> Items,
        int TotalCount,
        int TotalNetAmount           // 전체 정산 합계 (completed만)
    );
    ```
  - `SettlementDetailRespDto.cs` 작성 (상세):
    ```csharp
    public record SettlementDetailRespDto(
        long Id,
        long TransactionId,
        string EventTitle,
        string BuyerNickname,        // 구매자 닉네임
        int Amount,
        int Fee,
        int NetAmount,
        string Status,
        string? StatusNameKo,
        string BankName,             // 정산 계좌 은행명
        string AccountNumber,        // 계좌번호 (마스킹: 뒤 4자리 제외 *)
        string AccountHolder,        // 예금주
        DateTime ScheduledAt,
        DateTime? ProcessedAt,
        string? FailureReason,
        int RetryCount,
        DateTime CreatedAt
    );
    ```

  **Must NOT do**:
  - DTO에 비즈니스 로직 포함 금지
  - Entity 직접 참조 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (Task 1, 3과 동시)
  - **Blocks**: Task 6, 8
  - **Blocked By**: 없음

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Dispute/` — DTO 파일 구조 및 네이밍 패턴 참조
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Payment/` — 기존 Payment DTO 스타일 참조

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: DTO 컴파일 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer --no-restore
    Expected Result: Build succeeded, 0 Error(s)
    Evidence: .sisyphus/evidence/task-009-2-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-2-build.txt`
  **Commit**: YES (groups with Task 1)
  - Message: `chore(db): add korean_holidays table and settlement notification seeds`
  - Files: `TicketPlatFormServer/TicketPlatFormServer/DTO/Settlement/*.cs`

---

- [ ] 3. TossPayments 설정 확장 + TossPayoutService 클라이언트

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/Config/TossPaymentsSettings.cs` 확인 및 보강:
    - 기존 설정을 그대로 사용 (SecretKey, ApiBaseUrl)
    - Toss Payments 지급대행 API는 동일한 SecretKey + ApiBaseUrl 사용
    - **추가 설정 필요**: `SecurityKey` (64자 Hex) — JWE 암호화용 (Toss Developer Center에서 발급)
    - `appsettings.TossPayments.json`에 추가:
      ```json
      {
        "TossPayments": {
          "SecurityKey": "your-64-char-hex-security-key-here",
          // ... 기존 설정 유지
        }
      }
      ```
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Settlement/TossPayoutService.cs` 생성:
    ```csharp
    /// <summary>
    /// Toss Payments 지급대행 API 클라이언트
    /// </summary>
    public interface ITossPayoutService
    {
        /// <summary>
        /// 판매자 계좌로 정산금 송금
        /// </summary>
        /// <param name="bankCode">은행 코드 (예: "004" = KB국민)</param>
        /// <param name="accountNumber">계좌번호</param>
        /// <param name="holderName">예금주</param>
        /// <param name="amount">송금 금액 (원)</param>
        /// <param name="settlementId">정산 ID (멱등키로 사용)</param>
        /// <returns>송금 결과</returns>
        Task<TossPayoutResult> TransferAsync(
            string bankCode,
            string accountNumber,
            string holderName,
            int amount,
            long settlementId
        );
    }
    public record TossPayoutResult(
        bool Success,
        string? PayoutId,         // Toss 발급 지급 ID
        string? ErrorCode,        // 실패 시 오류 코드
        string? ErrorMessage      // 실패 시 오류 메시지
    );
    ```
  - `TossPayoutService` 구현:
    - `IHttpClientFactory`를 DI로 주입 (Named client: `"TossPayout"`)
    - Authorization 헤더: `Basic {Base64(SecretKey + ":")}` (기존 PaymentService의 Toss API 호출 패턴과 동일)
    - **추가 헤더**: `TossPayments-api-security-mode: ENCRYPTION` (JWE 암호화 필수)
    - **Idempotency-Key**: `settlement-{settlementId}` (중복 송금 방지)
    - **실제 API 호출** (Toss v2 지급대행):
      ```
      POST {ApiBaseUrl}/v2/payouts
      Content-Type: application/json
      Authorization: Basic {base64(secretKey:)}
      TossPayments-api-security-mode: ENCRYPTION
      Idempotency-Key: settlement-{settlementId}
      // Body는 JWE(A256GCM) 암호화 필요 (SecurityKey 사용)
      // 암호화 전 원문:
      [
        {
          "refPayoutId": "SETTLE_{settlementId}",
          "destination": "{sellerTossId}",  // Toss에 등록된 판매자 ID
          "scheduleType": "EXPRESS",
          "amount": {
            "currency": "KRW",
            "value": 50000
          },
          "transactionDescription": "티켓판매대금"
        }
      ]
      ```
    - **⚠️ 중요**: Toss 지급대행은 **별도 계약 + 판매자 사전등록(`POST /v2/sellers`)** 필요.
      현재는 **인터페이스 추상화 + 테스트 모드 stub** 패턴으로 구현하여,
      `IsTestMode = true`이면 실제 API 호출 없이 성공 응답 반환.
      프로덕션 전환 시: (1) Toss 지급대행 계약, (2) SecurityKey 발급, (3) 판매자 등록 API 연동, (4) JWE 암호화 구현.
    - **테스트 모드 동작** (`IsTestMode == true`):
      ```csharp
      if (options.IsTestMode)
      {
          logger.LogInformation("[TossPayoutService] 테스트 모드 - 정산 시뮬레이션 성공: SettlementId={Id}", settlementId);
          return new TossPayoutResult(true, $"test_payout_{settlementId}", null, null);
      }
      ```
    - **프로덕션 모드 에러 코드 처리**:
      | 에러 코드 | 의미 | 처리 |
      |-----------|------|------|
      | `INSUFFICIENT_BALANCE` | 정산 계좌 잔액 부족 | 재시도 (잔액 충전 후) |
      | `INVALID_ACCOUNT` | 판매자 계좌 무효/해지 | 실패 처리 + 판매자 알림 |
      | `BANK_MAINTENANCE` | 은행 시스템 점검 | 재시도 |
      | `ALREADY_PROCESSED_PAYOUT` | 중복 Idempotency-Key | 성공으로 간주 (멱등) |
    - **에러 처리**: HttpClient 타임아웃, HTTP 4xx/5xx, JSON 파싱 오류 모두 catch → `TossPayoutResult(false, ...)` 반환 (예외 throw 금지, 호출자가 재시도 판단)
    - **Polly 재시도**: 이 레벨에서는 적용하지 않음 (BackgroundService 레벨에서 1h 간격 재시도)
  - `Program.cs`에 HttpClient + DI 등록:
    ```csharp
    builder.Services.AddHttpClient("TossPayout", client =>
    {
        client.BaseAddress = new Uri(tossConfig.ApiBaseUrl);
        client.Timeout = TimeSpan.FromSeconds(tossConfig.TimeoutSeconds);
    });
    builder.Services.AddScoped<ITossPayoutService, TossPayoutService>();
    ```
  **Must NOT do**:
  - 기존 `PaymentService`의 Toss API 호출 코드 수정 금지 (별도 서비스로 분리)
  - SecretKey를 코드에 하드코딩 금지 (appsettings에서 읽기)
  - `IsTestMode == false`일 때도 예외를 throw하지 말 것 → `TossPayoutResult` 반환
  - JWE 암호화 로직은 테스트 모드에서는 구현하지 않음 (프로덕션 전환 시 구현)
  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: []
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (Task 1, 2와 동시)
  - **Blocks**: Task 6
  - **Blocked By**: 없음
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs` — 기존 Toss API HttpClient 호출 패턴 (Authorization 헤더, Base64 인코딩)
  - `TicketPlatFormServer/TicketPlatFormServer/appsettings.TossPayments.json` — 설정 구조
  - `TicketPlatFormServer/TicketPlatFormServer/Config/` — Options 바인딩 패턴
  - Toss Payments 지급대행 API 가이드: https://docs.tosspayments.com/guides/v2/payouts
  - Toss Payments 지급대행 API 레퍼런스: https://docs.tosspayments.com/reference/additional#지급대행-요청

  **Acceptance Criteria**:
  - [ ] `IsTestMode == true`일 때 API 호출 없이 성공 반환
  - [ ] `IsTestMode == false`일 때 HttpClient 호출 코드 존재 (실행은 Toss 계약 후)
  - [ ] Authorization 헤더가 Basic 인증 형식

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer --no-restore
    Expected Result: Build succeeded, 0 Error(s)
    Evidence: .sisyphus/evidence/task-009-3-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-3-build.txt`
  **Commit**: NO (Task 6과 함께)

---

- [ ] 4. SettlementRepository 구현

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Settlement/ISettlementRepository.cs` 인터페이스:
    ```csharp
    public interface ISettlementRepository
    {
        /// <summary>
        /// 정산 예정 시각이 현재 이전이고, pending 상태이며, 재시도 5회 미만인 정산 목록 조회
        /// </summary>
        Task<IEnumerable<Settlement>> GetPendingSettlementsAsync(DateTime now, int maxRetryCount);

        /// <summary>
        /// 정산 상태 업데이트 (processing, completed, failed)
        /// </summary>
        Task UpdateSettlementStatusAsync(long settlementId, long statusId, DateTime? processedAt, string? failureReason);

        /// <summary>
        /// 재시도 횟수 증가 + 실패 사유 기록
        /// </summary>
        Task IncrementRetryCountAsync(long settlementId, string failureReason);

        /// <summary>
        /// 특정 날짜가 공휴일인지 확인
        /// </summary>
        Task<bool> IsHolidayAsync(DateOnly date);

        /// <summary>
        /// 다음 영업일 계산 (주말 + 공휴일 건너뛰기, but 공휴일이면 즉시 정산이므로 다르게 사용)
        /// </summary>
        Task<DateTime> CalculateScheduledAtAsync(DateTime confirmedAt);

        /// <summary>
        /// 판매자 ID로 정산 목록 조회 (페이지네이션)
        /// </summary>
        Task<IEnumerable<Settlement>> GetBySellerIdAsync(long sellerId, int page, int size, string? statusFilter);

        /// <summary>
        /// 판매자 ID로 정산 총 건수 조회
        /// </summary>
        Task<int> CountBySellerIdAsync(long sellerId, string? statusFilter);

        /// <summary>
        /// 판매자의 완료 정산 합계 금액 조회
        /// </summary>
        Task<int> GetTotalCompletedNetAmountAsync(long sellerId);

        /// <summary>
        /// 정산 ID + 판매자 ID로 상세 조회 (소유권 검증 포함)
        /// </summary>
        Task<Settlement?> GetByIdAndSellerIdAsync(long settlementId, long sellerId);
    }
    ```
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Settlement/SettlementRepository.cs` 구현:
    - `GetPendingSettlementsAsync`: Dapper로 조회
      ```sql
      SELECT s.*, ss.code AS StatusCode
      FROM settlements s
      INNER JOIN settlement_statuses ss ON s.status_id = ss.id
      WHERE s.scheduled_at <= @Now
        AND ss.code = 'pending'
        AND COALESCE(s.retry_count, 0) < @MaxRetryCount
      ORDER BY s.scheduled_at ASC
      ```
    - `UpdateSettlementStatusAsync`: Dapper UPDATE
      ```sql
      UPDATE settlements
      SET status_id = @StatusId,
          processed_at = @ProcessedAt,
          failure_reason = @FailureReason,
          updated_at = @UpdatedAt
      WHERE id = @SettlementId
      ```
    - `IncrementRetryCountAsync`: Dapper
      ```sql
      UPDATE settlements
      SET retry_count = COALESCE(retry_count, 0) + 1,
          failure_reason = @FailureReason,
          updated_at = @UpdatedAt
      WHERE id = @SettlementId
      ```
    - `IsHolidayAsync`: Dapper SELECT COUNT from `korean_holidays`
    - `CalculateScheduledAtAsync`: D+1 기본. 공휴일이면 당일 즉시 정산 (공휴일 건너뛰기 = 정산 앞당기기)
      ```csharp
      // 기본: confirmedAt + 1일
      var scheduledDate = DateOnly.FromDateTime(confirmedAt.AddDays(1));
      // 주말이면 다음 월요일
      while (scheduledDate.DayOfWeek == DayOfWeek.Saturday || scheduledDate.DayOfWeek == DayOfWeek.Sunday)
          scheduledDate = scheduledDate.AddDays(1);
      // 공휴일이면 건너뛰고 바로 정산 = scheduledAt을 confirmedAt + 약간의 여유로 앞당김
      if (await IsHolidayAsync(scheduledDate))
      {
          // "공휴일이면 건너뛰고 바로 정산" = 공휴일 날에도 스케줄러가 처리하도록 ScheduledAt을 공휴일 당일로 설정
          // (스케줄러는 공휴일 여부를 무시하고 ScheduledAt <= NOW 기준으로 처리)
          return scheduledDate.ToDateTime(TimeOnly.MinValue, DateTimeKind.Utc);
      }
      return scheduledDate.ToDateTime(TimeOnly.MinValue, DateTimeKind.Utc);
      ```
      **핵심 로직 해석**: "공휴일이 있다면 건너뛰고 바로 정산"은 **공휴일이 D+1에 해당하면 공휴일을 기다리지 않고 즉시(당일) 정산**한다는 의미.
      즉, `ScheduledAt`을 공휴일 이전으로 앞당기거나, 스케줄러가 공휴일에도 실행되어 처리.
      구현: `ScheduledAt = 공휴일 당일 00:00 UTC` → 스케줄러 다음 실행 주기에 즉시 처리됨.
    - `GetBySellerIdAsync`: Dapper JOIN (settlements ← settlement_statuses ← transactions ← transaction_items ← tickets ← events) — 이벤트명 포함
    - `CountBySellerIdAsync`: Dapper COUNT
    - `GetTotalCompletedNetAmountAsync`: Dapper `SUM(net_amount) WHERE status = 'completed'`
    - `GetByIdAndSellerIdAsync`: EF Core 단건 조회 (Include Status, BankAccount, Transaction)
  - `Program.cs`에 DI 등록:
    `builder.Services.AddScoped<ISettlementRepository, SettlementRepository>();`

  **Must NOT do**:
  - AGENTS.md 규칙: `IDbConnection`은 Scoped → `Task.WhenAll` + repository 조합 절대 금지
  - 비즈니스 로직 포함 금지 (상태 전이 판단은 Service 레이어)
  - DTO 직접 참조 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (Task 5, 6과 동시)
  - **Blocks**: Task 6
  - **Blocked By**: Task 1

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Dispute/DisputeRepository.cs` — Repository 구현 패턴 (Dapper 쿼리 스타일, Primary Constructor 주입)
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Payment/PaymentRepository.cs` — Dapper + EF Core 혼용, IMemoryCache 사용 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/README.md` — Dapper + EF Core 혼용 패턴, 트랜잭션 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/Settlement.cs` — Entity 구조

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드 및 DI 등록 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer
    Expected Result: Build succeeded, 0 Error(s)
    Evidence: .sisyphus/evidence/task-009-4-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-4-build.txt`
  **Commit**: NO (Task 6과 함께)

---

- [ ] 5. PaymentService.ReleaseEscrowAsync() — ScheduledAt 계산 개선

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs` 수정 (line ~475-486):
    - 기존 `ScheduledAt = DateTime.UtcNow.AddDays(1)` → `ISettlementRepository.CalculateScheduledAtAsync()` 호출로 교체
    - `ISettlementRepository`를 PaymentService 생성자에 주입
    - 수정 코드:
      ```csharp
      // 기존: ScheduledAt = DateTime.UtcNow.AddDays(1),
      // 변경:
      ScheduledAt = await settlementRepository.CalculateScheduledAtAsync(DateTime.UtcNow),
      ```
  - `BankAccountId = defaultBankAccount?.Id ?? 0` 문제 수정:
    - 인증 계좌 없으면 `BankAccountId = 0` 대신 **Settlement은 생성하되 로그 경고 유지**
    - 스케줄러가 처리 시점에 계좌가 없으면 실패 처리 (SettlementService에서 검증)
    - 변경 코드:
      ```csharp
      // 기존: BankAccountId = defaultBankAccount?.Id ?? 0,
      // 변경:
      BankAccountId = defaultBankAccount?.Id ?? 0,  // 0이면 스케줄러에서 계좌 미등록 실패 처리
      ```
    - **결론**: `BankAccountId = 0` 유지하되, SettlementService에서 `BankAccountId == 0`이면 `"인증된 정산 계좌가 없습니다"` 실패 처리. 이렇게 하면 판매자가 나중에 계좌를 등록하면 재시도 시 성공 가능.

  **Must NOT do**:
  - `ReleaseEscrowAsync`의 기존 에스크로 해제/거래 확정 로직 수정 금지
  - Settlement 생성 자체를 막지 않기 (계좌 없어도 Settlement 레코드는 생성)

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (Task 4 완료 후 실행)
  - **Parallel Group**: Wave 2 후반 (Task 4 이후)
  - **Blocks**: 없음 (독립적)
  - **Blocked By**: Task 4 (ISettlementRepository.CalculateScheduledAtAsync 메서드 필요)

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs` (line 458-489) — Settlement 생성 코드 정확한 위치

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer
    Expected Result: Build succeeded, 0 Error(s)
    Evidence: .sisyphus/evidence/task-009-5-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-5-build.txt`
  **Commit**: YES
  - Message: `fix(payment): improve ScheduledAt calculation with holiday awareness`
  - Files: `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs`

---

- [ ] 6. SettlementService 구현

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Settlement/ISettlementService.cs` 인터페이스:
    ```csharp
    public interface ISettlementService
    {
        /// <summary>
        /// 대기 중인 정산 건들을 배치 처리
        /// </summary>
        Task ProcessPendingSettlementsAsync();

        /// <summary>
        /// 판매자 정산 목록 조회
        /// </summary>
        Task<SettlementListRespDto> GetBySellerAsync(long sellerId, int page, int size, string? statusFilter);

        /// <summary>
        /// 정산 상세 조회
        /// </summary>
        Task<SettlementDetailRespDto> GetDetailAsync(long settlementId, long sellerId);
    }
    ```
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Settlement/SettlementService.cs` 구현:

    **ProcessPendingSettlementsAsync 비즈니스 로직** (핵심):
    1. `_settlementRepo.GetPendingSettlementsAsync(DateTime.UtcNow, maxRetryCount: 5)` 호출
    2. 결과 없으면 로그 후 return
    3. **각 Settlement 순차 처리** (IDbConnection Scoped 규칙 → 병렬 금지):
       a. 상태를 `processing`으로 변경 (낙관적 잠금: 이미 processing이면 skip)
       b. `BankAccountId == 0` 검증 → 0이면 실패 처리 (`"인증된 정산 계좌가 없습니다"`)
       c. BankAccount 정보 조회 (EF Core로 BankAccount Load)
       d. `_tossPayoutService.TransferAsync()` 호출
       e. 성공: status → `completed`, `ProcessedAt = DateTime.UtcNow`
       f. 실패: `RetryCount++`, `FailureReason` 기록, status → `pending`으로 복원 (다음 주기에 재시도)
       g. `RetryCount >= 5`: status → `failed` (더 이상 재시도 안 함)
       h. 각 건 처리 결과 로깅
    4. 정산 완료 건에 대해 FCM 발송 (Task 9에서 연결)

    **에러 처리**:
    - 개별 Settlement 처리 실패 시 다른 건에 영향 없음 (foreach + try/catch)
    - `TossPayoutResult.Success == false`는 예외가 아닌 정상 응답 → 재시도 로직 적용

    **GetBySellerAsync 로직**:
    - Repository에서 페이지네이션 조회
    - Entity → DTO 변환 (이벤트명은 JOIN으로 가져옴)
    - TotalCompletedNetAmount 포함

    **GetDetailAsync 로직**:
    - `_settlementRepo.GetByIdAndSellerIdAsync()` 호출 → null이면 404
    - 계좌번호 마스킹 처리 (`accountNumber.Length > 4 ? new string('*', len-4) + accountNumber[^4..] : accountNumber`)
    - Entity → SettlementDetailRespDto 변환

  - `Program.cs`에 DI 등록:
    `builder.Services.AddScoped<ISettlementService, SettlementService>();`

  **Must NOT do**:
  - Repository 병렬 호출 금지 (`Task.WhenAll` + IDbConnection Scoped 위반)
  - 트랜잭션 없이 상태 변경 분리 실행 — 하지만 이 경우 단건 UPDATE라 트랜잭션 없이도 안전
  - Toss API 호출 시 예외 throw 금지 (TossPayoutResult로 결과 전달)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (Task 3, 4 완료 후)
  - **Parallel Group**: Wave 2 후반
  - **Blocks**: Task 7, 8, 9
  - **Blocked By**: Task 2, 3, 4

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Dispute/DisputeService.cs` — Service 구현 패턴 (AppException 사용, 순차 조회 패턴)
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs` — 트랜잭션 패턴, Entity→DTO 변환 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/Common/Exception/AppException.cs` — 예외 처리 패턴

  **Acceptance Criteria**:
  - [ ] pending 상태 Settlement → processing → completed 전이 확인
  - [ ] BankAccountId=0 → 실패 처리 + FailureReason 기록
  - [ ] RetryCount >= 5 → failed 상태 전이, 더 이상 조회되지 않음
  - [ ] 개별 실패가 다른 건에 영향 없음

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer
    Expected Result: Build succeeded, 0 Error(s)
    Evidence: .sisyphus/evidence/task-009-6-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-6-build.txt`
  **Commit**: NO (Task 7과 함께)

---

- [ ] 7. SettlementProcessingService — BackgroundService 구현

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BackgroundServices/SettlementProcessingService.cs` 생성:
    ```csharp
    /// <summary>
    /// 정산 처리 백그라운드 서비스 — 1시간 주기 실행
    /// </summary>
    public class SettlementProcessingService(
        IServiceProvider serviceProvider,
        ILogger<SettlementProcessingService> logger) : BackgroundService
    {
        private static readonly TimeSpan ProcessingInterval = TimeSpan.FromHours(1);

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            logger.LogInformation("[SettlementProcessingService] 정산 처리 서비스 시작. 실행 주기: {Hours}시간",
                ProcessingInterval.TotalHours);

            while (!stoppingToken.IsCancellationRequested)
            {
                try
                {
                    using var scope = serviceProvider.CreateScope();
                    var settlementService = scope.ServiceProvider.GetRequiredService<ISettlementService>();
                    await settlementService.ProcessPendingSettlementsAsync();
                }
                catch (Exception ex)
                {
                    logger.LogError(ex, "[SettlementProcessingService] 정산 처리 중 오류 발생");
                }

                await Task.Delay(ProcessingInterval, stoppingToken);
            }
        }
    }
    ```
  - `Program.cs`에 HostedService 등록:
    ```csharp
    builder.Services.AddHostedService<SettlementProcessingService>();
    ```
  - `TransactionReservationCleanupService.cs` 패턴을 정확히 따름:
    - Primary Constructor 패턴
    - `using var scope = serviceProvider.CreateScope();` 로 scoped 서비스 resolve
    - foreach 내 개별 try/catch (SettlementService 내부에서 이미 처리하지만, 최상위 catch도 유지)

  **Must NOT do**:
  - `TransactionReservationCleanupService` 수정 금지
  - 병렬 실행 (Task.WhenAll) 금지
  - 간격을 1시간 미만으로 설정 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (Task 8, 9와 동시)
  - **Blocks**: 없음 (최종 통합)
  - **Blocked By**: Task 6

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BackgroundServices/TransactionReservationCleanupService.cs` — 패턴 완전 참조 (Primary Constructor, scope 생성, while 루프, 개별 catch)

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인 + 서비스 등록 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer
    Expected Result: Build succeeded, 0 Error(s)
    Evidence: .sisyphus/evidence/task-009-7-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-7-build.txt`
  **Commit**: YES
  - Message: `feat(settlement): implement settlement processing background service`
  - Pre-commit: `dotnet build TicketPlatFormServer`
  - Files:
    - `TicketPlatFormServer/TicketPlatFormServer/Services/BackgroundServices/SettlementProcessingService.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/Services/Settlement/ISettlementService.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/Services/Settlement/SettlementService.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/Services/Settlement/TossPayoutService.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/Repository/Settlement/ISettlementRepository.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/Repository/Settlement/SettlementRepository.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/Program.cs`

---

- [ ] 8. SettlementController — API 엔드포인트 구현

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/SettlementController.cs` 생성:
    ```csharp
    [ApiController]
    [Route("api/settlements")]
    [Authorize]
    public class SettlementController(
        ISettlementService settlementService,
        ILogger<SettlementController> logger) : ControllerBase
    {
        // GET /api/settlements?page=1&size=20&status=completed
        // GET /api/settlements/{id}
    }
    ```
  - `GET /api/settlements` 구현:
    - `[HttpGet]`, `[Authorize]`
    - QueryString: `page=1&size=20&status=completed|pending|failed` (status는 선택)
    - JWT에서 userId 추출 → sellerId로 사용
    - 응답: `200 OK`, `ApiResponse<SettlementListRespDto>`
  - `GET /api/settlements/{id}` 구현:
    - `[HttpGet("{id}")]`, `[Authorize]`
    - 응답: `200 OK`, `ApiResponse<SettlementDetailRespDto>`
    - 다른 판매자의 정산 조회 시 404 반환 (소유권 검증)
  - Swagger XML 주석 추가 (`/// <summary>`)

  **Must NOT do**:
  - 비즈니스 로직 Controller에 포함 금지
  - Entity 직접 노출 금지
  - 정산 수동 트리거 API 구현 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (Task 7, 9와 동시)
  - **Blocks**: 없음
  - **Blocked By**: Task 6

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/DisputeController.cs` — Controller 구조 및 Swagger 주석 패턴 참조
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/PaymentController.cs` — JWT userId 추출 패턴 (`User.GetUserId()`)
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/ApiResponse.cs` — 응답 래퍼 형식 참조

  **Acceptance Criteria**:
  - [ ] `GET /api/settlements` → 판매자 본인의 정산 목록 반환 (TotalCount, TotalNetAmount 포함)
  - [ ] `GET /api/settlements?status=completed` → 완료 건만 필터링
  - [ ] `GET /api/settlements/{id}` → 상세 조회 (계좌번호 마스킹 확인)
  - [ ] 다른 판매자의 정산 ID → 404 반환

  **QA Scenarios**:

  ```
  Scenario: 정산 목록 조회 (Happy Path)
    Tool: Bash (curl)
    Preconditions:
      - 판매자 JWT: {sellerToken}
      - 해당 판매자에게 Settlement 레코드 존재
    Steps:
      1. curl -X GET http://localhost:5224/api/settlements?page=1&size=20 \
           -H "Authorization: Bearer {sellerToken}"
      2. 응답 statusCode 확인 → 200
      3. data.items 배열 확인
      4. data.totalCount, data.totalNetAmount 확인
    Expected Result:
      - 200 OK
      - items 배열에 Settlement 정보 포함
      - totalNetAmount = completed 건들의 netAmount 합
    Evidence: .sisyphus/evidence/task-009-8-list.json

  Scenario: 정산 상세 조회 — 계좌 마스킹
    Tool: Bash (curl)
    Preconditions: 위 목록에서 settlementId 확인
    Steps:
      1. curl -X GET http://localhost:5224/api/settlements/{id} \
           -H "Authorization: Bearer {sellerToken}"
    Expected Result:
      - 200 OK
      - accountNumber 마스킹 확인 (예: "****567890")
      - bankName, accountHolder 정상 반환
    Evidence: .sisyphus/evidence/task-009-8-detail.json

  Scenario: 타인 정산 조회 → 404
    Tool: Bash (curl)
    Preconditions: 다른 판매자의 settlementId
    Steps:
      1. curl -X GET http://localhost:5224/api/settlements/{otherSellerId} \
           -H "Authorization: Bearer {sellerToken}"
    Expected Result: 404 Not Found
    Evidence: .sisyphus/evidence/task-009-8-forbidden.json
  ```

  **Evidence**: `.sisyphus/evidence/task-009-8-*.json`
  **Commit**: YES
  - Message: `feat(settlement): add settlement list and detail API endpoints`
  - Pre-commit: `dotnet build TicketPlatFormServer`
  - Files:
    - `TicketPlatFormServer/TicketPlatFormServer/Controllers/SettlementController.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/DTO/Settlement/*.cs`

---

- [ ] 9. FCM 알림 연동 — 정산 완료/실패 시 발송

  **What to do**:
  - `SettlementService.ProcessPendingSettlementsAsync()` 내에서:
    - 정산 **완료** 시:
      ```csharp
      await _notificationService.CreateAndSendAsync(
          userId: settlement.SellerId,
          typeCode: "SETTLEMENT_COMPLETED",
          title: "정산 완료",
          body: $"정산금 {settlement.NetAmount:N0}원이 계좌에 입금되었습니다.",
          data: new Dictionary<string, string>
          {
              ["settlementId"] = settlement.Id.ToString(),
          }
      );
      ```
    - 정산 **최종 실패** (RetryCount >= 5) 시:
      ```csharp
      await _notificationService.CreateAndSendAsync(
          userId: settlement.SellerId,
          typeCode: "SETTLEMENT_FAILED",
          title: "정산 실패",
          body: "정산 처리에 실패했습니다. 계좌 정보를 확인해주세요.",
          data: new Dictionary<string, string>
          {
              ["settlementId"] = settlement.Id.ToString(),
          }
      );
      ```
  - FCM 발송 실패가 정산 처리에 영향을 주지 않도록 try/catch로 감싸기
  - `_notificationService`는 기존 `ChatService.cs`에서 사용하는 동일한 서비스 DI

  **Must NOT do**:
  - 채팅방 메시지 생성 금지 (FCM만)
  - FCM 실패 시 정산 상태 변경 금지 (정산은 이미 완료/실패 확정)
  - 재시도 중간 단계(1~4회차)에서 FCM 발송 금지 (최종 실패만)

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (Task 7, 8과 동시)
  - **Blocks**: 없음
  - **Blocked By**: Task 6

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Chat/ChatService.cs` — FCM 발송 패턴 (`_notificationService` 호출 방식)
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/NotificationController.cs` — 알림 타입 코드 참조

  **Acceptance Criteria**:
  - [ ] 정산 완료 시 SETTLEMENT_COMPLETED FCM 발송
  - [ ] 5회 실패 시 SETTLEMENT_FAILED FCM 발송
  - [ ] FCM 발송 실패가 정산 상태에 영향 없음

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer
    Expected Result: Build succeeded, 0 Error(s)
    Evidence: .sisyphus/evidence/task-009-9-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-9-build.txt`
  **Commit**: YES
  - Message: `feat(settlement): add FCM notification on settlement completion and failure`
  - Files: `TicketPlatFormServer/TicketPlatFormServer/Services/Settlement/SettlementService.cs`

---

## Final Verification Wave

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Must Have 12개 항목 전체 curl로 검증. 멱등성 확인 (동일 Settlement 2회 처리 시도). RetryCount 5회 후 failed 전이 확인. 공휴일 ScheduledAt 계산 확인. `.sisyphus/evidence/` 파일 존재 확인.
  Output: `Must Have [N/12] | Must NOT Have [N/6] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  `dotnet build TicketPlatFormServer` + 레이어 분리 위반 탐색 (DTO in Repository, Entity in Controller). `Task.WhenAll` + repository 병렬 호출 패턴 탐색. `as any` 다운캐스팅 오용 탐색. BackgroundService의 scope 생성 확인.
  Output: `Build [PASS/FAIL] | Layer Violations [N] | Parallel Violations [N] | VERDICT`

- [ ] F3. **Real Manual QA** — `unspecified-high`
  전체 QA 시나리오 실행. 정산 목록 → 상세 → 계좌 마스킹 엔드투엔드. BackgroundService 실행 로그 확인 (테스트 모드).
  Output: `Scenarios [N/N pass] | VERDICT`

---

## Commit Strategy

| 커밋 | 메시지 | 주요 파일 |
|------|--------|-----------|
| 1 | `chore(db): add korean_holidays table and settlement notification seeds` | `database_history/TASK-009-migration.sql`, `DBModel/KoreanHoliday.cs`, `Repository/TicketContext.cs`, `DTO/Settlement/*.cs` |
| 2 | `fix(payment): improve ScheduledAt calculation with holiday awareness` | `Services/Payment/PaymentService.cs` |
| 3 | `feat(settlement): implement settlement processing background service` | `Services/Settlement/*`, `Repository/Settlement/*`, `Services/BackgroundServices/SettlementProcessingService.cs`, `Program.cs` |
| 4 | `feat(settlement): add settlement list and detail API endpoints` | `Controllers/SettlementController.cs`, `DTO/Settlement/*.cs` |
| 5 | `feat(settlement): add FCM notification on settlement completion and failure` | `Services/Settlement/SettlementService.cs` |

---

## Success Criteria

```bash
# 빌드 확인
dotnet build TicketPlatFormServer
# Expected: Build succeeded, 0 Error(s)

# 정산 목록 조회
curl -X GET http://localhost:5224/api/settlements?page=1&size=20 \
  -H "Authorization: Bearer {sellerToken}"
# Expected: HTTP 200, {"data": {"items": [...], "totalCount": N, "totalNetAmount": N}}

# 정산 상세 조회
curl -X GET http://localhost:5224/api/settlements/{id} \
  -H "Authorization: Bearer {sellerToken}"
# Expected: HTTP 200, {"data": {"id": N, "accountNumber": "****1234", ...}}
```

### Final Checklist
- [ ] BackgroundService 1시간 주기 실행 (TransactionReservationCleanupService 패턴)
- [ ] pending 정산 배치 조회 + 순차 처리
- [ ] Toss Payments 지급대행 API 연동 (테스트 모드 stub 포함)
- [ ] 실패 시 RetryCount++, 5회 도달 시 failed 전이
- [ ] 공휴일 테이블 + D+1 ScheduledAt 계산 개선
- [ ] BankAccountId=0 시 실패 처리 (계좌 미등록)
- [ ] 정산 완료/실패 시 FCM 발송
- [ ] GET /api/settlements 판매자 정산 목록 API
- [ ] GET /api/settlements/{id} 정산 상세 API (계좌 마스킹)
- [ ] 멱등성 보장 (동일 Settlement 중복 처리 방지)
