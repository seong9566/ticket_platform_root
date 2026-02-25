# TASK-012 판매 내역 리팩토링 — Backend 계획

## TL;DR

> **요약**: 판매 내역 화면을 "판매 관리 대시보드"로 전환하기 위해 Backend API를 확장한다.
> 기존 `GET /api/sell/my-tickets`(flat 티켓 목록)는 유지하고, 새 엔드포인트 2개를 추가한다:
> (1) Event 기준 그룹핑 + 상태별 수량 집계 대시보드 API,
> (2) 특정 공연의 개별 티켓 상세 목록 API.
>
> **산출물**:
> - `GET /api/sell/sales-dashboard` — 공연별 그룹핑 판매 대시보드 (상태 필터, page/size 페이지네이션)
> - `GET /api/sell/sales-dashboard/{eventId}` — 특정 공연의 개별 티켓 + 거래 정보 목록
> - `SalesDashboardRespDto` / `EventGroupItemDto` — 그룹핑 응답 DTO
> - `EventTicketListRespDto` / `EventTicketItemDto` — 공연별 티켓 상세 응답 DTO
> - `SalesDashboardReadModel` / `EventTicketReadModel` — Dapper ReadModel
> - `SellRepository` 확장 — 2개 Dapper SQL 쿼리 추가
> - `SellService` 확장 — 대시보드 / 공연별 상세 비즈니스 로직
> - `SellController` 확장 — 2개 엔드포인트 추가
>
> **예상 공수**: Medium
> **병렬 실행**: YES — 3 Wave

---

## Context

### ⚠️ 작업 기준 디렉토리
**모든 소스 파일은 `TicketPlatFormServer/TicketPlatFormServer/` 아래에 위치합니다.**
- 예: Controller → `TicketPlatFormServer/TicketPlatFormServer/Controllers/`
- 예: DTO → `TicketPlatFormServer/TicketPlatFormServer/DTO/`
- 빌드 명령: `dotnet build TicketPlatFormServer` (솔루션 파일 기준, 루트에서 실행)

### 기획 확정 사항
| 항목 | 결정 |
|------|------|
| 화면 전환 | 기존 '판매 내역' 탭을 완전 교체 → 판매 관리 대시보드 |
| 그룹핑 기준 | `event_id` 기준 (같은 공연이면 같은 그룹) |
| 수량 단위 | 등록 건(티켓 글) 기준 COUNT (장수 합계 아님) |
| 상태 카테고리 | 총 티켓 / 판매중 / 판매 완료 / 정산 중 (4가지) |
| 정산 상태 매핑 | transaction confirmed = "정산 중", settlement completed = "판매 완료" |
| 필터 | 상태 필터만 (전체/판매중/완료/정산 중). 기간/정렬 필터 제거 |
| Backend API 전략 | 기존 sell/my-tickets 유지 + 새 엔드포인트 2개 추가 |
| 기존 API 보호 | `GET /api/transactions/sales` 건드리지 않음 |

### 근본 원인 분석
1. **신규 등록 티켓 미표시**: 판매 내역 API가 `transactions` 기준 `INNER JOIN` → 거래 없는 등록 티켓 누락
2. **같은 공연 개별 표시**: `GROUP BY` 없이 `transaction_item` 단위 flat 반환

### 상태 매핑 규칙 (4가지 카테고리)

```
총 티켓    = 해당 event_id로 등록된 내 티켓 중 ticket.status_id != 5(cancelled) 인 건수
판매중    = ticket.status_id = 1 (available)
판매 완료  = 연관 settlement.status = 'completed' 인 건수
정산 중   = 연관 transaction.status = 'confirmed' 이고 (settlement 미존재 OR settlement.status IN ('pending','processing')) 인 건수
```

> ⚠️ **Metis 가드레일**: `reserved`(예약중) 티켓은 "판매중"에 포함하지 않는다. 예약은 진행 중 거래이므로 별도 표시하지 않고 "총 티켓"에만 포함한다.

> ⚠️ **하위호환**: 기존 `GET /api/sell/my-tickets` 응답 shape/의미를 절대 변경하지 않는다. 새 엔드포인트로 분리한다.

### 현재 구현 상태
- `SellController.cs` — `GET /api/sell/my-tickets` 존재 (flat 티켓 목록, page/size)
- `SellService.cs` — `GetMyTicketsAsync()` 존재
- `SellRepository.cs` — Dapper 쿼리로 `tickets JOIN events` 조회
- `SellQueries.cs` — SQL 쿼리 상수 관리
- `MyTicketReadModel.cs` — TicketId, EventId, EventTitle, SeatGradeName, AreaName, Price, Quantity, RemainingQuantity, StatusId, StatusCode, StatusName, CreatedAt
- `MyTicketListReqDto.cs` / `MyTicketListRespDto.cs` — 요청/응답 DTO
- `Settlement` 테이블 — Id, TransactionId, SellerId, Amount, Fee, NetAmount, BankAccountId, StatusId, ScheduledAt, ProcessedAt, FailureReason, RetryCount
- `SettlementStatus` 시드 — pending, processing, completed, failed

