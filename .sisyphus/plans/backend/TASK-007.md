# TASK-007: 홈 화면 마감 임박 핫딜 API

## TL;DR

> **Quick Summary**: `GET /api/home` 응답에 `deadlineDeals` 필드를 추가한다. 공연일 D-3 이내 + 판매 중 티켓이 있는 이벤트를 할인율 내림차순으로 최대 10건 반환. 기존 banners/popularEvents/recommendedEvents 필드는 변경 없이 유지.
> 
> **Deliverables**:
> - `docs/backend/tasks/TASK-007_Home_Deadline_Deals.md` — 백엔드 Agent 작업 지시서
> - `TicketPlatFormServer/.../DTO/Home/DeadlineDealDto.cs` — 신규 DTO
> - `TicketPlatFormServer/.../Repository/Home/HomeQueries.cs` — SQL 쿼리 추가
> - `TicketPlatFormServer/.../Repository/Home/HomeRepository.cs` — Repository 구현
> - `TicketPlatFormServer/.../Repository/Home/IHomeRepository.cs` — 인터페이스 추가
> - `TicketPlatFormServer/.../DTO/Home/HomeRespDto.cs` — DeadlineDeals 필드 추가
> - `TicketPlatFormServer/.../Services/Home/HomeService.cs` — GetDeadlineDeals 호출 추가
> 
> **Estimated Effort**: Short (0.5일)
> **Parallel Execution**: YES - 2 waves
> **Critical Path**: Task 1 (문서) → Task 2 (API 구현) → F1 (검증)

---

## Context

### Original Request
홈 화면 상단의 광고 배너 슬라이더가 중고 티켓 거래 플랫폼에 불필요하므로, "마감 임박 핫딜" 데이터를 제공하는 API를 구현한다.

### Interview Summary
**Key Discussions**:
- 대체 콘텐츠: **마감 임박 핫딜** (공연 D-3 이내 + 판매 중 티켓, 할인율 강조)
- 마감 기준: **D-3 이내** (오늘 포함 ~ 3일 후)
- API 설계: **기존 /api/home 확장** (deadlineDeals 필드 추가, 별도 엔드포인트 불필요)
- 빈 상태: 0건이면 빈 배열 `[]` 반환

### Research Findings
- 배너 데이터는 서버에서 `GetBanners()` 쿼리로 조회하나, 모바일에서 하드코딩된 더미 데이터만 사용 중
- `PopularEvents` 쿼리 패턴(Dapper + 복합 점수 정렬)이 `DeadlineDeals` 쿼리의 기반이 됨
- IDbConnection은 scoped — 리포지토리 호출은 반드시 순차적으로

### 관련 Mobile 계획
- `.sisyphus/plans/mobile/TASK-007.md` — 모바일 UI 구현 (이 계획의 Task 2 완료 후 실행 가능)

---

## Work Objectives

### Core Objective
기존 `GET /api/home` 응답에 `deadlineDeals` 필드를 추가하여, 공연일 D-3 이내 마감 임박 핫딜 데이터를 제공한다.

### Concrete Deliverables
- `docs/backend/tasks/TASK-007_Home_Deadline_Deals.md` — 백엔드 작업 지시서
- `TicketPlatFormServer/.../DTO/Home/DeadlineDealDto.cs` — 신규 DTO
- `TicketPlatFormServer/.../Repository/Home/HomeQueries.cs` — SQL 쿼리 추가
- `TicketPlatFormServer/.../Repository/Home/HomeRepository.cs` — Repository 구현
- `TicketPlatFormServer/.../Repository/Home/IHomeRepository.cs` — 인터페이스 추가
- `TicketPlatFormServer/.../DTO/Home/HomeRespDto.cs` — DeadlineDeals 필드 추가
- `TicketPlatFormServer/.../Services/Home/HomeService.cs` — GetDeadlineDeals 호출 추가

