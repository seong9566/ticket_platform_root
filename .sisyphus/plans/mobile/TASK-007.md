# TASK-007: 홈 화면 마감 임박 핫딜 UI (모바일)

## TL;DR

> **Quick Summary**: 홈 화면 상단의 광고 배너 카루셀(`HomeBannerCarousel`)을 제거하고, Backend TASK-007에서 추가한 `deadlineDeals` 필드를 사용하여 **마감 임박 핫딜 카드 슬라이더**로 대체한다. DTO/Entity/UiModel에 DeadlineDeal 클래스를 추가하고, 카드 슬라이더 위젯을 신규 생성한다.
> 
> **Deliverables**:
> - `docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md` — 모바일 Agent 작업 지시서
> - `ticket_platform_mobile/.../widgets/home_deadline_deal_slider.dart` — 신규 카드 슬라이더 위젯
> - `ticket_platform_mobile/.../dto/response/home_resp_dto.dart` — DeadlineDealDto 추가
> - `ticket_platform_mobile/.../entities/home_entity.dart` — DeadlineDealEntity 추가
> - `ticket_platform_mobile/.../ui_models/home_ui_model.dart` — DeadlineDealUiModel 추가
> - `ticket_platform_mobile/.../views/home_view.dart` — 배너→핫딜 교체
> 
> **Estimated Effort**: Medium (1일)
> **Parallel Execution**: YES - 2 waves
> **Critical Path**: Task 1 (문서) → Task 2 (UI 구현) → F1 (검증)

---

## Context

### Original Request
홈 화면 상단의 광고 배너 슬라이더가 중고 티켓 거래 플랫폼에 불필요하므로, 마감 임박 핫딜 카드 슬라이더로 대체한다.

### Interview Summary
**Key Discussions**:
- UI 레이아웃: **카드 슬라이더** (기존 배너 위치, 자동 슬라이드 3초)
- 빈 상태 처리: **섹션 숨김** (마감 임박 0건이면 카테고리 아이콘이 바로 나옴)
- D-day 배지 색상: D-0=#EF4444(빨강), D-1=#F97316(주황), D-2=#EAB308(노랑), D-3=#3B82F6(파랑)
- 카드 탭 시 `/ticketing/{eventId}` 이동

### Research Findings
- 현재 배너 카루셀은 하드코딩된 3개 더미 텍스트('Summer Rock Festival' 등)와 고정 Unsplash 이미지만 사용
- 서버 `Banners` 필드는 모바일이 완전히 무시 중
- Clean MVVM 아키텍처: DataSource → Repository → UseCase → ViewModel → View
- `@freezed` + `fromJson()` + `toEntity()` 패턴 준수 필요

### 선행 Backend 계획
- `.sisyphus/plans/backend/TASK-007.md` — Backend API 구현 (이 계획의 선행 작업)

---

## Work Objectives

### Core Objective
광고 배너를 제거하고, 공연 D-3 이내 마감 임박 핫딜을 카드 슬라이더로 표시하여 구매 전환율을 높인다.

### Concrete Deliverables
- `docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md` — 모바일 작업 지시서
- `ticket_platform_mobile/.../widgets/home_deadline_deal_slider.dart` — 신규 위젯
- `ticket_platform_mobile/.../dto/response/home_resp_dto.dart` — DeadlineDealDto 추가
- `ticket_platform_mobile/.../entities/home_entity.dart` — DeadlineDealEntity 추가
- `ticket_platform_mobile/.../ui_models/home_ui_model.dart` — DeadlineDealUiModel 추가
- `ticket_platform_mobile/.../views/home_view.dart` — 배너→핫딜 교체

### Definition of Done
- [ ] 홈 화면에서 배너 대신 핫딜 카드 슬라이더 표시됨
- [ ] 각 카드에 D-day 배지, 할인율, 최저가, 잔여 티켓 수 표시
- [ ] 카드 탭 시 공연 상세 화면(`/ticketing/{eventId}`)으로 이동
- [ ] 자동 슬라이드 동작 (3초 간격, 순환)
- [ ] 마감 임박 0건 시 섹션 숨김
- [ ] DTO/Entity/UiModel에 `deadlineDeals` 필드 정상 파싱
- [ ] 기존 기능(카테고리, 인기 공연, 추천 공연) 정상 동작
- [ ] `flutter build apk --debug` 성공

