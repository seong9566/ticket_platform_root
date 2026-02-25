# TASK-008 평판 시스템 — Backend 계획

## TL;DR

> **요약**: 구매확정 후 구매자가 판매자를 별점(1~5)으로 평가하는 평판 API를 구현한다.
> 별점은 MannerTemperature와 선형 연동되며, 프로필 및 티켓 상세 화면에 노출된다.
>
> **산출물**:
> - `POST /api/reputations` — 리뷰 작성
> - `GET /api/users/{userId}/reputations` — 받은 리뷰 목록
> - `GET /api/reputations/check/{transactionId}` — 리뷰 작성 가능 여부 확인
> - DB 마이그레이션 (유니크 제약, 시드 데이터, 비정규화 컬럼)
> - `ChatService.ConfirmPurchase()` + AutoConfirm 서비스에 FCM 훅 추가
> - `UserProfileDto`, `ChatRoomDetailRespDto` 필드 확장
>
> **예상 공수**: Medium
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
| 리뷰 방향 | 구매자 → 판매자 단방향 |
| 리뷰 UX | 채팅방 버블 → 별도 화면 |
| 매너온도 공식 | `delta = (score - 3) × 1.0°` / 범위: 0°~100° 클램핑 |
| 리뷰 기간 | 구매확정 후 7일 이내 |
| 별점 노출 | 프로필 화면 + 티켓 상세 판매자 정보 |
| 분쟁 중 리뷰 | 분쟁 OPEN 상태면 리뷰 차단 |
| AverageRating 저장 | 비정규화 컬럼 (user_profile에 추가) |
| 자동확정 FCM | 자동확정(AutoConfirm) 시에도 FCM 발송 |

### 기존 DB 모델 (수정 불필요)
- `UserReputation`: Id, UserId, ReviewerId, TransactionId, RatingTypeId, Score(1-5), Comment, CreatedAt
- `ReputationRatingType`: Id, Code, NameKo, IsActive, SortOrder — **시드 데이터 없음 → Task 1에서 추가**
- `Transaction`: BuyerId, SellerId, StatusId, ConfirmedAt, CancelledAt, AutoConfirmAt
- `User` / `UserProfile`: MannerTemperature(float?) 존재, AverageRating 없음 → **Task 1에서 추가**

### 참조 패턴
- `TicketPlatFormServer/TicketPlatFormServer/Controllers/DisputeController.cs` — API 구조 참조
- `TicketPlatFormServer/TicketPlatFormServer/Repository/Dispute/` — Repository 패턴 참조
- `TicketPlatFormServer/TicketPlatFormServer/Services/Dispute/` — Service 패턴 참조
- `TicketPlatFormServer/TicketPlatFormServer/Services/Chat/ChatService.cs` (ConfirmPurchase ~580번째 줄) — FCM 훅 위치

---

## Work Objectives

### 핵심 목표
구매확정 완료 거래에 대해 구매자가 판매자를 별점 평가할 수 있는 API 시스템 구축.
별점 데이터가 MannerTemperature 및 AverageRating에 원자적으로 반영되어야 한다.

### Must Have
- 동일 거래에 리뷰 중복 작성 불가 (DB UNIQUE + 앱 레벨 이중 검증)
- 구매확정 후 7일 초과 시 리뷰 작성 불가
- 분쟁(Dispute) OPEN 상태 거래에 리뷰 작성 불가
- 취소된 거래(CancelledAt != null) 리뷰 작성 불가
- MannerTemperature 원자적 SQL 업데이트 (`manner_temperature + delta`)
- 리뷰 작성 트랜잭션: INSERT + MannerTemperature UPDATE + AverageRating UPDATE 원자성 보장
- 수동/자동 구매확정 모두에서 `REVIEW_REQUEST` FCM 발송