### Definition of Done
- [ ] `GET /api/home` 응답에 `deadlineDeals` 배열이 포함됨
- [ ] D-3 이내 + 판매 중 티켓이 있는 이벤트만 필터링됨
- [ ] 할인율(ticketDiscountRate) 내림차순 정렬
- [ ] 최대 10건 반환, 0건이면 빈 배열 `[]`
- [ ] 각 항목에 남은 일수(`daysLeft`) 필드 포함 (0~3 범위)
- [ ] 기존 `popularEvents`, `recommendedEvents`, `banners` 필드는 변경 없이 유지
- [ ] `dotnet build` 성공

### Must Have
- D-3 기준 필터링 (오늘 ≤ start_at ≤ 오늘+3일)
- 할인율 내림차순 정렬
- 남은 일수(daysLeft) 표시 (D-0=0, D-1=1, D-2=2, D-3=3)
- 빈 상태 시 빈 배열 반환
- 기존 banners 필드 하위 호환 유지

### Must NOT Have (Guardrails)
- banners 필드 삭제
- 별도 API 엔드포인트 신설 금지 (기존 /api/home 확장만)
- Controller 변경 불필요 (반환 타입 HomeRespDto 동일)
- 기존 쿼리/필드 수정 금지
- 리포지토리 병렬 호출 금지 (IDbConnection scoped)

---

## Verification Strategy

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed.

### Test Decision
- **Infrastructure exists**: NO (테스트 프로젝트 미존재)
- **Automated tests**: None
- **Framework**: N/A

### QA Policy
- **Backend**: Bash (curl) — API 호출 후 응답 필드 검증
- **Build**: `dotnet build` 성공 확인

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Start Immediately — 문서 작성):
└── Task 1: Backend TASK-007 문서 작성 [quick]

Wave 2 (After Wave 1 — 백엔드 구현):
└── Task 2: Backend API 구현 (DeadlineDealDto + SQL + Service) [unspecified-high]

Wave FINAL (After ALL tasks):
└── Task F1: 백엔드 통합 검증 [deep]

