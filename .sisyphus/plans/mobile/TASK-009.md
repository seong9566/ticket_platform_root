# TASK-009 정산 스케줄러 — Mobile 계획

## TL;DR

> **요약**: 판매자가 정산 내역을 확인할 수 있는 전용 화면을 구현한다.
> 프로필 메뉴에 '정산 내역' 항목을 추가하고, 정산 리스트(예정일, 금액, 상태) + 상세 화면을 제공한다.
>
> **산출물**:
> - `ticket_platform_mobile/lib/features/settlement/` — 정산 피처 디렉토리 (data + domain + presentation 레이어)
> - `ticket_platform_mobile/lib/features/settlement/presentation/views/settlement_history_view.dart` — 정산 내역 리스트 화면 (상태별 필터, 페이지네이션)
> - `ticket_platform_mobile/lib/features/settlement/presentation/views/settlement_detail_view.dart` — 정산 상세 화면 (계좌 정보 마스킹 표시)
> - `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart` — '정산 내역' 메뉴 타일 추가
> - `ticket_platform_mobile/lib/features/profile/presentation/widgets/transaction_history_item.dart` — 거래내역 아이템에 정산 상태 뱃지 추가
>
> **예상 공수**: Medium
> **병렬 실행**: YES — 3 Wave
> **Android-First**: 모든 QA는 Android 에뮬레이터 기준

---

## Context

### ⚠️ 작업 기준 디렉토리
모든 파일 경로는 `ticket_platform_mobile/lib/` 아래를 기준으로 한다. (예: `features/...` → `ticket_platform_mobile/lib/features/...`)

### 기획 확정 사항
| 항목 | 결정 |
|------|------|
| UI 범위 | 중간: 프로필에서 '정산 내역' 메뉴 → 정산 리스트 화면 (예정일, 금액, 상태) |
| 정산 알림 | FCM만 (채팅방 메시지 없음) — 알림 탭 시 정산 상세 화면으로 이동 |
| 정산 상태 | pending(대기), processing(처리중), completed(완료), failed(실패) |
| 코드 생성 | freezed/json_serializable 재생성 필요 (`*.freezed.dart`, `*.g.dart`) |

### 백엔드 API (TASK-009 Backend에서 제공)
- `GET /api/settlements?page=1&size=20&status=completed` — 판매자 정산 목록
- `GET /api/settlements/{id}` — 정산 상세 (계좌번호 마스킹 처리됨)

### 수정 대상 기존 파일
- `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart` — '정산 내역' 메뉴 타일 추가
- `ticket_platform_mobile/lib/features/profile/presentation/widgets/transaction_history_item.dart` — 정산 상태 뱃지 추가 (선택적)
- `ticket_platform_mobile/lib/core/router/app_router_path.dart` — 정산 라우트 추가
- `ticket_platform_mobile/lib/core/network/api_endpoint.dart` — 정산 API 엔드포인트 상수 추가

### 신규 생성 디렉토리
```
ticket_platform_mobile/lib/features/settlement/
├── data/
│   ├── dto/
│   │   └── response/
│   │       ├── settlement_resp_dto.dart
│   │       ├── settlement_list_resp_dto.dart
│   │       └── settlement_detail_resp_dto.dart
│   ├── datasources/settlement_remote_data_source.dart   ← 인터페이스 + 구현체 동일 파일 (dispute 패턴)
│   └── repositories/settlement_repository_impl.dart     ← domain SettlementRepository 구현체
├── domain/
│   ├── entities/settlement_entity.dart
│   └── repositories/settlement_repository.dart          ← abstract 인터페이스
└── presentation/
    ├── providers/settlement_providers_di.dart            ← @riverpod DI 등록 (dispute_providers_di.dart 패턴)
    ├── viewmodels/
    │   ├── settlement_history_viewmodel.dart
    │   └── settlement_detail_viewmodel.dart
    ├── views/
    │   ├── settlement_history_view.dart
    │   └── settlement_detail_view.dart
    └── widgets/
        ├── settlement_status_badge.dart
        └── settlement_history_item.dart
```

### 참조 패턴
- `ticket_platform_mobile/lib/features/dispute/` — 피처 디렉토리 구조 완전 참조
- `ticket_platform_mobile/lib/features/profile/presentation/views/transaction_history_view.dart` — 리스트 화면 + TabBar + 필터 패턴
- `ticket_platform_mobile/lib/features/profile/presentation/widgets/transaction_history_item.dart` — 리스트 아이템 패턴
- `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart` — 메뉴 타일 추가 위치

---

## Work Objectives