### 참조 패턴
- `TicketPlatFormServer/TicketPlatFormServer/Controllers/SellController.cs` — 기존 sell 엔드포인트 패턴 (JWT 인증, SellerId 추출)
- `TicketPlatFormServer/TicketPlatFormServer/Services/Sell/SellService.cs` — 기존 서비스 패턴 (이미지 URL 처리 포함)
- `TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/SellRepository.cs` — Dapper 쿼리 패턴 (ReadModel → Entity 변환)
- `TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/SellQueries.cs` — SQL 쿼리 상수 패턴
- `TicketPlatFormServer/TicketPlatFormServer/Repository/ReadModels/MyTicketReadModel.cs` — ReadModel 클래스 패턴
- `TicketPlatFormServer/TicketPlatFormServer/Repository/Transaction/TransactionHistoryRepository.cs` — 복잡 SQL + LEFT JOIN 패턴 (lines 173-218)
- `TicketPlatFormServer/TicketPlatFormServer/DTO/Sell/MyTicketListRespDto.cs` — 기존 응답 DTO 패턴

---

## Work Objectives

### 핵심 목표
판매자의 등록 티켓을 Event(공연) 기준으로 그룹핑하고, 각 공연별 상태 수량(총/판매중/완료/정산중)을 집계하는 API를 제공한다.
거래 없는 순수 등록 티켓도 포함되며, 상태 필터링을 지원한다.

### Must Have
- `GET /api/sell/sales-dashboard` — Event 그룹핑 대시보드 API
  - 응답: 공연별 그룹 목록 (posterUrl, title, venueName, eventDatetime, 상태별 COUNT)
  - `tickets` 테이블 기준 LEFT JOIN으로 transactions/settlements 상태 집계
  - 상태 필터 파라미터: `status` (all/on_sale/completed/settling)
  - page/size 페이지네이션 (기존 sell API 패턴 유지)
  - 공연별 대표 포스터 이미지 (event.poster_image_url)
  - 데이터 결손 fallback: event null → "공연 정보 없음", venue null → "장소 미정"
- `GET /api/sell/sales-dashboard/{eventId}` — 공연별 티켓 상세 목록 API
  - 응답: 해당 공연의 개별 티켓 목록 (좌석정보, 가격, 수량, 상태, 거래정보)
  - ticket + transaction + settlement LEFT JOIN
  - 거래 없는 티켓도 포함 (status = "판매중")
  - page/size 페이지네이션
- 상태 매핑 로직 서비스 레이어에 구현 (ticket status + transaction status + settlement status → 4가지 카테고리)
- JWT 인증 필수 (SellerId = userId)
- IDbConnection scoped 원칙 준수 — 병렬 DB 호출 금지

### Must NOT Have (가드레일)
- 기존 `GET /api/sell/my-tickets` 응답 형태/로직 변경 금지
- 기존 `GET /api/transactions/sales` 수정 금지
- 정산 처리 로직 (TASK-009 영역) 포함 금지
- 매출 합계/수익 통계 등 분석 기능 포함 금지
- 정산 실패/계좌 미등록 안내 UI 로직 포함 금지 (이번 범위 아님)
- 총 수량(장수) 합계가 아닌 등록 건(티켓 글) 기준 COUNT만 사용
- `as any`, `@ts-ignore`, `console.log` 등 AI 슬롭 패턴 금지
- 불필요한 추상화 레이어 추가 금지 (기존 패턴 따를 것)

---

## Verification Strategy

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed.

### Test Decision
- **Infrastructure exists**: NO (프로젝트에 테스트 프로젝트 미존재)
- **Automated tests**: None
- **Framework**: N/A

### QA Policy
Every task MUST include agent-executed QA scenarios.
Evidence saved to `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`.

- **API/Backend**: Use Bash (curl) — Send requests, assert status + response fields
- **Build**: Use Bash (dotnet build) — Verify compilation success

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Foundation — DTO + ReadModel + Repository Interface):
├── Task 1: ReadModel 2개 정의 [quick]
├── Task 2: Response DTO 2개 정의 [quick]
└── Task 3: Repository Interface 확장 + SQL 쿼리 상수 [quick]

Wave 2 (Core — Repository 구현 + Service 확장):
├── Task 4: Repository 구현 — 대시보드 SQL (depends: 1, 3) [unspecified-high]
├── Task 5: Repository 구현 — 공연별 상세 SQL (depends: 1, 3) [unspecified-high]
└── Task 6: Service 확장 — 비즈니스 로직 (depends: 2, 4, 5) [unspecified-high]

Wave 3 (API — Controller + 빌드 검증):
├── Task 7: Controller 엔드포인트 2개 추가 (depends: 6) [quick]
└── Task 8: 빌드 검증 + API 스펙 문서 (depends: 7) [quick]

Wave FINAL (독립 검증, 4개 병렬):
├── F1: Plan compliance audit (oracle)
├── F2: Code quality review (unspecified-high)
├── F3: API QA — curl 검증 (unspecified-high)
└── F4: Scope fidelity check (deep)