Critical Path: Task 1 → Task 2 → F1
```

### Dependency Matrix

| Task | Depends On | Blocks | Wave |
|------|-----------|--------|------|
| 1 | — | 2 | 1 |
| 2 | 1 | F1 | 2 |
| F1 | 2 | — | FINAL |

### Agent Dispatch Summary

- **Wave 1**: 1 task — T1 → `quick`
- **Wave 2**: 1 task — T2 → `unspecified-high`
- **FINAL**: 1 task — F1 → `deep`

---

## TODOs

> Implementation + verification = ONE Task.
> EVERY task MUST have: Recommended Agent Profile + Parallelization info + QA Scenarios.

- [ ] 1. Backend TASK-007 문서 작성

  **What to do**:
  - `docs/backend/tasks/TASK-007_Home_Deadline_Deals.md` 파일을 생성하라
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
  - `docs/backend/tasks/TASK-006_Dispute_System.md` — 문서 형식/톤 참조 (534줄)

  **API/Type References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Home/HomeQueries.cs` — SQL 쿼리 패턴 참조
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Home/HomeRespDto.cs` — 현재 응답 구조
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Home/PopularEventDto.cs` — DTO 필드 참조
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Home/HomeService.cs` — 현재 서비스 로직
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Home/IHomeRepository.cs` — 현재 Repository 인터페이스

  **Acceptance Criteria**:
  - [ ] `docs/backend/tasks/TASK-007_Home_Deadline_Deals.md` 파일 생성됨
  - [ ] 아래 '문서 전문'의 전체 내용이 빠짐없이 포함됨
  - [ ] 기존 TASK-006 문서와 동일한 마크다운 형식

  **QA Scenarios:**
  ```
  Scenario: Backend TASK-007 문서 생성 확인
    Tool: Bash
    Steps:
      1. test -f docs/backend/tasks/TASK-007_Home_Deadline_Deals.md && echo "EXISTS"
      2. head -1 docs/backend/tasks/TASK-007_Home_Deadline_Deals.md
      3. grep -c 'deadlineDeals' docs/backend/tasks/TASK-007_Home_Deadline_Deals.md
    Expected Result: 파일 존재, 첫 줄 "# TASK-007: 홈 화면 마감 임박 핫딜 API", deadlineDeals 키워드 10회 이상
    Failure Indicators: 파일 미존재, 키워드 누락
    Evidence: .sisyphus/evidence/task-1-backend-doc.txt
  ```

  **Commit**: YES
  - Message: `docs: add TASK-007 Home Deadline Deals backend spec`
  - Files: `docs/backend/tasks/TASK-007_Home_Deadline_Deals.md`

  <details>
  <summary>📄 문서 전문 (클릭하여 펼치기)</summary>

  # TASK-007: 홈 화면 마감 임박 핫딜 API

  **작성일**: 2026-02-23
  **작성자**: PM
  **담당 팀**: Backend
  **담당자**: Backend Agent
  **상태**:  완료
  **우선순위**: 🟡 Medium

  ---

  ## 📋 작업 개요

  ### 작업 설명
  홈 화면의 기존 광고 배너 영역을 대체할 **마감 임박 핫딜** 데이터를 제공하라. 공연일 기준 D-3 이내이면서 판매 중인 티켓이 있는 이벤트를 조회하여 `GET /api/home` 응답에 `deadlineDeals` 필드를 추가하라.

  ### 목표
  - 기존 `GET /api/home` 응답에 `deadlineDeals` 필드 추가
  - 공연일 D-3 이내 + 판매 중 티켓 보유 이벤트만 필터링
  - 할인율 높은 순 정렬, 최대 10건 반환
  - 마감 임박 이벤트가 0건이면 빈 배열(`[]`) 반환

  ### 배경
  현재 홈 화면 상단에 광고 배너 슬라이더가 있으나, 중고 티켓 거래 플랫폼 특성상 광고 배너는 불필요하다. 대신 **공연일이 임박하여 급처분이 필요한 티켓**을 강조 표시하면 구매 전환율을 높일 수 있다. 중고 거래의 핵심 심리(급처분 = 큰 할인)를 활용하는 전략이다.

  ---

  ## 🎯 완료 기준 (Acceptance Criteria)

  - [ ] `GET /api/home` 응답에 `deadlineDeals` 필드가 포함됨
  - [ ] `deadlineDeals`는 공연일 D-3 이내 + 판매 중 티켓 보유 이벤트만 포함
  - [ ] 할인율(TicketDiscountRate) 내림차순 정렬
  - [ ] 최대 10건 반환, 0건이면 빈 배열 `[]`
  - [ ] 각 항목에 남은 일수(`daysLeft`) 필드 포함 (0=오늘, 1=내일, 2=모레, 3=3일 후)
  - [ ] 기존 `popularEvents`, `recommendedEvents` 필드는 변경 없이 유지
  - [ ] 기존 `banners` 필드는 유지하되 모바일에서 미사용 (하위 호환)
  - [ ] 에러 없이 빌드 및 Swagger 문서에 반영

  ---

  ## 🔧 기술 스펙 (Backend)

  ### 마감 임박 기준

  | 조건 | 설명 |
  |------|------|
  | 공연일 범위 | `오늘 ≤ start_at ≤ 오늘 + 3일` |
  | 티켓 상태 | `status_id = 1` (판매 중) AND `deleted_at IS NULL` |
  | 이벤트 상태 | `is_active = 1` |
  | 판매 가능 티켓 | `remaining_quantity > 0` 인 티켓이 1건 이상 |

  ### API 응답 변경

  #### GET /api/home — 응답에 `deadlineDeals` 추가

  **변경된 응답 (200 OK)**
  ```json
  {
    "message": "홈 화면 데이터 조회 성공",
    "data": {
      "banners": [...],
      "categories": [...],
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
          "posterImageUrl": "https://storage.example.com/posters/bts.jpg",
          "availableTicketCount": 12,
          "categoryId": 1
        }
      ],
      "popularEvents": [...],
      "recommendedEvents": [...]
    },
    "statusCode": 200
  }
  ```

  ### DeadlineDealDto 필드 정의

  | 필드 | 타입 | 설명 |
  |------|------|------|
  | `eventId` | int | 공연 ID |
  | `eventTitle` | string | 공연 제목 |
  | `eventDate` | string | 공연 날짜 (형식: "2026.02.25") |
  | `venue` | string | 공연 장소 |
  | `daysLeft` | int | 남은 일수 (0=오늘, 1=내일, 2=모레, 3=3일후) |
  | `minTicketPrice` | int | 최저 판매가 (원) |
  | `originalMinTicketPrice` | int | 최저가 티켓의 원가 (원) |
  | `ticketDiscountRate` | int | 할인율 (%). UI에서 "-35%" 표시 |
  | `posterImageUrl` | string? | 포스터 이미지 URL |
  | `availableTicketCount` | int | 판매 가능 티켓 수량 |
  | `categoryId` | int | 카테고리 ID |

  ### 정렬 규칙

  1차: `ticketDiscountRate` 내림차순 (할인율 높은 것 먼저)
  2차: `daysLeft` 오름차순 (가장 임박한 것 먼저)
  3차: `availableTicketCount` 내림차순 (선택지 많은 것 먼저)

  ---

  ## 📂 파일 구조

  ### 수정 대상
  ```
  TicketPlatFormServer/
  ├── Services/Home/
  │   ├── IHomeService.cs              — 변경 없음 (반환 타입 HomeRespDto 동일)
  │   └── HomeService.cs               — GetDeadlineDeals 호출 추가
  ├── Repository/Home/
  │   ├── IHomeRepository.cs           — GetDeadlineDeals() 메서드 추가
  │   ├── HomeRepository.cs            — GetDeadlineDeals() 구현
  │   └── HomeQueries.cs               — GetDeadlineDeals SQL 추가
  └── DTO/Home/
      ├── HomeRespDto.cs               — DeadlineDeals 필드 추가
      └── DeadlineDealDto.cs           — 신규 DTO 생성
  ```

  ---

  ## ✅ 작업 체크리스트

  ### 개발
  - [ ] `DeadlineDealDto.cs` 생성 (위 필드 정의 참고)
  - [ ] `HomeRespDto.cs`에 `DeadlineDeals` 프로퍼티 추가
  - [ ] `IHomeRepository.cs`에 `GetDeadlineDeals(int limit = 10)` 메서드 추가
  - [ ] `HomeQueries.cs`에 마감 임박 조회 SQL 추가
  - [ ] `HomeRepository.cs`에 `GetDeadlineDeals` 구현
  - [ ] `HomeService.cs`에서 `GetDeadlineDeals` 호출 및 응답에 포함

  ### 테스트
  - [ ] 마감 임박 이벤트가 있을 때 정상 반환 확인
  - [ ] 마감 임박 이벤트가 0건일 때 빈 배열 반환 확인
  - [ ] 할인율 정렬 확인
  - [ ] daysLeft 계산 정확성 확인 (오늘=0, 내일=1)
  - [ ] 기존 popularEvents, recommendedEvents 필드 영향 없음 확인
  - [ ] Swagger 문서 정상 표시 확인

  ### 코드 품질
  - [ ] 린팅 에러 없음
  - [ ] 코딩 컨벤션 준수 (AGENTS.md 참조)
  - [ ] 자체 코드 리뷰 완료

  ---

  ## 🧪 테스트 시나리오

  ### 시나리오 1: 마감 임박 이벤트 정상 조회
  ```
  전제:
  - 공연일이 오늘~3일 이내인 이벤트 3건 존재
  - 각 이벤트에 판매 중(status_id=1) 티켓 존재

  입력:
  - GET /api/home

  예상 결과:
  - 200 OK
  - deadlineDeals 배열에 3건 포함
  - 각 항목에 daysLeft 값이 0~3 범위
  - ticketDiscountRate 내림차순 정렬
  - 기존 popularEvents, recommendedEvents 정상 포함
  ```

  ### 시나리오 2: 마감 임박 이벤트 0건
  ```
  전제:
  - 공연일이 3일 이내인 이벤트가 없음

  입력:
  - GET /api/home

  예상 결과:
  - 200 OK
  - deadlineDeals: [] (빈 배열)
  - 기존 필드 정상 반환
  ```

  ### 시나리오 3: 오늘 공연인 이벤트
  ```
  전제:
  - 공연일이 오늘(start_at = 오늘 날짜)인 이벤트 존재

  입력:
  - GET /api/home

  예상 결과:
  - deadlineDeals에 해당 이벤트 포함
  - daysLeft = 0
  ```

  ### 시나리오 4: D-4 이상 이벤트 제외 확인
  ```
  전제:
  - 공연일이 4일 이후인 이벤트만 존재

  입력:
  - GET /api/home

  예상 결과:
  - deadlineDeals: [] (빈 배열)
  ```

  ### 시나리오 5: 판매 완료 티켓만 있는 이벤트 제외
  ```
  전제:
  - 공연일이 D-2인 이벤트 존재
  - 해당 이벤트의 모든 티켓이 판매 완료(remaining_quantity = 0)

  입력:
  - GET /api/home

  예상 결과:
  - deadlineDeals에 해당 이벤트 미포함
  ```

  ### 시나리오 6: 최대 10건 제한
  ```
  전제:
  - 공연일 D-3 이내 이벤트가 15건 존재

  입력:
  - GET /api/home

  예상 결과:
  - deadlineDeals 배열 최대 10건
  ```

  ---

  ## 🔗 의존성

  ### 선행 작업
  - 없음 (기존 DB 테이블 및 HomeService 활용, 즉시 구현 가능)

  ### 관련 테이블
  - `events` — 공연 정보 (start_at으로 D-day 계산)
  - `tickets` — 티켓 정보 (판매 상태, 가격, 잔여 수량)
  - `event_seat_grades` — 좌석 등급별 원가 (original_price)

  ### 후속 작업
  - [ ] Mobile TASK-007: 홈 화면 마감 임박 핫딜 UI 구현

  ---

  ## ⏱️ 예상 소요 시간

  | 항목 | 시간 |
  |------|------|
  | DeadlineDealDto 생성 | 0.5시간 |
  | HomeRespDto 수정 | 0.5시간 |
  | SQL 쿼리 작성 (HomeQueries) | 1시간 |
  | Repository 구현 | 0.5시간 |
  | Service 수정 | 0.5시간 |
  | 테스트 | 1시간 |
  | **총 예상 시간** | **4시간 (약 0.5일)** |

  ---

  ## 🚨 리스크 및 고려사항

  ### 기술적 리스크
  - **타임존 처리**: `daysLeft` 계산 시 서버 시간(UTC/KST) 기준이 명확해야 함 → MySQL `CURDATE()` 사용 시 DB 타임존 설정 확인
  - **성능**: D-3 필터링은 `events.start_at` 인덱스 활용 가능 → 없으면 추가 권장

  ### 하위 호환
  - 기존 `banners` 필드는 제거하지 않고 유지 (구버전 앱 호환)
  - `deadlineDeals`는 새 필드이므로 구버전 앱에서는 무시됨 (안전)

  ---

  ## 📝 구현 노트

  ### SQL 쿼리 가이드 (HomeQueries.GetDeadlineDeals)

  기존 `GetPopularEvents` 쿼리 패턴을 따르되, D-3 필터와 `daysLeft` 계산을 추가하라.

  ```sql
  SELECT
      e.id AS EventId,
      e.title AS EventTitle,
      DATE_FORMAT(e.start_at, '%Y.%m.%d') AS EventDate,
      e.venue_name AS Venue,
      DATEDIFF(DATE(e.start_at), CURDATE()) AS DaysLeft,
      MIN(t.price) AS MinTicketPrice,
      MIN(COALESCE(esg.original_price, t.price)) AS OriginalMinTicketPrice,
      MAX(CASE
          WHEN COALESCE(esg.original_price, t.price) > 0
          THEN ROUND((COALESCE(esg.original_price, t.price) - t.price) / COALESCE(esg.original_price, t.price) * 100)
          ELSE 0
      END) AS TicketDiscountRate,
      e.poster_image_url AS PosterImageUrl,
      COUNT(t.id) AS AvailableTicketCount,
      e.category_id AS CategoryId
  FROM events e
  INNER JOIN tickets t ON e.id = t.event_id
  LEFT JOIN event_seat_grades esg ON t.seat_grade_id = esg.id
  WHERE e.is_active = 1
      AND t.status_id = 1
      AND t.deleted_at IS NULL
      AND t.remaining_quantity > 0
      AND DATE(e.start_at) >= CURDATE()
      AND DATE(e.start_at) <= DATE_ADD(CURDATE(), INTERVAL 3 DAY)
  GROUP BY e.id, e.title, e.start_at, e.venue_name, e.poster_image_url, e.category_id
  HAVING AvailableTicketCount > 0
  ORDER BY TicketDiscountRate DESC, DaysLeft ASC, AvailableTicketCount DESC
  LIMIT @Limit
  ```

  ---

  ## 📚 참고 자료

  - 기존 HomeRepository: `TicketPlatFormServer/Repository/Home/HomeQueries.cs` (GetPopularEvents 패턴 참조)
  - 기존 HomeRespDto: `TicketPlatFormServer/DTO/Home/HomeRespDto.cs`
  - 기존 PopularEventDto: `TicketPlatFormServer/DTO/Home/PopularEventDto.cs` (필드 구조 참조)
  - Mobile TASK-007: `docs/mobile/tasks/TASK-007_Home_Deadline_Deals.md`

  ---

  **리뷰어**: Backend Lead
  **상태**: ⏳ 대기

  </details>