### Must NOT Have (가드레일)
- 구매자가 아닌 사용자의 리뷰 작성 (403 반환)
- 판매자 본인이 자기 자신에게 리뷰 작성 불가
- 리뷰 수정/삭제 API (작성 완료 후 불변)
- 발신 리뷰 목록 API (받은 리뷰만)
- 리뷰 신고/숨김 기능 (어드민 기능 별도)
- 판매자의 리뷰 답글 기능
- MannerTemperature 0°~100° 범위 이탈

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
Wave 1 (즉시 시작 — DB + DTO 기반 작업):
├── Task 1: DB 마이그레이션 (유니크 제약, 시드, 컬럼 추가)
└── Task 2: DTO 클래스 생성 (Request/Response)

Wave 2 (Wave 1 완료 후 — 핵심 로직):
├── Task 3: ReputationRepository 구현 (depends: 1, 2)
├── Task 4: ChatRoomDetailRespDto + UserProfileDto 필드 확장 (depends: 1, 2)
└── Task 5: AutoConfirmService FCM 훅 (depends: 2)

Wave 3 (Wave 2 완료 후 — 통합):
├── Task 6: ReputationService 구현 (depends: 3, 4)
├── Task 7: ChatService.ConfirmPurchase() FCM 훅 (depends: 4)
└── Task 8: ReputationController API 엔드포인트 (depends: 6)

Critical Path: Task 1 → Task 3 → Task 6 → Task 8
```

### Agent Dispatch
- **Wave 1**: Task 1 → `quick`, Task 2 → `quick`
- **Wave 2**: Task 3 → `unspecified-high`, Task 4 → `quick`, Task 5 → `quick`
- **Wave 3**: Task 6 → `unspecified-high`, Task 7 → `quick`, Task 8 → `unspecified-high`

---

## TODOs

---

- [ ] 1. DB 마이그레이션 — 유니크 제약 + 시드 + 비정규화 컬럼

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/database_history/TASK-008-migration.sql` 파일 생성:
    ```sql
    -- 1. user_reputation 복합 유니크 제약 추가
    ALTER TABLE user_reputation
    ADD UNIQUE KEY uk_reputation_tx_reviewer (transaction_id, reviewer_id);

    -- 2. reputation_rating_types 시드 데이터
    INSERT INTO reputation_rating_types (id, code, name_ko, is_active, sort_order)
    VALUES (1, 'GENERAL', '전체 평가', 1, 1);

    -- 3. user_profile 비정규화 컬럼 추가
    ALTER TABLE user_profile
    ADD COLUMN average_rating DECIMAL(3,2) NULL DEFAULT NULL,
    ADD COLUMN review_count INT NOT NULL DEFAULT 0;

    -- 4. notification_types 시드
    INSERT INTO notification_types (code, name_ko, is_active)
    VALUES ('REVIEW_REQUEST', '리뷰 요청', 1)
    ON DUPLICATE KEY UPDATE is_active = 1;
    ```
  - `TicketPlatFormServer/TicketPlatFormServer/database_history/db_restore.sh` 및 `db_restore.bat` 파일 상단에 마이그레이션 실행 순서 주석 추가

  **Must NOT do**:
  - EF Core 마이그레이션 파일 생성 금지 (프로젝트는 수동 SQL 방식 사용)
  - 기존 데이터 삭제 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: SQL 파일 작성 단순 작업
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (Task 2와 동시)
  - **Blocks**: Task 3, 4, 5, 6, 7, 8
  - **Blocked By**: 없음

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/database_history/` — 기존 덤프 파일 구조 참조
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/UserReputation.cs` — 테이블 구조 확인
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/UserProfile.cs` — 컬럼 추가 대상 확인

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: DB 마이그레이션 적용 확인
    Tool: Bash (mysql)
    Steps:
      1. mysql -u root -p tickethub -e "SHOW INDEX FROM user_reputation WHERE Key_name='uk_reputation_tx_reviewer';"
      2. mysql -u root -p tickethub -e "SELECT * FROM reputation_rating_types;"
      3. mysql -u root -p tickethub -e "SHOW COLUMNS FROM user_profile LIKE 'average_rating';"
    Expected Result:
      1. uk_reputation_tx_reviewer 인덱스 행 존재
      2. Code='GENERAL' 레코드 1건 반환
      3. average_rating DECIMAL(3,2) 컬럼 존재
    Evidence: .sisyphus/evidence/task-1-db-migration.txt

  Scenario: 유니크 제약 중복 차단 확인
    Tool: Bash (mysql)
    Steps:
      1. 테스트 레코드 INSERT (transaction_id=9999, reviewer_id=9999)
      2. 동일 (transaction_id=9999, reviewer_id=9999)로 재 INSERT 시도
    Expected Result: ERROR 1062 (Duplicate entry) 반환
    Evidence: .sisyphus/evidence/task-1-unique-constraint.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-1-*.txt`
  **Commit**: YES (groups with Task 2)
  - Message: `chore(db): add reputation migration - unique constraint, seed, columns`
  - Files: `TicketPlatFormServer/TicketPlatFormServer/database_history/TASK-008-migration.sql`