Critical Path: Task 1 → Task 4 → Task 6 → Task 7 → Task 8 → F1-F4
```

### Dependency Matrix
| Task | Depends On | Blocks |
|------|-----------|--------|
| 1 | — | 4, 5 |
| 2 | — | 6 |
| 3 | — | 4, 5 |
| 4 | 1, 3 | 6 |
| 5 | 1, 3 | 6 |
| 6 | 2, 4, 5 | 7 |
| 7 | 6 | 8 |
| 8 | 7 | F1-F4 |

### Agent Dispatch Summary
- **Wave 1**: 3 tasks — T1 `quick`, T2 `quick`, T3 `quick`
- **Wave 2**: 3 tasks — T4 `unspecified-high`, T5 `unspecified-high`, T6 `unspecified-high`
- **Wave 3**: 2 tasks — T7 `quick`, T8 `quick`
- **FINAL**: 4 tasks — F1 `oracle`, F2 `unspecified-high`, F3 `unspecified-high`, F4 `deep`

---

## TODOs

- [ ] 1. ReadModel 2개 정의

  **What to do**:
  - `SalesDashboardReadModel.cs` 생성 — `TicketPlatFormServer/TicketPlatFormServer/Repository/ReadModels/` 아래
    - Fields: `int EventId`, `string EventTitle`, `string? PosterImageUrl`, `string? VenueName`, `DateTime? EarliestEventDatetime`, `int TotalCount`, `int OnSaleCount`, `int CompletedCount`, `int SettlingCount`
    - Dapper가 SQL 결과를 직접 매핑할 ReadModel
  - `EventTicketReadModel.cs` 생성 — 같은 디렉토리
    - Fields: `int TicketId`, `int EventId`, `string EventTitle`, `string? SeatGradeName`, `string? AreaName`, `string? Row`, `int Quantity`, `int RemainingQuantity`, `int Price`, `int OriginalPrice`, `string StatusCode`, `string StatusName`, `string? TransactionStatusCode`, `string? TransactionStatusName`, `long? TransactionId`, `string? SettlementStatusCode`, `DateTime CreatedAt`, `string? ThumbnailPath`
    - ticket + transaction + settlement JOIN 결과 매핑용
  - 기존 `MyTicketReadModel.cs` 패턴 따를 것 (simple POCO, no logic)

  **Must NOT do**:
  - `MyTicketReadModel.cs` 수정 금지
  - 비즈니스 로직이나 변환 메서드 포함 금지 (pure data class)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 단순 POCO 클래스 2개 생성, 로직 없음
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 2, 3)
  - **Blocks**: Tasks 4, 5
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/ReadModels/MyTicketReadModel.cs` — ReadModel POCO 패턴 (필드 명명, nullable 사용)

  **API/Type References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/Ticket.cs` — EventId, StatusId, Quantity, RemainingQuantity 필드 확인
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/Settlement.cs` — StatusId 필드 확인

  **WHY Each Reference Matters**:
  - `MyTicketReadModel.cs`: Dapper ReadModel의 정확한 패턴 (PascalCase 프로퍼티, SQL 컬럼 alias와 일치시키는 방식)
  - `Ticket.cs`: 실제 DB 컬럼 타입과 nullable 여부 확인 → ReadModel 필드 타입 결정

  **Acceptance Criteria**:
  - [ ] `SalesDashboardReadModel.cs` 파일 존재, 8개 필드 포함
  - [ ] `EventTicketReadModel.cs` 파일 존재, 17개 필드 포함
  - [ ] `dotnet build TicketPlatFormServer` → Build succeeded

  **QA Scenarios**:

  ```
  Scenario: ReadModel 파일 생성 및 빌드 성공
    Tool: Bash
    Preconditions: 프로젝트 클린 상태
    Steps:
      1. ls TicketPlatFormServer/TicketPlatFormServer/Repository/ReadModels/SalesDashboardReadModel.cs → 파일 존재 확인
      2. ls TicketPlatFormServer/TicketPlatFormServer/Repository/ReadModels/EventTicketReadModel.cs → 파일 존재 확인
      3. dotnet build TicketPlatFormServer → Build succeeded
    Expected Result: 두 파일 모두 존재하고 빌드 성공
    Failure Indicators: 파일 미존재 또는 빌드 에러
    Evidence: .sisyphus/evidence/task-1-readmodel-build.txt
  ```

  **Commit**: YES (groups with Wave 1)
  - Message: `feat(sell): add sales dashboard ReadModels`
  - Files: `Repository/ReadModels/SalesDashboardReadModel.cs`, `Repository/ReadModels/EventTicketReadModel.cs`
  - Pre-commit: `dotnet build TicketPlatFormServer`

