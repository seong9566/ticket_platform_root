# TASK-012 판매 내역 리팩토링 — Mobile 계획

## TL;DR

> **요약**: 기존 '판매 내역' 탭을 공연 기준 그룹핑 "판매 관리 대시보드"로 완전 교체한다.
> Backend의 새 API(`GET /api/sell/sales-dashboard`, `GET /api/sell/sales-dashboard/{eventId}`)를 연동하고,
> 공연 그룹 카드 UI + 공연별 티켓 상세 화면을 구현한다.
>
> **산출물**:
> - `ticket_platform_mobile/lib/features/sales_dashboard/` — 판매 대시보드 피처 디렉토리 (data + domain + presentation)
> - `SalesDashboardView` — 공연별 그룹 카드 리스트 (상태 필터, page/size 페이지네이션)
> - `EventTicketListView` — 특정 공연의 개별 티켓 목록 화면 (새 화면 push)
> - `EventGroupCard` — 공연 그룹 카드 위젯 (포스터+공연명+일시+장소+상태별 수량 뱃지)
> - 기존 `TransactionHistoryView` 판매 탭 → 대시보드 연결 변경
> - `ApiEndpoint` + `AppRouterPath` 확장
>
> **예상 공수**: Medium
> **병렬 실행**: YES — 3 Wave
> **Android-First**: 모든 QA는 Android 에뮬레이터 기준

---

## Context

### ⚠️ 작업 기준 디렉토리
모든 파일 경로는 `ticket_platform_mobile/lib/` 아래를 기준으로 한다.
(예: `features/...` → `ticket_platform_mobile/lib/features/...`)

### 기획 확정 사항
| 항목 | 결정 |
|------|------|
| 화면 전환 | 기존 '판매 내역' 탭을 완전 교체 → 판매 관리 대시보드 |
| 상세 진입 | 공연 카드 탭 → 새 화면 push (RULE-01 준수) |
| 상태 카테고리 | 총 티켓 / 판매중 / 판매 완료 / 정산 중 (4가지) |
| 필터 | 상태 필터만 (전체/판매중/완료/정산 중). 기간/정렬 제거 |
| 코드 생성 | freezed/json_serializable 재생성 필요 (`*.freezed.dart`, `*.g.dart`) |
| UI 가이드 | `ticket_platform_mobile/.agent/rules/ui_ux_guide.md` 10개 Rule 준수 |

### 백엔드 API (TASK-012 Backend에서 제공)
- `GET /api/sell/sales-dashboard?status=on_sale&page=1&size=20` — 공연별 그룹핑 대시보드
  - Response: `{ eventGroups: [{ eventId, eventTitle, posterImageUrl, venueName, earliestEventDatetime, totalCount, onSaleCount, completedCount, settlingCount }], page, size, totalCount, hasMore }`
- `GET /api/sell/sales-dashboard/{eventId}?page=1&size=20` — 공연별 티켓 상세
  - Response: `{ eventId, eventTitle, tickets: [{ ticketId, seatInfo, quantity, remainingQuantity, price, originalPrice, statusCode, statusName, transactionId, thumbnailUrl, createdAt }], page, size, totalCount, hasMore }`

### 수정 대상 기존 파일
- `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart` — '판매 내역' 메뉴 라우트 변경
- `ticket_platform_mobile/lib/core/router/app_router_path.dart` — 대시보드 라우트 추가
- `ticket_platform_mobile/lib/core/network/api_endpoint.dart` — 대시보드 API 엔드포인트 추가

### 신규 생성 디렉토리
```
ticket_platform_mobile/lib/features/sales_dashboard/
├── data/
│   ├── dto/
│   │   └── response/
│   │       ├── sales_dashboard_resp_dto.dart       ← @freezed, EventGroupItemDto 포함
│   │       └── event_ticket_list_resp_dto.dart     ← @freezed, EventTicketItemDto 포함
│   ├── datasources/sales_dashboard_remote_data_source.dart
│   └── repositories/sales_dashboard_repository_impl.dart
├── domain/
│   ├── entities/
│   │   ├── event_group_entity.dart
│   │   └── event_ticket_entity.dart
│   ├── repositories/sales_dashboard_repository.dart
│   └── usecases/
│       ├── get_sales_dashboard_usecase.dart
│       └── get_event_tickets_usecase.dart
└── presentation/
    ├── providers/sales_dashboard_providers_di.dart   ← @riverpod DI
    ├── viewmodels/
    │   ├── sales_dashboard_viewmodel.dart
    │   └── event_ticket_list_viewmodel.dart
    ├── views/
    │   ├── sales_dashboard_view.dart
    │   └── event_ticket_list_view.dart
    ├── ui_models/
    │   ├── event_group_ui_model.dart
    │   └── event_ticket_ui_model.dart
    └── widgets/
        ├── event_group_card.dart
        ├── event_ticket_item.dart
        └── sales_status_filter_bar.dart
```

### 참조 패턴
- `ticket_platform_mobile/lib/features/dispute/` — 피처 디렉토리 구조 완전 참조 (data/domain/presentation 레이어)
- `ticket_platform_mobile/lib/features/profile/presentation/views/transaction_history_view.dart` — 리스트 화면 + 필터 패턴
- `ticket_platform_mobile/lib/features/profile/presentation/widgets/transaction_history_item.dart` — 카드 아이템 위젯 패턴
- `ticket_platform_mobile/lib/features/profile/presentation/viewmodels/transaction_history_viewmodel.dart` — ViewModel 패턴 (페이지네이션, 필터, 상태 관리)
- `ticket_platform_mobile/lib/features/profile/presentation/ui_models/transaction_history_ui_model.dart` — UI Model 패턴 (포맷팅, 색상 매핑)
- `ticket_platform_mobile/lib/features/profile/data/dto/response/transaction_item_resp_dto.dart` — @freezed DTO 패턴
- `ticket_platform_mobile/.agent/rules/ui_ux_guide.md` — UI/UX 10개 Rule Set

### UI/UX 가이드 핵심 Rule (반드시 준수)
- **RULE-01**: 하나의 화면에는 하나의 핵심 목적만 존재
- **RULE-02**: 시각적 밀도를 낮추고 정보 계층을 명확히. 3초 안에 화면 목적 인지
- **RULE-04**: 공간은 전략적 도구. 여백 > 구분선
- **RULE-09**: 트렌드는 UX를 강화할 때만 사용. 가독성·접근성을 해치면 즉시 제거

---

## Work Objectives

### 핵심 목표
프로필 화면에서 '판매 내역'을 탭하면, 공연 기준 그룹핑된 판매 대시보드를 보여준다.
각 공연 카드에 상태별 수량(총/판매중/완료/정산중)이 표시되며,
카드를 탭하면 해당 공연의 개별 티켓 목록 화면으로 이동한다.

### Must Have
- 판매 대시보드 화면: 공연 그룹 카드 리스트, 상태 필터(전체/판매중/완료/정산중), page/size 페이지네이션
- 공연 그룹 카드: 포스터 이미지 + 공연명 + 일시 + 장소 + 상태별 수량 뱃지
- 공연별 티켓 상세 화면: 개별 티켓 목록 (좌석정보, 가격, 수량, 상태)
- 상태 뱃지 색상: on_sale=성공(녹색), completed=완료(파란색), settling=경고(노란색)
- 데이터 결손 fallback: 공연명 없음 → "공연 정보 없음", 장소 없음 → "장소 미정", 포스터 없음 → 기본 아이콘
- Clean Architecture: DTO(@freezed+fromJson+toEntity) → Entity → UiModel → Widget
- 모든 Provider는 @riverpod annotation 사용
- 기존 '판매 내역' 탭에서 새 대시보드 화면으로 라우팅 변경
- `dart run build_runner build` 코드 생성 성공

### Must NOT Have (가드레일)
- 기존 구매 내역 탭 변경 금지
- 기존 TransactionHistoryView의 구매 탭 로직 수정 금지
- 판매 등록 플로우 변경 금지
- 티켓 상세 화면(ticketDetail) 변경 금지
- 매출/수익 통계 화면 포함 금지
- 정산 처리 UI (TASK-009 Mobile 영역) 포함 금지
- 과도한 애니메이션/시각 효과 금지 (RULE-09)
- 화면 당 2개 이상 목적 금지 (RULE-01)

---

## Verification Strategy

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed.

### Test Decision
- **Infrastructure exists**: NO
- **Automated tests**: None
- **Framework**: N/A

### QA Policy
Every task MUST include agent-executed QA scenarios.
Evidence saved to `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`.

- **Build**: Use Bash — `flutter build apk --debug` or `flutter analyze`
- **Code Gen**: Use Bash — `dart run build_runner build --delete-conflicting-outputs`
- **UI**: Use Playwright skill — Android 에뮬레이터에서 UI 검증

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Foundation — DTO + Entity + Repository Interface + Route/Endpoint):
├── Task 1: 응답 DTO 2개 (@freezed) [quick]
├── Task 2: Domain Entity 2개 + Repository Interface + UseCase 2개 [quick]
├── Task 3: ApiEndpoint + AppRouterPath 확장 [quick]
└── Task 4: DataSource + Repository 구현체 [quick]

Wave 2 (Core — ViewModel + UiModel + DI Provider):
├── Task 5: UiModel 2개 (포맷팅/색상) (depends: 2) [quick]
├── Task 6: ViewModel 2개 (depends: 2, 4) [unspecified-high]
└── Task 7: DI Provider + 코드 생성 (depends: 1, 2, 4, 6) [quick]

Wave 3 (UI — Widget + View + 라우팅 통합):
├── Task 8: EventGroupCard + EventTicketItem + SalesStatusFilterBar 위젯 (depends: 5) [visual-engineering]
├── Task 9: SalesDashboardView + EventTicketListView 화면 (depends: 6, 7, 8) [visual-engineering]
└── Task 10: 라우팅 통합 + 빌드 검증 (depends: 3, 9) [quick]

Wave FINAL (독립 검증, 4개 병렬):
├── F1: Plan compliance audit (oracle)
├── F2: Code quality review (unspecified-high)
├── F3: UI QA — Android 에뮬레이터 (unspecified-high + playwright)
└── F4: Scope fidelity check (deep)