### 핵심 목표
판매자가 프로필 화면에서 '정산 내역' 메뉴를 탭하여 정산 리스트를 확인하고,
각 정산 건의 상세 정보(금액, 상태, 계좌, 예정일)를 조회할 수 있어야 한다.

### Must Have
- 프로필 화면 '거래' 섹션에 '정산 내역' 메뉴 타일 추가
- 정산 내역 리스트 화면: 상태별 필터(전체/대기/완료/실패), 무한 스크롤 페이지네이션
- 정산 아이템 UI: 이벤트명, 정산 금액(NetAmount), 상태 뱃지, 예정일/완료일
- 정산 상세 화면: 거래 정보, 정산 금액 상세(총액/수수료/순액), 계좌 정보(마스킹), 상태 타임라인
- 상태 뱃지 색상: pending=노란색, processing=파란색, completed=녹색, failed=빨간색
- freezed/json_serializable 재생성 (`flutter pub run build_runner build --delete-conflicting-outputs`)
- GoRouter 라우트 등록 (settlementHistory, settlementDetail)

### Must NOT Have (가드레일)
- 정산 수동 요청 UI (서버 자동 처리)
- 정산 취소/환불 UI
- 계좌 등록/변경 UI (별도 Task)
- 정산 통계 차트 (월별 수익 등)
- 실시간 정산 상태 업데이트 (Pull-to-refresh만)

---

## Verification Strategy

### 테스트 인프라
- **자동 테스트**: 없음
- **검증 방법**: Agent-Executed QA (Playwright + Android 에뮬레이터)

### QA 정책
- 모든 UI QA는 `playwright` 스킬로 Android 에뮬레이터에서 실행
- 스크린샷을 `.sisyphus/evidence/` 경로에 저장
- `flutter pub run build_runner build --delete-conflicting-outputs` 명령으로 코드 생성 파일 검증

---

## Execution Strategy

### 병렬 실행 Wave

```
Wave 1 (즉시 시작 — 기반 레이어):
├── Task 1: 정산 피처 DTO + Entity + Repository 인터페이스
└── Task 2: 라우트 + API 엔드포인트 상수 + 프로필 메뉴 추가

Wave 2 (Wave 1 완료 후 — 핵심 구현):
├── Task 3: SettlementRemoteDataSource + RepositoryImpl + DI 등록 (depends: 1)
└── Task 4: SettlementStatusBadge + SettlementHistoryItem 위젯 (depends: 1)

Wave 3 (Wave 2 완료 후 — 통합 UI):
├── Task 5: SettlementHistoryView + ViewModel 구현 (depends: 2, 3, 4)
└── Task 6: SettlementDetailView + ViewModel 구현 (depends: 2, 3)

Critical Path: Task 1 → Task 3 → Task 5
```

### Agent Dispatch
- **Wave 1**: Task 1 → `quick`, Task 2 → `quick`
- **Wave 2**: Task 3 → `unspecified-high`, Task 4 → `visual-engineering`
- **Wave 3**: Task 5 → `visual-engineering`, Task 6 → `visual-engineering`

---

## TODOs

---