- [ ] 2. Response DTO 2개 정의

  **What to do**:
  - `SalesDashboardRespDto.cs` 생성 — `TicketPlatFormServer/TicketPlatFormServer/DTO/Sell/` 아래
    ```csharp
    public class SalesDashboardRespDto
    {
        public List<EventGroupItemDto> EventGroups { get; set; } = new();
        public int Page { get; set; }
        public int Size { get; set; }
        public int TotalCount { get; set; }
        public bool HasMore { get; set; }
    }

    public class EventGroupItemDto
    {
        public int EventId { get; set; }
        public string EventTitle { get; set; } = null!;
        public string? PosterImageUrl { get; set; }
        public string? VenueName { get; set; }
        public DateTime? EarliestEventDatetime { get; set; }
        public int TotalCount { get; set; }
        public int OnSaleCount { get; set; }
        public int CompletedCount { get; set; }
        public int SettlingCount { get; set; }
    }
    ```
  - `EventTicketListRespDto.cs` 생성 — 같은 디렉토리
    ```csharp
    public class EventTicketListRespDto
    {
        public int EventId { get; set; }
        public string EventTitle { get; set; } = null!;
        public List<EventTicketItemDto> Tickets { get; set; } = new();
        public int Page { get; set; }
        public int Size { get; set; }
        public int TotalCount { get; set; }
        public bool HasMore { get; set; }
    }

    public class EventTicketItemDto
    {
        public int TicketId { get; set; }
        public string? SeatInfo { get; set; }  // "VIP석 A구역 3열" 형태로 조합
        public int Quantity { get; set; }
        public int RemainingQuantity { get; set; }
        public int Price { get; set; }
        public int OriginalPrice { get; set; }
        public string StatusCode { get; set; } = null!;  // on_sale, completed, settling
        public string StatusName { get; set; } = null!;  // 판매중, 판매 완료, 정산 중
        public long? TransactionId { get; set; }
        public string? ThumbnailUrl { get; set; }
        public DateTime CreatedAt { get; set; }
    }
    ```
  - `SalesDashboardReqDto.cs` 생성 — 요청 DTO
    ```csharp
    public class SalesDashboardReqDto
    {
        public string? Status { get; set; }  // all, on_sale, completed, settling
        public int Page { get; set; } = 1;
        public int Size { get; set; } = 20;
    }
    ```

  **Must NOT do**:
  - 기존 `MyTicketListRespDto.cs` 수정 금지
  - Freezed/코드생성 사용 금지 (Backend는 순수 C# DTO)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: DTO 클래스 3개 생성, 순수 데이터 구조
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 3)
  - **Blocks**: Task 6
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Sell/MyTicketListRespDto.cs` — 기존 판매 DTO 패턴 (페이지네이션 필드, 리스트 구조)
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Sell/MyTicketListReqDto.cs` — 요청 DTO 패턴 (기본값 설정)

  **WHY Each Reference Matters**:
  - `MyTicketListRespDto.cs`: 페이지네이션 필드명(Page/Size/TotalCount/HasMore)을 동일하게 유지하여 모바일과 일관된 계약 형성

  **Acceptance Criteria**:
  - [ ] `SalesDashboardRespDto.cs` 파일에 `SalesDashboardRespDto` + `EventGroupItemDto` 클래스 존재
  - [ ] `EventTicketListRespDto.cs` 파일에 `EventTicketListRespDto` + `EventTicketItemDto` 클래스 존재
  - [ ] `SalesDashboardReqDto.cs` 파일에 Status, Page, Size 필드 존재
  - [ ] `dotnet build TicketPlatFormServer` → Build succeeded

  **QA Scenarios**:

  ```
  Scenario: DTO 파일 생성 및 빌드 성공
    Tool: Bash
    Preconditions: 프로젝트 클린 상태
    Steps:
      1. ls TicketPlatFormServer/TicketPlatFormServer/DTO/Sell/SalesDashboardRespDto.cs → 파일 존재
      2. ls TicketPlatFormServer/TicketPlatFormServer/DTO/Sell/EventTicketListRespDto.cs → 파일 존재
      3. ls TicketPlatFormServer/TicketPlatFormServer/DTO/Sell/SalesDashboardReqDto.cs → 파일 존재
      4. dotnet build TicketPlatFormServer → Build succeeded
    Expected Result: 3개 DTO 파일 존재, 빌드 성공
    Failure Indicators: 파일 미존재 또는 타입 에러
    Evidence: .sisyphus/evidence/task-2-dto-build.txt
  ```

  **Commit**: YES (groups with Wave 1)
  - Message: `feat(sell): add sales dashboard DTOs`
  - Files: `DTO/Sell/SalesDashboardRespDto.cs`, `DTO/Sell/EventTicketListRespDto.cs`, `DTO/Sell/SalesDashboardReqDto.cs`
  - Pre-commit: `dotnet build TicketPlatFormServer`

