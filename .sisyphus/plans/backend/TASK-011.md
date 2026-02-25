# TASK-011 결제 정보 가독성 개선 — Backend 계획

## TL;DR

> **요약**: `/api/payment/request` 응답을 확장해 결제 화면에서 문자열 파싱 없이
> "구매할 티켓 정보"와 "공연 정보"를 구조화된 데이터로 표시할 수 있게 한다.
>
> **핵심 방향**:
> - 클라이언트 조합 문자열(`orderName`) 의존 제거
> - 서버 권위 데이터로 결제 미리보기 메타데이터 제공
> - 기존 결제 위젯 필드(`orderId`, `amount`, `clientKey` 등) 하위 호환 유지
>
> **산출물**:
> - `PaymentRequestResponseDto` 확장 (Ticket/Event 메타데이터)
> - Transaction 기반 결제 미리보기 조회 메서드 추가
> - `PaymentService.InitiatePaymentAsync()` 응답 매핑 강화
> - API 스펙 문서 추가 (`api_spec/TASK-011_PAYMENT_REQUEST_CONTRACT.md`)

---

## Context

### 작업 기준 디렉토리
모든 소스 파일은 `TicketPlatFormServer/TicketPlatFormServer/` 하위 기준.

### 현재 문제점
- 현재 `PaymentRequestResponseDto`는 `orderName` 중심 단일 문자열 표현이며, 티켓/공연 정보를 분리 전달하지 않음.
  - `DTO/Payment/PaymentRequestDto.cs`
- `PaymentService.InitiatePaymentAsync()`는 OrderId 생성 및 금액 검증 위주로 동작하며, 결제 화면용 구조화 메타데이터를 생성하지 않음.
  - `Services/Payment/PaymentService.cs`
- 모바일은 결제 진입 전 채팅 데이터로 `orderName`을 조합해 UI에 노출하고 있어 서버/클라이언트 정보 기준이 분리됨.

### 목표 UX 연결점
모바일 결제 화면에서 다음 2개 섹션을 명확히 분리 노출해야 함:
1. 구매할 티켓 정보 (좌석/매수/금액)
2. 공연 정보 (공연명/일시/장소)

---

## Work Objectives

### 핵심 목표
`POST /api/payment/request` 응답을 결제 미리보기 목적에 맞게 확장해,
프론트가 별도 추론/문자열 파싱 없이 바로 화면 구성할 수 있도록 한다.

### Must Have
- `PaymentRequestResponseDto`에 구조화 메타데이터 추가:
  - `ticketInfo`: ticketId, seatInfo, quantity, unitPrice, totalAmount, thumbnailUrl
  - `eventInfo`: eventId, title, eventDateTime, venueName
- `transactionId` 기준 + `userId` 권한 검증 하에 조회된 실제 DB값으로 응답 구성
- 기존 필드(`orderId`, `amount`, `orderName`, `successUrl`, `failUrl`, `clientKey`) 유지
- 데이터 누락 시 null-safe 처리(서버 예외 대신 안전한 기본값/nullable 반환)
- API 스펙 문서화 및 샘플 응답 제공

### Must NOT Have
- 클라이언트 전달 `orderName` 문자열 역파싱 금지
- 권한 없는 사용자의 거래 메타데이터 노출 금지
- Repository 병렬 호출 금지 (`IDbConnection` scoped 규칙 준수)
- 결제 승인(`confirm`) 로직 변경 금지 (이번 범위는 request 응답 개선)

---

## 계약 초안 (Draft)

```json
{
  "orderId": "TXN_123_...",
  "amount": 360000,
  "orderName": "Bunnies Camp 2024 티켓 결제",
  "customerName": "홍길동",
  "customerEmail": "buyer@test.com",
  "successUrl": "...",
  "failUrl": "...",
  "clientKey": "...",
  "ticketInfo": {
    "ticketId": 871,
    "thumbnailUrl": "https://...",
    "seatInfo": "1층 R석 B구역 119열",
    "quantity": 1,
    "unitPrice": 360000,
    "totalAmount": 360000
  },
  "eventInfo": {
    "eventId": 91,
    "title": "Bunnies Camp 2024",
    "eventDateTime": "2026-02-26T18:00:00Z",
    "venueName": "고척스카이돔"
  }
}
```

---

## Execution Strategy

### 병렬 실행 Wave

