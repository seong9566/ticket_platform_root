# TASK-008 평판 시스템 — Mobile 계획

## TL;DR

> **요약**: 구매확정 후 채팅방 버블에서 판매자 별점 리뷰를 작성하고, 프로필/티켓 상세 화면에 별점을 노출하는 Flutter 기능을 구현한다.
>
> **산출물**:
> - `purchase_confirmed_bubble.dart` — "리뷰 남기기" 버튼 추가 (작성 완료 시 숨김)
> - `reputation_write_view.dart` — 별점 선택 + 코멘트 입력 전용 화면
> - `profile_header_section.dart` — ★4.8 (23개) 형태 별점 표시 추가
> - `TicketingSellerUiModel` — averageRating, reviewCount 필드 추가
> - 평판 피처 디렉토리: `features/reputation/` (data + domain + presentation 레이어)
>
> **예상 공수**: Medium  
> **병렬 실행**: YES — 3 Wave  
> **Android-First**: 모든 QA는 Android 에뮬레이터 기준

---

## Context

### 기획 확정 사항
| 항목 | 결정 |
|------|------|
| 리뷰 UX | 채팅방 구매확정 버블 → "리뷰 남기기" 버튼 → 별도 화면 |
| 별점 노출 | 프로필 화면 + 티켓 상세 판매자 정보 |
| 리뷰 버튼 상태 | `CanWriteReview`, `HasReviewedSeller` 플래그 (`ChatRoomRespDto`에서 수신, 파일: `features/chat/data/dto/response/chat_resp_dto.dart`) |
| 리뷰 기간 | 서버에서 판단 (7일), 모바일은 버튼 상태만 표시 |
| 코드 생성 | freezed/json_serializable 재생성 필요 (`*.freezed.dart`, `*.g.dart`) |

### 수정 대상 기존 파일
- `features/chat/presentation/widgets/purchase_confirmed_bubble.dart` — 리뷰 버튼 추가
- `features/profile/presentation/widgets/profile_header_section.dart` — 별점 UI 추가
- `features/profile/data/dto/response/my_profile_resp_dto.dart` — averageRating, reviewCount 추가
- `features/ticketing/presentation/ui_models/ticketing_ticket_ui_model.dart` — averageRating 추가
 `features/chat/data/dto/response/chat_resp_dto.dart` (`ChatRoomRespDto`) — canWriteReview, hasReviewedSeller 추가

### 신규 생성 디렉토리
```
features/reputation/
├── data/
│   ├── dto/
│   │   ├── request/create_reputation_req_dto.dart
│   │   └── response/reputation_resp_dto.dart
│   │       reputation_check_resp_dto.dart
│   │       reputation_list_resp_dto.dart
│   ├── datasources/reputation_remote_data_source.dart   ← 인터페이스 + 구현체 동일 파일 (dispute 패턴)
│   └── repositories/reputation_repository_impl.dart     ← domain ReputationRepository 구현체
├── domain/
│   ├── entities/reputation_entity.dart
│   ├── repositories/reputation_repository.dart          ← abstract 인터페이스
│   └── usecases/
│       ├── create_reputation_usecase.dart
│       └── check_reputation_usecase.dart
└── presentation/
    ├── providers/reputation_providers_di.dart            ← @riverpod DI 등록 (dispute_providers_di.dart 패턴)
    ├── viewmodels/reputation_write_viewmodel.dart
    └── views/reputation_write_view.dart
        widgets/star_rating_widget.dart
```

### 참조 패턴
- `features/dispute/` — 피처 디렉토리 구조 완전 참조
- `features/profile/presentation/widgets/profile_header_section.dart` — 별점 추가 대상
- `features/chat/presentation/widgets/purchase_confirmed_bubble.dart` — 버튼 추가 대상
- `features/ticketing/presentation/ui_models/ticketing_ticket_ui_model.dart` — 필드 추가 대상

---

## Work Objectives

### 핵심 목표
구매확정 완료 거래에서 구매자가 채팅방 내에서 판매자를 별점 평가하고,
그 결과가 프로필 및 티켓 상세 화면에 즉시 반영되어야 한다.

### Must Have
 `ChatRoomRespDto.canWriteReview` 기반 버튼 표시/숨김 (파일: `features/chat/data/dto/response/chat_resp_dto.dart`)