- [ ] 3. Repository Interface 확장 + SQL 쿼리 상수

  **What to do**:
  - `ISellRepository.cs`에 2개 메서드 시그니처 추가:
    ```csharp
    Task<(List<SalesDashboardReadModel> Items, int TotalCount)> GetSalesDashboardAsync(int sellerId, string? statusFilter, int page, int size);
    Task<(List<EventTicketReadModel> Items, int TotalCount)> GetEventTicketsAsync(int sellerId, int eventId, int page, int size);
    ```
  - `SellQueries.cs`에 2개 SQL 쿼리 상수 추가:
    - `GetSalesDashboard`: tickets 기준 EVENT GROUP BY + 상태별 서브쿼리 COUNT
      ```sql
      SELECT
          e.id AS EventId,
          COALESCE(e.title, '공연 정보 없음') AS EventTitle,
          e.poster_image_url AS PosterImageUrl,
          COALESCE(e.venue_name, '장소 미정') AS VenueName,
          MIN(t.event_datetime) AS EarliestEventDatetime,
          COUNT(*) AS TotalCount,
          SUM(CASE WHEN t.status_id = 1 THEN 1 ELSE 0 END) AS OnSaleCount,
          SUM(CASE WHEN EXISTS (
              SELECT 1 FROM transaction_items ti
              INNER JOIN transactions tr ON ti.transaction_id = tr.id
              INNER JOIN settlements s ON tr.id = s.transaction_id
              INNER JOIN settlement_statuses ss ON s.status_id = ss.id
              WHERE ti.ticket_id = t.id AND ss.code = 'completed'
          ) THEN 1 ELSE 0 END) AS CompletedCount,
          SUM(CASE WHEN EXISTS (
              SELECT 1 FROM transaction_items ti
              INNER JOIN transactions tr ON ti.transaction_id = tr.id
              INNER JOIN transaction_statuses trs ON tr.status_id = trs.id
              WHERE ti.ticket_id = t.id AND trs.code = 'confirmed'
              AND NOT EXISTS (
                  SELECT 1 FROM settlements s
                  INNER JOIN settlement_statuses ss ON s.status_id = ss.id
                  WHERE s.transaction_id = tr.id AND ss.code = 'completed'
              )
          ) THEN 1 ELSE 0 END) AS SettlingCount
      FROM tickets t
      LEFT JOIN events e ON t.event_id = e.id
      WHERE t.seller_id = @SellerId
        AND t.deleted_at IS NULL
        AND t.status_id != 5
      GROUP BY e.id, e.title, e.poster_image_url, e.venue_name
      ORDER BY MIN(t.created_at) DESC
      LIMIT @Size OFFSET @Offset
      ```
    - `GetSalesDashboardCount`: 총 공연 그룹 수 COUNT
    - `GetEventTickets`: 특정 공연의 개별 티켓 + transaction/settlement LEFT JOIN
    - `GetEventTicketsCount`: 특정 공연의 티켓 총 수
  - 상태 필터 WHERE 조건:
    - `on_sale` → `HAVING OnSaleCount > 0`
    - `completed` → `HAVING CompletedCount > 0`
    - `settling` → `HAVING SettlingCount > 0`
    - `all` 또는 null → 조건 없음

  **Must NOT do**:
  - 기존 `SellQueries.GetMyTickets` SQL 수정 금지
  - 기존 `ISellRepository` 메서드 시그니처 변경 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 인터페이스 시그니처 2개 + SQL 상수 추가
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2)
  - **Blocks**: Tasks 4, 5
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/ISellRepository.cs` — 기존 인터페이스 시그니처 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/SellQueries.cs` — SQL 상수 패턴 (static class, const string)
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Transaction/TransactionHistoryRepository.cs:173-218` — 복잡 SQL LEFT JOIN 패턴

  **API/Type References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/Settlement.cs` — settlements 테이블 구조
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/Transaction.cs` — transactions 테이블 구조

  **WHY Each Reference Matters**:
  - `SellQueries.cs`: SQL 쿼리를 const string으로 관리하는 프로젝트 관례 확인
  - `TransactionHistoryRepository.cs:173-218`: 복잡한 LEFT JOIN + 서브쿼리 패턴의 실제 예시

  **Acceptance Criteria**:
  - [ ] `ISellRepository.cs`에 `GetSalesDashboardAsync`, `GetEventTicketsAsync` 시그니처 존재
  - [ ] `SellQueries.cs`에 `GetSalesDashboard`, `GetEventTickets` 쿼리 상수 존재
  - [ ] `dotnet build TicketPlatFormServer` → Build succeeded (구현체 미완성이라 warning 가능, error 없어야 함)

  **QA Scenarios**:

  ```
  Scenario: Interface + SQL 상수 추가 및 빌드
    Tool: Bash
    Preconditions: Task 1 ReadModel 완료
    Steps:
      1. grep -c "GetSalesDashboardAsync" TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/ISellRepository.cs → 1 이상
      2. grep -c "GetEventTicketsAsync" TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/ISellRepository.cs → 1 이상
      3. grep -c "GetSalesDashboard" TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/SellQueries.cs → 1 이상
      4. dotnet build TicketPlatFormServer → Build succeeded (warning 허용)
    Expected Result: 시그니처와 SQL 상수 존재, 빌드 성공
    Failure Indicators: 시그니처 누락 또는 SQL 문법 에러
    Evidence: .sisyphus/evidence/task-3-interface-sql.txt
  ```

  **Commit**: YES (groups with Wave 1)
  - Message: `feat(sell): add sales dashboard repository interface and queries`
  - Files: `Repository/Sell/ISellRepository.cs`, `Repository/Sell/SellQueries.cs`
  - Pre-commit: `dotnet build TicketPlatFormServer`

- [ ] 4. Repository 구현 — 대시보드 SQL

  **What to do**:
  - `SellRepository.cs`에 `GetSalesDashboardAsync()` 구현
    - `SellQueries.GetSalesDashboard` SQL 실행 (Dapper `QueryAsync<SalesDashboardReadModel>`)
    - `SellQueries.GetSalesDashboardCount` SQL로 총 그룹 수 조회
    - statusFilter 파라미터에 따라 HAVING 조건 동적 추가:
      - `on_sale` → `HAVING OnSaleCount > 0`
      - `completed` → `HAVING CompletedCount > 0`
      - `settling` → `HAVING SettlingCount > 0`
      - `all`/null → 조건 없음
    - page/size → OFFSET/LIMIT 계산
    - 기존 `GetMyTicketsAsync()` 패턴 따를 것 (IDbConnection 주입, QueryAsync 사용)
  - ⚠️ IDbConnection scoped — 대시보드 쿼리와 카운트 쿼리를 순차 호출할 것 (Task.WhenAll 금지)

  **Must NOT do**:
  - 기존 `GetMyTicketsAsync()` 메서드 수정 금지
  - Task.WhenAll 등 병렬 DB 호출 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Dapper SQL 실행 + 동적 HAVING 조건 구성 로직
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Task 5)
  - **Blocks**: Task 6
  - **Blocked By**: Tasks 1, 3

  **References**:
  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/SellRepository.cs` — 기존 `GetMyTicketsAsync()` Dapper 구현 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Transaction/TransactionHistoryRepository.cs:265-324` — 동적 WHERE 조건 구성 패턴
  **API/Type References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/ReadModels/SalesDashboardReadModel.cs` — Task 1에서 생성한 ReadModel
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/SellQueries.cs` — Task 3에서 추가한 SQL 상수
  **WHY Each Reference Matters**:
  - `SellRepository.cs`: DI 주입 패턴(IDbConnection), Dapper QueryAsync 호출 방식 확인
  - `TransactionHistoryRepository.cs:265-324`: 동적 SQL 조건 구성의 프로젝트 표준 패턴

  **Acceptance Criteria**:
  - [ ] `SellRepository.cs`에 `GetSalesDashboardAsync()` 메서드 구현 존재
  - [ ] statusFilter별 HAVING 조건 분기 로직 존재
  - [ ] 순차적 DB 호출 (병렬 호출 없음)
  - [ ] `dotnet build TicketPlatFormServer` → Build succeeded

  **QA Scenarios**:
  ```
  Scenario: 대시보드 Repository 구현 및 빌드
    Tool: Bash
    Preconditions: Wave 1 완료
    Steps:
      1. grep -c "GetSalesDashboardAsync" TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/SellRepository.cs → 1 이상
      2. grep -c "Task.WhenAll" TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/SellRepository.cs → 0
      3. dotnet build TicketPlatFormServer → Build succeeded
    Expected Result: 구현 존재, 병렬 호출 없음, 빌드 성공
    Evidence: .sisyphus/evidence/task-4-dashboard-repo.txt
  ```
  **Commit**: NO (Wave 2 그룹 커밋)

- [ ] 5. Repository 구현 — 공연별 상세 SQL

  **What to do**:
  - `SellRepository.cs`에 `GetEventTicketsAsync()` 구현
    - `SellQueries.GetEventTickets` SQL 실행 (Dapper `QueryAsync<EventTicketReadModel>`)
    - WHERE: `t.seller_id = @SellerId AND t.event_id = @EventId AND t.deleted_at IS NULL AND t.status_id != 5`
    - ticket + transaction_items + transactions + settlements LEFT JOIN
    - 좌석 정보: `CONCAT_WS(' ', sg.name_ko, a.area_name, t.row) AS SeatInfo`
    - 상태 매핑은 SQL에서 raw data 반환, Service에서 카테고리 변환
    - page/size 페이지네이션

  **Must NOT do**:
  - 상태 매핑 비즈니스 로직을 Repository에 넣지 않을 것 (raw data만 반환)
  - 기존 메서드 수정 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: 복잡한 LEFT JOIN SQL + Dapper 구현
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Task 4)
  - **Blocks**: Task 6
  - **Blocked By**: Tasks 1, 3

  **References**:
  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Transaction/TransactionHistoryRepository.cs:173-218` — 복잡 SQL: CONCAT_WS 좌석 조합, 다중 LEFT JOIN
  **API/Type References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/ReadModels/EventTicketReadModel.cs` — Task 1 ReadModel
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/TransactionItem.cs` — transaction_items 테이블 구조

  **Acceptance Criteria**:
  - [ ] `SellRepository.cs`에 `GetEventTicketsAsync()` 구현 존재
  - [ ] LEFT JOIN으로 transactions/settlements 조인
  - [ ] CONCAT_WS로 좌석 정보 조합
  - [ ] `dotnet build TicketPlatFormServer` → Build succeeded

  **QA Scenarios**:
  ```
  Scenario: 공연별 상세 Repository 구현 및 빌드
    Tool: Bash
    Preconditions: Wave 1 완료
    Steps:
      1. grep -c "GetEventTicketsAsync" TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/SellRepository.cs → 1 이상
      2. grep -c "CONCAT_WS" TicketPlatFormServer/TicketPlatFormServer/Repository/Sell/SellQueries.cs → 1 이상
      3. dotnet build TicketPlatFormServer → Build succeeded
    Expected Result: 구현 존재, 좌석 조합 SQL 존재, 빌드 성공
    Evidence: .sisyphus/evidence/task-5-event-tickets-repo.txt
  ```
  **Commit**: NO (Wave 2 그룹 커밋)