```
Wave 1 (계약/조회 기반 준비):
├── Task 1: PaymentRequestResponseDto 계약 확장
└── Task 2: Transaction 결제 미리보기 조회 메서드 추가

Wave 2 (서비스 통합):
└── Task 3: PaymentService.InitiatePaymentAsync 매핑 통합 (depends: 1,2)

Wave 3 (문서/검증):
├── Task 4: API 스펙 문서 추가 (depends: 3)
└── Task 5: curl 기반 응답 검증 및 Evidence 수집 (depends: 3)

Critical Path: Task 1 → Task 2 → Task 3 → Task 5
```

---

## TODOs

- [ ] 1. DTO 계약 확장 — 결제 미리보기 메타데이터 추가

  **What to do**:
  - `DTO/Payment/PaymentRequestDto.cs` 내 `PaymentRequestResponseDto` 확장
  - 하위 DTO 추가:
    - `PaymentTicketInfoDto`
    - `PaymentEventInfoDto`
  - nullable 정책 명시 (썸네일/좌석/장소 누락 허용)

  **Acceptance Criteria**:
  - [ ] 기존 필드는 유지되고 직렬화 구조가 깨지지 않음
  - [ ] `ticketInfo`, `eventInfo`가 Swagger 응답 예시에서 확인됨

  **References**:
  - `DTO/Payment/PaymentRequestDto.cs`

---

- [ ] 2. 결제 미리보기 조회 메서드 추가

  **What to do**:
  - `Repository/Transaction/ITransactionRepository.cs` 확장:
    - `GetPaymentPreviewAsync(long transactionId, int buyerId)` 추가
  - `Repository/Transaction/TransactionRepository.cs` 구현:
    - `transactions`, `transaction_items`, `tickets`, `events`, 좌석 테이블 조인
    - `TransactionHistoryRepository`의 좌석 조합 패턴(`CONCAT_WS`) 재사용
  - 필요 시 read model 추가:
    - `Repository/ReadModels/PaymentPreviewReadModel.cs`

  **Must NOT do**:
  - Task.WhenAll 기반 병렬 조회 금지
  - Service에서 SQL 직접 작성 금지

  **Acceptance Criteria**:
  - [ ] 거래+구매자 소유권 조건으로 1건 조회 가능
  - [ ] seatInfo/quantity/unitPrice/totalAmount/eventTitle/eventDateTime/venueName 반환

  **References**:
  - `Repository/Transaction/TransactionRepository.cs`
  - `Repository/Transaction/TransactionHistoryRepository.cs`

---

- [ ] 3. PaymentService 응답 매핑 통합

  **What to do**:
  - `Services/Payment/PaymentService.cs` `InitiatePaymentAsync()`에서
    Task 2 조회 결과를 DTO에 매핑
  - 클라이언트 입력 `orderName`은 Toss 결제용 텍스트로만 사용,
    화면용 정보는 preview 데이터 기준으로 반환
  - preview 누락 시에도 결제 요청 자체는 동작하도록 방어 로직 추가

  **Acceptance Criteria**:
  - [ ] `/api/payment/request` 응답에 `ticketInfo`, `eventInfo` 포함
  - [ ] 금액 검증/주문 생성 기존 로직 영향 없음

  **References**:
  - `Services/Payment/PaymentService.cs`
  - `Controllers/PaymentController.cs`

---

- [ ] 4. API 스펙 문서화

  **What to do**:
  - `api_spec/TASK-011_PAYMENT_REQUEST_CONTRACT.md` 작성
  - 요청/응답 필드 표 + nullable 규칙 + 샘플 JSON 포함

  **Acceptance Criteria**:
  - [ ] 모바일 팀이 문서만으로 DTO 작성 가능
  - [ ] `ticketInfo`/`eventInfo` 필수/선택 필드 구분 명확

---

- [ ] 5. QA & Evidence

  **QA Scenarios**:
  - Scenario A: 정상 거래 결제 요청
    - `ticketInfo`, `eventInfo` 포함 200 응답
  - Scenario B: 소유권 불일치 거래
    - 403 또는 비즈니스 에러 반환
  - Scenario C: 이벤트 일부 누락 데이터
    - nullable 필드 null 반환 + API 200 유지

  **Evidence**:
  - `.sisyphus/evidence/task-011-be-request-success.json`
  - `.sisyphus/evidence/task-011-be-request-forbidden.json`
  - `.sisyphus/evidence/task-011-be-request-nullable.json`

---

## Success Criteria

- [ ] 결제 요청 응답에 티켓/공연 정보가 구조화되어 제공됨
- [ ] 클라이언트가 `orderName` 문자열 파싱 없이 UI 구성 가능
- [ ] 기존 Toss 위젯 결제 흐름과 하위 호환 유지
- [ ] 권한/금액 검증 로직 회귀 없음