Critical Path: Task 1 → Task 7 → Task 9 → Task 10 → F1-F4
```

### Dependency Matrix
| Task | Depends On | Blocks |
|------|-----------|--------|
| 1 | — | 7 |
| 2 | — | 5, 6, 7 |
| 3 | — | 10 |
| 4 | — | 6, 7 |
| 5 | 2 | 8 |
| 6 | 2, 4 | 7, 9 |
| 7 | 1, 2, 4, 6 | 9 |
| 8 | 5 | 9 |
| 9 | 6, 7, 8 | 10 |
| 10 | 3, 9 | F1-F4 |

### Agent Dispatch Summary
- **Wave 1**: 4 tasks — T1-T4 `quick`
- **Wave 2**: 3 tasks — T5 `quick`, T6 `unspecified-high`, T7 `quick`
- **Wave 3**: 3 tasks — T8 `visual-engineering`, T9 `visual-engineering`, T10 `quick`
- **FINAL**: 4 tasks — F1 `oracle`, F2 `unspecified-high`, F3 `unspecified-high`+`playwright`, F4 `deep`

---

## TODOs

- [ ] 1. 응답 DTO 2개 (@freezed)

  **What to do**:
  - `sales_dashboard_resp_dto.dart` 생성 — `features/sales_dashboard/data/dto/response/` 아래
    ```dart
    @freezed
    abstract class SalesDashboardRespDto with _$SalesDashboardRespDto {
      const factory SalesDashboardRespDto({
        required List<EventGroupItemDto> eventGroups,
        required int page,
        required int size,
        required int totalCount,
        required bool hasMore,
      }) = _SalesDashboardRespDto;
      factory SalesDashboardRespDto.fromJson(Map<String, dynamic> json) =>
          _$SalesDashboardRespDtoFromJson(json);
    }

    @freezed
    abstract class EventGroupItemDto with _$EventGroupItemDto {
      const factory EventGroupItemDto({
        required int eventId,
        required String eventTitle,
        String? posterImageUrl,
        String? venueName,
        String? earliestEventDatetime,
        required int totalCount,
        required int onSaleCount,
        required int completedCount,
        required int settlingCount,
      }) = _EventGroupItemDto;
      factory EventGroupItemDto.fromJson(Map<String, dynamic> json) =>
          _$EventGroupItemDtoFromJson(json);
    }
    ```
  - `toEntity()` extension 추가 (같은 파일 하단):
    - `SalesDashboardRespDtoX.toEntity()` → `SalesDashboardEntity` 변환
    - `EventGroupItemDtoX.toEntity()` → `EventGroupEntity` 변환
    - `earliestEventDatetime` String? → `DateTime.parse()` 변환
  - `event_ticket_list_resp_dto.dart` 생성 — 같은 디렉토리
    ```dart
    @freezed
    abstract class EventTicketListRespDto with _$EventTicketListRespDto {
      const factory EventTicketListRespDto({
        required int eventId,
        required String eventTitle,
        required List<EventTicketItemDto> tickets,
        required int page,
        required int size,
        required int totalCount,
        required bool hasMore,
      }) = _EventTicketListRespDto;
      factory EventTicketListRespDto.fromJson(Map<String, dynamic> json) =>
          _$EventTicketListRespDtoFromJson(json);
    }

    @freezed
    abstract class EventTicketItemDto with _$EventTicketItemDto {
      const factory EventTicketItemDto({
        required int ticketId,
        String? seatInfo,
        required int quantity,
        required int remainingQuantity,
        required int price,
        required int originalPrice,
        required String statusCode,
        required String statusName,
        int? transactionId,
        String? thumbnailUrl,
        required String createdAt,
      }) = _EventTicketItemDto;
      factory EventTicketItemDto.fromJson(Map<String, dynamic> json) =>
          _$EventTicketItemDtoFromJson(json);
    }
    ```
  - `toEntity()` extension 추가:
    - `EventTicketListRespDtoX.toEntity()` → `EventTicketListEntity` 변환
    - `EventTicketItemDtoX.toEntity()` → `EventTicketEntity` 변환
    - `createdAt` String → `DateTime.parse()` 변환
  - 각 파일에 `part '*.freezed.dart'` + `part '*.g.dart'` 선언 필수

  **Must NOT do**:
  - 기존 profile 피처의 DTO 파일 수정 금지
  - 요청 DTO에 @freezed 사용 금지 (응답 DTO만 @freezed — CLAUDE.md 규칙)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: @freezed DTO 2개 + toEntity extension, 기존 패턴 복제
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 2, 3, 4)
  - **Blocks**: Task 7
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/features/profile/data/dto/response/transaction_item_resp_dto.dart` — @freezed DTO 완전 패턴: part 선언, fromJson factory, toEntity() extension, 내부 DTO 클래스 분리

  **API/Type References**:
  - `.sisyphus/plans/backend/TASK-012.md` Task 2 — Backend DTO 필드 정의 (EventGroupItemDto 9개 필드, EventTicketItemDto 11개 필드 1:1 매칭)

  **WHY Each Reference Matters**:
  - `transaction_item_resp_dto.dart`: @freezed + part 선언 + fromJson + toEntity extension의 정확한 프로젝트 표준. 이 패턴을 그대로 따라야 build_runner 코드 생성이 정상 동작함

  **Acceptance Criteria**:
  - [ ] `features/sales_dashboard/data/dto/response/sales_dashboard_resp_dto.dart` 존재 — `SalesDashboardRespDto` + `EventGroupItemDto` + `toEntity()` extension 2개
  - [ ] `features/sales_dashboard/data/dto/response/event_ticket_list_resp_dto.dart` 존재 — `EventTicketListRespDto` + `EventTicketItemDto` + `toEntity()` extension 2개
  - [ ] 각 파일에 `part '*.freezed.dart'`, `part '*.g.dart'` 선언 존재
  - [ ] Backend API 응답 필드와 DTO 필드 1:1 매칭 (필드명 camelCase 일치)

  **QA Scenarios**:

  ```
  Scenario: DTO 파일 생성 및 패턴 검증
    Tool: Bash
    Preconditions: sales_dashboard 디렉토리 구조 생성 완료
    Steps:
      1. ls ticket_platform_mobile/lib/features/sales_dashboard/data/dto/response/sales_dashboard_resp_dto.dart → 파일 존재
      2. ls ticket_platform_mobile/lib/features/sales_dashboard/data/dto/response/event_ticket_list_resp_dto.dart → 파일 존재
      3. grep -c "@freezed" ticket_platform_mobile/lib/features/sales_dashboard/data/dto/response/sales_dashboard_resp_dto.dart → 2 (SalesDashboardRespDto + EventGroupItemDto)
      4. grep -c "toEntity" ticket_platform_mobile/lib/features/sales_dashboard/data/dto/response/sales_dashboard_resp_dto.dart → 2 이상
    Expected Result: 두 파일 존재, @freezed 어노테이션 2개씩 + toEntity extension 포함
    Failure Indicators: 파일 미존재 또는 @freezed/toEntity 누락
    Evidence: .sisyphus/evidence/task-1-dto-files.txt
  ```

  **Commit**: YES (groups with Wave 1)
  - Message: `feat(sales-dashboard): add response DTOs with freezed`
  - Files: `sales_dashboard_resp_dto.dart`, `event_ticket_list_resp_dto.dart`
  - Pre-commit: `flutter analyze`