- [ ] 6. Service 확장 — 비즈니스 로직

  **What to do**:
  - `ISellService.cs`에 2개 메서드 시그니처 추가:
    - `Task<SalesDashboardRespDto> GetSalesDashboardAsync(int sellerId, SalesDashboardReqDto request);`
    - `Task<EventTicketListRespDto> GetEventTicketsAsync(int sellerId, int eventId, int page, int size);`
  - `SellService.cs`에 구현:
    - `GetSalesDashboardAsync()`: Repository 호출 → ReadModel→EventGroupItemDto 변환 → SalesDashboardRespDto 조립
    - `GetEventTicketsAsync()`: Repository 호출 → **상태 매핑 핵심 로직**:
      - `SettlementStatusCode == 'completed'` → StatusCode: `completed`, StatusName: `판매 완료`
      - `TransactionStatusCode == 'confirmed'` AND settlement 미완료 → StatusCode: `settling`, StatusName: `정산 중`
      - `ticket.StatusCode == 'available'` → StatusCode: `on_sale`, StatusName: `판매중`
      - 그 외 (`reserved`, `expired` 등) → StatusCode/StatusName 원본 유지
    - ThumbnailPath → Supabase 서명 URL 생성 (기존 SellService 패턴)
    - 데이터 결손 fallback: EventTitle null → "공연 정보 없음"

  **Must NOT do**:
  - 기존 `GetMyTicketsAsync()` 수정 금지
  - 매출/수익 통계 등 분석 기능 포함 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: 상태 매핑 비즈니스 로직 + DTO 변환 + 썸네일 URL 처리
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 2 (after Tasks 4, 5)
  - **Blocks**: Task 7
  - **Blocked By**: Tasks 2, 4, 5

  **References**:
  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Sell/SellService.cs` — 기존 `GetMyTicketsAsync()` 구현 (이미지 URL 서명 로직, DTO 변환)
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Sell/ISellService.cs` — 인터페이스 패턴
  **API/Type References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Sell/SalesDashboardRespDto.cs` — Task 2 응답 DTO
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Sell/EventTicketListRespDto.cs` — Task 2 공연별 DTO

  **Acceptance Criteria**:
  - [ ] `SellService.cs`에 두 메서드 구현 존재
  - [ ] 상태 매핑 로직: completed/settling/on_sale 3개 분기 존재
  - [ ] `dotnet build TicketPlatFormServer` → Build succeeded

  **QA Scenarios**:
  ```
  Scenario: Service 구현 및 상태 매핑 확인
    Tool: Bash
    Preconditions: Tasks 2, 4, 5 완료
    Steps:
      1. grep -c "GetSalesDashboardAsync" TicketPlatFormServer/TicketPlatFormServer/Services/Sell/SellService.cs → 1 이상
      2. grep -c "settling" TicketPlatFormServer/TicketPlatFormServer/Services/Sell/SellService.cs → 1 이상
      3. dotnet build TicketPlatFormServer → Build succeeded
    Expected Result: 두 메서드 구현, 상태 매핑 존재, 빌드 성공
    Evidence: .sisyphus/evidence/task-6-service.txt
  ```
  **Commit**: YES (Wave 2)
  - Message: `feat(sell): implement sales dashboard repository and service`
  - Files: `Repository/Sell/SellRepository.cs`, `Services/Sell/ISellService.cs`, `Services/Sell/SellService.cs`
  - Pre-commit: `dotnet build TicketPlatFormServer`