### Must Have
- D-day 배지 색상 (D-0 빨강, D-1 주황, D-2 노랑, D-3 파랑)
- 자동 슬라이드 3초 간격, 순환
- 0건 시 섹션 완전 숨김 (헤더 + 슬라이더 모두)
- 카드 탭 → `/ticketing/{eventId}` 이동
- `@freezed` + `fromJson()` + `toEntity()` 패턴 준수
- `dart run build_runner build --delete-conflicting-outputs` 실행

### Must NOT Have (Guardrails)
- `home_banner_carousel.dart` 파일 삭제 금지 (import만 제거)
- 기존 `PopularEventList`, `RecommendedEventList` 수정 금지
- D-day 배지 외 불필요한 애니메이션 추가 금지
- 하드코딩 색상/폰트 사용 금지 (`AppColors`, `AppTextStyles`, `AppSpacing` 사용)
- 기존 `home_banner_carousel.dart` 파일 내용 수정 금지

---

## Verification Strategy

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed.

### Test Decision
- **Infrastructure exists**: NO
- **Automated tests**: None
- **Framework**: N/A
- **플랫폼**: Android-First (모든 테스트는 Android에서 진행)

### QA Policy
- **Build**: `flutter build apk --debug` 성공 확인
- **Code Generation**: `dart run build_runner build` 성공 확인

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Start Immediately — 문서 작성):
└── Task 1: Mobile TASK-007 문서 작성 [quick]

Wave 2 (After Wave 1 + Backend TASK-007 Task 2 완료 — 모바일 구현):
└── Task 2: Mobile UI 구현 (DTO + 슬라이더 위젯 + 배너 제거) [visual-engineering]

Wave FINAL (After ALL tasks):
└── Task F1: 모바일 통합 검증 [deep]

