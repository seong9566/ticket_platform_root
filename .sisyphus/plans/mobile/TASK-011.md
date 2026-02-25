# TASK-011 결제 화면 가독성 개선 — Mobile 계획

## TL;DR

> **요약**: 결제 화면(`payment_view.dart`)을 구매자 관점 정보 구조로 재설계한다.
> 핵심은 "구매할 티켓 정보"와 "공연 정보"를 분리해 한눈에 이해되게 만드는 것이다.
>
> **디자인 기준 이미지**: `.sisyphus/plans/mobile/image.png`
>
> **핵심 방향**:
> - 서버 응답 기반 구조화 데이터 사용
> - 로컬 문자열 조합(`orderName`) 중심 표시 제거
> - 결제 금액/좌석/공연일시를 시각적 우선순위 상단 배치
>
> **산출물**:
> - `PaymentRequestRespDto`, `PaymentRequestEntity` 계약 확장 반영
> - `PaymentViewModel.requestPayment()` 입력 간소화
> - `payment_view.dart` UI 재구성 (티켓 정보 카드 + 공연 정보 카드)
> - Android 기준 QA 스크린샷 Evidence

---

## Context

### 작업 기준 디렉토리
모든 파일 경로는 `ticket_platform_mobile/lib/` 기준.

### 현재 문제점
- 결제 화면에서 핵심 정보가 `orderName` 문자열과 일반 메타정보에 혼합되어 가독성이 낮음.
  - `ticket_platform_mobile/lib/features/payment/presentation/views/payment_view.dart`
- 결제 메타데이터 일부(`eventTitle`, `seatInfo` 등)를 서버 응답이 아닌 클라이언트에서 주입/보관하고 있음.
  - `ticket_platform_mobile/lib/features/payment/presentation/viewmodels/payment_viewmodel.dart`
  - `ticket_platform_mobile/lib/features/payment/data/repositories/payment_repository_impl.dart`
- 결제 진입 시 채팅 화면에서 긴 문자열을 조합해 전달하고 있어 UI/데이터 책임 경계가 모호함.
  - `ticket_platform_mobile/lib/features/chat/presentation/view/chat_room_view.dart`
- 결제 진입 파라미터와 ViewModel 시그니처 간 불일치(`sellerName` 전달) 정합성 이슈가 있어,
  구현 시 파라미터 계약 정리가 선행되어야 함.

### 참고 UX 진단 (image.png)
- 핵심 정보 대비 약함, 타이틀 말줄임, 섹션 계층 모호
- 주문/안내 텍스트 밀집으로 구매 결정 정보 탐색 속도 저하

---

## Work Objectives

### 핵심 목표
결제 화면에서 사용자가 3초 이내에 아래를 파악할 수 있어야 한다.
1. 내가 무엇을 사는지 (좌석/수량/금액)
2. 어떤 공연인지 (공연명/일시/장소)
3. 결제 액수가 맞는지 (총 결제금액)

### Must Have
- 결제 화면 정보 계층 재구성:
  - 상단: 총 결제금액
  - 섹션1: 구매할 티켓 정보
  - 섹션2: 공연 정보
  - 섹션3: 주문자 정보(보조)
- 서버 응답 `ticketInfo`, `eventInfo`를 화면 데이터의 단일 소스로 사용
- 긴 공연명/좌석 정보 줄바꿈 처리(말줄임 최소화)
- 정보 레이블과 값 대비 개선 (디자인 토큰 사용)
- CTA(`결제하기`)는 현재 동작/문구 유지

### Must NOT Have
- `orderName` 문자열 분해/파싱 로직 추가 금지
- 하드코딩 색상/폰트 추가 금지 (AppColors/AppTextStyles 사용)
- 결제 승인/실패 라우팅 로직 변경 금지
- 서버 미연동 임시 목데이터 상시 의존 금지

---

## 데이터 계약 반영안 (모바일)

`PaymentRequestEntity` 확장 필드(예시):

```dart
PaymentTicketInfoEntity? ticketInfo;
PaymentEventInfoEntity? eventInfo;
```

세부 필드:
- ticketInfo: ticketId, thumbnailUrl, seatInfo, quantity, unitPrice, totalAmount
- eventInfo: eventId, title, eventDateTime, venueName

> 기존 `eventTitle/eventDate/seatInfo/ticketImageUrl/venueName`의
> 클라이언트 주입 경로는 제거하고 서버 응답으로 단일화.

---

## Execution Strategy

### 병렬 실행 Wave

```
Wave 1 (데이터 계층 정리):
├── Task 1: Payment DTO/Entity 계약 확장
└── Task 2: ViewModel/Repository 파라미터 정리 (서버 응답 단일 소스화)

Wave 2 (UI 재설계):
├── Task 3: payment_view.dart 레이아웃 재구성
└── Task 4: 세부 위젯 분리(티켓/공연/주문자 카드)

Wave 3 (연결/검증):
├── Task 5: 채팅 결제 진입점 전달값 정리
└── Task 6: Android QA + 빌드 검증

Critical Path: Task 1 → Task 2 → Task 3 → Task 6
```