---

- [ ] 2. Backend API 구현 — `GET /api/home`에 `deadlineDeals` 추가

  **What to do**:
  - `DeadlineDealDto.cs` 신규 생성 (필드: EventId, EventTitle, EventDate, Venue, DaysLeft, MinTicketPrice, OriginalMinTicketPrice, TicketDiscountRate, PosterImageUrl, AvailableTicketCount, CategoryId)
  - `HomeRespDto.cs`에 `DeadlineDeals` 프로퍼티 추가
  - `IHomeRepository.cs`에 `GetDeadlineDeals(int limit = 10)` 메서드 추가
  - `HomeQueries.cs`에 D-3 필터 SQL 추가 (기존 `GetPopularEvents` 쿼리 패턴 따름)
  - `HomeRepository.cs`에 `GetDeadlineDeals` 구현
  - `HomeService.cs`에서 `GetDeadlineDeals` 호출 후 응답에 포함
  - 기존 `banners`, `categories`, `popularEvents`, `recommendedEvents` 필드는 변경 없이 유지

  **Must NOT do**:
  - `banners` 필드 삭제 금지 (하위 호환)
  - 별도 API 엔드포인트 신설 금지
  - 기존 쿼리/필드 수정 금지
  - Controller 변경 불필요 (반환 타입 HomeRespDto 동일)
  - 리포지토리 병렬 호출 금지 (IDbConnection scoped — `Task.WhenAll` 사용 금지)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Dapper SQL 쿼리 작성 + 다수 파일 수정 (DTO, Repository, Service)
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 2 (단독)
  - **Blocks**: F1
  - **Blocked By**: Task 1

  **References**:

  **Pattern References**:
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Home/HomeQueries.cs` — `GetPopularEvents` SQL 패턴 (268줄, 라인 39-75). 이 패턴을 따라 `GetDeadlineDeals` 쿼리 작성
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Home/HomeRepository.cs` — Repository 구현 패턴 (67줄, 라인 32-40). Dapper `QueryAsync<T>` 호출 방식 참조

  **API/Type References**:
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Home/HomeRespDto.cs` — `DeadlineDeals` 필드 추가 위치 (28줄)
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/Home/PopularEventDto.cs` — DTO 필드 패턴 참조 (62줄)
  - `TicketPlatFormServer/TicketPlatFormServer/Repository/Home/IHomeRepository.cs` — 인터페이스 추가 위치 (30줄)
  - `TicketPlatFormServer/TicketPlatFormServer/Services/Home/HomeService.cs` — `GetDeadlineDeals` 호출 추가 위치 (35줄, 라인 18-33)
  - `TicketPlatFormServer/TicketPlatFormServer/Controllers/HomeController.cs` — 변경 불필요 확인 (37줄)

  **Task Spec Reference**:
  - `docs/backend/tasks/TASK-007_Home_Deadline_Deals.md` — 상세 스펙 (Task 1에서 생성). SQL 쿼리 가이드, 필드 정의, 테스트 시나리오 포함

  **Acceptance Criteria**:
  - [ ] `dotnet build` 성공
  - [ ] `curl http://localhost:5224/api/home` 응답에 `deadlineDeals` 배열 존재
  - [ ] `deadlineDeals` 각 항목에 `daysLeft` 필드 포함 (0~3 범위)
  - [ ] `ticketDiscountRate` 내림차순 정렬됨
  - [ ] 기존 `popularEvents`, `recommendedEvents` 필드 변경 없음

  **QA Scenarios:**
  ```
  Scenario: API 응답에 deadlineDeals 필드 존재 확인
    Tool: Bash (curl)
    Preconditions: dotnet run --project TicketPlatFormServer 실행 중
    Steps:
      1. curl -s http://localhost:5224/api/home | jq '.data | keys'
      2. 응답 키에 "deadlineDeals" 포함 확인
      3. curl -s http://localhost:5224/api/home | jq '.data.deadlineDeals | length'
      4. curl -s http://localhost:5224/api/home | jq '.data.popularEvents | length'
    Expected Result: deadlineDeals 키 존재, 배열 (빈 배열 또는 데이터), popularEvents 기존 동작 유지
    Failure Indicators: deadlineDeals 키 없음, 500 에러, popularEvents 누락
    Evidence: .sisyphus/evidence/task-2-api-response.json

  Scenario: 빌드 성공 확인
    Tool: Bash
    Steps:
      1. dotnet build TicketPlatFormServer
    Expected Result: Build succeeded, 0 Warning(s), 0 Error(s)
    Failure Indicators: Build FAILED, error CS 에러
    Evidence: .sisyphus/evidence/task-2-build.txt
  ```

  **Commit**: YES
  - Message: `feat(home): add deadline deals to home API response`
  - Files: `TicketPlatFormServer/TicketPlatFormServer/DTO/Home/DeadlineDealDto.cs`, `TicketPlatFormServer/TicketPlatFormServer/DTO/Home/HomeRespDto.cs`, `TicketPlatFormServer/TicketPlatFormServer/Repository/Home/HomeQueries.cs`, `TicketPlatFormServer/TicketPlatFormServer/Repository/Home/HomeRepository.cs`, `TicketPlatFormServer/TicketPlatFormServer/Repository/Home/IHomeRepository.cs`, `TicketPlatFormServer/TicketPlatFormServer/Services/Home/HomeService.cs`
  - Pre-commit: `dotnet build`