---

- [ ] 2. DTO 클래스 생성

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Reputation/` 디렉토리 생성
  - `CreateReputationReqDto.cs` 작성:
    ```csharp
    public record CreateReputationReqDto(
        long TransactionId,
        [Range(1, 5)] int Score,
        [MaxLength(500)] string? Comment
    );
    ```
  - `ReputationRespDto.cs` 작성 (리뷰 목록 아이템):
    ```csharp
    public record ReputationRespDto(
        long Id,
        string ReviewerNickname,
        string? ReviewerProfileImageUrl,
        int Score,
        string? Comment,
        DateTime CreatedAt
    );
    ```
  - `ReputationCheckRespDto.cs` 작성:
    ```csharp
    public record ReputationCheckRespDto(
        bool CanReview,
        bool HasReviewed,
        DateTime? ReviewDeadline  // null이면 기간 만료
    );
    ```
  - `ReputationListRespDto.cs` 작성 (페이지네이션):
    ```csharp
    public record ReputationListRespDto(
        IEnumerable<ReputationRespDto> Items,
        int TotalCount,
        float? AverageRating
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
  - **Parallel Group**: Wave 1 (Task 1과 동시)
  - **Blocks**: Task 3, 4, 5, 6, 7, 8
  - **Blocked By**: 없음

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Dispute/` — DTO 파일 구조 및 네이밍 패턴 참조
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/User/UserProfileDto.cs` — 기존 DTO 스타일 참조

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: DTO 컴파일 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer --no-restore
    Expected Result: Build succeeded, 0 Error(s)
    Evidence: .sisyphus/evidence/task-2-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-2-build.txt`
  **Commit**: YES (groups with Task 1)
  - Message: `chore(db): add reputation migration - unique constraint, seed, columns`
  - Files: `TicketPlatFormServer/TicketPlatFormServer/DTO/Reputation/*.cs`

---

- [ ] 3. ReputationRepository 구현

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Reputation/IReputationRepository.cs` 인터페이스 작성:
    ```csharp
    public interface IReputationRepository
    {
        Task<bool> ExistsAsync(long transactionId, long reviewerId);
        Task InsertAsync(UserReputation reputation);
        Task<IEnumerable<UserReputation>> GetByTargetUserIdAsync(long userId, int page, int size);
        Task<int> CountByTargetUserIdAsync(long userId);
        Task UpdateUserProfileStatsAsync(long userId, float delta, float newAvgRating, int newReviewCount);
    }
    ```
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Reputation/ReputationRepository.cs` 구현:
    - `ExistsAsync`: Dapper `SELECT COUNT(1)` — 중복 체크
    - `InsertAsync`: EF Core로 INSERT
    - `GetByTargetUserIdAsync`: Dapper JOIN (user_reputation ← users) — reviewer 닉네임, 프로필 이미지 포함, 최신순 정렬, offset 페이지네이션
    - `CountByTargetUserIdAsync`: Dapper COUNT
    - `UpdateUserProfileStatsAsync`: Dapper **원자적 UPDATE**:
      ```sql
      UPDATE user_profile
      SET manner_temperature = LEAST(100.0, GREATEST(0.0, COALESCE(manner_temperature, 36.5) + @Delta)),
          average_rating = @NewAvgRating,
          review_count = review_count + 1
      WHERE user_id = @UserId
      ```
  - `TicketPlatFormServer/TicketPlatFormServer/Program.cs`에 DI 등록:
    `builder.Services.AddScoped<IReputationRepository, ReputationRepository>();`

  **Must NOT do**:
  - AGENTS.md 규칙: `IDbConnection`은 Scoped → `Task.WhenAll` + repository 조합 절대 금지
  - 비즈니스 로직 포함 금지 (유효성 검사는 Service 레이어)
  - DTO 직접 참조 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (Task 4, 5와 동시)
  - **Blocks**: Task 6
  - **Blocked By**: Task 1, 2

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Dispute/DisputeRepository.cs` — Repository 구현 패턴 (Dapper 쿼리 스타일, 생성자 주입)
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Dispute/IDisputeRepository.cs` — 인터페이스 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/DBModel/UserReputation.cs` — Entity 구조
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/README.md` — Dapper + EF Core 혼용 패턴, 트랜잭션 패턴

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드 및 DI 등록 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer
    Expected Result: Build succeeded, 0 Error(s)
    Evidence: .sisyphus/evidence/task-3-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-3-build.txt`
  **Commit**: NO (Task 6, 8과 함께)

---

- [ ] 4. UserProfileDto + ChatRoomDetailRespDto 필드 확장

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/User/UserProfileDto.cs` 수정 — `AverageRating`, `ReviewCount` 필드 추가:
    ```csharp
    // 기존 MannerTemperature, TotalTradeCount 아래에 추가
    public float? AverageRating { get; init; }
    public int ReviewCount { get; init; }
    ```
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Chat/ChatRoomDetailRespDto.cs` 수정 — 리뷰 상태 플래그 추가:
    ```csharp
    // 기존 CanConfirmPurchase, CanCancelTransaction 아래에 추가
    public bool CanWriteReview { get; init; }    // isBuyer && confirmed && !hasReviewed && within7days && noOpenDispute
    public bool HasReviewedSeller { get; init; } // 이미 리뷰 완료 여부
    ```
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Chat/ChatService.cs`에서 `ChatRoomDetailRespDto` 생성 위치를 찾아 두 플래그 값 채우는 로직 추가:
    - `IReputationRepository`를 ChatService 생성자에 주입
    - `CanWriteReview = isBuyer && confirmedAt != null && !hasReviewed && (DateTime.UtcNow - confirmedAt.Value).TotalDays <= 7 && !hasOpenDispute`
    - `HasReviewedSeller = await _reputationRepo.ExistsAsync(transactionId, currentUserId)`

  **Must NOT do**:
  - `UserProfileDto`에서 `MannerTemperature` 필드 제거 금지 (기존 필드 유지)
  - Entity를 직접 Controller로 노출 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (Task 3, 5와 동시)
  - **Blocks**: Task 6, 7
  - **Blocked By**: Task 1, 2

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/User/UserProfileDto.cs` — 기존 필드 구조 확인 (MannerTemperature, TotalTradeCount 위치)
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Chat/ChatRoomDetailRespDto.cs` — 기존 필드 구조 확인 (CanConfirmPurchase, CanCancelTransaction 위치)
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Chat/ChatService.cs` — ChatRoomDetailRespDto 생성 위치 탐색 후 수정

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer
    Expected Result: Build succeeded
    Evidence: .sisyphus/evidence/task-4-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-4-build.txt`
  **Commit**: NO (Task 6, 8과 함께)

---

- [ ] 5. AutoConfirmService — 리뷰 요청 FCM 훅 추가

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BackgroundServices/` 내 자동 구매확정 관련 서비스 파일 탐색 (TransactionReservationCleanupService.cs 참조)
  - 자동 구매확정 처리 로직 완료 후 구매자(BuyerId)에게 FCM 발송 코드 추가:
    ```csharp
    // AutoConfirm 처리 완료 후
    await _notificationService.SendAsync(
        userId: transaction.BuyerId,
        type: "REVIEW_REQUEST",
        title: "거래는 어떠셨나요?",
        body: $"{sellerNickname} 판매자에 대한 리뷰를 남겨주세요.",
        data: new { transactionId = transaction.Id }
    );
    ```
  - FCM 발송 메서드 시그니처는 `TicketPlatFormServer/TicketPlatFormServer/Services/Chat/ChatService.cs`의 기존 호출 방식과 동일하게 사용

  **Must NOT do**:
  - 수동 확정(ChatService) 로직 수정 금지 (Task 7에서 별도 처리)

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (Task 3, 4와 동시)
  - **Blocks**: 없음
  - **Blocked By**: Task 2

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/BackgroundServices/TransactionReservationCleanupService.cs` — AutoConfirm 처리 로직 위치 참조
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Chat/ChatService.cs` — FCM 발송 패턴 참조 (`_notificationService` 호출 방식)
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/NotificationController.cs` — 알림 타입 코드 참조
  - `TicketPlatFormServer/TicketPlatFormServer/database_history/TASK-008-migration.sql` — REVIEW_REQUEST 시드 확인 (Task 1 완료 후)

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer
    Expected Result: Build succeeded
    Evidence: .sisyphus/evidence/task-5-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-5-build.txt`
  **Commit**: NO (Task 7과 함께)

---

- [ ] 6. ReputationService 구현

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Reputation/IReputationService.cs` 인터페이스 작성:
    ```csharp
    public interface IReputationService
    {
        Task CreateAsync(long requestUserId, CreateReputationReqDto dto);
        Task<ReputationListRespDto> GetByUserIdAsync(long targetUserId, int page, int size);
        Task<ReputationCheckRespDto> CheckAsync(long requestUserId, long transactionId);
    }
    ```
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Reputation/ReputationService.cs` 구현:

    **CreateAsync 비즈니스 로직 순서** (순서 엄수):
    1. `transactionId`로 Transaction 조회 → 없으면 `AppException(404)`
    2. `requestUserId == Transaction.BuyerId` 검증 → 아니면 `AppException(403, "구매자만 리뷰 작성 가능")`
    3. `Transaction.CancelledAt != null` → `AppException(400, "취소된 거래")`
    4. `Transaction.ConfirmedAt == null` → `AppException(400, "구매확정 전")`
    5. `(DateTime.UtcNow - Transaction.ConfirmedAt.Value).TotalDays > 7` → `AppException(400, "리뷰 작성 기간 만료")`
    6. 분쟁 OPEN 여부 확인 (Dispute 테이블 조회) → 있으면 `AppException(400, "분쟁 진행 중")`
    7. `_reputationRepo.ExistsAsync(transactionId, requestUserId)` → true면 `AppException(409, "이미 리뷰 작성")`
    8. **EF 트랜잭션 시작** (원자성 보장):
       - `UserReputation` INSERT (`RatingTypeId = 1` 고정)
       - `delta = (dto.Score - 3) * 1.0f` 계산
       - 현재 `average_rating`, `review_count` 조회 → `newAvgRating` 계산
       - `_reputationRepo.UpdateUserProfileStatsAsync(sellerId, delta, newAvgRating, newCount)`
       - `await efTransaction.CommitAsync()`
    9. 판매자에게 FCM 발송 (선택적: "새 리뷰가 도착했어요")

    **CheckAsync 로직**:
    - Transaction 조회 → 구매자 확인 → CanReview/HasReviewed/ReviewDeadline 계산 후 `ReputationCheckRespDto` 반환

  - `TicketPlatFormServer/TicketPlatFormServer/Program.cs`에 DI 등록:
    `builder.Services.AddScoped<IReputationService, ReputationService>();`

  **Must NOT do**:
  - Repository 병렬 호출 금지 (`Task.WhenAll` + IDbConnection Scoped 위반)
  - 트랜잭션 없이 INSERT + UPDATE 분리 실행 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES (Task 7과 동시, Task 8은 이후)
  - **Parallel Group**: Wave 3
  - **Blocks**: Task 8
  - **Blocked By**: Task 3, 4

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Dispute/DisputeService.cs` — Service 구현 패턴 (AppException 사용, 순차 조회 패턴)
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Chat/ChatService.cs` (ConfirmPurchase ~580줄) — EF 트랜잭션 + FCM 패턴 완전 참조
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/README.md` — EF 트랜잭션 + Dapper 공유 패턴
  - `TicketPlatFormServer/TicketPlatFormServer/Common/Exception/AppException.cs` — 예외 처리 패턴

  **Acceptance Criteria**:
  - [ ] Score 0 또는 6 입력 시 400 반환
  - [ ] 구매자가 아닌 사람이 리뷰 시도 → 403 반환
  - [ ] 7일 초과 후 리뷰 시도 → 400 반환
  - [ ] 동일 거래 2번째 리뷰 → 409 반환
  - [ ] 취소된 거래 리뷰 → 400 반환

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer
    Expected Result: Build succeeded, 0 Error(s)
    Evidence: .sisyphus/evidence/task-6-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-6-build.txt`
  **Commit**: NO (Task 8과 함께)

---

- [ ] 7. ChatService.ConfirmPurchase() — 리뷰 요청 FCM 훅 추가

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Chat/ChatService.cs`의 `ConfirmPurchase()` 메서드에서 기존 판매자 FCM 발송 코드 다음 줄에 구매자 대상 REVIEW_REQUEST FCM 추가:
    ```csharp
    // 기존 판매자 FCM 발송 코드 다음에 추가
    await _notificationService.SendAsync(
        userId: transaction.BuyerId,
        type: "REVIEW_REQUEST",
        title: "거래는 어떠셨나요?",
        body: $"{sellerNickname} 판매자에 대한 리뷰를 남겨주세요.",
        data: new { transactionId = transaction.Id }
    );
    ```
  - 삽입 위치: `ConfirmPurchase()` 내 에스크로 해제 → PURCHASE_CONFIRMED 메시지 생성 → 판매자 FCM 완료 직후 (~580번째 줄)
  - Task 1의 마이그레이션으로 `notification_types`에 REVIEW_REQUEST 시드가 이미 추가됐다고 가정

  **Must NOT do**:
  - ConfirmPurchase 기존 로직(에스크로 해제, 메시지 생성, 판매자 FCM) 수정 금지
  - AutoConfirm 서비스 수정 금지 (Task 5에서 처리)

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (Task 6과 동시)
  - **Blocks**: 없음
  - **Blocked By**: Task 4

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Chat/ChatService.cs` (~580줄 `ConfirmPurchase`) — 정확한 삽입 위치 확인 (파일 열어서 메서드 끝 부분 확인)

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer
    Expected Result: Build succeeded
    Evidence: .sisyphus/evidence/task-7-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-7-build.txt`
  **Commit**: YES (Task 5와 함께)
  - Message: `feat(notification): add REVIEW_REQUEST FCM on purchase confirm (manual + auto)`
  - Files: `TicketPlatFormServer/TicketPlatFormServer/BackgroundServices/[AutoConfirmService].cs`, `TicketPlatFormServer/TicketPlatFormServer/Services/Chat/ChatService.cs`

---

- [ ] 8. ReputationController — API 엔드포인트 구현

  **What to do**:
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/ReputationController.cs` 생성:
    ```csharp
    [ApiController]
    [Route("api/reputations")]
    [Authorize]
    public class ReputationController : ControllerBase
    {
        // POST /api/reputations
        // GET /api/reputations/check/{transactionId}
    }
    ```
  - `POST /api/reputations` 구현:
    - `[HttpPost]`, `[Authorize]`
    - Body: `CreateReputationReqDto`
    - 응답: `201 Created`, `ApiResponse<long>` (생성된 ReputationId)
    - 오류: 400, 403, 404, 409
  - `GET /api/reputations/check/{transactionId}` 구현:
    - `[HttpGet("check/{transactionId}")]`, `[Authorize]`
    - 응답: `200 OK`, `ApiResponse<ReputationCheckRespDto>`
  - `GET /api/users/{userId}/reputations` 구현 (UserController에 추가 또는 ReputationController에 별도):
    - QueryString: `page=1&size=20`
    - 응답: `200 OK`, `ApiResponse<ReputationListRespDto>`
    - `[AllowAnonymous]` (공개 프로필)
  - Swagger XML 주석 추가 (`/// <summary>`)

  **Must NOT do**:
  - 비즈니스 로직 Controller에 포함 금지
  - Entity 직접 노출 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (Task 6 완료 후)
  - **Parallel Group**: Wave 3 후반
  - **Blocks**: 없음 (최종 통합)
  - **Blocked By**: Task 6

  **References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/DisputeController.cs` — Controller 구조 및 Swagger 주석 패턴 완전 참조
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/ApiResponse.cs` — 응답 래퍼 형식 (`ApiResponse<T>` 생성 방식)
  - `TicketPlatFormServer/TicketPlatFormServer/Common/Exception/AppException.cs` — 예외 → HTTP 상태코드 매핑 확인

  **Acceptance Criteria**:
  - [ ] `POST /api/reputations` → 정상 리뷰 → 201, ReputationId 반환
  - [ ] `POST /api/reputations` (중복) → 409 반환
  - [ ] `POST /api/reputations` (7일 초과) → 400 반환
  - [ ] `POST /api/reputations` (구매자 아님) → 403 반환
  - [ ] `GET /api/reputations/check/{transactionId}` → CanReview/HasReviewed/ReviewDeadline 정확히 반환
  - [ ] `GET /api/users/{userId}/reputations` → 리뷰 목록 + TotalCount + AverageRating 반환
  - [ ] 리뷰 작성 후 `GET /api/users/{sellerId}/profile` → AverageRating, ReviewCount 갱신 반영
  - [ ] 리뷰 작성 후 MannerTemperature 0°~100° 범위 내 유지

  **QA Scenarios**:

  ```
  Scenario: 정상 리뷰 작성 (Happy Path)
    Tool: Bash (curl)
    Preconditions:
      - 구매확정 완료된 거래 ID: {confirmedTransactionId}
      - 구매자 JWT: {buyerToken}
    Steps:
      1. curl -X POST http://localhost:5224/api/reputations \
           -H "Authorization: Bearer {buyerToken}" \
           -H "Content-Type: application/json" \
           -d '{"transactionId": {confirmedTransactionId}, "score": 5, "comment": "친절한 판매자였어요!"}'
      2. 응답 statusCode 확인 → 201
      3. curl http://localhost:5224/api/users/{sellerId}/reputations
      4. AverageRating, ReviewCount 변경 확인
      5. curl http://localhost:5224/api/users/{sellerId}/profile → AverageRating 반영 확인
    Expected Result:
      - 201 Created, reputationId 반환
      - 판매자 프로필 AverageRating = 5.00, ReviewCount = 1
      - MannerTemperature += 2 (max 100)
    Evidence: .sisyphus/evidence/task-8-happy-path.json

  Scenario: 중복 리뷰 차단 (Failure Path)
    Tool: Bash (curl)
    Preconditions: 위 Happy Path 완료 후
    Steps:
      1. 동일 transactionId로 재 POST /api/reputations
    Expected Result: 409 Conflict, "이미 리뷰 작성" 메시지
    Evidence: .sisyphus/evidence/task-8-duplicate.json

  Scenario: 비구매자 리뷰 시도 (Failure Path)
    Tool: Bash (curl)
    Preconditions: 판매자 JWT: {sellerToken}
    Steps:
      1. 판매자 JWT로 POST /api/reputations (자기 거래)
    Expected Result: 403 Forbidden
    Evidence: .sisyphus/evidence/task-8-forbidden.json

  Scenario: 기간 만료 리뷰 시도 (Failure Path)
    Tool: Bash (mysql + curl)
    Steps:
      1. mysql: UPDATE transactions SET confirmed_at = DATE_SUB(NOW(), INTERVAL 8 DAY) WHERE id = {confirmedTransactionId}
      2. curl -X POST http://localhost:5224/api/reputations ... (동일 payload)
    Expected Result: 400 Bad Request, "리뷰 작성 기간 만료" 메시지
    Evidence: .sisyphus/evidence/task-8-expired.json
  ```

  **Evidence**: `.sisyphus/evidence/task-8-*.json`
  **Commit**: YES
  - Message: `feat(reputation): implement reputation API - create, list, check endpoints`
  - Pre-commit: `dotnet build TicketPlatFormServer`
  - Files:
    - `TicketPlatFormServer/TicketPlatFormServer/Controllers/ReputationController.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/Services/Reputation/IReputationService.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/Services/Reputation/ReputationService.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/Repository/Reputation/IReputationRepository.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/Repository/Reputation/ReputationRepository.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/DTO/Reputation/*.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/DTO/User/UserProfileDto.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/DTO/Chat/ChatRoomDetailRespDto.cs`
    - `TicketPlatFormServer/TicketPlatFormServer/Program.cs`

---

## Final Verification Wave

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Must Have 7개 항목 전체 curl로 검증. DB 유니크 제약 실제 동작 확인. MannerTemperature 0~100 클램핑 확인. `.sisyphus/evidence/` 파일 존재 확인.
  Output: `Must Have [N/7] | Must NOT Have [N/5] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  `dotnet build TicketPlatFormServer` + 레이어 분리 위반 탐색 (DTO in Repository, Entity in Controller). `Task.WhenAll` + repository 병렬 호출 패턴 탐색. `as` 다운캐스팅 오용 탐색.
  Output: `Build [PASS/FAIL] | Layer Violations [N] | Parallel Violations [N] | VERDICT`

- [ ] F3. **Real Manual QA** — `unspecified-high`
  전체 QA 시나리오 실행. 리뷰 작성 → 프로필 AverageRating 반영 엔드투엔드. MannerTemperature 실제 DB 값 확인.
  Output: `Scenarios [N/N pass] | VERDICT`

---

## Commit Strategy

| 커밋 | 메시지 | 주요 파일 |
|------|--------|-----------|
| 1 | `chore(db): add reputation migration - unique constraint, seed, columns` | `database_history/TASK-008-migration.sql`, `DTO/Reputation/*.cs` |
| 2 | `feat(notification): add REVIEW_REQUEST FCM on purchase confirm` | `Services/Chat/ChatService.cs`, `BackgroundServices/[AutoConfirmService].cs` |
| 3 | `feat(reputation): implement reputation API` | `Controllers/ReputationController.cs`, `Services/Reputation/`, `Repository/Reputation/`, `DTO/User/UserProfileDto.cs`, `DTO/Chat/ChatRoomDetailRespDto.cs`, `Program.cs` |

---

## Success Criteria

```bash
# 빌드 확인
dotnet build TicketPlatFormServer
# Expected: Build succeeded, 0 Error(s)

# 정상 리뷰 작성
curl -X POST http://localhost:5224/api/reputations \
  -H "Authorization: Bearer {buyerToken}" \
  -H "Content-Type: application/json" \
  -d '{"transactionId": N, "score": 5}'
# Expected: HTTP 201, {"data": reputationId}

# 리뷰 목록 조회
curl http://localhost:5224/api/users/{sellerId}/reputations
# Expected: HTTP 200, {"data": {"items": [...], "totalCount": 1, "averageRating": 5.0}}
```

### Final Checklist
- [ ] API 3개 엔드포인트 정상 동작
- [ ] DB 유니크 제약으로 중복 리뷰 차단
- [ ] 7일 기간 제한 적용
- [ ] 분쟁 중 리뷰 차단
- [ ] MannerTemperature 0~100 범위 내 원자적 업데이트 (LEAST/GREATEST/COALESCE)
- [ ] 수동/자동 구매확정 모두 FCM 발송
- [ ] AverageRating, ReviewCount 비정규화 컬럼 갱신