Critical Path: Task 1 → Task 2 → F1
External Dependency: Backend TASK-007 Task 2 완료 필요 (Task 2 시작 전)
```

### Dependency Matrix

| Task | Depends On | Blocks | Wave |
|------|-----------|--------|------|
| 1 | — | 2 | 1 |
| 2 | 1, Backend TASK-007 Task 2 | F1 | 2 |
| F1 | 2 | — | FINAL |

### Agent Dispatch Summary

- **Wave 1**: 1 task — T1 → `quick`
- **Wave 2**: 1 task — T2 → `visual-engineering`
- **FINAL**: 1 task — F1 → `deep`

---

## TODOs

> Implementation + verification = ONE Task.
> EVERY task MUST have: Recommended Agent Profile + Parallelization info + QA Scenarios.

- [ ] 1. Mobile TASK-007 문서 작성

  **What to do**:
  - `docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md` 파일을 생성하라
  - 아래 '문서 전문' 섹션의 전체 내용을 그대로 파일에 작성하라
  - 기존 TASK-006 문서의 형식과 톤을 따르라

  **Must NOT do**:
  - 문서 내용을 임의로 축약하거나 변경하지 말 것
  - 기존 TASK-006 문서 형식에서 벗어나지 말 것

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 단순 문서 파일 생성, 로직 복잡도 없음
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 1 (단독)
  - **Blocks**: Task 2
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `docs/mobile/tasks/TASK-006_Dispute_System.md` — 문서 형식/톤 참조 (474줄)

  **API/Type References**:
  - `ticket_platform_mobile/lib/features/home/presentation/views/home_view.dart` — 현재 홈 화면 구현 (155줄)
  - `ticket_platform_mobile/lib/features/home/presentation/widgets/home_banner_carousel.dart` — 제거 대상 배너 위젯 (138줄)
  - `ticket_platform_mobile/lib/features/home/data/dto/response/home_resp_dto.dart` — 현재 DTO 구조 (97줄)
  - `ticket_platform_mobile/lib/features/home/domain/entities/home_entity.dart` — 현재 Entity 구조 (46줄)
  - `ticket_platform_mobile/lib/features/home/presentation/ui_models/home_ui_model.dart` — 현재 UiModel 구조 (105줄)
  - `ticket_platform_mobile/CLAUDE.md` — Flutter 프로젝트 코딩 규칙

  **Acceptance Criteria**:
  - [ ] `docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md` 파일 생성됨
  - [ ] 아래 '문서 전문'의 전체 내용이 빠짐없이 포함됨
  - [ ] 기존 TASK-006 문서와 동일한 마크다운 형식

  **QA Scenarios:**
  ```
  Scenario: Mobile TASK-007 문서 생성 확인
    Tool: Bash
    Steps:
      1. test -f docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md && echo "EXISTS"
      2. head -1 docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md
      3. grep -c 'deadlineDeals\|DeadlineDeal\|home_deadline' docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md
    Expected Result: 파일 존재, 첫 줄 "# TASK-007: 홈 화면 마감 임박 핫딜 UI (모바일)", 관련 키워드 10회 이상
    Failure Indicators: 파일 미존재, 키워드 누락
    Evidence: .sisyphus/evidence/task-1-mobile-doc.txt
  ```

  **Commit**: YES
  - Message: `docs: add TASK-007 Home Deadline Deals mobile spec`
  - Files: `docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md`

  <details>
  <summary>📄 문서 전문 (클릭하여 펼치기)</summary>

  # TASK-007: 홈 화면 마감 임박 핫딜 UI (모바일)

  **작성일**: 2026-02-23
  **작성자**: PM
  **담당 팀**: Mobile
  **담당자**: Mobile Agent
  **상태**:  완료
  **우선순위**: 🟡 Medium

  ---

  ## 📋 작업 개요

  ### 작업 설명
  홈 화면 상단의 광고 배너 카루셀(`HomeBannerCarousel`)을 제거하고, **마감 임박 핫딜 카드 슬라이더**로 대체하라. Backend TASK-007에서 추가한 `deadlineDeals` 필드를 사용하여 공연 D-3 이내 핫딜을 표시하라.

  ### 목표
  - 기존 `HomeBannerCarousel` 위젯을 `HomeDeadlineDealSlider`로 교체
  - 카드 슬라이더 UI: 포스터 이미지 + D-day 배지 + 할인율 + 최저가 + 잔여 티켓 수
  - 자동 슬라이드 (3초 간격)
  - 마감 임박 이벤트 0건 시 섹션 숨김 (카테고리 아이콘이 바로 표시)
  - DTO/Entity/UiModel에 `deadlineDeals` 필드 추가

  ### 배경
  현재 홈 화면의 배너 카루셀은 하드코딩된 더미 데이터('Summer Rock Festival' 등)와 Unsplash 고정 이미지만 표시하고 있어 중고 티켓 거래 플랫폼에 전혀 가치가 없다. 공연 임박 핫딜을 강조하면 급처분 심리를 활용하여 구매 전환율을 높일 수 있다.

  ### ⚠️ 플랫폼 전략: Android-First
  Apple 유료 개발자 계정 미확보 상태. Flutter 코드는 양 플랫폼 공유이므로 코드 작성에는 영향 없음. **모든 테스트는 Android에서 진행**하고, iOS는 계정 확보 후 일괄 검증한다.

  ---

  ## 🎯 완료 기준 (Acceptance Criteria)

  - [ ] 홈 화면에서 기존 배너 카루셀 대신 마감 임박 핫딜 카드 슬라이더가 표시됨
  - [ ] 각 카드에 D-day 배지 (D-0, D-1, D-2, D-3), 할인율, 최저가, 잔여 티켓 수 표시
  - [ ] 카드 탭 시 해당 공연 상세 화면(`/ticketing/{eventId}`)으로 이동
  - [ ] 자동 슬라이드 동작 (3초 간격, 마지막 카드 후 첫 카드로 순환)
  - [ ] 마감 임박 이벤트 0건 시 해당 섹션 완전 숨김
  - [ ] DTO/Entity/UiModel에 `deadlineDeals` 필드 정상 파싱
  - [ ] 기존 기능(카테고리, 인기 공연, 추천 공연) 정상 동작
  - [ ] `flutter build apk --debug` 성공

  ---

  ## 🔧 기술 스펙 (Mobile)

  ### API 응답 변경 (Backend TASK-007)

  기존 `GET /api/home` 응답에 `deadlineDeals` 필드가 추가됨:

  ```json
  {
    "data": {
      "deadlineDeals": [
        {
          "eventId": 15,
          "eventTitle": "BTS Yet To Come 부산 콘서트",
          "eventDate": "2026.02.25",
          "venue": "부산 아시아드 주경기장",
          "daysLeft": 2,
          "minTicketPrice": 85000,
          "originalMinTicketPrice": 132000,
          "ticketDiscountRate": 35,
          "posterImageUrl": "https://...",
          "availableTicketCount": 12,
          "categoryId": 1
        }
      ],
      "popularEvents": [...],
      "recommendedEvents": [...]
    }
  }
  ```

  ### 카드 UI 구성

  ```
  ┌─────────────────────────────────────┐
  │  [포스터 이미지 (전체 배경)]          │
  │                                     │
  │  ┌─────┐                            │
  │  │ D-2 │  ← D-day 배지 (좌상단)     │
  │  └─────┘                            │
  │                                     │
  │  ─── 하단 그라데이션 영역 ───        │
  │  BTS Yet To Come 부산 콘서트         │
  │  부산 아시아드 주경기장               │
  │                                     │
  │  -35%  85,000원     🎫 12건 판매중   │
  │                 ● ● ○ ← 페이지 인디케이터
  └─────────────────────────────────────┘
  ```

  ### D-day 배지 색상

  | daysLeft | 표시 | 배지 색상 |
  |----------|------|----------|
  | 0 | D-DAY | 빨간색 (#EF4444) |
  | 1 | D-1 | 주황색 (#F97316) |
  | 2 | D-2 | 노란색 (#EAB308) |
  | 3 | D-3 | 파란색 (#3B82F6) |

  ### 자동 슬라이드 규칙
  - 3초 간격 자동 넘김
  - 마지막 카드 도달 시 첫 카드로 순환
  - 사용자 수동 스와이프 시 자동 슬라이드 타이머 리셋
  - 화면 이탈 시 타이머 중지, 복귀 시 재개

  ---

  ## 📱 화면 구조

  ### 홈 화면 변경 전후 비교

  **변경 전 (현재)**:
  ```
  HomeHeader
  HomeBannerCarousel (200px, 하드코딩 배너)   ← 제거
  HomeEventRow (카테고리 4개)
  🔥 현재 인기 공연
  ✨ 이런 공연은 어때요?
  ```

  **변경 후**:
  ```
  HomeHeader
  🔥 마감 임박 핫딜 (섹션 헤더)               ← 신규 (0건 시 숨김)
  HomeDeadlineDealSlider (200px, 카드)        ← 신규 (0건 시 숨김)
  HomeEventRow (카테고리 4개)
  🔥 현재 인기 공연
  ✨ 이런 공연은 어때요?
  ```

  ---

  ## 📂 파일 구조

  ### 신규 파일
  ```
  lib/features/home/
  └── presentation/widgets/
      └── home_deadline_deal_slider.dart     — 마감 임박 핫딜 카드 슬라이더 위젯
  ```

  ### 수정 대상
  ```
  lib/features/home/
  ├── data/
  │   └── dto/response/
  │       └── home_resp_dto.dart             — DeadlineDealDto 추가, toEntity() 수정
  ├── domain/
  │   └── entities/
  │       └── home_entity.dart               — DeadlineDealEntity 추가
  ├── presentation/
  │   ├── views/
  │   │   └── home_view.dart                 — HomeBannerCarousel → HomeDeadlineDealSlider 교체
  │   └── ui_models/
  │       └── home_ui_model.dart             — DeadlineDealUiModel 추가
  ```

  ### 미사용 처리 (삭제 금지)
  ```
  lib/features/home/
  └── presentation/widgets/
      └── home_banner_carousel.dart          — import 제거만, 파일 유지
  ```

  ---

  ## ✅ 작업 체크리스트

  ### 개발
  - [ ] `home_resp_dto.dart`에 `DeadlineDealDto` 클래스 + `toEntity()` 추가
  - [ ] `home_entity.dart`에 `DeadlineDealEntity` 클래스 추가
  - [ ] `home_ui_model.dart`에 `DeadlineDealUiModel` 클래스 + `fromEntity()` 추가
  - [ ] `HomeRespDto`에 `deadlineDeals` 필드 추가
  - [ ] `HomeEntity`에 `deadlineDeals` 필드 추가
  - [ ] `HomeUiModel`에 `deadlineDeals` 필드 추가
  - [ ] `home_deadline_deal_slider.dart` 위젯 생성 (카드 슬라이더 + 자동 슬라이드 + D-day 배지)
  - [ ] `home_view.dart`에서 `HomeBannerCarousel` → `HomeDeadlineDealSlider` 교체
  - [ ] `home_view.dart`에서 `_bannerController`, `_banners`, `_currentBanner` 관련 코드 제거
  - [ ] 마감 임박 0건 시 섹션 숨김 처리
  - [ ] code generation 실행 (`dart run build_runner build`)

  ### 테스트 (Android)
  - [ ] 마감 임박 이벤트가 있을 때 카드 슬라이더 표시 확인
  - [ ] 마감 임박 0건일 때 섹션 숨김 확인
  - [ ] 카드 탭 시 공연 상세 이동 확인
  - [ ] 자동 슬라이드 동작 확인 (3초 간격)
  - [ ] 수동 스와이프 동작 확인
  - [ ] D-day 배지 색상 확인 (D-0 빨강, D-1 주황, D-2 노랑, D-3 파랑)
  - [ ] 기존 기능 정상 동작 확인 (카테고리, 인기 공연, 추천 공연)
  - [ ] ⏸️ iOS 테스트 — Apple Developer 계정 확보 후 일괄 검증

  ### 코드 품질
  - [ ] 린팅 에러 없음
  - [ ] 코딩 컨벤션 준수 (CLAUDE.md 참조)
  - [ ] 자체 코드 리뷰 완료

  ---

  ## 🧪 테스트 시나리오

  ### 시나리오 1: 마감 임박 핫딜 정상 표시
  ```
  전제:
  - API 응답에 deadlineDeals 3건 포함

  동작:
  1. 앱 실행, 홈 화면 진입

  예상 결과:
  - 홈 헤더 아래에 "🔥 마감 임박 핫딜" 섹션 헤더 표시
  - 카드 슬라이더에 첫 번째 카드 표시
  - 카드에 포스터 이미지, D-day 배지, 공연명, 장소, 할인율, 최저가, 잔여 티켓 수 표시
  - 페이지 인디케이터 3개 (첫 번째 활성)
  ```

  ### 시나리오 2: 자동 슬라이드
  ```
  전제:
  - deadlineDeals 3건

  동작:
  1. 홈 화면 진입 후 대기

  예상 결과:
  - 3초 후 두 번째 카드로 자동 전환
  - 6초 후 세 번째 카드로 자동 전환
  - 9초 후 첫 번째 카드로 순환
  ```

  ### 시나리오 3: 빈 상태 (0건)
  ```
  전제:
  - API 응답에 deadlineDeals: []

  동작:
  1. 홈 화면 진입

  예상 결과:
  - "🔥 마감 임박 핫딜" 섹션 헤더 미표시
  - 카드 슬라이더 미표시
  - HomeHeader 바로 아래에 HomeEventRow(카테고리) 표시
  ```

  ### 시나리오 4: 카드 탭 → 공연 상세
  ```
  동작:
  1. 홈 화면에서 핫딜 카드 탭

  예상 결과:
  - /ticketing/{eventId} 경로로 네비게이션
  - 해당 공연 상세 화면 표시
  ```

  ### 시나리오 5: D-day 배지 색상
  ```
  전제:
  - daysLeft=0, daysLeft=1, daysLeft=2, daysLeft=3 이벤트 각 1건

  동작:
  1. 각 카드의 D-day 배지 색상 확인

  예상 결과:
  - D-DAY: 빨간색(#EF4444)
  - D-1: 주황색(#F97316)
  - D-2: 노란색(#EAB308)
  - D-3: 파란색(#3B82F6)
  ```

  ### 시나리오 6: Pull-to-Refresh
  ```
  동작:
  1. 홈 화면에서 아래로 당기기 (Pull-to-Refresh)

  예상 결과:
  - 홈 데이터 새로고침
  - 마감 임박 핫딜 데이터도 갱신
  ```

  ---

  ## 🔗 의존성

  ### 선행 작업
  - [ ] Backend TASK-007: `GET /api/home`에 `deadlineDeals` 필드 추가 완료

  ### 필요한 패키지
  - `cached_network_image` (이미 설치됨 — PopularEventList에서 사용 중)

  ### 후속 작업
  - 없음

  ---

  ## ⏱️ 예상 소요 시간

  | 단계 | 시간 |
  |------|------|
  | DTO/Entity/UiModel 수정 | 1시간 |
  | DeadlineDealSlider 위젯 생성 | 3시간 |
  | home_view.dart 수정 (교체 + 삭제) | 1시간 |
  | code generation | 0.5시간 |
  | 테스트 | 1.5시간 |
  | 버그 수정 | 1시간 |
  | **총 예상 시간** | **8시간 (약 1일)** |

  ---

  ## 📅 일정

  - **시작일**: -
  - **목표 완료일**: -
  - **실제 완료일**: -

  ---

  ## 🚨 리스크 및 고려사항

  ### 기술적 리스크
  - **자동 슬라이드 타이머 메모리 누수**: `dispose()`에서 타이머 해제 필수 → StatefulWidget 또는 ConsumerStatefulWidget 사용
  - **Freezed code generation**: DTO/Entity에 `deadlineDeals` 필드 추가 후 `build_runner` 실행 필수

  ### 블로커
  - Backend TASK-007 완료 전까지 빈 배열 `[]` 응답으로 UI 선행 개발 가능 (0건 시 숨김 동작 확인)

  ---

  ## ⚠️ 주의사항

  ### UX
  - 자동 슬라이드는 3초 간격 (너무 빠르면 읽기 어려움)
  - D-DAY(오늘) 카드는 빨간색 배지로 긴급성 강조
  - 카드 탭 영역은 카드 전체 (이미지 + 텍스트 모두)

  ### Clean Architecture
  - Data → Domain → Presentation 의존성 방향 엄수
  - `DeadlineDealDto` → `DeadlineDealEntity` → `DeadlineDealUiModel` 변환 흐름
  - 기존 `HomeBannerCarousel` 파일은 삭제하지 않고 import만 제거

  ### 코딩 컨벤션 (CLAUDE.md)
  - 위젯 50줄 이상 시 분리 또는 `_buildXxx()` 헬퍼 메서드 사용
  - 하드코딩 금지: `AppColors`, `AppTextStyles`, `AppSpacing` 사용
  - `@freezed` + `fromJson()` + `toEntity()` 패턴 준수

  ---

  ## 📚 참고 자료

  - Backend TASK-007: `docs/backend/tasks/TASK-007_Home_Deadline_Deals.md`
  - 기존 HomeBannerCarousel — 교체 대상: `lib/features/home/presentation/widgets/home_banner_carousel.dart`
  - 기존 PopularEventList — 카드 레이아웃 패턴 참조: `lib/features/home/presentation/widgets/popular_event_list.dart`
  - 기존 HomeRespDto — DTO 구조 참조: `lib/features/home/data/dto/response/home_resp_dto.dart`
  - 기존 HomeEntity — Entity 구조 참조: `lib/features/home/domain/entities/home_entity.dart`
  - 기존 HomeUiModel — UiModel 구조 참조: `lib/features/home/presentation/ui_models/home_ui_model.dart`
  - Flutter 코딩 규칙: `ticket_platform_mobile/CLAUDE.md`

  ---

  **리뷰어**: Mobile Lead
  **리뷰 완료일**: -
  **상태**: ⏳ 대기

  </details>

---

- [ ] 2. Mobile UI 구현 — 배너 → 마감 임박 핫딜 카드 슬라이더

  **What to do**:
  - `home_resp_dto.dart`에 `DeadlineDealDto` 클래스 추가 (`@freezed` + `fromJson()` + `toEntity()`)
  - `home_entity.dart`에 `DeadlineDealEntity` 클래스 추가 (`@freezed`)
  - `home_ui_model.dart`에 `DeadlineDealUiModel` 클래스 추가 (`fromEntity()`)
  - `HomeRespDto`, `HomeEntity`, `HomeUiModel`에 각각 `deadlineDeals` 리스트 필드 추가
  - `home_deadline_deal_slider.dart` 신규 위젯 생성:
    - 높이 200px, PageView 기반 카드 슬라이더
    - 포스터 이미지 배경 + 하단 그라데이션 + D-day 배지(좌상단) + 공연명/장소/할인율/최저가/잔여 티켓
    - 자동 슬라이드 3초, Timer 기반, dispose 시 정리
    - 페이지 인디케이터 (하단 중앙)
    - 카드 탭 시 `/ticketing/{eventId}` 이동
  - `home_view.dart` 수정:
    - `HomeBannerCarousel` import/사용 제거
    - `_bannerController`, `_banners`, `_currentBanner` 관련 코드 제거
    - `homeState.when(data:)` 블록 안에서 `deadlineDeals`가 비어있지 않으면 섹션 헤더 + `HomeDeadlineDealSlider` 표시
    - 비어있으면 해당 영역 미표시 (기존 `HomeEventRow`가 바로 나옴)
  - `dart run build_runner build --delete-conflicting-outputs` 실행

  **Must NOT do**:
  - `home_banner_carousel.dart` 파일 삭제 금지 (import만 제거)
  - 기존 `PopularEventList`, `RecommendedEventList` 수정 금지
  - D-day 배지 외 불필요한 애니메이션 추가 금지
  - 하드코딩 색상/폰트 사용 금지 (`AppColors`, `AppTextStyles`, `AppSpacing` 사용)

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
    - Reason: UI 위젯 신규 생성, 카드 슬라이더 레이아웃, 색상/애니메이션 구현
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 2 (단독)
  - **Blocks**: F1
  - **Blocked By**: Task 1, Backend TASK-007 Task 2

  **References**:

  **Pattern References**:
  - `ticket_platform_mobile/lib/features/home/presentation/widgets/home_banner_carousel.dart` — 교체 대상 위젯 패턴 (138줄, PageView + PageController + 인디케이터). 이 구조를 참고하되 내용을 완전히 대체
  - `ticket_platform_mobile/lib/features/home/presentation/widgets/popular_event_list.dart` — 카드 UI 패턴 참조 (183줄, CachedNetworkImage + InkWell). 카드 레이아웃과 이미지 처리 방식 참고

  **API/Type References**:
  - `ticket_platform_mobile/lib/features/home/presentation/views/home_view.dart` — 배너 교체 위치 (155줄, 라인 74-80 HomeBannerCarousel 사용부). `_bannerController`, `_banners`, `_currentBanner` 코드 제거 대상
  - `ticket_platform_mobile/lib/features/home/data/dto/response/home_resp_dto.dart` — DTO 추가 위치 (97줄, @freezed 패턴). `DeadlineDealDto` 클래스와 `deadlineDeals` 필드 추가
  - `ticket_platform_mobile/lib/features/home/domain/entities/home_entity.dart` — Entity 추가 위치 (46줄, @freezed 패턴). `DeadlineDealEntity` 클래스 추가
  - `ticket_platform_mobile/lib/features/home/presentation/ui_models/home_ui_model.dart` — UiModel 추가 위치 (105줄, fromEntity 패턴). `DeadlineDealUiModel` 클래스 추가
  - `ticket_platform_mobile/lib/features/home/presentation/viewmodels/home_viewmodel.dart` — ViewModel (변경 불필요, HomeUiModel에 필드 추가만으로 자동 반영)
  - `ticket_platform_mobile/lib/features/home/presentation/widgets/home_common_widgets.dart` — `HomeSectionHeader` 재사용 (라인 78-103)

  **Style References**:
  - `ticket_platform_mobile/lib/core/theme/app_colors.dart` — 색상 상수
  - `ticket_platform_mobile/lib/core/theme/app_text_styles.dart` — 텍스트 스타일 상수
  - `ticket_platform_mobile/lib/core/theme/app_spacing.dart` — 간격 상수

  **Convention References**:
  - `ticket_platform_mobile/CLAUDE.md` — Flutter 코딩 규칙

  **Task Spec Reference**:
  - `docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md` — 상세 스펙 (Task 1에서 생성). 카드 UI 구성, D-day 배지 색상, 자동 슬라이드 규칙 포함

  **Acceptance Criteria**:
  - [ ] `flutter build apk --debug` 성공
  - [ ] 홈 화면에서 배너 대신 핫딜 카드 슬라이더 표시됨
  - [ ] 핫딜 0건 시 섹션 숨김됨
  - [ ] 카드 탭 시 공연 상세로 이동
  - [ ] 기존 카테고리/인기공연/추천공연 정상 동작

  **QA Scenarios:**
  ```
  Scenario: Flutter 빌드 성공 확인
    Tool: Bash
    Steps:
      1. flutter build apk --debug (workdir: ticket_platform_mobile)
    Expected Result: BUILD SUCCESSFUL, APK 생성
    Failure Indicators: BUILD FAILED, Dart 컴파일 에러
    Evidence: .sisyphus/evidence/task-2-build.txt

  Scenario: code generation 성공 확인
    Tool: Bash
    Steps:
      1. dart run build_runner build --delete-conflicting-outputs (workdir: ticket_platform_mobile)
    Expected Result: Succeeded after X.Xs, 0 error(s)
    Failure Indicators: Could not generate, error
    Evidence: .sisyphus/evidence/task-2-codegen.txt

  Scenario: 배너 import 제거 확인
    Tool: Bash
    Steps:
      1. grep -c 'home_banner_carousel' ticket_platform_mobile/lib/features/home/presentation/views/home_view.dart
    Expected Result: 0 (import 및 사용 코드 모두 제거됨)
    Failure Indicators: 1 이상 (배너 참조 잔존)
    Evidence: .sisyphus/evidence/task-2-banner-removed.txt

  Scenario: 신규 슬라이더 파일 존재 확인
    Tool: Bash
    Steps:
      1. test -f ticket_platform_mobile/lib/features/home/presentation/widgets/home_deadline_deal_slider.dart && echo "EXISTS"
    Expected Result: EXISTS
    Failure Indicators: 파일 미존재
    Evidence: .sisyphus/evidence/task-2-slider-exists.txt

  Scenario: 배너 파일 삭제되지 않았는지 확인
    Tool: Bash
    Steps:
      1. test -f ticket_platform_mobile/lib/features/home/presentation/widgets/home_banner_carousel.dart && echo "PRESERVED"
    Expected Result: PRESERVED (파일 유지됨)
    Failure Indicators: 파일 삭제됨
    Evidence: .sisyphus/evidence/task-2-banner-preserved.txt
  ```

  **Commit**: YES
  - Message: `feat(home): replace banner with deadline deals slider`
  - Files: `ticket_platform_mobile/lib/features/home/presentation/widgets/home_deadline_deal_slider.dart`, `ticket_platform_mobile/lib/features/home/presentation/views/home_view.dart`, `ticket_platform_mobile/lib/features/home/data/dto/response/home_resp_dto.dart`, `ticket_platform_mobile/lib/features/home/domain/entities/home_entity.dart`, `ticket_platform_mobile/lib/features/home/presentation/ui_models/home_ui_model.dart`, generated files (`.g.dart`, `.freezed.dart`)
  - Pre-commit: `flutter build apk --debug`

---

## Final Verification Wave

- [ ] F1. **모바일 통합 검증** — `deep`
  모바일 빌드(`flutter build apk --debug`) 성공 확인. 기존 기능(카테고리, 인기공연, 추천공연) 정상 동작 확인. 신규 파일이 기존 코드 컨벤션(CLAUDE.md) 준수하는지 점검. `home_banner_carousel.dart` 파일이 삭제되지 않았는지 확인. `docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md` 파일 존재 확인. `home_view.dart`에서 `home_banner_carousel` import가 제거되었는지 확인.

---

## Commit Strategy

- **Task 1**: `docs: add TASK-007 Home Deadline Deals mobile spec` — docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md
- **Task 2**: `feat(home): replace banner with deadline deals slider` — home_deadline_deal_slider.dart, home_view.dart, home_resp_dto.dart, home_entity.dart, home_ui_model.dart, generated files

---

## Success Criteria

### Verification Commands
```bash
# 모바일 빌드
flutter build apk --debug  # Expected: BUILD SUCCESSFUL

# code generation
dart run build_runner build --delete-conflicting-outputs  # Expected: Succeeded

# 배너 import 제거 확인
grep -c 'home_banner_carousel' ticket_platform_mobile/lib/features/home/presentation/views/home_view.dart  # Expected: 0

# 문서 존재 확인
test -f docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md && echo "OK"

# 배너 파일 유지 확인
test -f ticket_platform_mobile/lib/features/home/presentation/widgets/home_banner_carousel.dart && echo "PRESERVED"
```

### Final Checklist
- [ ] 배너 대신 핫딜 카드 슬라이더 표시
- [ ] 0건 시 섹션 숨김
- [ ] 카드 탭 → 공연 상세 이동
- [ ] 기존 기능 정상 동작
- [ ] TASK-007 모바일 문서 작성 완료
- [ ] `flutter build apk --debug` 성공
- [ ] `home_banner_carousel.dart` 삭제되지 않음