- [ ] 7. Controller 엔드포인트 2개 추가

  **What to do**:
  - `SellController.cs`에 2개 엔드포인트 추가:
    - `[HttpGet("sales-dashboard")]` → `GetSalesDashboard([FromQuery] SalesDashboardReqDto request)`
    - `[HttpGet("sales-dashboard/{eventId}")]` → `GetEventTickets(int eventId, [FromQuery] int page = 1, [FromQuery] int size = 20)`
  - 두 엔드포인트 모두 `[Authorize]` 적용, `GetUserId()` 헬퍼 사용
  - `ApiResponse<T>.Success()` 래핑 (기존 SellController 패턴)
  - Swagger XML 주석 한글 설명 추가

  **Must NOT do**:
  - 기존 `GetMyTickets` 엔드포인트 수정 금지
  - 새 Controller 파일 생성 금지 (기존 SellController에 추가)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 얇은 Controller 레이어, 서비스 호출만 위임
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 3
  - **Blocks**: Task 8
  - **Blocked By**: Task 6

  **References**:
  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/SellController.cs` — 기존 엔드포인트: Route, Authorize, GetUserId(), ApiResponse<T>.Success()

  **Acceptance Criteria**:
  - [ ] `SellController.cs`에 `GetSalesDashboard`, `GetEventTickets` 액션 메서드 존재
  - [ ] `[Authorize]` 적용
  - [ ] `dotnet build TicketPlatFormServer` → Build succeeded

  **QA Scenarios**:
  ```
  Scenario: Controller 엔드포인트 및 인증 확인
    Tool: Bash
    Preconditions: Task 6 완료
    Steps:
      1. grep -c "sales-dashboard" TicketPlatFormServer/TicketPlatFormServer/Controllers/SellController.cs → 2 이상
      2. dotnet build TicketPlatFormServer → Build succeeded
      3. 서버 실행 후 curl -s -o /dev/null -w "%{http_code}" http://localhost:5224/api/sell/sales-dashboard → 401
    Expected Result: 엔드포인트 존재, 빌드 성공, 인증 없이 401
    Evidence: .sisyphus/evidence/task-7-controller.txt
  ```
  **Commit**: NO (Wave 3 그룹 커밋)

- [ ] 8. 빌드 검증 + API 스펙 문서 + 하위호환 확인

  **What to do**:
  - `dotnet build TicketPlatFormServer` 전체 빌드 성공 확인 (0 errors)
  - `api_spec/sell-sales-dashboard.md` 파일 생성 — 2개 엔드포인트 Request/Response 스키마, 상태 필터 값, 페이지네이션
  - 기존 `GET /api/sell/my-tickets` curl 호출 → 무변경 확인
  - 기존 `GET /api/transactions/sales` curl 호출 → 무변경 확인
  - Swagger UI에 두 엔드포인트 표시 확인

  **Must NOT do**:
  - 소스 코드 수정 금지 (검증만)
  - 기존 API 스펙 파일 수정 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 빌드 확인 + 문서 작성 + curl 검증
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 3 (after Task 7)
  - **Blocks**: F1-F4
  - **Blocked By**: Task 7

  **References**:
  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/api_spec/` — 기존 API 스펙 문서 패턴

  **Acceptance Criteria**:
  - [ ] `dotnet build TicketPlatFormServer` → Build succeeded, 0 errors
  - [ ] `api_spec/sell-sales-dashboard.md` 존재
  - [ ] `GET /api/sell/my-tickets` curl → 기존과 동일한 응답
  - [ ] `GET /api/transactions/sales` curl → 기존과 동일한 응답

  **QA Scenarios**:
  ```
  Scenario: 전체 빌드 및 하위호환 확인
    Tool: Bash
    Preconditions: Task 7 완료, 서버 실행 중
    Steps:
      1. dotnet build TicketPlatFormServer → Build succeeded, 0 errors
      2. curl -H "Authorization: Bearer $TOKEN" http://localhost:5224/api/sell/my-tickets → 200
      3. curl -H "Authorization: Bearer $TOKEN" http://localhost:5224/api/sell/sales-dashboard → 200
      4. curl -H "Authorization: Bearer $TOKEN" http://localhost:5224/api/transactions/sales → 200
      5. ls TicketPlatFormServer/TicketPlatFormServer/api_spec/sell-sales-dashboard.md → 존재
    Expected Result: 빌드 성공, 모든 API 정상, 스펙 존재
    Evidence: .sisyphus/evidence/task-8-final-build.txt
  ```
  **Commit**: YES (Wave 3)
  - Message: `feat(sell): add sales dashboard API endpoints and spec`
  - Files: `Controllers/SellController.cs`, `api_spec/sell-sales-dashboard.md`
  - Pre-commit: `dotnet build TicketPlatFormServer`