- [ ] 1. 정산 피처 — DTO + Entity + Repository 인터페이스

  **What to do**:
  - `features/settlement/data/dto/response/settlement_resp_dto.dart`:
    ```dart
    @freezed
    class SettlementRespDto with _$SettlementRespDto {
      const factory SettlementRespDto({
        required int id,
        required int transactionId,
        required String eventTitle,
        required int amount,
        required int fee,
        required int netAmount,
        required String status,
        String? statusNameKo,
        required DateTime scheduledAt,
        DateTime? processedAt,
        String? failureReason,
        @Default(0) int retryCount,
        required DateTime createdAt,
      }) = _SettlementRespDto;
      factory SettlementRespDto.fromJson(Map<String, dynamic> json) =>
          _$SettlementRespDtoFromJson(json);
    }
    ```
  - `features/settlement/data/dto/response/settlement_list_resp_dto.dart`:
    ```dart
    @freezed
    class SettlementListRespDto with _$SettlementListRespDto {
      const factory SettlementListRespDto({
        required List<SettlementRespDto> items,
        required int totalCount,
        @Default(0) int totalNetAmount,
      }) = _SettlementListRespDto;
      factory SettlementListRespDto.fromJson(Map<String, dynamic> json) =>
          _$SettlementListRespDtoFromJson(json);
    }
    ```
  - `features/settlement/data/dto/response/settlement_detail_resp_dto.dart`:
    ```dart
    @freezed
    class SettlementDetailRespDto with _$SettlementDetailRespDto {
      const factory SettlementDetailRespDto({
        required int id,
        required int transactionId,
        required String eventTitle,
        String? buyerNickname,
        required int amount,
        required int fee,
        required int netAmount,
        required String status,
        String? statusNameKo,
        String? bankName,
        String? accountNumber,   // 서버에서 마스킹 처리됨
        String? accountHolder,
        required DateTime scheduledAt,
        DateTime? processedAt,
        String? failureReason,
        @Default(0) int retryCount,
        required DateTime createdAt,
      }) = _SettlementDetailRespDto;
      factory SettlementDetailRespDto.fromJson(Map<String, dynamic> json) =>
          _$SettlementDetailRespDtoFromJson(json);
    }
    ```
  - `features/settlement/domain/entities/settlement_entity.dart` (freezed):
    ```dart
    @freezed
    abstract class SettlementListEntity with _$SettlementListEntity {
      const factory SettlementListEntity({
        @Default([]) List<SettlementEntity> items,
        @Default(0) int totalCount,
        @Default(0) int totalNetAmount,
      }) = _SettlementListEntity;
    }

    @freezed
    abstract class SettlementEntity with _$SettlementEntity {
      const factory SettlementEntity({
        required int id,
        required int transactionId,
        required String eventTitle,
        required int amount,
        required int fee,
        required int netAmount,
        required String status,
        String? statusNameKo,
        required DateTime scheduledAt,
        DateTime? processedAt,
        String? failureReason,
        required int retryCount,
        required DateTime createdAt,
      }) = _SettlementEntity;
    }

    @freezed
    abstract class SettlementDetailEntity with _$SettlementDetailEntity {
      const factory SettlementDetailEntity({
        required int id,
        required int transactionId,
        required String eventTitle,
        String? buyerNickname,
        required int amount,
        required int fee,
        required int netAmount,
        required String status,
        String? statusNameKo,
        String? bankName,
        String? accountNumber,   // 서버에서 마스킹 처리됨
        String? accountHolder,
        required DateTime scheduledAt,
        DateTime? processedAt,
        String? failureReason,
        @Default(0) int retryCount,
        required DateTime createdAt,
      }) = _SettlementDetailEntity;
    }
    ```
  - `SettlementRespDto`에 `toEntity()` extension 추가 (dispute 패턴 준수):
    ```dart
    extension SettlementRespDtoX on SettlementRespDto {
      SettlementEntity toEntity() => SettlementEntity(
        id: id, transactionId: transactionId, eventTitle: eventTitle,
        amount: amount, fee: fee, netAmount: netAmount, status: status,
        statusNameKo: statusNameKo, scheduledAt: scheduledAt,
        processedAt: processedAt, failureReason: failureReason,
        retryCount: retryCount, createdAt: createdAt,
      );
    }
    ```
  - `SettlementListRespDto`에 `toEntity()` extension 추가:
    ```dart
    extension SettlementListRespDtoX on SettlementListRespDto {
      SettlementListEntity toEntity() => SettlementListEntity(
        items: items.map((e) => e.toEntity()).toList(),
        totalCount: totalCount,
        totalNetAmount: totalNetAmount,
      );
    }
    ```
  - `SettlementDetailRespDto`에 `toEntity()` extension 추가:
    ```dart
    extension SettlementDetailRespDtoX on SettlementDetailRespDto {
      SettlementDetailEntity toEntity() => SettlementDetailEntity(
        id: id, transactionId: transactionId, eventTitle: eventTitle,
        buyerNickname: buyerNickname, amount: amount, fee: fee,
        netAmount: netAmount, status: status, statusNameKo: statusNameKo,
        bankName: bankName, accountNumber: accountNumber,
        accountHolder: accountHolder, scheduledAt: scheduledAt,
        processedAt: processedAt, failureReason: failureReason,
        retryCount: retryCount, createdAt: createdAt,
      );
    }
    ```
  - `features/settlement/domain/repositories/settlement_repository.dart` 인터페이스 (**Entity 반환 — dispute 패턴 준수**):
    ```dart
    abstract class SettlementRepository {
      Future<SettlementListEntity> getSettlements({int page = 1, int size = 20, String? status});
      Future<SettlementDetailEntity> getSettlementDetail(int settlementId);
    }
    ```
  - `flutter pub run build_runner build --delete-conflicting-outputs` 실행

  **Must NOT do**:
  - 비즈니스 로직 DTO에 포함 금지
  - `*.freezed.dart`, `*.g.dart` 수동 편집 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (Task 2와 동시)
  - **Blocks**: Task 3, 4, 5, 6
  - **Blocked By**: 없음

  **References**:
  - `ticket_platform_mobile/lib/features/dispute/data/dto/response/` — freezed DTO 구조 참조
  - `ticket_platform_mobile/lib/features/dispute/domain/repositories/dispute_repository.dart` — Repository 인터페이스 패턴
  - `ticket_platform_mobile/lib/features/profile/data/dto/response/transaction_item_resp_dto.dart` — 거래 관련 DTO 참조

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드러너 코드 생성 확인
    Tool: Bash
    Steps:
      1. flutter pub run build_runner build --delete-conflicting-outputs
      2. flutter build apk --debug
    Expected Result: 오류 없이 빌드 완료
    Evidence: .sisyphus/evidence/task-009-m1-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-m1-build.txt`
  **Commit**: YES (groups with Task 2)
  - Message: `feat(settlement): add settlement feature DTOs, entities, repository interface`
  - Files: `ticket_platform_mobile/lib/features/settlement/data/dto/**`, `ticket_platform_mobile/lib/features/settlement/domain/**`

---

- [ ] 2. 라우트 + API 엔드포인트 + 프로필 메뉴 추가

  **What to do**:
  - `ticket_platform_mobile/lib/core/router/app_router_path.dart` 수정 — 정산 라우트 추가:
    ```dart
    static const settlementHistory = _Route('/settlements', 'settlementHistory');
    static const settlementDetail = _Route('/settlements/detail', 'settlementDetail');
    ```
  - `ticket_platform_mobile/lib/core/network/api_endpoint.dart` 수정 — 엔드포인트 상수 추가:
    ```dart
    static const settlements = '/api/settlements';
    static String settlementDetail(int id) => '/api/settlements/$id';
    ```
  - GoRouter 설정 파일에 라우트 등록 (기존 dispute 라우트 등록 패턴 참조)
  - `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart` 수정:
    - '거래' 섹션 `ProfileSection`에 '정산 내역' 메뉴 타일 추가 (기존 '판매 내역', '구매 내역' 아래):
      ```dart
      ProfileMenuTile(
        icon: Icons.account_balance_wallet_outlined,
        title: '정산 내역',
        onTap: () => context.pushNamed(AppRouterPath.settlementHistory.name),
      ),
      ```

  **Must NOT do**:
  - 기존 '판매 내역', '구매 내역' 메뉴 수정 금지
  - 하드코딩된 아이콘/텍스트 사용 금지 (AppColors 규칙은 유지)

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (Task 1과 동시)
  - **Blocks**: Task 5, 6
  - **Blocked By**: 없음

  **References**:
  - `ticket_platform_mobile/lib/core/router/app_router_path.dart` — 기존 라우트 패턴 (disputeList, disputeCreate, disputeDetail)
  - `ticket_platform_mobile/lib/core/network/api_endpoint.dart` — 기존 엔드포인트 상수
  - `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart` — '거래' 섹션 위치 (line 101~127)
  - GoRouter 설정 파일 (기존 dispute 라우트 등록 패턴 참조)

  **Acceptance Criteria**:
  - [ ] 프로필 화면 '거래' 섹션에 '정산 내역' 메뉴 표시
  - [ ] '정산 내역' 탭 시 정산 화면으로 navigate (화면 미구현이면 빈 Scaffold)

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인
    Tool: Bash
    Steps:
      1. flutter build apk --debug
    Expected Result: 오류 없이 빌드 완료
    Evidence: .sisyphus/evidence/task-009-m2-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-m2-build.txt`
  **Commit**: YES (groups with Task 1)
  - Message: `feat(settlement): add settlement feature DTOs, entities, repository interface`
  - Files: `ticket_platform_mobile/lib/core/router/app_router_path.dart`, `ticket_platform_mobile/lib/core/network/api_endpoint.dart`, `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart`, GoRouter 설정 파일

---

- [ ] 3. SettlementRemoteDataSource + RepositoryImpl + DI 등록

  **What to do**:
  - `features/settlement/data/datasources/settlement_remote_data_source.dart` — 인터페이스 + 구현체 (dispute 패턴):
    ```dart
    abstract class SettlementRemoteDataSource {
      Future<BaseResponse<SettlementListRespDto>> getSettlements({
        int page = 1,
        int size = 20,
        String? status,
      });
      Future<BaseResponse<SettlementDetailRespDto>> getSettlementDetail(int settlementId);
    }

    class SettlementRemoteDataSourceImpl implements SettlementRemoteDataSource {
      final Dio _dio;
      SettlementRemoteDataSourceImpl(this._dio);

      @override
      Future<BaseResponse<SettlementListRespDto>> getSettlements({
        int page = 1,
        int size = 20,
        String? status,
      }) async {
        final queryParams = <String, dynamic>{
          'page': page,
          'size': size,
          if (status != null) 'status': status,
        };
        return safeApiCall<SettlementListRespDto>(
          apiCall: (options) => _dio.get(
            ApiEndpoint.settlements,
            queryParameters: queryParams,
            options: options,
          ),
          apiName: 'getSettlements',
          dataParser: (json) => SettlementListRespDto.fromJson(json as Map<String, dynamic>),
        );
      }

      @override
      Future<BaseResponse<SettlementDetailRespDto>> getSettlementDetail(int settlementId) async {
        return safeApiCall<SettlementDetailRespDto>(
          apiCall: (options) => _dio.get(
            ApiEndpoint.settlementDetail(settlementId),
            options: options,
          ),
          apiName: 'getSettlementDetail',
          dataParser: (json) => SettlementDetailRespDto.fromJson(json as Map<String, dynamic>),
        );
      }
    }
    ```
  - `features/settlement/data/repositories/settlement_repository_impl.dart`:
    ```dart
    class SettlementRepositoryImpl implements SettlementRepository {
      final SettlementRemoteDataSource _remoteDataSource;
      SettlementRepositoryImpl(this._remoteDataSource);
      @override
      Future<SettlementListEntity> getSettlements({int page = 1, int size = 20, String? status}) async {
        final response = await _remoteDataSource.getSettlements(page: page, size: size, status: status);
        return response.mapOrThrow((dto) => dto.toEntity(), errorMessage: '정산 내역을 불러올 수 없습니다.');
      }

      @override
      Future<SettlementDetailEntity> getSettlementDetail(int settlementId) async {
        final response = await _remoteDataSource.getSettlementDetail(settlementId);
        return response.mapOrThrow((dto) => dto.toEntity(), errorMessage: '정산 상세 정보를 불러올 수 없습니다.');
      }
    }
    ```
  - `features/settlement/presentation/providers/settlement_providers_di.dart` — **Riverpod DI 등록** (dispute_providers_di.dart 패턴 완전 참조):
    ```dart
    import 'package:riverpod_annotation/riverpod_annotation.dart';
    import 'package:ticket_platform_mobile/core/network/dio_provider.dart';
    // ... settlement 관련 import

    part 'settlement_providers_di.g.dart';

    @riverpod
    SettlementRemoteDataSource settlementRemoteDataSource(Ref ref) {
      return SettlementRemoteDataSourceImpl(ref.watch(dioProvider));
    }

    @riverpod
    SettlementRepository settlementRepository(Ref ref) {
      return SettlementRepositoryImpl(ref.watch(settlementRemoteDataSourceProvider));
    }
    ```
  - `flutter pub run build_runner build --delete-conflicting-outputs` 실행

  **Must NOT do**:
  - API 엔드포인트 URL 하드코딩 금지 (상수 파일 사용)
  - UseCase 레이어는 불필요 (단순 조회만이므로 Repository 직접 사용 가능)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (Task 4와 동시)
  - **Blocks**: Task 5, 6
  - **Blocked By**: Task 1

  **References**:
  - `features/dispute/data/datasources/dispute_remote_data_source.dart` — RemoteDataSource 패턴 (safeApiCall, dataParser)
  - `features/dispute/data/repositories/dispute_repository_impl.dart` — RepositoryImpl 패턴 (mapOrThrow)
  - `features/dispute/presentation/providers/dispute_providers_di.dart` — Riverpod DI 패턴 완전 참조
  - `core/network/dio_provider.dart` — Dio 클라이언트 DI

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인
    Tool: Bash
    Steps:
      1. flutter pub run build_runner build --delete-conflicting-outputs
      2. flutter build apk --debug
    Expected Result: 오류 없이 빌드 완료
    Evidence: .sisyphus/evidence/task-009-m3-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-m3-build.txt`
  **Commit**: NO (Task 5와 함께)

---

- [ ] 4. SettlementStatusBadge + SettlementHistoryItem 위젯

  **What to do**:
  - `features/settlement/presentation/widgets/settlement_status_badge.dart` 생성:
    ```dart
    // 상태별 색상 매핑
    // pending → AppColors.warning (노란색) + "대기"
    // processing → AppColors.info (파란색) + "처리중"
    // completed → AppColors.success (녹색) + "완료"
    // failed → AppColors.error (빨간색) + "실패"
    // 작은 둥근 뱃지 형태, 텍스트 + 배경색
    ```
    - `AppColors`에 `warning`, `info`, `success` 색상이 없으면 추가
    - 패턴: 기존 `TransactionHistoryItem` 내 상태 표시 위젯 참조
  - `features/settlement/presentation/widgets/settlement_history_item.dart` 생성:
    ```dart
    // Card 형태의 리스트 아이템
    // 좌측: 이벤트명 (1줄, 말줄임), 정산 예정일 or 완료일
    // 우측: 정산 금액 (NetAmount, 볼드), SettlementStatusBadge
    // 탭 시 상세 화면으로 navigate
    // 디자인: transaction_history_item.dart 스타일 참조
    ```

  **Must NOT do**:
  - 하드코딩 색상 사용 금지 (AppColors 사용)
  - 하드코딩 텍스트 스타일 사용 금지 (AppTextStyles 사용)

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
  - **Skills**: [`frontend-ui-ux`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (Task 3과 동시)
  - **Blocks**: Task 5
  - **Blocked By**: Task 1

  **References**:
  - `features/profile/presentation/widgets/transaction_history_item.dart` — 리스트 아이템 UI 패턴
  - `core/theme/app_colors.dart` — 색상 토큰 (warning, info, success 추가 여부 확인)
  - `core/theme/app_text_styles.dart` — 텍스트 스타일
  - `core/theme/app_spacing.dart` — 간격 상수

  **Acceptance Criteria**:
  - [ ] 상태별 뱃지 색상 4종 정상 표시
  - [ ] 아이템 탭 시 상세 화면 navigate

  **QA Scenarios**:

  ```
  Scenario: 위젯 빌드 확인
    Tool: Bash
    Steps:
      1. flutter build apk --debug
    Expected Result: 오류 없이 빌드 완료
    Evidence: .sisyphus/evidence/task-009-m4-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-009-m4-build.txt`
  **Commit**: NO (Task 5와 함께)

---

- [ ] 5. SettlementHistoryView + ViewModel 구현

  **What to do**:
  - `features/settlement/presentation/viewmodels/settlement_history_viewmodel.dart` 구현:
    ```dart
    // @riverpod 어노테이션 사용
    // 상태: items, totalCount, totalNetAmount, isLoadingMore, selectedStatus, page
    // 메서드:
    //   - loadInitial() → page=1 로드
    //   - loadMore() → 다음 페이지 로드 (무한 스크롤)
    //   - filterByStatus(String? status) → 상태 필터 변경 후 재로드
    //   - refresh() → pull-to-refresh
    ```
  - `features/settlement/presentation/views/settlement_history_view.dart` 구현:
    - AppBar: "정산 내역" 타이틀
    - 상단 요약 카드: 총 정산 금액 (totalNetAmount, completed 기준)
    - 상태 필터 Chip: 전체 | 대기 | 완료 | 실패
    - ListView.separated: SettlementHistoryItem 사용
    - 무한 스크롤: ScrollController + loadMore()
    - Pull-to-refresh: RefreshIndicator
    - 빈 상태: "정산 내역이 없습니다" + 아이콘
    - 에러 상태: 에러 메시지 + "다시 시도" 버튼
    - 디자인: `transaction_history_view.dart` 레이아웃 패턴 완전 참조 (gradient 배경, AppBar 스타일)

  **Must NOT do**:
  - 정산 수동 요청 버튼 추가 금지
  - 하드코딩 색상/폰트 사용 금지

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
  - **Skills**: [`frontend-ui-ux`]

  **Parallelization**:
  - **Can Run In Parallel**: NO (Task 3, 4 완료 후)
  - **Parallel Group**: Wave 3 (Task 6과 동시)
  - **Blocks**: 없음
  - **Blocked By**: Task 2, 3, 4

  **References**:
  - `features/profile/presentation/views/transaction_history_view.dart` — 리스트 화면 패턴 (무한 스크롤, 필터, 빈 상태, 에러 상태)
  - `features/profile/presentation/viewmodels/transaction_history_viewmodel.dart` — ViewModel 패턴 (Riverpod)
  - `features/dispute/presentation/` — View + ViewModel 구현 패턴
  - `core/theme/` — 디자인 토큰

  **Acceptance Criteria**:
  - [ ] 정산 내역 리스트 정상 표시 (이벤트명, 금액, 상태 뱃지)
  - [ ] 상태 필터 Chip 동작 (전체/대기/완료/실패)
  - [ ] 무한 스크롤 동작 (20건 단위)
  - [ ] Pull-to-refresh 동작
  - [ ] 빈 상태 UI 표시
  - [ ] 총 정산 금액 표시

  **QA Scenarios**:

  ```
  Scenario: 정산 내역 리스트 표시 (Happy Path)
    Tool: Playwright
    Preconditions:
      - 판매자 계정 로그인
      - 해당 판매자에게 Settlement 레코드 존재
      - 백엔드 서버 실행 중 (localhost:5224)
    Steps:
      1. 프로필 화면 진입
      2. '정산 내역' 메뉴 탭
      3. 정산 리스트 화면 확인
      4. 이벤트명, 금액, 상태 뱃지 확인
      5. 스크린샷 캡처
    Expected Result:
      - 정산 내역 리스트 정상 표시
      - 상태 뱃지 색상 정상
    Evidence: .sisyphus/evidence/task-009-m5-list.png

  Scenario: 상태 필터 동작
    Tool: Playwright
    Steps:
      1. '완료' 필터 Chip 탭
      2. completed 상태 건만 표시 확인
      3. '전체' 필터 Chip 탭
      4. 모든 상태 건 표시 확인
    Expected Result: 필터에 따라 리스트 갱신
    Evidence: .sisyphus/evidence/task-009-m5-filter.png

  Scenario: 빈 상태 (정산 없음)
    Tool: Playwright
    Preconditions: 정산 내역 없는 판매자 계정
    Steps:
      1. 정산 내역 화면 진입
      2. 빈 상태 UI 확인 (아이콘 + 텍스트)
    Expected Result: "정산 내역이 없습니다" 표시
    Evidence: .sisyphus/evidence/task-009-m5-empty.png
  ```

  **Evidence**: `.sisyphus/evidence/task-009-m5-*.png`
  **Commit**: YES
  - Message: `feat(settlement): implement settlement history view with filter and pagination`
  - Pre-commit: `flutter build apk --debug`
  - Files: `ticket_platform_mobile/lib/features/settlement/presentation/**`, `ticket_platform_mobile/lib/features/settlement/data/datasources/**`, `ticket_platform_mobile/lib/features/settlement/data/repositories/**`

---

- [ ] 6. SettlementDetailView + ViewModel 구현

  **What to do**:
  - `features/settlement/presentation/viewmodels/settlement_detail_viewmodel.dart` 구현:
    ```dart
    // @riverpod 어노테이션, family 패턴 (settlementId 파라미터)
    // 상태: SettlementDetailEntity (AsyncValue)
    // 메서드: loadDetail(int settlementId)
    ```
  - `features/settlement/presentation/views/settlement_detail_view.dart` 구현:
    - AppBar: "정산 상세" 타이틀
    - 상단: 상태 뱃지 (크게) + 상태명
    - 거래 정보 섹션:
      - 이벤트명
      - 구매자 닉네임
      - 거래 ID
    - 정산 금액 섹션:
      - 총 금액 (amount)
      - 수수료 (fee)
      - 구분선
      - **순 정산 금액 (netAmount, 볼드, 강조색)**
    - 정산 계좌 섹션:
      - 은행명
      - 계좌번호 (마스킹됨, 서버에서 처리)
      - 예금주
    - 정산 일정 섹션:
      - 정산 예정일 (scheduledAt)
      - 정산 완료일 (processedAt, null이면 "-")
    - 실패 정보 (status == "failed" 일 때만):
      - 실패 사유 (failureReason)
      - 재시도 횟수 (retryCount)
    - 로딩/에러 상태 처리

  **Must NOT do**:
  - 정산 취소/재요청 버튼 추가 금지
  - 계좌 변경 버튼 추가 금지
  - 계좌번호 모바일에서 언마스킹 시도 금지 (서버에서 마스킹 처리)

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
  - **Skills**: [`frontend-ui-ux`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (Task 5와 동시)
  - **Blocks**: 없음
  - **Blocked By**: Task 2, 3

  **References**:
  - `features/dispute/presentation/views/` — 상세 화면 구현 패턴
  - `features/profile/presentation/views/transaction_history_view.dart` — 스타일 참조
  - `core/theme/` — 디자인 토큰

  **Acceptance Criteria**:
  - [ ] 정산 상세 정보 정상 표시 (금액, 상태, 계좌, 일정)
  - [ ] 계좌번호 마스킹 정상 표시 (서버에서 처리됨)
  - [ ] 실패 건: failureReason, retryCount 표시
  - [ ] 완료 건: processedAt 표시
  - [ ] 대기 건: processedAt = "-"
  - [ ] 로딩 상태 처리 (CircularProgressIndicator)
  - [ ] 에러 상태 처리 (에러 메시지 + 재시도)

  **QA Scenarios**:

  ```
  Scenario: 정산 상세 조회 (완료 건)
    Tool: Playwright
    Preconditions:
      - completed 상태 Settlement 존재
      - 정산 내역 화면에서 해당 아이템 탭
    Steps:
      1. 정산 아이템 탭 → 상세 화면 진입
      2. 이벤트명, 금액, 수수료, 순 정산 금액 확인
      3. 계좌 정보 (마스킹) 확인
      4. 정산 완료일 표시 확인
      5. 스크린샷 캡처
    Expected Result:
      - 모든 정보 정상 표시
      - 계좌번호 마스킹 (****1234 형태)
      - 완료일 표시
    Evidence: .sisyphus/evidence/task-009-m6-detail-completed.png

  Scenario: 정산 상세 조회 (실패 건)
    Tool: Playwright
    Preconditions: failed 상태 Settlement 존재
    Steps:
      1. 실패 건 아이템 탭
      2. 실패 사유, 재시도 횟수 표시 확인
    Expected Result:
      - failureReason 텍스트 표시
      - retryCount 표시
    Evidence: .sisyphus/evidence/task-009-m6-detail-failed.png
  ```

  **Evidence**: `.sisyphus/evidence/task-009-m6-*.png`
  **Commit**: YES
  - Message: `feat(settlement): implement settlement detail view`
  - Pre-commit: `flutter build apk --debug`
  - Files: `ticket_platform_mobile/lib/features/settlement/presentation/views/settlement_detail_view.dart`, `ticket_platform_mobile/lib/features/settlement/presentation/viewmodels/settlement_detail_viewmodel.dart`

---

## Final Verification Wave

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Must Have 7개 항목 전체 Playwright로 검증. 프로필 → 정산 내역 → 상세 엔드투엔드 확인. 상태 필터 동작 확인. `.sisyphus/evidence/` 파일 존재 확인.
  Output: `Must Have [N/7] | Must NOT Have [N/5] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  `flutter build apk --debug` + 하드코딩 색상 탐색 (`Color(0x` 직접 사용). `*.freezed.dart`, `*.g.dart` 재생성 완료 여부. Null safety 위반 탐색 (`!` 강제 언박싱). Riverpod DI 등록 확인.
  Output: `Build [PASS/FAIL] | Hardcoded Colors [N] | Null Issues [N] | VERDICT`

- [ ] F3. **Real Manual QA** — `unspecified-high` + `playwright`
  전체 QA 시나리오 실행. 프로필 → 정산 내역 → 필터 → 상세 엔드투엔드. Android 에뮬레이터 기준.
  Output: `Scenarios [N/N pass] | VERDICT`

---

## Commit Strategy

| 커밋 | 메시지 | 파일 |
|------|--------|------|
| 1 | `feat(settlement): add settlement feature DTOs, entities, repository interface` | `ticket_platform_mobile/lib/features/settlement/data/dto/`, `ticket_platform_mobile/lib/features/settlement/domain/`, `ticket_platform_mobile/lib/core/router/`, `ticket_platform_mobile/lib/core/network/`, `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart`, `*.freezed.dart`, `*.g.dart` |
| 2 | `feat(settlement): implement settlement history view with filter and pagination` | `ticket_platform_mobile/lib/features/settlement/presentation/**`, `ticket_platform_mobile/lib/features/settlement/data/datasources/`, `ticket_platform_mobile/lib/features/settlement/data/repositories/` |
| 3 | `feat(settlement): implement settlement detail view` | `ticket_platform_mobile/lib/features/settlement/presentation/views/settlement_detail_view.dart`, `ticket_platform_mobile/lib/features/settlement/presentation/viewmodels/settlement_detail_viewmodel.dart` |

---

## Success Criteria

```bash
# 빌드 확인
flutter build apk --debug
# Expected: Build successful

# 코드 생성 확인
flutter pub run build_runner build --delete-conflicting-outputs
# Expected: 오류 없이 완료
```

### Final Checklist
- [ ] 프로필 '거래' 섹션에 '정산 내역' 메뉴 타일 표시
- [ ] 정산 내역 리스트 (이벤트명, 금액, 상태 뱃지, 예정일)
- [ ] 상태 필터 (전체/대기/완료/실패)
- [ ] 무한 스크롤 + Pull-to-refresh
- [ ] 정산 상세 (금액 상세, 계좌 마스킹, 상태)
- [ ] 빈 상태 / 에러 상태 처리
- [ ] freezed/json_serializable 재생성 완료
- [ ] 하드코딩 색상 없음
- [ ] GoRouter 라우트 등록 완료