---

## TODOs

- [ ] 1. Payment DTO/Entity 계약 확장

  **What to do**:
  - `ticket_platform_mobile/lib/features/payment/data/dto/response/payment_request_resp_dto.dart`
    - 서버 확장 응답(`ticketInfo`, `eventInfo`) 매핑 추가
  - `ticket_platform_mobile/lib/features/payment/domain/entities/payment_entities.dart`
    - `PaymentTicketInfoEntity`, `PaymentEventInfoEntity` 추가
    - `PaymentRequestEntity`에 신규 필드 연결

  **Acceptance Criteria**:
  - [ ] DTO → Entity 변환에서 ticket/event 정보 손실 없음
  - [ ] nullable 필드 누락 시 앱 크래시 없음

---

- [ ] 2. ViewModel/Repository 파라미터 정리

  **What to do**:
  - `ticket_platform_mobile/lib/features/payment/presentation/viewmodels/payment_viewmodel.dart`
    - `requestPayment()`에서 클라이언트 주입 메타(`eventTitle` 등) 제거
  - `ticket_platform_mobile/lib/features/payment/data/repositories/payment_repository_impl.dart`
    - `dto.toEntity(...)` 수동 주입 제거, 서버 응답 직접 변환
  - `ticket_platform_mobile/lib/features/payment/domain/usecases/payment_params.dart`
    - 결제 요청 파라미터 최소화

  **Must NOT do**:
  - 서버 확장 전 임시 파싱 로직 추가 금지

  **Acceptance Criteria**:
  - [ ] 결제 요청 파이프라인에서 중복 데이터 소스 제거
  - [ ] request->response->entity 경로가 단일 계약으로 유지

---

- [ ] 3. PaymentView 정보 구조 재설계

  **What to do**:
  - `ticket_platform_mobile/lib/features/payment/presentation/views/payment_view.dart` 수정
  - 섹션 구성:
    1) 금액 강조 카드
    2) 구매할 티켓 정보 카드
    3) 공연 정보 카드
    4) 주문자/주문번호 카드
    5) 결제 안내 카드
  - 좌석/공연일시는 본문보다 높은 시각적 우선순위로 표현

  **Acceptance Criteria**:
  - [ ] "구매할 티켓 정보"와 "공연 정보" 헤더가 명확히 분리
  - [ ] 공연명 긴 텍스트 2줄 이상 자연 줄바꿈
  - [ ] 말줄임은 보조 필드에서만 제한적으로 사용

---

- [ ] 4. 위젯 분리로 유지보수성 확보

  **What to do**:
  - payment_view.dart 내부 private widget 분리:
    - `_PaymentAmountCard`
    - `_PaymentTicketInfoCard`
    - `_PaymentEventInfoCard`
    - `_PaymentOrderMetaCard`
  - 중복 label/value row는 공통 컴포넌트 재사용

  **Acceptance Criteria**:
  - [ ] 화면 파일 복잡도 감소
  - [ ] 각 카드 단위 UI 수정이 독립적으로 가능

---

- [ ] 5. 채팅 결제 진입점 정리

  **What to do**:
  - `ticket_platform_mobile/lib/features/chat/presentation/view/chat_room_view.dart`
    - 결제 요청 시 서버 계약 기준 필수 필드만 전달
    - UI 구성 데이터는 결제 응답(entity) 사용으로 통일
  - 결제 진입 `extra` 전달 구조 유지 (`PaymentRequestEntity`)

  **Acceptance Criteria**:
  - [ ] 결제 진입 성공/실패 플로우 기존 동작 유지
  - [ ] 결제 화면 렌더링에 필요한 핵심 정보가 서버 응답에서 충족

---

- [ ] 6. Android QA + 빌드 검증

  **QA Scenarios**:
  - Scenario A: 정상 데이터(썸네일/좌석/일시/장소 모두 있음)
  - Scenario B: 썸네일 없음(placeholder 처리)
  - Scenario C: 공연명 40자 이상 긴 텍스트
  - Scenario D: 좌석 정보 길이 긴 케이스

  **Commands**:
  - `flutter pub run build_runner build --delete-conflicting-outputs`
  - `flutter build apk --debug`

  **Evidence**:
  - `.sisyphus/evidence/task-011-m-a-normal.png`
  - `.sisyphus/evidence/task-011-m-b-no-thumb.png`
  - `.sisyphus/evidence/task-011-m-c-long-title.png`
  - `.sisyphus/evidence/task-011-m-d-long-seatinfo.png`
  - `.sisyphus/evidence/task-011-m-build.txt`

---

## Success Criteria

- [ ] 결제 화면에 구매 티켓/공연 정보가 명확히 분리 표시됨
- [ ] 화면 핵심 정보가 서버 응답 단일 소스로 구성됨
- [ ] 사용자 입장에서 좌석/공연일시/금액 확인 시간이 단축됨
- [ ] Android 빌드 통과 및 QA 스크린샷 확보