---

## Final Verification Wave

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Read the plan end-to-end. For each "Must Have": verify implementation exists (read file, curl endpoint, run command). For each "Must NOT Have": search codebase for forbidden patterns — reject with file:line if found. Check evidence files exist in .sisyphus/evidence/. Compare deliverables against plan.
  Output: `Must Have [N/N] | Must NOT Have [N/N] | Tasks [N/N] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  Run `dotnet build TicketPlatFormServer`. Review all changed/new files for: empty catches, commented-out code, unused usings. Check AI slop: 과도한 주석, 불필요한 추상화, generic names (data/result/item/temp). 기존 SellController/SellService 패턴과 일관성 검증.
  Output: `Build [PASS/FAIL] | Files [N clean/N issues] | VERDICT`

- [ ] F3. **API QA** — `unspecified-high`
  서버를 시작하고 curl로 두 엔드포인트 호출:
  1. `GET /api/sell/sales-dashboard` — 200 응답, eventGroups 배열 존재, 각 그룹에 totalCount/onSaleCount/completedCount/settlingCount 필드 확인
  2. `GET /api/sell/sales-dashboard/{eventId}` — 200 응답, tickets 배열 존재, 각 항목에 ticketId/status/seatInfo 필드 확인
  3. 인증 없이 호출 → 401 확인
  4. 상태 필터 `?status=on_sale` → 필터링 동작 확인
  Output: `Endpoints [N/N pass] | Auth [PASS/FAIL] | Filter [PASS/FAIL] | VERDICT`

- [ ] F4. **Scope Fidelity Check** — `deep`
  For each task: read "What to do", read actual diff (git log/diff). Verify 1:1 — everything in spec was built, nothing beyond spec was built. Check "Must NOT do" compliance: 기존 sell/my-tickets 변경 없음, transactions/sales 변경 없음. Detect cross-task contamination. Flag unaccounted changes.
  Output: `Tasks [N/N compliant] | Contamination [CLEAN/N issues] | Unaccounted [CLEAN/N files] | VERDICT`

---

## Commit Strategy

- **Wave 1**: `feat(sell): add sales dashboard ReadModels and DTOs` — ReadModel, DTO 파일들
- **Wave 2**: `feat(sell): implement sales dashboard repository and service` — Repository, Service 파일들
- **Wave 3**: `feat(sell): add sales dashboard API endpoints` — Controller, API spec

---

## Success Criteria

### Verification Commands
```bash
dotnet build TicketPlatFormServer  # Expected: Build succeeded
curl -H "Authorization: Bearer $TOKEN" http://localhost:5224/api/sell/sales-dashboard  # Expected: 200, JSON with eventGroups
curl -H "Authorization: Bearer $TOKEN" http://localhost:5224/api/sell/sales-dashboard/1  # Expected: 200, JSON with tickets
curl http://localhost:5224/api/sell/sales-dashboard  # Expected: 401 Unauthorized
```

### Final Checklist
- [ ] 신규 등록 티켓(거래 0건)이 대시보드에 "판매중"으로 표시됨
- [ ] 같은 event_id 티켓들이 하나의 그룹으로 집계됨
- [ ] 상태별 수량 (총/판매중/완료/정산중) 정확히 계산됨
- [ ] 상태 필터 동작 (on_sale → 판매중 수량>0인 공연만 반환)
- [ ] 기존 `GET /api/sell/my-tickets` 무변경
- [ ] 기존 `GET /api/transactions/sales` 무변경
- [ ] `dotnet build` 성공