- 리뷰 작성 완료 후 버튼 → "리뷰 완료" 텍스트로 전환 (재진입 시 유지)
- 별점 위젯: 1~5 선택, 코멘트 선택 입력(최대 500자)
- 제출 성공 후 채팅방으로 pop + 버블 상태 갱신
- 프로필 화면: `averageRating`, `reviewCount` 표시
- 티켓 상세 화면: 판매자 별점 표시
- freezed/json_serializable 재생성 (`flutter pub run build_runner build --delete-conflicting-outputs`)

### Must NOT Have (가드레일)
- 내가 남긴 리뷰 목록 화면 (받은 리뷰 목록만)
- 리뷰 수정/삭제 UI
- 판매자가 리뷰에 답글 쓰는 UI
- 별점 분포 차트 (1~5 개수)
- 채팅방 이외 경로에서의 리뷰 작성 진입 (FCM 탭으로 바로 진입은 별도 Task)
- MannerTemperature 계산 로직 모바일에 구현 (서버 전담)

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
├── Task 1: 평판 피처 DTO + Entity + Repository 인터페이스
└── Task 2: 기존 DTO/UiModel 필드 확장 + 코드 재생성

Wave 2 (Wave 1 완료 후 — 핵심 구현):
├── Task 3: ReputationRemoteDataSource + UseCase 구현 (depends: 1)
├── Task 4: purchase_confirmed_bubble.dart 리뷰 버튼 추가 (depends: 2)
└── Task 5: profile_header_section.dart 별점 UI 추가 (depends: 2)

Wave 3 (Wave 2 완료 후 — 통합 UI):
├── Task 6: ReputationWriteView + ViewModel 구현 (depends: 3, 4)
└── Task 7: 티켓 상세 판매자 별점 UI 추가 (depends: 2)

Critical Path: Task 1 → Task 3 → Task 6
```

### Agent Dispatch
- **Wave 1**: Task 1 → `quick`, Task 2 → `quick`
- **Wave 2**: Task 3 → `unspecified-high`, Task 4 → `visual-engineering`, Task 5 → `visual-engineering`
- **Wave 3**: Task 6 → `visual-engineering`, Task 7 → `visual-engineering`

---

## TODOs

---

- [ ] 1. 평판 피처 — DTO + Entity + Repository 인터페이스

  **What to do**:
  - `features/reputation/data/dto/request/create_reputation_req_dto.dart`:
    ```dart
    @freezed
    class CreateReputationReqDto with _$CreateReputationReqDto {
      const factory CreateReputationReqDto({
        required int transactionId,
        required int score,       // 1~5
        String? comment,          // optional, 최대 500자
      }) = _CreateReputationReqDto;
      factory CreateReputationReqDto.fromJson(Map<String, dynamic> json) =>
          _$CreateReputationReqDtoFromJson(json);
    }
    ```
  - `features/reputation/data/dto/response/reputation_check_resp_dto.dart`:
    ```dart
    @freezed
    class ReputationCheckRespDto with _$ReputationCheckRespDto {
      const factory ReputationCheckRespDto({
        required bool canReview,
        required bool hasReviewed,
        DateTime? reviewDeadline,
      }) = _ReputationCheckRespDto;
      factory ReputationCheckRespDto.fromJson(Map<String, dynamic> json) =>
          _$ReputationCheckRespDtoFromJson(json);
    }
    ```
  - `features/reputation/domain/entities/reputation_entity.dart` (freezed):
    - id, reviewerNickname, reviewerProfileImageUrl, score, comment, createdAt
  - `features/reputation/domain/repositories/reputation_repository.dart` 인터페이스:
    ```dart
    abstract class ReputationRepository {
      Future<void> createReputation(CreateReputationReqDto dto);
      Future<ReputationCheckRespDto> checkReputation(int transactionId);
    }
    ```
  - `flutter pub run build_runner build --delete-conflicting-outputs` 실행

  **Must NOT do**:
  - 비즈니스 로직 DTO에 포함 금지
  - `*.freezed.dart`, `*.g.dart` 수동 편집 금지 (빌드러너로 자동 생성)

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (Task 2와 동시)
  - **Blocks**: Task 3, 6
  - **Blocked By**: 없음

  **References**:
  - `features/dispute/data/dto/` — freezed DTO 구조 참조
  - `features/dispute/domain/repositories/` — Repository 인터페이스 패턴 참조 (`dispute_repository.dart`)
  - `features/dispute/data/datasources/dispute_remote_data_source.dart` — DataSource 인터페이스+구현체 패턴
  - `features/profile/data/dto/response/my_profile_resp_dto.dart` — freezed 스타일 참조

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드러너 코드 생성 확인
    Tool: Bash
    Steps:
      1. flutter pub run build_runner build --delete-conflicting-outputs
      2. flutter build apk --debug (빌드 오류 확인)
    Expected Result: 오류 없이 빌드 완료
    Evidence: .sisyphus/evidence/task-1-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-1-build.txt`
  **Commit**: YES (groups with Task 2)
  - Message: `feat(reputation): add reputation feature DTOs, entities, repository interface`
  - Files: `features/reputation/data/dto/**`, `features/reputation/domain/**`