---

## Final Verification Wave

- [ ] F1. **백엔드 통합 검증** — `deep`
  서버 실행 상태에서 `curl GET /api/home` 호출하여 `deadlineDeals` 필드 존재 확인. `dotnet build` 성공 확인. 기존 `popularEvents`, `recommendedEvents` 필드 변경 없음 확인. 신규 파일이 기존 코드 컨벤션(AGENTS.md) 준수하는지 점검. `docs/backend/tasks/TASK-007_Home_Deadline_Deals.md` 파일 존재 확인.

---

## Commit Strategy

- **Task 1**: `docs: add TASK-007 Home Deadline Deals backend spec` — docs/backend/tasks/TASK-007_Home_Deadline_Deals.md
- **Task 2**: `feat(home): add deadline deals to home API response` — DeadlineDealDto.cs, HomeRespDto.cs, HomeQueries.cs, HomeRepository.cs, IHomeRepository.cs, HomeService.cs

---

## Success Criteria

### Verification Commands
```bash
# 백엔드 빌드
dotnet build  # Expected: Build succeeded

# API 응답 확인
curl -s http://localhost:5224/api/home | jq '.data.deadlineDeals'  # Expected: array (possibly empty)

# 문서 존재 확인
test -f docs/backend/tasks/TASK-007_Home_Deadline_Deals.md && echo "OK"
```

### Final Checklist
- [ ] `deadlineDeals` 필드가 API 응답에 존재
- [ ] 할인율 내림차순 정렬
- [ ] D-3 이내 필터링 동작
- [ ] 기존 기능 정상 동작
- [ ] TASK-007 백엔드 문서 작성 완료
- [ ] `dotnet build` 성공