- [ ] 2. Domain Entity 2개 + Repository Interface + UseCase 2개

  **What to do**:
  - `event_group_entity.dart` 생성 — `features/sales_dashboard/domain/entities/` 아래
    ```dart
    @freezed
    abstract class EventGroupEntity with _$EventGroupEntity {
      const factory EventGroupEntity({
        required int eventId,
        required String eventTitle,
        String? posterImageUrl,
        String? venueName,
        DateTime? earliestEventDatetime,
        required int totalCount,
        required int onSaleCount,
        required int completedCount,
        required int settlingCount,
      }) = _EventGroupEntity;
    }

    @freezed
    abstract class SalesDashboardEntity with _$SalesDashboardEntity {
      const factory SalesDashboardEntity({
        required List<EventGroupEntity> eventGroups,
        required int page,
        required int size,
        required int totalCount,
        required bool hasMore,
      }) = _SalesDashboardEntity;
    }
    ```
  - `event_ticket_entity.dart` 생성 — 같은 디렉토리
    ```dart
    @freezed
    abstract class EventTicketEntity with _$EventTicketEntity {
      const factory EventTicketEntity({
        required int ticketId,
        String? seatInfo,
        required int quantity,
        required int remainingQuantity,
        required int price,
        required int originalPrice,
        required String statusCode,
        required String statusName,
        int? transactionId,
        String? thumbnailUrl,
        required DateTime createdAt,
      }) = _EventTicketEntity;
    }

    @freezed
    abstract class EventTicketListEntity with _$EventTicketListEntity {
      const factory EventTicketListEntity({
        required int eventId,
        required String eventTitle,
        required List<EventTicketEntity> tickets,
        required int page,
        required int size,
        required int totalCount,
        required bool hasMore,
      }) = _EventTicketListEntity;
    }
    ```
  - `sales_dashboard_repository.dart` 생성 — `features/sales_dashboard/domain/repositories/` 아래
    ```dart
    abstract class SalesDashboardRepository {
      Future<SalesDashboardEntity> getSalesDashboard({
        String? status,
        int page = 1,
        int size = 20,
      });
      Future<EventTicketListEntity> getEventTickets({
        required int eventId,
        int page = 1,
        int size = 20,
      });
    }
    ```
  - `get_sales_dashboard_usecase.dart` 생성 — `features/sales_dashboard/domain/usecases/` 아래
    - Repository 주입, `call()` 메서드로 `getSalesDashboard()` 위임
  - `get_event_tickets_usecase.dart` 생성 — 같은 디렉토리
    - Repository 주입, `call()` 메서드로 `getEventTickets()` 위임
  - 각 Entity 파일에 `part '*.freezed.dart'` 선언 필수 (Entity는 `.g.dart` 불필요)

  **Must NOT do**:
  - Entity에 `fromJson()` 추가 금지 (DTO→Entity 변환은 DTO의 toEntity()에서 처리)
  - 기존 profile/domain 수정 금지
  - UseCase에 비즈니스 로직 추가 금지 (순수 위임만)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: @freezed Entity 4개 + abstract Repository + UseCase 2개, 보일러플레이트 생성
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 3, 4)
  - **Blocks**: Tasks 5, 6, 7
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/features/profile/domain/entities/transaction_history_entity.dart` — @freezed Entity 패턴: part 선언, 순수 데이터 필드, DateTime 타입 사용
  - `ticket_platform_mobile/lib/features/dispute/domain/repositories/dispute_repository.dart` — abstract Repository 패턴: Entity 반환, named parameters
  - `ticket_platform_mobile/lib/features/dispute/domain/usecases/get_disputes_usecase.dart` — UseCase 패턴: Repository 주입, call() 메서드

  **WHY Each Reference Matters**:
  - `transaction_history_entity.dart`: Entity에 @freezed만 사용하고 fromJson은 포함하지 않는 프로젝트 규칙 확인
  - `dispute_repository.dart`: Repository가 Entity만 반환하는 Clean Architecture 패턴 (DTO 반환 금지)
  - `get_disputes_usecase.dart`: UseCase가 Repository를 생성자 주입받고 call()로 위임하는 표준 패턴

  **Acceptance Criteria**:
  - [ ] `event_group_entity.dart` — `EventGroupEntity` + `SalesDashboardEntity` @freezed 클래스
  - [ ] `event_ticket_entity.dart` — `EventTicketEntity` + `EventTicketListEntity` @freezed 클래스
  - [ ] `sales_dashboard_repository.dart` — abstract class, Entity 반환, DTO 반환 없음
  - [ ] `get_sales_dashboard_usecase.dart`, `get_event_tickets_usecase.dart` — Repository 주입 + call() 위임
  - [ ] `flutter analyze` 에러 없음

  **QA Scenarios**:

  ```
  Scenario: Domain 레이어 생성 검증
    Tool: Bash
    Preconditions: 디렉토리 구조 생성 완료
    Steps:
      1. ls ticket_platform_mobile/lib/features/sales_dashboard/domain/entities/event_group_entity.dart → 존재
      2. ls ticket_platform_mobile/lib/features/sales_dashboard/domain/entities/event_ticket_entity.dart → 존재
      3. ls ticket_platform_mobile/lib/features/sales_dashboard/domain/repositories/sales_dashboard_repository.dart → 존재
      4. ls ticket_platform_mobile/lib/features/sales_dashboard/domain/usecases/get_sales_dashboard_usecase.dart → 존재
      5. ls ticket_platform_mobile/lib/features/sales_dashboard/domain/usecases/get_event_tickets_usecase.dart → 존재
      6. grep -c "@freezed" ticket_platform_mobile/lib/features/sales_dashboard/domain/entities/event_group_entity.dart → 2
      7. grep -c "abstract class SalesDashboardRepository" ticket_platform_mobile/lib/features/sales_dashboard/domain/repositories/sales_dashboard_repository.dart → 1
    Expected Result: 5개 파일 존재, Entity @freezed, Repository abstract
    Failure Indicators: 파일 미존재 또는 Repository가 DTO 타입 반환
    Evidence: .sisyphus/evidence/task-2-domain-layer.txt

  Scenario: Entity에 fromJson 없음 확인 (Clean Architecture 규칙)
    Tool: Bash
    Preconditions: Task 2 완료
    Steps:
      1. grep -c "fromJson" ticket_platform_mobile/lib/features/sales_dashboard/domain/entities/event_group_entity.dart → 0
      2. grep -c "fromJson" ticket_platform_mobile/lib/features/sales_dashboard/domain/entities/event_ticket_entity.dart → 0
    Expected Result: Entity 파일에 fromJson 없음
    Failure Indicators: fromJson 존재 시 아키텍처 위반
    Evidence: .sisyphus/evidence/task-2-no-fromjson.txt
  ```

  **Commit**: YES (groups with Wave 1)
  - Message: `feat(sales-dashboard): add domain entities, repository interface and usecases`
  - Files: `domain/entities/*.dart`, `domain/repositories/*.dart`, `domain/usecases/*.dart`
  - Pre-commit: `flutter analyze`

- [ ] 3. ApiEndpoint + AppRouterPath 확장

  **What to do**:
  - `api_endpoint.dart` 수정 — `core/network/api_endpoint.dart`
    - `// Sell` 섹션 하단에 판매 대시보드 엔드포인트 2개 추가:
    ```dart
    // Sales Dashboard
    static const String salesDashboard = '/api/sell/sales-dashboard';
    static String salesDashboardEventTickets(int eventId) =>
        '/api/sell/sales-dashboard/$eventId';
    ```
  - `app_router_path.dart` 수정 — `core/router/app_router_path.dart`
    - 판매 대시보드 라우트 2개 추가:
    ```dart
    static const salesDashboard = _Route(
      '/sales-dashboard',
      'salesDashboard',
    );
    static const eventTicketList = _Route(
      '/sales-dashboard/event-tickets',
      'eventTicketList',
    );
    ```

  **Must NOT do**:
  - 기존 `sellMyTickets`, `salesHistory` 엔드포인트 수정/삭제 금지
  - 기존 `transactionHistory` 라우트 수정 금지
  - GoRouter 설정 파일은 이 Task에서 수정하지 않음 (Task 10에서 처리)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 기존 파일에 상수 4줄 추가, 단순 확장
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 4)
  - **Blocks**: Task 10
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/core/network/api_endpoint.dart` — 기존 엔드포인트 상수 패턴: static const String + static String 함수형(pathParam 포함)
  - `ticket_platform_mobile/lib/core/router/app_router_path.dart` — _Route 클래스 패턴: path + name 쌍

  **API/Type References**:
  - `.sisyphus/plans/backend/TASK-012.md` Task 7 — Backend 엔드포인트 경로: `/api/sell/sales-dashboard`, `/api/sell/sales-dashboard/{eventId}`

  **WHY Each Reference Matters**:
  - `api_endpoint.dart`: pathParam이 있는 엔드포인트의 static String 함수 패턴 확인 (disputeDetail, disputeEvidence 등 기존 예시)
  - `app_router_path.dart`: _Route 클래스의 path/name 네이밍 컨벤션 확인

  **Acceptance Criteria**:
  - [ ] `api_endpoint.dart`에 `salesDashboard`, `salesDashboardEventTickets` 상수 존재
  - [ ] `app_router_path.dart`에 `salesDashboard`, `eventTicketList` 라우트 존재
  - [ ] 기존 엔드포인트/라우트 무변경 확인
  - [ ] `flutter analyze` 에러 없음

  **QA Scenarios**:

  ```
  Scenario: 엔드포인트 및 라우트 추가 검증
    Tool: Bash
    Preconditions: 기존 파일 정상 상태
    Steps:
      1. grep -c "salesDashboard" ticket_platform_mobile/lib/core/network/api_endpoint.dart → 2 이상
      2. grep -c "salesDashboard" ticket_platform_mobile/lib/core/router/app_router_path.dart → 1 이상
      3. grep -c "eventTicketList" ticket_platform_mobile/lib/core/router/app_router_path.dart → 1 이상
      4. grep -c "sellMyTickets" ticket_platform_mobile/lib/core/network/api_endpoint.dart → 1 (무변경 확인)
      5. flutter analyze → No issues found
    Expected Result: 새 상수 존재, 기존 상수 무변경, 분석 통과
    Failure Indicators: 상수 누락 또는 기존 상수 변경됨
    Evidence: .sisyphus/evidence/task-3-endpoint-route.txt
  ```

  **Commit**: YES (groups with Wave 1)
  - Message: `feat(sales-dashboard): add API endpoints and router paths`
  - Files: `core/network/api_endpoint.dart`, `core/router/app_router_path.dart`
  - Pre-commit: `flutter analyze`

- [ ] 4. DataSource + Repository 구현체

  **What to do**:
  - `sales_dashboard_remote_data_source.dart` 생성 — `features/sales_dashboard/data/datasources/` 아래
    - abstract class + impl 패턴 (dispute DataSource 패턴 따를 것)
    ```dart
    abstract class SalesDashboardRemoteDataSource {
      Future<BaseResponse<SalesDashboardRespDto>> getSalesDashboard({
        String? status,
        int page = 1,
        int size = 20,
      });
      Future<BaseResponse<EventTicketListRespDto>> getEventTickets({
        required int eventId,
        int page = 1,
        int size = 20,
      });
    }

    class SalesDashboardRemoteDataSourceImpl
        implements SalesDashboardRemoteDataSource {
      final Dio _dio;
      SalesDashboardRemoteDataSourceImpl(this._dio);

      @override
      Future<BaseResponse<SalesDashboardRespDto>> getSalesDashboard({
        String? status,
        int page = 1,
        int size = 20,
      }) async {
        return safeApiCall<SalesDashboardRespDto>(
          apiCall: (options) => _dio.get(
            ApiEndpoint.salesDashboard,
            queryParameters: {
              if (status != null) 'status': status,
              'page': page,
              'size': size,
            },
            options: options,
          ),
          apiName: 'getSalesDashboard',
          dataParser: (json) =>
              SalesDashboardRespDto.fromJson(json as Map<String, dynamic>),
        );
      }

      @override
      Future<BaseResponse<EventTicketListRespDto>> getEventTickets({
        required int eventId,
        int page = 1,
        int size = 20,
      }) async {
        return safeApiCall<EventTicketListRespDto>(
          apiCall: (options) => _dio.get(
            ApiEndpoint.salesDashboardEventTickets(eventId),
            queryParameters: {'page': page, 'size': size},
            options: options,
          ),
          apiName: 'getEventTickets',
          dataParser: (json) =>
              EventTicketListRespDto.fromJson(json as Map<String, dynamic>),
        );
      }
    }
    ```
  - `sales_dashboard_repository_impl.dart` 생성 — `features/sales_dashboard/data/repositories/` 아래
    - `SalesDashboardRepository` 구현체
    - DataSource 주입, `mapOrThrow()` + `dto.toEntity()` 패턴
    ```dart
    class SalesDashboardRepositoryImpl implements SalesDashboardRepository {
      final SalesDashboardRemoteDataSource _remoteDataSource;
      SalesDashboardRepositoryImpl(this._remoteDataSource);

      @override
      Future<SalesDashboardEntity> getSalesDashboard({
        String? status,
        int page = 1,
        int size = 20,
      }) async {
        final response = await _remoteDataSource.getSalesDashboard(
          status: status, page: page, size: size,
        );
        return response.mapOrThrow(
          (dto) => dto.toEntity(),
          errorMessage: '판매 대시보드를 불러오지 못했습니다.',
        );
      }

      @override
      Future<EventTicketListEntity> getEventTickets({
        required int eventId,
        int page = 1,
        int size = 20,
      }) async {
        final response = await _remoteDataSource.getEventTickets(
          eventId: eventId, page: page, size: size,
        );
        return response.mapOrThrow(
          (dto) => dto.toEntity(),
          errorMessage: '공연별 티켓 목록을 불러오지 못했습니다.',
        );
      }
    }
    ```

  **Must NOT do**:
  - 기존 profile DataSource/Repository 수정 금지
  - Repository에서 DTO 타입 반환 금지 (반드시 Entity 반환)
  - `safeApiCall` 외의 API 호출 패턴 사용 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: dispute 피처의 DataSource/Repository 패턴을 그대로 복제하는 보일러플레이트 작업
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 3)
  - **Blocks**: Tasks 6, 7
  - **Blocked By**: None (Task 1의 DTO 타입과 Task 2의 Entity 타입을 import하지만, 파일 자체는 동시에 생성 가능 — build_runner는 Wave 1 완료 후 Task 7에서 실행)

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/features/dispute/data/datasources/dispute_remote_data_source.dart` — DataSource 완전 패턴: abstract + impl, Dio 주입, safeApiCall 래핑, apiName/dataParser 구조
  - `ticket_platform_mobile/lib/features/dispute/data/repositories/dispute_repository_impl.dart` — Repository 구현 패턴: DataSource 주입, mapOrThrow() + toEntity(), 한글 에러 메시지

  **API/Type References**:
  - `ticket_platform_mobile/lib/core/network/base_response.dart` — BaseResponse<T> 타입, mapOrThrow() 메서드
  - `ticket_platform_mobile/lib/core/network/safe_api_call.dart` — safeApiCall<T>() 유틸리티 함수 시그니처

  **WHY Each Reference Matters**:
  - `dispute_remote_data_source.dart`: safeApiCall의 정확한 사용법 (apiCall 콜백 + apiName + dataParser 구조), GET 요청의 queryParameters 전달 방식
  - `dispute_repository_impl.dart`: mapOrThrow()의 정확한 호출 패턴 (첫 번째 인자: dto→entity 변환 람다, errorMessage: 한글 에러)

  **Acceptance Criteria**:
  - [ ] `sales_dashboard_remote_data_source.dart` — abstract class + impl, `safeApiCall` 사용, 2개 메서드
  - [ ] `sales_dashboard_repository_impl.dart` — `SalesDashboardRepository` 구현, `mapOrThrow` + `toEntity()`, Entity 반환
  - [ ] DataSource에 `ApiEndpoint.salesDashboard`, `ApiEndpoint.salesDashboardEventTickets` 사용
  - [ ] Repository에서 DTO 타입 직접 반환 없음 (Entity만 반환)
  - [ ] `flutter analyze` 에러 없음

  **QA Scenarios**:

  ```
  Scenario: DataSource + Repository 생성 및 패턴 검증
    Tool: Bash
    Preconditions: Task 1~3 완료 (DTO, Entity, Endpoint)
    Steps:
      1. ls ticket_platform_mobile/lib/features/sales_dashboard/data/datasources/sales_dashboard_remote_data_source.dart → 존재
      2. ls ticket_platform_mobile/lib/features/sales_dashboard/data/repositories/sales_dashboard_repository_impl.dart → 존재
      3. grep -c "safeApiCall" ticket_platform_mobile/lib/features/sales_dashboard/data/datasources/sales_dashboard_remote_data_source.dart → 2
      4. grep -c "mapOrThrow" ticket_platform_mobile/lib/features/sales_dashboard/data/repositories/sales_dashboard_repository_impl.dart → 2
      5. grep -c "toEntity" ticket_platform_mobile/lib/features/sales_dashboard/data/repositories/sales_dashboard_repository_impl.dart → 2
    Expected Result: 두 파일 존재, safeApiCall 2회, mapOrThrow 2회, toEntity 2회
    Failure Indicators: 파일 미존재, safeApiCall 미사용, DTO 직접 반환
    Evidence: .sisyphus/evidence/task-4-datasource-repo.txt

  Scenario: Repository가 Entity만 반환하는지 확인 (DTO 반환 금지)
    Tool: Bash
    Preconditions: Task 4 완료
    Steps:
      1. grep -c "RespDto" ticket_platform_mobile/lib/features/sales_dashboard/data/repositories/sales_dashboard_repository_impl.dart → import문만 존재 (반환 타입에 없음)
      2. grep "Future<" ticket_platform_mobile/lib/features/sales_dashboard/data/repositories/sales_dashboard_repository_impl.dart → SalesDashboardEntity, EventTicketListEntity만 반환
    Expected Result: 반환 타입이 Entity만 사용
    Failure Indicators: 반환 타입에 DTO 사용 시 아키텍처 위반
    Evidence: .sisyphus/evidence/task-4-entity-only-return.txt
  ```

  **Commit**: YES (groups with Wave 1)
  - Message: `feat(sales-dashboard): add data source and repository implementation`
  - Files: `data/datasources/sales_dashboard_remote_data_source.dart`, `data/repositories/sales_dashboard_repository_impl.dart`
  - Pre-commit: `flutter analyze`

- [ ] 5. UiModel 2개 (포맷팅/색상 매핑)

  **What to do**:
  - `event_group_ui_model.dart` 생성 — `features/sales_dashboard/presentation/ui_models/` 아래
    - `@freezed` 클래스, `fromEntity()` factory 메서드 포함
    - Entity → 화면 표시용 포맷 변환:
      - `eventTitle`: 그대로 (null fallback은 Entity에서 이미 처리)
      - `venueName`: 빈 문자열 → "장소 미정"
      - `eventDate`: `DateFormatUtil.formatCompactDate()` 사용 (earliestEventDatetime)
      - `posterImageUrl`: 그대로 전달 (null이면 위젯에서 기본 아이콘 표시)
      - 상태별 수량 뱃지 텍스트:
        - `totalCountText`: `"총 ${entity.totalCount}건"`
        - `onSaleCountText`: `"판매중 ${entity.onSaleCount}"` (0이면 빈 문자열)
        - `completedCountText`: `"완료 ${entity.completedCount}"` (0이면 빈 문자열)
        - `settlingCountText`: `"정산중 ${entity.settlingCount}"` (0이면 빈 문자열)
      - 상태별 뱃지 색상:
        - `onSaleBadgeColor`: `AppColors.success` (녹색)
        - `completedBadgeColor`: `AppColors.info` 또는 파란 계열 (파란색)
        - `settlingBadgeColor`: `AppColors.badgeWaitingBackground` (노란색)
    ```dart
    @freezed
    abstract class EventGroupUiModel with _$EventGroupUiModel {
      const factory EventGroupUiModel({
        required int eventId,
        required String eventTitle,
        String? posterImageUrl,
        required String venueName,
        required String eventDate,
        required String totalCountText,
        required String onSaleCountText,
        required String completedCountText,
        required String settlingCountText,
        required Color onSaleBadgeColor,
        required Color completedBadgeColor,
        required Color settlingBadgeColor,
      }) = _EventGroupUiModel;

      factory EventGroupUiModel.fromEntity(EventGroupEntity entity) { ... }
    }
    ```
  - `event_ticket_ui_model.dart` 생성 — 같은 디렉토리
    - `@freezed` 클래스, `fromEntity()` factory 메서드 포함
    - Entity → 화면 표시용 포맷 변환:
      - `seatInfo`: null → "좌석 정보 없음"
      - `priceText`: `NumberFormatUtil.formatPrice(entity.price)`
      - `originalPriceText`: `NumberFormatUtil.formatPrice(entity.originalPrice)`
      - `quantityText`: `"${entity.remainingQuantity}/${entity.quantity}장"`
      - `statusText`: entity.statusName 그대로
      - `statusColor`: statusCode 기준 색상 매핑
        - `on_sale` → `AppColors.success` (녹색)
        - `completed` → 파란 계열
        - `settling` → `AppColors.badgeWaitingBackground` (노란색)
        - 기타 → `AppColors.textSecondary`
      - `statusTextColor`: settling일 때 `AppColors.badgeWaitingText`, 나머지 null
      - `dateText`: `DateFormatUtil.formatCompactDate(entity.createdAt)`
      - `thumbnailUrl`: 그대로 전달
    ```dart
    @freezed
    abstract class EventTicketUiModel with _$EventTicketUiModel {
      const factory EventTicketUiModel({
        required int ticketId,
        required String seatInfo,
        required String priceText,
        required String originalPriceText,
        required String quantityText,
        required String statusText,
        required Color statusColor,
        Color? statusTextColor,
        int? transactionId,
        String? thumbnailUrl,
        required String dateText,
      }) = _EventTicketUiModel;

      factory EventTicketUiModel.fromEntity(EventTicketEntity entity) { ... }
    }
    ```

  **Must NOT do**:
  - UiModel에서 API 호출 또는 비즈니스 로직 포함 금지
  - 색상 하드코딩 금지 — 반드시 `AppColors` 상수 사용
  - 문자열 하드코딩 금지 — 포맷팅 유틸리티 사용
  - 기존 `transaction_history_ui_model.dart` 수정 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 기존 TransactionHistoryUiModel 패턴을 따르는 포맷팅/색상 매핑, 로직 복잡도 낮음
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 6, 7)
  - **Blocks**: Task 8
  - **Blocked By**: Task 2 (Entity 타입 필요)

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/features/profile/presentation/ui_models/transaction_history_ui_model.dart` — UiModel 완전 패턴: @freezed + fromEntity() factory, switch 색상 매핑, DateFormatUtil/NumberFormatUtil 사용, AppColors 참조

  **API/Type References**:
  - `ticket_platform_mobile/lib/core/theme/app_colors.dart` — AppColors.success, AppColors.error, AppColors.badgeWaitingBackground, AppColors.badgeWaitingText 등 색상 상수
  - `ticket_platform_mobile/lib/core/utils/date_format_util.dart` — DateFormatUtil.formatCompactDate() 시그니처
  - `ticket_platform_mobile/lib/core/utils/number_format_util.dart` — NumberFormatUtil.formatPrice() 시그니처

  **WHY Each Reference Matters**:
  - `transaction_history_ui_model.dart`: switch(statusCode) 기반 색상 매핑의 정확한 프로젝트 패턴. stateColor + stateTextColor 이중 색상 구조도 이 파일에서 확인
  - `AppColors`: 하드코딩 색상 사용 시 flutter analyze 경고 및 ui_ux_guide.md RULE 위반

  **Acceptance Criteria**:
  - [ ] `event_group_ui_model.dart` — @freezed + fromEntity(), 상태별 뱃지 텍스트 3개 + 색상 3개
  - [ ] `event_ticket_ui_model.dart` — @freezed + fromEntity(), statusCode 기반 색상 switch, 가격/수량/날짜 포맷팅
  - [ ] 모든 색상이 `AppColors` 상수 사용 (하드코딩 없음)
  - [ ] 모든 날짜/가격 포맷이 `DateFormatUtil`/`NumberFormatUtil` 사용
  - [ ] `flutter analyze` 에러 없음

  **QA Scenarios**:

  ```
  Scenario: UiModel 패턴 검증
    Tool: Bash
    Preconditions: Task 2 Entity 완료
    Steps:
      1. ls ticket_platform_mobile/lib/features/sales_dashboard/presentation/ui_models/event_group_ui_model.dart → 존재
      2. ls ticket_platform_mobile/lib/features/sales_dashboard/presentation/ui_models/event_ticket_ui_model.dart → 존재
      3. grep -c "fromEntity" ticket_platform_mobile/lib/features/sales_dashboard/presentation/ui_models/event_group_ui_model.dart → 1 이상
      4. grep -c "AppColors" ticket_platform_mobile/lib/features/sales_dashboard/presentation/ui_models/event_group_ui_model.dart → 2 이상 (색상 매핑)
      5. grep -c "AppColors" ticket_platform_mobile/lib/features/sales_dashboard/presentation/ui_models/event_ticket_ui_model.dart → 2 이상
      6. grep -rc "Color(0x" ticket_platform_mobile/lib/features/sales_dashboard/presentation/ui_models/ → 0 (하드코딩 색상 없음)
    Expected Result: 두 파일 존재, fromEntity 포함, AppColors 사용, 하드코딩 색상 없음
    Failure Indicators: 파일 미존재, 하드코딩 색상 발견, fromEntity 누락
    Evidence: .sisyphus/evidence/task-5-ui-models.txt
  ```

  **Commit**: NO (Wave 2 그룹 커밋)

- [ ] 6. ViewModel 2개 (페이지네이션, 필터, 상태 관리)

  **What to do**:
  - `sales_dashboard_viewmodel.dart` 생성 — `features/sales_dashboard/presentation/viewmodels/` 아래
    - `@riverpod` annotation 사용 (code generation)
    - `SalesDashboardState` @freezed 상태 클래스 (같은 파일 또는 별도 파일):
      ```dart
      @freezed
      abstract class SalesDashboardState with _$SalesDashboardState {
        const factory SalesDashboardState({
          @Default([]) List<EventGroupUiModel> items,
          @Default(1) int currentPage,
          @Default(false) bool hasMore,
          @Default(false) bool isLoadingMore,
          String? selectedStatus,  // null = 전체
          int? totalCount,
        }) = _SalesDashboardState;
      }
      ```
    - `SalesDashboardViewModel extends _$SalesDashboardViewModel`:
      - `build()` → 초기 데이터 fetch → `SalesDashboardState` 반환
      - `_fetchDashboard({int page, String? status})` — UseCase 호출 → Entity → UiModel 변환
      - `loadMore()` — hasMore 확인 → page+1 fetch → items 누적
      - `refresh()` — state = loading → 재fetch
      - `applyStatusFilter(String? status)` — 필터 변경 → 처음부터 재fetch
    - page/size 페이지네이션 (cursor 아닌 page 기반 — Backend API 사양)
    - 기존 `TransactionHistoryViewModel` 패턴 참조하되, cursor 대신 page 사용
  - `event_ticket_list_viewmodel.dart` 생성 — 같은 디렉토리
    - `@riverpod` annotation 사용
    - family parameter: `int eventId`
    - `EventTicketListState` @freezed 상태 클래스:
      ```dart
      @freezed
      abstract class EventTicketListState with _$EventTicketListState {
        const factory EventTicketListState({
          required String eventTitle,
          @Default([]) List<EventTicketUiModel> items,
          @Default(1) int currentPage,
          @Default(false) bool hasMore,
          @Default(false) bool isLoadingMore,
          int? totalCount,
        }) = _EventTicketListState;
      }
      ```
    - `EventTicketListViewModel extends _$EventTicketListViewModel`:
      - `build(int eventId)` → 초기 fetch → `EventTicketListState` 반환
      - `loadMore()` — 추가 페이지 로드
      - `refresh()` — 재fetch
    - 필터 없음 (공연별 상세에서는 필터 미지원)

  **Must NOT do**:
  - ViewModel에서 직접 Repository/DataSource 접근 금지 (UseCase만 사용)
  - 기존 `TransactionHistoryViewModel` 수정 금지
  - `state = AsyncValue.data(...)` 외의 상태 갱신 패턴 금지
  - cursor 기반 페이지네이션 사용 금지 (Backend는 page/size)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: 페이지네이션 + 필터 + 상태 관리 + AsyncValue 패턴이 결합된 로직 복잡도
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES (Task 5와 동시 가능, Task 7은 이후)
  - **Parallel Group**: Wave 2 (with Task 5)
  - **Blocks**: Tasks 7, 9
  - **Blocked By**: Tasks 2, 4 (UseCase + Repository 타입 필요)

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/features/profile/presentation/viewmodels/transaction_history_viewmodel.dart` — ViewModel 완전 패턴: @riverpod + @freezed State, build()→초기fetch, loadMore()→items 누적, refresh()→AsyncValue.loading, applyFilter()→낙관적 업데이트 후 재fetch, AppLogger 에러 처리
  - `ticket_platform_mobile/lib/features/dispute/presentation/viewmodels/dispute_list_viewmodel.dart` — 리스트 ViewModel 패턴 참조

  **API/Type References**:
  - `features/sales_dashboard/domain/usecases/get_sales_dashboard_usecase.dart` — Task 2 UseCase (call() 시그니처)
  - `features/sales_dashboard/domain/usecases/get_event_tickets_usecase.dart` — Task 2 UseCase
  - `features/sales_dashboard/presentation/ui_models/event_group_ui_model.dart` — Task 5 UiModel (fromEntity)
  - `features/sales_dashboard/presentation/ui_models/event_ticket_ui_model.dart` — Task 5 UiModel

  **WHY Each Reference Matters**:
  - `transaction_history_viewmodel.dart`: AsyncValue 상태 관리, loadMore의 isLoadingMore 플래그 관리, applyFilter의 낙관적 업데이트 → 재fetch 패턴. 이 ViewModel은 cursor 기반이므로 page 기반으로 변환 필요
  - `dispute_list_viewmodel.dart`: 리스트 ViewModel의 간결한 패턴 참조

  **Acceptance Criteria**:
  - [ ] `sales_dashboard_viewmodel.dart` — @riverpod, SalesDashboardState @freezed, build()/loadMore()/refresh()/applyStatusFilter() 4개 메서드
  - [ ] `event_ticket_list_viewmodel.dart` — @riverpod, family(eventId), EventTicketListState @freezed, build()/loadMore()/refresh() 3개 메서드
  - [ ] UseCase만 통해 데이터 접근 (Repository/DataSource 직접 접근 없음)
  - [ ] page 기반 페이지네이션 (cursor 미사용)
  - [ ] Entity → UiModel 변환은 fromEntity() 사용
  - [ ] `flutter analyze` 에러 없음

  **QA Scenarios**:

  ```
  Scenario: ViewModel 생성 및 패턴 검증
    Tool: Bash
    Preconditions: Tasks 2, 4, 5 완료
    Steps:
      1. ls ticket_platform_mobile/lib/features/sales_dashboard/presentation/viewmodels/sales_dashboard_viewmodel.dart → 존재
      2. ls ticket_platform_mobile/lib/features/sales_dashboard/presentation/viewmodels/event_ticket_list_viewmodel.dart → 존재
      3. grep -c "@riverpod" ticket_platform_mobile/lib/features/sales_dashboard/presentation/viewmodels/sales_dashboard_viewmodel.dart → 1
      4. grep -c "@freezed" ticket_platform_mobile/lib/features/sales_dashboard/presentation/viewmodels/sales_dashboard_viewmodel.dart → 1
      5. grep -c "loadMore" ticket_platform_mobile/lib/features/sales_dashboard/presentation/viewmodels/sales_dashboard_viewmodel.dart → 1 이상
      6. grep -c "applyStatusFilter" ticket_platform_mobile/lib/features/sales_dashboard/presentation/viewmodels/sales_dashboard_viewmodel.dart → 1 이상
      7. grep -c "@riverpod" ticket_platform_mobile/lib/features/sales_dashboard/presentation/viewmodels/event_ticket_list_viewmodel.dart → 1
    Expected Result: 두 파일 존재, @riverpod + @freezed 적용, 핵심 메서드 포함
    Failure Indicators: 파일 미존재, @riverpod 누락, cursor 패턴 사용
    Evidence: .sisyphus/evidence/task-6-viewmodels.txt

  Scenario: ViewModel이 UseCase만 사용하는지 확인 (직접 Repository 접근 금지)
    Tool: Bash
    Preconditions: Task 6 완료
    Steps:
      1. grep -c "Repository" ticket_platform_mobile/lib/features/sales_dashboard/presentation/viewmodels/sales_dashboard_viewmodel.dart → 0
      2. grep -c "DataSource" ticket_platform_mobile/lib/features/sales_dashboard/presentation/viewmodels/sales_dashboard_viewmodel.dart → 0
      3. grep -c "Usecase\|UseCase" ticket_platform_mobile/lib/features/sales_dashboard/presentation/viewmodels/sales_dashboard_viewmodel.dart → 1 이상
    Expected Result: Repository/DataSource 참조 0, UseCase 참조 1 이상
    Failure Indicators: Repository 또는 DataSource 직접 참조 시 아키텍처 위반
    Evidence: .sisyphus/evidence/task-6-usecase-only.txt
  ```

  **Commit**: NO (Wave 2 그룹 커밋)

- [ ] 7. DI Provider + 코드 생성

  **What to do**:
  - `sales_dashboard_providers_di.dart` 생성 — `features/sales_dashboard/presentation/providers/` 아래
    - 모든 Provider에 `@riverpod` annotation 사용
    - DI 체인 구성 (dispute_providers_di.dart 패턴 따를 것):
    ```dart
    @riverpod
    SalesDashboardRemoteDataSource salesDashboardRemoteDataSource(Ref ref) {
      return SalesDashboardRemoteDataSourceImpl(ref.watch(dioProvider));
    }

    @riverpod
    SalesDashboardRepository salesDashboardRepository(Ref ref) {
      return SalesDashboardRepositoryImpl(
        ref.watch(salesDashboardRemoteDataSourceProvider),
      );
    }

    @riverpod
    GetSalesDashboardUsecase getSalesDashboardUsecase(Ref ref) {
      return GetSalesDashboardUsecase(
        ref.watch(salesDashboardRepositoryProvider),
      );
    }

    @riverpod
    GetEventTicketsUsecase getEventTicketsUsecase(Ref ref) {
      return GetEventTicketsUsecase(
        ref.watch(salesDashboardRepositoryProvider),
      );
    }
    ```
  - `part 'sales_dashboard_providers_di.g.dart'` 선언 필수
  - `dart run build_runner build --delete-conflicting-outputs` 실행하여 모든 코드 생성 파일 재생성:
    - `*.freezed.dart` (DTO, Entity, UiModel, State)
    - `*.g.dart` (DTO fromJson, Provider)
  - 코드 생성 성공 확인 + `flutter analyze` 통과 확인

  **Must NOT do**:
  - 수동 Provider 작성 금지 (@riverpod code generation 필수 — CLAUDE.md 규칙)
  - 기존 `profile_providers_di.dart` 수정 금지
  - `ref.read` 대신 `ref.watch` 사용 (DI Provider에서는 watch가 표준)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: DI 배선 + build_runner 실행, dispute_providers_di 패턴 복제
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 2 (after Tasks 1, 2, 4, 6 모두 완료 후)
  - **Blocks**: Task 9
  - **Blocked By**: Tasks 1, 2, 4, 6 (모든 @freezed/@riverpod 파일이 존재해야 build_runner 성공)

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/features/dispute/presentation/providers/dispute_providers_di.dart` — DI Provider 완전 패턴: @riverpod, part 선언, DataSource→Repository→UseCase 체인, ref.watch() 사용
  - `ticket_platform_mobile/lib/features/profile/presentation/providers/profile_providers_di.dart` — DI Provider 대안 패턴 참조 (ref.read vs ref.watch 차이점 확인)

  **API/Type References**:
  - `ticket_platform_mobile/lib/core/network/dio_provider.dart` — `dioProvider` (Dio 인스턴스 Provider)

  **WHY Each Reference Matters**:
  - `dispute_providers_di.dart`: DataSource→Repository→UseCase 3단 DI 체인의 정확한 패턴. `ref.watch(dioProvider)` 사용과 각 Provider의 반환 타입 확인
  - `dio_provider.dart`: Dio 인스턴스의 Provider 이름 확인 (import 경로)

  **Acceptance Criteria**:
  - [ ] `sales_dashboard_providers_di.dart` — @riverpod 4개 Provider (DataSource, Repository, UseCase×2)
  - [ ] `part 'sales_dashboard_providers_di.g.dart'` 선언 존재
  - [ ] `dart run build_runner build --delete-conflicting-outputs` → 성공 (exit code 0)
  - [ ] 생성된 파일 존재: `*.freezed.dart` (최소 6개), `*.g.dart` (최소 4개)
  - [ ] `flutter analyze` → No issues found

  **QA Scenarios**:

  ```
  Scenario: DI Provider 생성 및 코드 생성 성공
    Tool: Bash
    Preconditions: Tasks 1, 2, 4, 6 모두 완료
    Steps:
      1. ls ticket_platform_mobile/lib/features/sales_dashboard/presentation/providers/sales_dashboard_providers_di.dart → 존재
      2. grep -c "@riverpod" ticket_platform_mobile/lib/features/sales_dashboard/presentation/providers/sales_dashboard_providers_di.dart → 4
      3. cd ticket_platform_mobile && dart run build_runner build --delete-conflicting-outputs → exit code 0
      4. ls ticket_platform_mobile/lib/features/sales_dashboard/data/dto/response/sales_dashboard_resp_dto.freezed.dart → 존재
      5. ls ticket_platform_mobile/lib/features/sales_dashboard/data/dto/response/sales_dashboard_resp_dto.g.dart → 존재
      6. ls ticket_platform_mobile/lib/features/sales_dashboard/presentation/providers/sales_dashboard_providers_di.g.dart → 존재
      7. cd ticket_platform_mobile && flutter analyze → No issues found
    Expected Result: DI 파일 존재, @riverpod 4개, build_runner 성공, 생성 파일 존재, 분석 통과
    Failure Indicators: build_runner 실패 (part 선언 누락 또는 import 오류), flutter analyze 에러
    Evidence: .sisyphus/evidence/task-7-di-codegen.txt

  Scenario: build_runner 실패 시 에러 진단
    Tool: Bash
    Preconditions: build_runner 실패 발생 시
    Steps:
      1. cd ticket_platform_mobile && dart run build_runner build --delete-conflicting-outputs 2>&1 | tail -30
      2. 에러 메시지에서 누락된 part 선언 또는 import 경로 확인
      3. 해당 파일 수정 후 재실행
    Expected Result: 에러 원인 파악 및 수정
    Failure Indicators: 반복 실패 — @freezed/@riverpod annotation 누락 가능성
    Evidence: .sisyphus/evidence/task-7-codegen-error.txt
  ```

  **Commit**: YES (Wave 2)
  - Message: `feat(sales-dashboard): add viewmodels, UI models, DI providers and generate code`
  - Files: `presentation/ui_models/*.dart`, `presentation/viewmodels/*.dart`, `presentation/providers/*.dart`, 생성된 `*.freezed.dart`, `*.g.dart`
  - Pre-commit: `flutter analyze`

- [ ] 8. EventGroupCard + EventTicketItem + SalesStatusFilterBar 위젯

  **What to do**:
  - `event_group_card.dart` 생성 — `features/sales_dashboard/presentation/widgets/` 아래
    - 공연 그룹 카드 위젯 (대시보드 리스트의 각 항목)
    - 레이아웃 구성:
      - 좌측: 포스터 이미지 (ClipRRect 라운드, null이면 Icons.event 기본 아이콘)
      - 우측: 공연명 (1줄, ellipsis) + 일시 + 장소
      - 하단: 상태별 수량 뱃지 (Row — 판매중/완료/정산중, 0이면 미표시)
    - 뱃지 스타일: Container + 배경색 + 텍스트 (AppColors 사용, 라운드 패딩)
    - onTap 콜백으로 eventId 전달 (카드 전체 탭 영역)
    - `EventGroupUiModel`을 입력으로 받음
    - UI/UX Rule 준수:
      - RULE-02: 정보 계층 명확 (공연명 > 일시 > 장소 > 뱃지)
      - RULE-04: 여백 > 구분선 (카드 간 패딩으로 구분)
      - RULE-09: 과도한 시각효과 금지
    ```dart
    class EventGroupCard extends StatelessWidget {
      final EventGroupUiModel model;
      final VoidCallback onTap;
      const EventGroupCard({required this.model, required this.onTap, super.key});
      // ...
    }
    ```
  - `event_ticket_item.dart` 생성 — 같은 디렉토리
    - 공연별 개별 티켓 아이템 위젯 (상세 화면의 리스트 항목)
    - 레이아웃 구성:
      - 좌측: 썸네일 이미지 (null이면 Icons.confirmation_number 기본 아이콘)
      - 우측: 좌석정보 + 가격(할인표시) + 수량 + 등록일
      - 우상단: 상태 뱃지 (statusColor 배경 + statusText)
    - `EventTicketUiModel`을 입력으로 받음
    ```dart
    class EventTicketItem extends StatelessWidget {
      final EventTicketUiModel model;
      const EventTicketItem({required this.model, super.key});
      // ...
    }
    ```
  - `sales_status_filter_bar.dart` 생성 — 같은 디렉토리
    - 상태 필터 바 위젯 (대시보드 상단 고정)
    - 필터 옵션: 전체 / 판매중 / 완료 / 정산 중
    - 구현 방식: 가로 스크롤 Row (ChoiceChip 또는 커스텀 필터 버튼)
    - 선택된 필터 강조 표시 (AppColors.primary 배경)
    - 콜백: `onFilterChanged(String? status)` — null = 전체
    ```dart
    class SalesStatusFilterBar extends StatelessWidget {
      final String? selectedStatus;
      final ValueChanged<String?> onFilterChanged;
      const SalesStatusFilterBar({
        this.selectedStatus,
        required this.onFilterChanged,
        super.key,
      });
      // ...
    }
    ```

  **Must NOT do**:
  - 색상/폰트 하드코딩 금지 — AppColors, AppTextStyles 사용
  - 여백 하드코딩 금지 — AppSpacing 사용
  - 위젯 50줄 초과 시 _buildXxx() 헬퍼 메서드로 분리 (CLAUDE.md 규칙)
  - 기존 `transaction_history_item.dart` 수정 금지
  - 복잡한 애니메이션/그라데이션/그림자 효과 금지 (RULE-09)

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
    - Reason: UI 위젯 3개 구현, 레이아웃 설계 + 스타일링 + 반응형 고려 필요
  - **Skills**: []
    - Skills Evaluated but Omitted:
      - `playwright`: 위젯 단위 검증이므로 불필요 (통합 QA는 F3에서 수행)

  **Parallelization**:
  - **Can Run In Parallel**: YES (Task 9와 동시 불가, Task 8 단독 또는 Task 9 전에)
  - **Parallel Group**: Wave 3 (Task 8 → Task 9 → Task 10)
  - **Blocks**: Task 9
  - **Blocked By**: Task 5 (UiModel 타입 필요)

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/features/profile/presentation/widgets/transaction_history_item.dart` — 리스트 아이템 위젯 패턴: 썸네일+정보 레이아웃, 상태 뱃지, onTap 콜백 구조
  - `ticket_platform_mobile/lib/features/dispute/presentation/widgets/dispute_item_widget.dart` — 카드 위젯 패턴 참조
  - `ticket_platform_mobile/lib/features/dispute/presentation/widgets/dispute_status_badge.dart` — 상태 뱃지 위젯 패턴

  **API/Type References**:
  - `features/sales_dashboard/presentation/ui_models/event_group_ui_model.dart` — Task 5 UiModel (위젯 입력 타입)
  - `features/sales_dashboard/presentation/ui_models/event_ticket_ui_model.dart` — Task 5 UiModel
  - `ticket_platform_mobile/lib/core/theme/app_colors.dart` — 색상 상수
  - `ticket_platform_mobile/lib/core/theme/app_text_styles.dart` — 텍스트 스타일 상수
  - `ticket_platform_mobile/lib/core/theme/app_spacing.dart` — 여백 상수

  **External References**:
  - `ticket_platform_mobile/.agent/rules/ui_ux_guide.md` — 10개 Rule Set (RULE-01, 02, 04, 09 중점)

  **WHY Each Reference Matters**:
  - `transaction_history_item.dart`: 리스트 아이템의 좌우 레이아웃 (이미지+정보) 패턴. 상태 뱃지 표시 방식
  - `dispute_status_badge.dart`: 상태별 색상 뱃지의 Container + padding + borderRadius 구현 패턴
  - `ui_ux_guide.md`: 위젯 스타일링 판단 기준 — 여백 > 구분선, 시각적 밀도 낮추기, 과도한 장식 금지

  **Acceptance Criteria**:
  - [ ] `event_group_card.dart` — 포스터+공연명+일시+장소+상태뱃지 레이아웃, onTap 콜백, null 포스터 fallback
  - [ ] `event_ticket_item.dart` — 썸네일+좌석+가격+수량+상태뱃지 레이아웃, null 썸네일 fallback
  - [ ] `sales_status_filter_bar.dart` — 4개 필터 옵션, 선택 상태 표시, onFilterChanged 콜백
  - [ ] 모든 색상 AppColors, 텍스트 AppTextStyles, 여백 AppSpacing 사용
  - [ ] `flutter analyze` 에러 없음

  **QA Scenarios**:

  ```
  Scenario: 위젯 파일 생성 및 패턴 검증
    Tool: Bash
    Preconditions: Task 5 UiModel 완료
    Steps:
      1. ls ticket_platform_mobile/lib/features/sales_dashboard/presentation/widgets/event_group_card.dart → 존재
      2. ls ticket_platform_mobile/lib/features/sales_dashboard/presentation/widgets/event_ticket_item.dart → 존재
      3. ls ticket_platform_mobile/lib/features/sales_dashboard/presentation/widgets/sales_status_filter_bar.dart → 존재
      4. grep -c "AppColors" ticket_platform_mobile/lib/features/sales_dashboard/presentation/widgets/event_group_card.dart → 1 이상
      5. grep -c "onTap" ticket_platform_mobile/lib/features/sales_dashboard/presentation/widgets/event_group_card.dart → 1 이상
      6. grep -c "onFilterChanged" ticket_platform_mobile/lib/features/sales_dashboard/presentation/widgets/sales_status_filter_bar.dart → 1 이상
      7. grep -rc "Color(0x" ticket_platform_mobile/lib/features/sales_dashboard/presentation/widgets/ → 0 (하드코딩 없음)
      8. cd ticket_platform_mobile && flutter analyze → No issues found
    Expected Result: 3개 위젯 파일 존재, AppColors 사용, 콜백 존재, 하드코딩 없음
    Failure Indicators: 파일 미존재, 하드코딩 색상, 콜백 누락
    Evidence: .sisyphus/evidence/task-8-widgets.txt
  ```

  **Commit**: NO (Wave 3 그룹 커밋)

- [ ] 9. SalesDashboardView + EventTicketListView 화면

  **What to do**:
  - `sales_dashboard_view.dart` 생성 — `features/sales_dashboard/presentation/views/` 아래
    - 판매 관리 대시보드 메인 화면
    - 레이아웃 구성:
      - AppBar: "판매 관리" 타이틀
      - 상단 고정: `SalesStatusFilterBar` (필터 바)
      - 본문: `ListView.builder` — EventGroupCard 리스트
      - 무한 스크롤: ScrollController.addListener → loadMore() 호출
      - 빈 상태: 등록 티켓 없을 때 안내 화면 ("등록된 판매 티켓이 없습니다")
      - 로딩 상태: CircularProgressIndicator (초기 로드)
      - 추가 로딩: 리스트 하단 로딩 인디케이터 (loadMore 중)
      - 에러 상태: 에러 메시지 + 재시도 버튼
    - ViewModel 연동:
      - `ref.watch(salesDashboardViewModelProvider)` — AsyncValue<SalesDashboardState>
      - `ref.read(salesDashboardViewModelProvider.notifier).applyStatusFilter(status)`
      - `ref.read(salesDashboardViewModelProvider.notifier).loadMore()`
    - 카드 탭 → `context.push` 로 EventTicketListView 이동 (eventId 전달)
    - ConsumerStatefulWidget 사용 (ScrollController lifecycle 관리)
    ```dart
    class SalesDashboardView extends ConsumerStatefulWidget {
      const SalesDashboardView({super.key});
      @override
      ConsumerState<SalesDashboardView> createState() => _SalesDashboardViewState();
    }
    ```
  - `event_ticket_list_view.dart` 생성 — 같은 디렉토리
    - 공연별 티켓 상세 목록 화면 (push로 진입)
    - 레이아웃 구성:
      - AppBar: 공연명 타이틀 (eventTitle)
      - 본문: `ListView.builder` — EventTicketItem 리스트
      - 무한 스크롤: ScrollController → loadMore()
      - 빈 상태/로딩/에러 동일 패턴
    - ViewModel 연동:
      - `ref.watch(eventTicketListViewModelProvider(eventId))` — family provider
    - ConsumerStatefulWidget 사용
    ```dart
    class EventTicketListView extends ConsumerStatefulWidget {
      final int eventId;
      const EventTicketListView({required this.eventId, super.key});
      @override
      ConsumerState<EventTicketListView> createState() => _EventTicketListViewState();
    }
    ```
  - RULE-01 준수: 각 화면은 하나의 핵심 목적만 (대시보드 = 공연별 현황 조회, 상세 = 개별 티켓 확인)
  - RULE-02 준수: 3초 안에 화면 목적 인지 가능한 정보 계층

  **Must NOT do**:
  - 기존 `TransactionHistoryView` 수정 금지 (이 화면은 구매 탭 유지)
  - 화면 50줄 초과 시 _buildXxx() 분리 필수
  - 하드코딩 색상/여백/텍스트 금지
  - 한 화면에 2개 이상 목적 금지 (RULE-01)
  - LazyLoad 없는 전체 로딩 금지 (반드시 페이지네이션)

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
    - Reason: 화면 2개 구현, 무한스크롤 + 필터 + 상태관리 + UI 레이아웃 통합
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (Task 8 위젯 완료 후)
  - **Parallel Group**: Wave 3 (after Task 8)
  - **Blocks**: Task 10
  - **Blocked By**: Tasks 6, 7, 8 (ViewModel + Provider + Widget 필요)

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/features/profile/presentation/views/transaction_history_view.dart` — 리스트 화면 패턴: TabBarView + ListView.builder + ScrollController + 무한스크롤 + 필터 + AsyncValue.when() 패턴
  - `ticket_platform_mobile/lib/features/dispute/presentation/views/dispute_list_view.dart` — 리스트 화면 + 빈 상태 + 에러 상태 + RefreshIndicator 패턴
  - `ticket_platform_mobile/lib/features/dispute/presentation/views/dispute_detail_view.dart` — 상세 화면 push 패턴

  **API/Type References**:
  - `features/sales_dashboard/presentation/viewmodels/sales_dashboard_viewmodel.dart` — Task 6 ViewModel provider 이름
  - `features/sales_dashboard/presentation/viewmodels/event_ticket_list_viewmodel.dart` — Task 6 family ViewModel
  - `features/sales_dashboard/presentation/widgets/*.dart` — Task 8 위젯들

  **WHY Each Reference Matters**:
  - `transaction_history_view.dart`: ScrollController 기반 무한스크롤의 정확한 구현 패턴 (listener 등록, dispose, 하단 감지 offset). AsyncValue.when() 으로 loading/data/error 상태 분기
  - `dispute_list_view.dart`: 빈 상태 화면 구현 패턴 (아이콘 + 설명 텍스트 중앙 배치)
  - `dispute_detail_view.dart`: push 방식 상세 화면 진입 패턴 (GoRouter context.push)

  **Acceptance Criteria**:
  - [ ] `sales_dashboard_view.dart` — ConsumerStatefulWidget, AppBar, SalesStatusFilterBar, ListView.builder, 무한스크롤, 빈 상태
  - [ ] `event_ticket_list_view.dart` — ConsumerStatefulWidget, family(eventId), AppBar(공연명), ListView.builder, 무한스크롤
  - [ ] AsyncValue.when() 또는 switch 패턴으로 loading/data/error 상태 분기
  - [ ] 빈 상태 화면 (등록 티켓 없을 때)
  - [ ] ScrollController dispose 처리
  - [ ] `flutter analyze` 에러 없음

  **QA Scenarios**:

  ```
  Scenario: View 파일 생성 및 핵심 패턴 검증
    Tool: Bash
    Preconditions: Tasks 6, 7, 8 완료
    Steps:
      1. ls ticket_platform_mobile/lib/features/sales_dashboard/presentation/views/sales_dashboard_view.dart → 존재
      2. ls ticket_platform_mobile/lib/features/sales_dashboard/presentation/views/event_ticket_list_view.dart → 존재
      3. grep -c "ConsumerStatefulWidget" ticket_platform_mobile/lib/features/sales_dashboard/presentation/views/sales_dashboard_view.dart → 1
      4. grep -c "ScrollController" ticket_platform_mobile/lib/features/sales_dashboard/presentation/views/sales_dashboard_view.dart → 1 이상
      5. grep -c "loadMore" ticket_platform_mobile/lib/features/sales_dashboard/presentation/views/sales_dashboard_view.dart → 1 이상
      6. grep -c "SalesStatusFilterBar" ticket_platform_mobile/lib/features/sales_dashboard/presentation/views/sales_dashboard_view.dart → 1 이상
      7. grep -c "ConsumerStatefulWidget" ticket_platform_mobile/lib/features/sales_dashboard/presentation/views/event_ticket_list_view.dart → 1
      8. grep -c "eventId" ticket_platform_mobile/lib/features/sales_dashboard/presentation/views/event_ticket_list_view.dart → 2 이상
    Expected Result: 두 화면 존재, ConsumerStatefulWidget, ScrollController, 무한스크롤 패턴
    Failure Indicators: 파일 미존재, StatelessWidget 사용, ScrollController 누락
    Evidence: .sisyphus/evidence/task-9-views.txt

  Scenario: 빈 상태 화면 존재 확인
    Tool: Bash
    Preconditions: Task 9 완료
    Steps:
      1. grep -c "등록된\|없습니다\|empty" ticket_platform_mobile/lib/features/sales_dashboard/presentation/views/sales_dashboard_view.dart → 1 이상
    Expected Result: 빈 상태 안내 텍스트 존재
    Failure Indicators: 빈 상태 처리 누락
    Evidence: .sisyphus/evidence/task-9-empty-state.txt
  ```

  **Commit**: NO (Wave 3 그룹 커밋)

- [ ] 10. 라우팅 통합 + 빌드 검증

  **What to do**:
  - GoRouter 설정 파일에 판매 대시보드 라우트 2개 추가:
    - `AppRouterPath.salesDashboard.path` → `SalesDashboardView()`
    - `AppRouterPath.eventTicketList.path` → `EventTicketListView(eventId: ...)` (pathParam 또는 queryParam으로 eventId 전달)
    - 기존 GoRouter 설정 파일 위치 확인 후 패턴에 맞게 추가
  - 프로필 화면에서 '판매 내역' 메뉴 탭 시 라우트 변경:
    - `profile_view.dart`에서 기존 `transactionHistory` 라우트 대신 `salesDashboard` 라우트로 변경
    - 또는 `TransactionHistoryView`의 판매 탭에서 대시보드 화면으로 리다이렉트
    - ⚠️ 구매 탭은 절대 변경하지 않음
  - 전체 빌드 검증:
    - `dart run build_runner build --delete-conflicting-outputs` → 성공
    - `flutter analyze` → No issues found
    - `flutter build apk --debug` → BUILD SUCCESSFUL
  - 라우트 검증:
    - GoRouter에 salesDashboard, eventTicketList 라우트 등록 확인
    - 프로필 화면에서 판매 내역 탭 시 대시보드 화면 이동 확인

  **Must NOT do**:
  - 기존 구매 내역 탭 라우트/로직 변경 금지
  - 기존 `transactionHistory` 라우트 삭제 금지 (구매 탭에서 사용 중일 수 있음)
  - GoRouter 설정의 기존 라우트 순서/구조 변경 금지
  - 프로필 화면의 구매 관련 UI 변경 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 라우트 2개 추가 + 메뉴 라우트 변경 + 빌드 검증, 소규모 통합 작업
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (모든 이전 Task 완료 필요)
  - **Parallel Group**: Wave 3 (after Task 9)
  - **Blocks**: F1-F4
  - **Blocked By**: Tasks 3, 9 (Route 상수 + View 필요)

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/core/router/` — GoRouter 설정 파일 (기존 라우트 등록 패턴 확인)
  - `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart` — '판매 내역' 메뉴 탭 핸들러 위치

  **API/Type References**:
  - `ticket_platform_mobile/lib/core/router/app_router_path.dart` — Task 3에서 추가한 salesDashboard, eventTicketList 라우트
  - `features/sales_dashboard/presentation/views/sales_dashboard_view.dart` — Task 9 대시보드 화면
  - `features/sales_dashboard/presentation/views/event_ticket_list_view.dart` — Task 9 상세 화면

  **WHY Each Reference Matters**:
  - GoRouter 설정 파일: 기존 라우트 등록 방식 (GoRoute, path, builder/pageBuilder 패턴) 확인. eventId pathParam 전달 방식 확인
  - `profile_view.dart`: '판매 내역' 메뉴의 onTap 핸들러 위치. 기존 라우트 이름 확인하여 정확한 교체 지점 파악

  **Acceptance Criteria**:
  - [ ] GoRouter에 `salesDashboard`, `eventTicketList` 라우트 등록됨
  - [ ] 프로필 → 판매 내역 메뉴 탭 시 `SalesDashboardView`로 이동
  - [ ] eventId 파라미터가 EventTicketListView에 정상 전달
  - [ ] 기존 구매 내역 관련 라우트/로직 무변경
  - [ ] `dart run build_runner build --delete-conflicting-outputs` → 성공
  - [ ] `flutter analyze` → No issues found
  - [ ] `flutter build apk --debug` → BUILD SUCCESSFUL

  **QA Scenarios**:

  ```
  Scenario: 라우팅 통합 및 빌드 검증
    Tool: Bash
    Preconditions: Tasks 1~9 모두 완료
    Steps:
      1. grep -c "salesDashboard" ticket_platform_mobile/lib/core/router/*.dart → 2 이상 (path 정의 + 라우트 등록)
      2. grep -c "eventTicketList" ticket_platform_mobile/lib/core/router/*.dart → 2 이상
      3. cd ticket_platform_mobile && dart run build_runner build --delete-conflicting-outputs → exit code 0
      4. cd ticket_platform_mobile && flutter analyze → No issues found
      5. cd ticket_platform_mobile && flutter build apk --debug → BUILD SUCCESSFUL
    Expected Result: 라우트 등록 확인, 코드 생성 성공, 분석 통과, APK 빌드 성공
    Failure Indicators: 라우트 미등록, 빌드 실패, analyze 에러
    Evidence: .sisyphus/evidence/task-10-routing-build.txt

  Scenario: 구매 내역 라우트 무변경 확인
    Tool: Bash
    Preconditions: Task 10 완료
    Steps:
      1. git diff ticket_platform_mobile/lib/features/profile/presentation/views/ | grep -c "purchases\|purchase" → 0 (구매 관련 변경 없음)
      2. grep -c "transactionHistory" ticket_platform_mobile/lib/core/router/app_router_path.dart → 1 (기존 유지)
    Expected Result: 구매 관련 코드 무변경, transactionHistory 라우트 유지
    Failure Indicators: 구매 탭 코드 변경 감지
    Evidence: .sisyphus/evidence/task-10-no-purchase-change.txt
  ```

  **Commit**: YES (Wave 3)
  - Message: `feat(sales-dashboard): add dashboard and ticket list views with routing`
  - Files: `presentation/widgets/*.dart`, `presentation/views/*.dart`, GoRouter 설정 파일, `profile_view.dart` 라우트 변경
  - Pre-commit: `flutter build apk --debug`

---

## Final Verification Wave

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Read the plan end-to-end. For each "Must Have": verify implementation exists. For each "Must NOT Have": search codebase for forbidden changes. Check evidence files in .sisyphus/evidence/.
  Output: `Must Have [N/N] | Must NOT Have [N/N] | Tasks [N/N] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  Run `flutter analyze`. Review all new files for: unused imports, TODO 미해결, 하드코딩된 색상/문자열. ui_ux_guide.md Rule 위반 확인. 기존 dispute/profile 피처 패턴과 일관성 검증.
  Output: `Analyze [PASS/FAIL] | Files [N clean/N issues] | VERDICT`

- [ ] F3. **UI QA** — `unspecified-high` + `playwright`
  Android 에뮬레이터에서 프로필 → 판매 내역 → 대시보드 화면 진입. 공연 그룹 카드 표시 확인. 상태 필터 동작 확인. 카드 탭 → 공연별 티켓 목록 화면 진입 확인. 빈 상태 화면 확인. 스크린샷 캡처.
  Output: `Scenarios [N/N pass] | Screenshots [N captured] | VERDICT`

- [ ] F4. **Scope Fidelity Check** — `deep`
  For each task: read "What to do", read actual diff. Verify 1:1. Check "Must NOT do": 구매 내역 탭 무변경, 기존 TransactionHistoryView 구매 로직 무변경. Detect unaccounted changes.
  Output: `Tasks [N/N compliant] | Contamination [CLEAN/N issues] | VERDICT`

---

## Commit Strategy

- **Wave 1**: `feat(sales-dashboard): add DTOs, entities, repository and data layer` — DTO, Entity, DataSource, Repository 파일들
- **Wave 2**: `feat(sales-dashboard): add viewmodels, UI models and DI providers` — ViewModel, UiModel, Provider 파일들
- **Wave 3**: `feat(sales-dashboard): add dashboard and ticket list views with routing` — Widget, View, 라우팅 파일들

---

## Success Criteria

### Verification Commands
```bash
cd ticket_platform_mobile && dart run build_runner build --delete-conflicting-outputs  # Expected: success
cd ticket_platform_mobile && flutter analyze  # Expected: No issues found
cd ticket_platform_mobile && flutter build apk --debug  # Expected: BUILD SUCCESSFUL
```

### Final Checklist
- [ ] 프로필 → '판매 내역' 탭 → 판매 대시보드 화면 표시
- [ ] 공연 그룹 카드에 포스터/공연명/일시/장소/상태별 수량 표시
- [ ] 상태 필터 동작 (전체/판매중/완료/정산중)
- [ ] 공연 카드 탭 → 개별 티켓 목록 화면 push
- [ ] 빈 상태 화면 (등록 티켓 없을 때)
- [ ] 기존 구매 내역 탭 무변경
- [ ] `flutter analyze` 이슈 없음
- [ ] `flutter build apk --debug` 성공