---

- [ ] 2. 기존 DTO/UiModel 필드 확장 + 코드 재생성

  **What to do**:
  - `features/chat/data/dto/response/chat_resp_dto.dart`의 `ChatRoomRespDto` 수정:
    ```dart
    // 기존 canConfirmPurchase, canCancelTransaction 아래에 추가
    @Default(false) bool canWriteReview,
    @Default(false) bool hasReviewedSeller,
    ```
  - `ChatRoomRespDtoX.toEntity()` 확장메서드에 새 필드 매핑 추가 → `ChatRoomEntity`에도 해당 필드 추가 필요
  - `features/chat/domain/entities/chat_room_entity.dart`에 `canWriteReview`, `hasReviewedSeller` 필드 추가
  - `features/chat/presentation/ui_models/chat_room_ui_model.dart` (존재 시) 에 해당 필드 전파
  - `features/profile/data/dto/response/my_profile_resp_dto.dart` 수정:
    ```dart
    // 기존 mannerTemperature, totalTradeCount 아래에 추가
    double? averageRating,
    @Default(0) int reviewCount,
    ```
  - `features/profile/data/dto/response/user_profile_resp_dto.dart` 수정 (타인 프로필용이 별도라면):
    - 동일하게 averageRating, reviewCount 추가
  - `features/ticketing/presentation/ui_models/ticketing_ticket_ui_model.dart` 수정:
    - `TicketingSellerUiModel`에 `double? averageRating`, `int reviewCount` 추가
  - `flutter pub run build_runner build --delete-conflicting-outputs` 실행
  - ChatViewModel/ProfileViewModel에서 새 필드 매핑 확인 (DTO → UiModel 변환 코드)

  **Must NOT do**:
  - 기존 `mannerTemperature` 필드 제거 금지
  - `*.freezed.dart`, `*.g.dart` 수동 편집 금지

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (Task 1과 동시)
  - **Blocks**: Task 3, 4, 5, 6, 7
  - **Blocked By**: 없음

  **References**:
  - `features/chat/data/dto/response/chat_resp_dto.dart` — `ChatRoomRespDto` 기존 필드 구조 (line 28~46)
  - `features/chat/domain/entities/chat_room_entity.dart` — Entity에 필드 추가 필요
  - `features/profile/data/dto/response/my_profile_resp_dto.dart` — 기존 필드 구조
  - `features/ticketing/presentation/ui_models/ticketing_ticket_ui_model.dart` — SellerUiModel 구조
  - `features/dispute/` — DTO 필드 추가 후 ViewModel 매핑 예시

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드러너 코드 재생성 확인
    Tool: Bash
    Steps:
      1. flutter pub run build_runner build --delete-conflicting-outputs
      2. flutter build apk --debug
    Expected Result: 오류 없이 빌드 완료
    Evidence: .sisyphus/evidence/task-2-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-2-build.txt`
  **Commit**: YES (groups with Task 1)
  - Message: `feat(reputation): add reputation feature DTOs, entities, repository interface`
  - Files: `features/chat/data/dto/response/chat_resp_dto.dart`, `features/chat/domain/entities/chat_room_entity.dart`, `features/profile/data/dto/response/my_profile_resp_dto.dart`, `features/ticketing/presentation/ui_models/ticketing_ticket_ui_model.dart`, `**/*.freezed.dart`, `**/*.g.dart`

---

- [ ] 3. ReputationRemoteDataSource + UseCase 구현

  **What to do**:
  - `features/reputation/data/datasources/reputation_remote_data_source.dart` — 인터페이스 + 구현체 (dispute 패턴: 하나의 파일에 abstract class + impl class):
    ```dart
    abstract class ReputationRemoteDataSource {
      Future<BaseResponse<void>> createReputation(CreateReputationReqDto dto);
      Future<BaseResponse<ReputationCheckRespDto>> checkReputation(int transactionId);
    }

    class ReputationRemoteDataSourceImpl implements ReputationRemoteDataSource {
      final Dio _dio;
      ReputationRemoteDataSourceImpl(this._dio);

      @override
      Future<BaseResponse<void>> createReputation(CreateReputationReqDto dto) async {
        return safeApiCall<void>(
          apiCall: (options) => _dio.post(ApiEndpoint.reputations, data: dto.toMap(), options: options),
          apiName: 'createReputation',
        );
      }
      // ... checkReputation 동일 패턴
    }
    ```
  - `features/reputation/data/repositories/reputation_repository_impl.dart` — domain Repository 구현체:
    ```dart
    class ReputationRepositoryImpl implements ReputationRepository {
      final ReputationRemoteDataSource _remoteDataSource;
      ReputationRepositoryImpl(this._remoteDataSource);

      @override
      Future<void> createReputation(CreateReputationReqDto dto) async {
        final response = await _remoteDataSource.createReputation(dto);
        response.mapOrThrow((_) {}, errorMessage: '리뷰 작성에 실패했습니다.');
      }
      // ... checkReputation 동일 패턴
    }
    ```
  - `features/reputation/domain/usecases/create_reputation_usecase.dart`:
    ```dart
    class CreateReputationUseCase {
      final ReputationRepository _repo;
      CreateReputationUseCase(this._repo);
      Future<void> call(CreateReputationReqDto dto) => _repo.createReputation(dto);
    }
    ```
  - `features/reputation/domain/usecases/check_reputation_usecase.dart`:
    ```dart
    class CheckReputationUseCase {
      final ReputationRepository _repo;
      CheckReputationUseCase(this._repo);
      Future<ReputationCheckRespDto> call(int transactionId) => _repo.checkReputation(transactionId);
    }
    ```
  - `features/reputation/presentation/providers/reputation_providers_di.dart` — **Riverpod DI 등록** (dispute_providers_di.dart 패턴 완전 참조):
    ```dart
    import 'package:riverpod_annotation/riverpod_annotation.dart';
    import 'package:ticket_platform_mobile/core/network/dio_provider.dart';
    // ... reputation 관련 import

    part 'reputation_providers_di.g.dart';

    @riverpod
    ReputationRemoteDataSource reputationRemoteDataSource(Ref ref) {
      return ReputationRemoteDataSourceImpl(ref.watch(dioProvider));
    }

    @riverpod
    ReputationRepository reputationRepository(Ref ref) {
      return ReputationRepositoryImpl(ref.watch(reputationRemoteDataSourceProvider));
    }

    @riverpod
    CreateReputationUseCase createReputationUsecase(Ref ref) {
      return CreateReputationUseCase(ref.watch(reputationRepositoryProvider));
    }

    @riverpod
    CheckReputationUseCase checkReputationUsecase(Ref ref) {
      return CheckReputationUseCase(ref.watch(reputationRepositoryProvider));
    }
    ```
  - `core/network/api_endpoint.dart`에 엔드포인트 상수 추가:
    ```dart
    static const reputations = '/api/reputations';
    static String reputationCheck(int transactionId) => '/api/reputations/check/$transactionId';
    ```

  **Must NOT do**:
  - API 엔드포인트 URL 하드코딩 금지 (상수 파일 사용)
  - UseCase에 UI 로직 포함 금지

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (Task 4, 5와 동시)
  - **Blocks**: Task 6
  - **Blocked By**: Task 1

  **References**:
  - `features/dispute/data/datasources/dispute_remote_data_source.dart` — RemoteDataSource 인터페이스+구현체 패턴
  - `features/dispute/data/repositories/dispute_repository_impl.dart` — RepositoryImpl 패턴
  - `features/dispute/domain/usecases/` — UseCase 패턴
  - `features/dispute/presentation/providers/dispute_providers_di.dart` — Riverpod DI 패턴 (완전 참조)
  - `core/network/api_endpoint.dart` — 엔드포인트 상수
  - `core/network/dio_provider.dart` — Dio 클라이언트 DI

  **Acceptance Criteria**:

  **QA Scenarios**:

  ```
  Scenario: 빌드 확인
    Tool: Bash
    Steps:
      1. flutter build apk --debug
    Expected Result: 오류 없이 빌드 완료
    Evidence: .sisyphus/evidence/task-3-build.txt
  ```

  **Evidence**: `.sisyphus/evidence/task-3-build.txt`
  **Commit**: NO (Task 6과 함께)

---

- [ ] 4. purchase_confirmed_bubble.dart — 리뷰 버튼 추가

  **What to do**:
  - `features/chat/presentation/widgets/purchase_confirmed_bubble.dart` 수정
  - 현재 버블은 `isBuyer` 플래그만 수신 → `canWriteReview`, `hasReviewedSeller` 파라미터 추가
  - 버블 하단 조건부 UI 추가:
    ```dart
    // canWriteReview == true && hasReviewedSeller == false
    → "리뷰 남기기" 텍스트 버튼 표시 (탭 시 ReputationWriteView로 navigate)
    
    // hasReviewedSeller == true
    → "리뷰 완료 ✓" 텍스트 (비활성, 회색)
    
    // canWriteReview == false && hasReviewedSeller == false
    → 아무것도 표시 안 함 (기간 만료 또는 구매자 아님)
    ```
  - `ReputationWriteView`로 이동 시 `transactionId`, `sellerNickname` 전달
  - 리뷰 완료 후 채팅방으로 복귀 시 버블 상태 갱신:
    - ChatViewModel에서 `hasReviewedSeller = true`로 업데이트 (서버 재조회 또는 로컬 상태 업데이트)
  - 버튼 스타일: 기존 채팅 버블 내 버튼 스타일 참조 (AppColors, AppTextStyles)

  **Must NOT do**:
  - 리뷰 작성 로직을 버블 위젯에 포함 금지 (navigate만)
  - 하드코딩된 색상/폰트 사용 금지 (AppColors, AppTextStyles 사용)

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (Task 3, 5와 동시)
  - **Blocks**: Task 6
  - **Blocked By**: Task 2

  **References**:
  - `features/chat/presentation/widgets/purchase_confirmed_bubble.dart` — 현재 구현 확인 (파라미터, 스타일)
  - `features/chat/presentation/viewmodels/` — ChatViewModel에서 canWriteReview 플래그 매핑 확인
  - `core/theme/app_colors.dart`, `core/theme/app_text_styles.dart` — 디자인 토큰
  - `features/dispute/presentation/` — 채팅방 내 다른 버튼 Navigate 패턴

  **Acceptance Criteria**:
  - [ ] canWriteReview=true, hasReviewedSeller=false → "리뷰 남기기" 버튼 표시
  - [ ] hasReviewedSeller=true → "리뷰 완료 ✓" 표시
  - [ ] canWriteReview=false → 버튼 없음

  **QA Scenarios**:

  ```
  Scenario: 리뷰 버튼 표시 확인 (Happy Path)
    Tool: Playwright (playwright 스킬)
    Preconditions:
      - 구매확정 완료된 채팅방 진입
      - canWriteReview=true, hasReviewedSeller=false 상태
    Steps:
      1. 채팅방 화면에서 구매확정 버블 확인
      2. 버블 하단 "리뷰 남기기" 버튼 존재 여부 확인
      3. 스크린샷 캡처
    Expected Result: "리뷰 남기기" 텍스트 버튼이 버블 하단에 표시됨
    Evidence: .sisyphus/evidence/task-4-review-button.png

  Scenario: 리뷰 완료 상태 표시 (완료 후)
    Tool: Playwright
    Preconditions: 리뷰 작성 완료 후 채팅방 복귀
    Steps:
      1. 채팅방 화면에서 구매확정 버블 확인
      2. "리뷰 완료 ✓" 텍스트 표시 확인
      3. 스크린샷 캡처
    Expected Result: "리뷰 완료 ✓" 텍스트 표시, 버튼 비활성
    Evidence: .sisyphus/evidence/task-4-review-done.png
  ```

  **Evidence**: `.sisyphus/evidence/task-4-*.png`
  **Commit**: NO (Task 6과 함께)

---

- [ ] 5. profile_header_section.dart — 별점 UI 추가

  **What to do**:
  - `features/profile/presentation/widgets/profile_header_section.dart` 수정
  - `mannerTemperatureText` 아래 또는 옆에 별점 표시 추가:
    ```dart
    // averageRating != null && reviewCount > 0 이면 표시
    Row(
      children: [
        Icon(Icons.star, color: AppColors.starYellow, size: 14),
        Text(' ${averageRating!.toStringAsFixed(1)}', style: AppTextStyles.caption),
        Text(' (${reviewCount}개)', style: AppTextStyles.captionGray),
      ],
    )
    // averageRating == null 또는 reviewCount == 0 이면 "리뷰 없음" 또는 표시 생략
    ```
  - 내 프로필 화면과 타인 프로필 화면 모두 적용
  - 타인 프로필 화면에서 별점 영역 탭 시 리뷰 목록 화면으로 이동 (별도 Task로 분리하지 않고 navigate만 준비, 리뷰 목록 화면은 Task 6 완료 후 연결)
  - AppColors에 `starYellow` 색상이 없으면 추가 (`const Color(0xFFFFC107)`)

  **Must NOT do**:
  - MannerTemperature 위젯 제거/수정 금지 (기존 유지)
  - 하드코딩된 색상 직접 사용 금지

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (Task 3, 4와 동시)
  - **Blocks**: 없음 (Task 7과 독립)
  - **Blocked By**: Task 2

  **References**:
  - `features/profile/presentation/widgets/profile_header_section.dart` — 현재 레이아웃 구조
  - `core/theme/app_colors.dart` — 색상 토큰 (starYellow 추가 여부 확인)
  - `features/profile/presentation/viewmodels/` — profileViewModel에서 averageRating 매핑 확인

  **Acceptance Criteria**:
  - [ ] averageRating=4.8, reviewCount=23 → "★4.8 (23개)" 형태 표시
  - [ ] averageRating=null → 별점 UI 표시 안 함 (graceful null handling)

  **QA Scenarios**:

  ```
  Scenario: 별점 표시 확인 (리뷰 있음)
    Tool: Playwright
    Preconditions: 리뷰가 1건 이상인 판매자 프로필 화면
    Steps:
      1. 판매자 프로필 화면 진입
      2. 별점 텍스트(★N.N) 표시 확인
      3. 스크린샷 캡처
    Expected Result: 별 아이콘 + 숫자 + 리뷰 수 표시
    Evidence: .sisyphus/evidence/task-5-profile-rating.png

  Scenario: 리뷰 없음 처리 (Null Safety)
    Tool: Playwright
    Preconditions: 리뷰가 0건인 판매자 프로필
    Steps:
      1. 리뷰 없는 판매자 프로필 화면 진입
      2. 별점 UI 표시 안 됨 확인 (크래시 없음)
      3. 스크린샷 캡처
    Expected Result: 별점 UI 없음, 앱 정상 동작
    Evidence: .sisyphus/evidence/task-5-profile-no-rating.png
  ```

  **Evidence**: `.sisyphus/evidence/task-5-*.png`
  **Commit**: YES
  - Message: `feat(profile): add star rating display to profile header`
  - Files: `features/profile/presentation/widgets/profile_header_section.dart`, `core/theme/app_colors.dart`

---

- [ ] 6. ReputationWriteView + ViewModel 구현

  **What to do**:
  - `features/reputation/presentation/viewmodels/reputation_write_viewmodel.dart` 구현:
    ```dart
    // 상태 관리
    int selectedScore = 0;     // 0 = 미선택
    String comment = '';
    bool isSubmitting = false;
    String? errorMessage;
    bool isSuccess = false;
    
    // 메서드
    void selectScore(int score);
    void updateComment(String text);
    Future<void> submit(int transactionId);
    ```
    - submit 성공 시 `isSuccess = true` → View에서 pop + 콜백 호출
    - submit 실패 시 `errorMessage` 설정 → SnackBar 표시
  - `features/reputation/presentation/views/reputation_write_view.dart` 구현:
    - AppBar: "리뷰 작성" 타이틀
    - 판매자 닉네임 표시
    - 별점 선택 위젯 (5개 별, 탭 시 색상 변경)
    - 코멘트 TextField (선택, 최대 500자, `maxLength: 500`)
    - "제출" 버튼: selectedScore > 0 이어야 활성화
    - 제출 중 로딩 표시
  - `features/reputation/presentation/widgets/star_rating_widget.dart` 분리:
    ```dart
    // 파라미터: currentScore, onScoreSelected
    // 별 5개, 선택 시 황색, 미선택 회색
    ```
  - `ChatRoomScreen`에서 "리뷰 남기기" 버튼 탭 시 `ReputationWriteView`로 navigate:
    ```dart
    Navigator.push(context, MaterialPageRoute(
      builder: (_) => ReputationWriteView(
        transactionId: widget.transactionId,
        sellerNickname: sellerNickname,
        onSuccess: () => chatViewModel.markReviewedSeller(),
      ),
    ));
    ```
  - ChatViewModel에 `markReviewedSeller()` 메서드 추가 (로컬 상태 업데이트)

  **Must NOT do**:
  - 별점 제출 후 자동으로 채팅방 메시지 발송 금지
  - 리뷰 목록 조회 이 화면에서 구현 금지

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (Task 3, 4 완료 후)
  - **Parallel Group**: Wave 3 (Task 7과 동시)
  - **Blocks**: 없음 (최종 통합)
  - **Blocked By**: Task 3, 4

  **References**:
  - `features/dispute/presentation/views/` — View + ViewModel 구현 패턴
  - `features/chat/presentation/widgets/purchase_confirmed_bubble.dart` — navigate 연결 위치
  - `features/chat/presentation/viewmodels/` — ChatViewModel markReviewedSeller 추가 위치
  - `core/theme/app_colors.dart`, `core/theme/app_text_styles.dart` — 디자인 토큰

  **Acceptance Criteria**:
  - [ ] 별점 미선택 시 "제출" 버튼 비활성
  - [ ] 별점 3선택 + 제출 → API 호출 → 성공 → 채팅방 복귀
  - [ ] 채팅방 복귀 후 구매확정 버블 "리뷰 완료 ✓" 표시
  - [ ] 네트워크 오류 시 SnackBar 표시, 화면 유지
  - [ ] 코멘트 500자 초과 입력 차단

  **QA Scenarios**:

  ```
  Scenario: 리뷰 작성 전체 플로우 (Happy Path)
    Tool: Playwright
    Preconditions:
      - 구매확정 완료 채팅방, canWriteReview=true
      - 백엔드 서버 실행 중 (localhost:5224)
    Steps:
      1. 채팅방에서 구매확정 버블 "리뷰 남기기" 버튼 탭
      2. ReputationWriteView 화면 진입 확인 (AppBar "리뷰 작성" 텍스트)
      3. 별 5개 중 4번째 별 탭 (4점 선택)
      4. 코멘트 입력: "믿을 수 있는 판매자였어요"
      5. "제출" 버튼 탭
      6. 로딩 후 채팅방으로 복귀 확인
      7. 구매확정 버블에 "리뷰 완료 ✓" 표시 확인
      8. 각 단계 스크린샷 캡처
    Expected Result:
      - 리뷰 작성 화면 정상 진입
      - 4점 선택 시 별 4개 황색, 1개 회색
      - 제출 후 채팅방 복귀
      - 버블에 "리뷰 완료 ✓" 표시
    Evidence: .sisyphus/evidence/task-6-flow-{1..7}.png

  Scenario: 별점 미선택 제출 시도 (Failure Path)
    Tool: Playwright
    Steps:
      1. ReputationWriteView 진입
      2. 별점 선택 없이 "제출" 버튼 탭 시도
    Expected Result: 버튼 비활성 상태 (탭 불가)
    Evidence: .sisyphus/evidence/task-6-no-score.png

  Scenario: 코멘트 500자 제한
    Tool: Playwright
    Steps:
      1. ReputationWriteView 진입
      2. 코멘트 TextField에 501자 입력 시도
    Expected Result: 500자에서 입력 차단 (maxLength 카운터 표시)
    Evidence: .sisyphus/evidence/task-6-comment-limit.png
  ```

  **Evidence**: `.sisyphus/evidence/task-6-*.png`
  **Commit**: YES
  - Message: `feat(reputation): implement reputation write view and viewmodel`
  - Pre-commit: `flutter build apk --debug`
  - Files: `features/reputation/presentation/**`, `features/chat/presentation/widgets/purchase_confirmed_bubble.dart`, `features/chat/presentation/viewmodels/`

---

- [ ] 7. 티켓 상세 — 판매자 별점 UI 추가

  **What to do**:
  - 티켓 상세 화면에서 판매자 정보 표시 위젯 탐색 (TicketingSellerUiModel 사용 위젯)
  - 판매자 닉네임/프로필 이미지 아래 또는 옆에 별점 표시 추가:
    ```dart
    // averageRating != null && reviewCount > 0 이면
    Row(
      children: [
        Icon(Icons.star, color: AppColors.starYellow, size: 12),
        Text(' ${averageRating.toStringAsFixed(1)}', style: AppTextStyles.caption),
        Text(' (${reviewCount})', style: AppTextStyles.captionGray),
      ],
    )
    ```
  - `TicketingSellerUiModel` → DTO 매핑 코드에서 `averageRating`, `reviewCount` 필드 연결 확인
  - `GET /api/tickets/{id}` 또는 `GET /api/ticketing/{id}` 응답에 판매자 averageRating이 포함되는지 확인:
    - 포함되지 않으면 백엔드 팀에 `SellerInfoDto`에 필드 추가 요청 (백엔드 계획 Task 4와 연동)
    - 포함된다면 UiModel 매핑만 추가

  **Must NOT do**:
  - 별점 탭 시 리뷰 목록 화면으로 이동 기능 구현 금지 (이번 스코프 외)
  - 하드코딩 색상 사용 금지

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (Task 6과 동시)
  - **Blocks**: 없음
  - **Blocked By**: Task 2

  **References**:
  - `features/ticketing/presentation/ui_models/ticketing_ticket_ui_model.dart` — SellerUiModel 구조
  - `features/ticketing/presentation/` — 판매자 정보 표시 위젯 파일 탐색
  - `core/theme/app_colors.dart` — starYellow 색상
  - Task 5 (profile_header_section.dart) — 동일한 별점 UI 패턴 재사용

  **Acceptance Criteria**:
  - [ ] 티켓 상세에서 판매자 별점 표시 (averageRating != null 조건)
  - [ ] averageRating=null → 별점 UI 없음 (크래시 없음)

  **QA Scenarios**:

  ```
  Scenario: 티켓 상세 판매자 별점 표시
    Tool: Playwright
    Preconditions: 리뷰가 있는 판매자의 티켓 상세 화면
    Steps:
      1. 티켓 목록에서 티켓 탭 → 상세 화면 진입
      2. 판매자 정보 영역에서 별점(★N.N) 표시 확인
      3. 스크린샷 캡처
    Expected Result: 판매자 닉네임 옆/아래에 별점 텍스트 표시
    Evidence: .sisyphus/evidence/task-7-ticketing-rating.png

  Scenario: 리뷰 없는 판매자 티켓 상세 (Null Safety)
    Tool: Playwright
    Preconditions: 리뷰 0건 판매자의 티켓
    Steps:
      1. 해당 티켓 상세 화면 진입
      2. 별점 UI 없음 확인, 크래시 없음 확인
      3. 스크린샷 캡처
    Expected Result: 별점 UI 없음, 정상 동작
    Evidence: .sisyphus/evidence/task-7-no-rating.png
  ```

  **Evidence**: `.sisyphus/evidence/task-7-*.png`
  **Commit**: YES
  - Message: `feat(ticketing): add seller rating display to ticket detail`
  - Files: `features/ticketing/presentation/**`

---

## Final Verification Wave

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Must Have 7개 항목 전체 Playwright로 검증. canWriteReview 플래그 기반 버튼 조건부 렌더링 확인. 리뷰 완료 후 버블 상태 갱신 확인. .sisyphus/evidence/ 파일 존재 확인.
  Output: `Must Have [N/7] | Must NOT Have [N/5] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  `flutter build apk --debug` + 하드코딩 색상 탐색 (`Color(0x` 직접 사용). `*.freezed.dart`, `*.g.dart` 재생성 완료 여부. Null safety 위반 탐색 (`!` 강제 언박싱).
  Output: `Build [PASS/FAIL] | Hardcoded Colors [N] | Null Issues [N] | VERDICT`

- [ ] F3. **Real Manual QA** — `unspecified-high` + `playwright`
  전체 QA 시나리오 실행. 리뷰 작성 → 채팅방 복귀 → 버블 상태 갱신 엔드투엔드. 프로필 화면/티켓 상세 별점 UI 확인. Android 에뮬레이터 기준.
  Output: `Scenarios [N/N pass] | VERDICT`

---

## Commit Strategy

| 커밋 | 메시지 | 파일 |
|------|--------|------|
| 1 | `feat(reputation): add reputation feature DTOs, entities, repository interface` | `features/reputation/data/`, `features/reputation/domain/`, 기존 DTO 수정, `*.freezed.dart`, `*.g.dart` |
| 2 | `feat(profile): add star rating display to profile header` | `profile_header_section.dart`, `app_colors.dart` |
| 3 | `feat(reputation): implement reputation write view and viewmodel` | `features/reputation/presentation/`, `purchase_confirmed_bubble.dart`, ChatViewModel |
| 4 | `feat(ticketing): add seller rating display to ticket detail` | `features/ticketing/presentation/` |

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
- [ ] 구매확정 버블에 "리뷰 남기기" 버튼 표시 (canWriteReview=true 조건)
- [ ] 리뷰 작성 화면에서 1~5 별점 선택 가능
- [ ] 제출 성공 후 채팅방 복귀 + 버블 "리뷰 완료 ✓" 상태
- [ ] 프로필 화면 별점 표시 (null safety 처리)
- [ ] 티켓 상세 판매자 별점 표시 (null safety 처리)
- [ ] freezed/json_serializable 재생성 완료
- [ ] 하드코딩 색상 없음
