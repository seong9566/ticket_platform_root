# TASK-006: 신고/분쟁 시스템

**작성일**: 2026-02-11
**작성자**: PM
**담당 팀**: Backend
**담당자**: Backend Agent
**상태**: 🟡 부분 완료 (백엔드 API 구현 완료, 테스트/문서화 진행 필요)
**우선순위**: 🔴 High

---

## 📋 작업 개요

### 작업 설명
거래 과정에서 발생하는 분쟁(신고)을 접수, 관리, 해결하는 시스템을 구현하라. 신고 생성 시 Escrow 상태를 FROZEN으로 전환하고, 해결 시 REFUNDED 또는 RELEASED로 전환하라.

### 목표
- 신고 생성/조회/취소 API 제공
- 증거 이미지 첨부 기능
- Escrow 상태 전환을 통한 자금 보호
- 신고 상태 관리 (PENDING → IN_REVIEW → RESOLVED/REJECTED/CANCELLED)

### 배경
티켓 거래 플랫폼 특성상 가짜 티켓, 잘못된 티켓, 미배송 등의 분쟁이 발생할 수 있다. 구매자를 보호하기 위해 신고 시 Escrow 자금을 동결하고, 관리자 검토 후 환불 또는 정산을 진행해야 한다. 기존 DB에 Dispute, DisputeEvidence, DisputeType, DisputeStatus, Escrow, EscrowStatus 테이블이 이미 존재한다.

---

## 🎯 완료 기준 (Acceptance Criteria)

- [x] `POST /api/disputes` — 신고 생성 API 동작
- [x] `GET /api/disputes` — 내 신고 목록 조회 API 동작
- [x] `GET /api/disputes/{id}` — 신고 상세 조회 API 동작
- [x] `POST /api/disputes/{id}/evidence` — 증거 이미지 첨부 API 동작
- [x] `PUT /api/disputes/{id}/cancel` — 신고 취소 API 동작
- [x] 신고 생성 시 Escrow 상태가 HOLD → FROZEN으로 전환
- [x] 신고 취소 시 Escrow 상태가 FROZEN → HOLD로 복원
- [x] 중복 신고 방지 (동일 거래에 활성 신고 1건만 허용)
- [x] 에러 처리 및 로깅 완료

---

## 🔧 기술 스펙 (Backend)

### 신고 유형 정의 (DisputeType)

| Code | 설명 |
|------|------|
| `FAKE_TICKET` | 가짜/위조 티켓 |
| `WRONG_TICKET` | 잘못된 티켓 (좌석/날짜 불일치) |
| `NO_DELIVERY` | 티켓 미배송 |
| `RUDE_BEHAVIOR` | 비매너 행위 |
| `OTHER` | 기타 |

### 신고 상태 정의 (DisputeStatus)

| Code | 설명 | Escrow 상태 |
|------|------|-------------|
| `PENDING` | 접수 대기 | FROZEN |
| `IN_REVIEW` | 검토 중 | FROZEN |
| `RESOLVED_BUYER` | 구매자 승 (환불) | REFUNDED |
| `RESOLVED_SELLER` | 판매자 승 (정산) | RELEASED |
| `REJECTED` | 신고 기각 | HOLD 복원 |
| `CANCELLED` | 신고자 취소 | HOLD 복원 |

### Escrow 상태 전환

```
정상 거래: HOLD → RELEASED (구매 확정 시)

신고 발생:
  HOLD → FROZEN (신고 생성 시)
  FROZEN → REFUNDED (구매자 승)
  FROZEN → RELEASED (판매자 승)
  FROZEN → HOLD (신고 기각/취소)
```

### API 엔드포인트

#### 1. POST /api/disputes — 신고 생성

**요청 (Request)**
```json
{
  "transactionId": 10,
  "typeCode": "FAKE_TICKET",
  "description": "구매한 티켓의 QR코드가 유효하지 않습니다."
}
```

**필드 검증 규칙**:
- `transactionId`: 필수, 존재하는 거래 ID
- `typeCode`: 필수, DisputeType.Code 중 하나
- `description`: 필수, 최소 10자 이상

**성공 응답 (201 Created)**
```json
{
  "message": "신고 접수 완료",
  "data": {
    "id": 5,
    "transactionId": 10,
    "typeCode": "FAKE_TICKET",
    "typeName": "가짜/위조 티켓",
    "statusCode": "PENDING",
    "statusName": "접수 대기",
    "description": "구매한 티켓의 QR코드가 유효하지 않습니다.",
    "createdAt": "2026-02-11T10:00:00Z"
  },
  "statusCode": 201
}
```

**에러 응답**
- `400 Bad Request` — 필수 필드 누락, description 10자 미만, 잘못된 typeCode
- `401 Unauthorized` — JWT 인증 실패
- `403 Forbidden` — 해당 거래의 구매자가 아님
- `404 Not Found` — 거래가 존재하지 않음
- `409 Conflict` — 해당 거래에 이미 활성 신고가 존재함

**비즈니스 규칙**:
- 신고자(ClaimantId)는 해당 거래의 구매자(BuyerId)만 가능
- 동일 거래에 활성 신고(PENDING 또는 IN_REVIEW)가 존재하면 중복 생성 불가
- 거래 상태가 결제 완료(PAID) 또는 배송 중(DELIVERING) 상태여야 신고 가능
- 신고 생성 시 Escrow 상태를 HOLD → FROZEN으로 전환
- TASK-005 구현 완료 시, 피신고자(판매자)에게 DISPUTE_OPENED 알림 발송

---

#### 2. GET /api/disputes — 내 신고 목록

**요청 (Query Parameters)**
```
GET /api/disputes?cursor={cursor}&limit={limit}
```

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| cursor | string? | null | 마지막 신고 ID 기반 커서 |
| limit | int | 20 | 조회 개수 (최대 50) |

**성공 응답 (200 OK)**
```json
{
  "message": "신고 목록 조회 성공",
  "data": {
    "items": [
      {
        "id": 5,
        "transactionId": 10,
        "typeCode": "FAKE_TICKET",
        "typeName": "가짜/위조 티켓",
        "statusCode": "PENDING",
        "statusName": "접수 대기",
        "description": "구매한 티켓의 QR코드가 유효하지 않습니다.",
        "evidenceCount": 2,
        "createdAt": "2026-02-11T10:00:00Z"
      }
    ],
    "nextCursor": "4",
    "hasMore": false
  },
  "statusCode": 200
}
```

**비즈니스 규칙**:
- 본인이 신고자(ClaimantId)인 신고만 조회
- 최신순 정렬

---

#### 3. GET /api/disputes/{id} — 신고 상세

**성공 응답 (200 OK)**
```json
{
  "message": "신고 상세 조회 성공",
  "data": {
    "id": 5,
    "transactionId": 10,
    "typeCode": "FAKE_TICKET",
    "typeName": "가짜/위조 티켓",
    "statusCode": "PENDING",
    "statusName": "접수 대기",
    "description": "구매한 티켓의 QR코드가 유효하지 않습니다.",
    "evidences": [
      {
        "id": 1,
        "imageUrl": "https://storage.example.com/disputes/evidence_1.jpg",
        "note": "무효 QR코드 스크린샷",
        "createdAt": "2026-02-11T10:05:00Z"
      }
    ],
    "transaction": {
      "transactionId": 10,
      "ticketTitle": "아이유 콘서트 S석",
      "amount": 150000,
      "buyerNickname": "구매자닉네임",
      "sellerNickname": "판매자닉네임"
    },
    "createdAt": "2026-02-11T10:00:00Z"
  },
  "statusCode": 200
}
```

**에러 응답**
- `401 Unauthorized` — JWT 인증 실패
- `403 Forbidden` — 본인의 신고가 아님 (신고자 또는 피신고 거래 당사자만 조회 가능)
- `404 Not Found` — 신고가 존재하지 않음

---

#### 4. POST /api/disputes/{id}/evidence — 증거 첨부

**요청 (Multipart Form)**
```
Content-Type: multipart/form-data

file: [이미지 파일] (JPEG, PNG, 최대 10MB)
note: "증거 설명 텍스트" (선택)
```

**성공 응답 (201 Created)**
```json
{
  "message": "증거 첨부 완료",
  "data": {
    "id": 2,
    "disputeId": 5,
    "imageUrl": "https://storage.example.com/disputes/evidence_2.jpg",
    "note": "구매 영수증",
    "createdAt": "2026-02-11T10:10:00Z"
  },
  "statusCode": 201
}
```

**에러 응답**
- `400 Bad Request` — 파일 없음, 지원하지 않는 파일 형식, 파일 크기 초과
- `401 Unauthorized` — JWT 인증 실패
- `403 Forbidden` — 본인의 신고가 아님
- `404 Not Found` — 신고가 존재하지 않음
- `409 Conflict` — 신고가 이미 종료 상태 (RESOLVED/REJECTED/CANCELLED)

**비즈니스 규칙**:
- 신고자만 증거를 첨부할 수 있음
- 신고 상태가 PENDING 또는 IN_REVIEW일 때만 첨부 가능
- 신고당 최대 증거 5건
- 기존 FileUploadService를 활용하여 이미지 업로드

---

#### 5. PUT /api/disputes/{id}/cancel — 신고 취소

**성공 응답 (200 OK)**
```json
{
  "message": "신고 취소 완료",
  "data": {
    "id": 5,
    "statusCode": "CANCELLED",
    "statusName": "신고자 취소"
  },
  "statusCode": 200
}
```

**에러 응답**
- `401 Unauthorized` — JWT 인증 실패
- `403 Forbidden` — 본인의 신고가 아님
- `404 Not Found` — 신고가 존재하지 않음
- `409 Conflict` — 이미 종료 상태이거나 IN_REVIEW 이후 취소 불가

**비즈니스 규칙**:
- 신고자만 취소 가능
- PENDING 상태에서만 취소 가능 (IN_REVIEW 이후 취소 불가)
- 취소 시 Escrow 상태를 FROZEN → HOLD로 복원

---

### 데이터베이스

기존 테이블 활용 (신규 테이블 생성 불필요):

- **Dispute** — 신고 (TransactionId, ClaimantId, TypeId, Description, StatusId, CreatedAt)
- **DisputeEvidence** — 증거 (DisputeId, ImageUrl, Note, CreatedAt)
- **DisputeType** — 신고 유형 코드 (Code, NameKo, IsActive, SortOrder)
- **DisputeStatus** — 신고 상태 코드 (Code, NameKo, IsActive, SortOrder)
- **Escrow** — 에스크로 (TransactionId, Amount, StatusId, CreatedAt, ReleasedAt, RefundedAt)
- **EscrowStatus** — 에스크로 상태 코드 (Code, NameKo)

DisputeType과 DisputeStatus 테이블에 위 코드값들이 등록되어 있는지 확인하고, 없으면 seed 데이터를 추가하라.

---

## 📂 파일 구조

### Backend
```
TicketPlatFormServer/
├── Controllers/
│   └── DisputeController.cs              — 신고 API 컨트롤러
├── Services/
│   └── Dispute/
│       ├── IDisputeService.cs            — 신고 서비스 인터페이스
│       └── DisputeService.cs             — 신고 비즈니스 로직
├── Repository/
│   └── Dispute/
│       ├── IDisputeRepository.cs         — 신고 저장소 인터페이스
│       ├── DisputeRepository.cs          — 신고 쿼리/저장
│       ├── IEscrowRepository.cs          — 에스크로 저장소 인터페이스
│       └── EscrowRepository.cs           — 에스크로 상태 전환
└── DTO/
    └── Dispute/
        ├── CreateDisputeReqDto.cs        — 신고 생성 요청
        ├── DisputeListRespDto.cs         — 신고 목록 응답
        ├── DisputeDetailRespDto.cs       — 신고 상세 응답
        └── DisputeEvidenceRespDto.cs     — 증거 응답
```

---

## ✅ 작업 체크리스트

### 개발
- [x] DisputeController 구현
- [x] DisputeService 구현
- [x] DisputeRepository 구현
- [x] EscrowRepository 구현 (상태 전환)
- [x] DTO 생성
- [x] DI 등록 (Program.cs)
- [x] 이미지 업로드 연동 (기존 FileUploadService 활용)

### 최신 점검 (2026-02-13)
- 구현 완료: 신고 생성/목록/상세/증거 첨부/취소 API
- 구현 완료: Escrow 상태 전환(HOLD→FROZEN, FROZEN→HOLD)
- 구현 완료: 활성 신고 중복 방지, 신고 유형/상태 seed, escrow_statuses `frozen` seed
- 구현 완료: DISPUTE_OPENED 알림 트리거 연동
- 미완료: 통합 테스트 체크리스트 완료, API 스펙 문서 정리

### 테스트
- [ ] 신고 생성/조회/취소 테스트
- [ ] 증거 첨부 테스트
- [ ] Escrow 상태 전환 테스트
- [ ] 중복 신고 방지 테스트
- [ ] 수동 테스트 완료

### 문서화
- [ ] API 명세서 업데이트
- [ ] Swagger 문서 확인

### 코드 품질
- [ ] 린팅 에러 없음
- [ ] 코딩 컨벤션 준수
- [ ] 자체 코드 리뷰 완료

---

## 🧪 테스트 시나리오

### 시나리오 1: 신고 생성 성공 + Escrow 동결
```
전제:
- 거래 ID=10, 구매자(userId=1), 결제 완료 상태
- Escrow 상태: HOLD

입력:
- POST /api/disputes
- Body: { "transactionId": 10, "typeCode": "FAKE_TICKET", "description": "QR코드가 유효하지 않습니다. 스캔이 되지 않습니다." }
- Header: Authorization: Bearer {구매자jwt}

예상 결과:
- 201 Created
- Dispute 테이블에 레코드 생성 (StatusId = PENDING)
- Escrow 상태: HOLD → FROZEN
```

### 시나리오 2: 중복 신고 차단
```
전제:
- 거래 ID=10에 이미 PENDING 상태 신고 존재

입력:
- POST /api/disputes
- Body: { "transactionId": 10, "typeCode": "WRONG_TICKET", "description": "좌석 번호가 일치하지 않습니다. 구매한 것과 다릅니다." }

예상 결과:
- 409 Conflict
- 메시지: "해당 거래에 이미 처리 중인 신고가 있습니다"
```

### 시나리오 3: 구매자가 아닌 사용자의 신고 시도
```
전제:
- 거래 ID=10, 판매자(userId=2)가 신고 시도

입력:
- POST /api/disputes
- Header: Authorization: Bearer {판매자jwt}

예상 결과:
- 403 Forbidden
- 메시지: "해당 거래의 구매자만 신고할 수 있습니다"
```

### 시나리오 4: 증거 이미지 첨부
```
전제:
- 신고 ID=5, PENDING 상태, 신고자 본인

입력:
- POST /api/disputes/5/evidence
- Form: file=screenshot.jpg, note="무효 QR코드 스크린샷"

예상 결과:
- 201 Created
- DisputeEvidence 테이블에 레코드 생성
- imageUrl에 업로드된 이미지 URL 반환
```

### 시나리오 5: 신고 취소 + Escrow 복원
```
전제:
- 신고 ID=5, PENDING 상태
- Escrow 상태: FROZEN

입력:
- PUT /api/disputes/5/cancel

예상 결과:
- 200 OK
- Dispute.StatusId → CANCELLED
- Escrow 상태: FROZEN → HOLD
```

### 시나리오 6: IN_REVIEW 상태에서 취소 시도
```
전제:
- 신고 ID=5, IN_REVIEW 상태 (관리자가 검토 중)

입력:
- PUT /api/disputes/5/cancel

예상 결과:
- 409 Conflict
- 메시지: "검토 중인 신고는 취소할 수 없습니다"
```

---

## 🔗 의존성

### 선행 작업
- 없음 (기존 DB 테이블 및 FileUploadService 활용, 즉시 구현 가능)

### 관련 이슈
- DisputeType, DisputeStatus, EscrowStatus seed 데이터 확인 필요
- 기존 FileUploadService 활용 (이미지 업로드)
- 기존 TransactionRepository 활용 (거래 조회/검증)

### 후속 작업
- [ ] Mobile TASK-006: 신고 시스템 (클라이언트)
- [ ] TASK-005 연동: 신고 생성/해결 시 알림 발송
- [ ] 관리자 패널: 신고 검토/해결 기능 (Phase 3)

---

## ⏱️ 예상 소요 시간

| 항목 | 시간 |
|------|------|
| DTO 생성 | 1시간 |
| DisputeRepository + EscrowRepository 구현 | 3시간 |
| DisputeService 구현 (비즈니스 로직) | 4시간 |
| DisputeController 구현 | 2시간 |
| 이미지 업로드 연동 | 1시간 |
| 테스트 | 2시간 |
| 버그 수정 | 1시간 |
| **총 예상 시간** | **14시간 (약 2일)** |

---

## 📅 일정

- **시작일**: 2026-02-12
- **목표 완료일**: 2026-02-14
- **실제 완료일**: 2026-02-13 (부분 완료)

---

## 🚨 리스크 및 고려사항

### 기술적 리스크
- **Escrow 상태 전환 동시성**: 신고 생성과 구매 확정이 동시에 발생할 수 있음 → 트랜잭션 격리 수준 설정 또는 낙관적 잠금 적용
- **이미지 업로드 실패**: 외부 스토리지(Supabase) 장애 시 증거 첨부 실패 → 에러 메시지로 재시도 안내

### 블로커
- DisputeType, DisputeStatus, EscrowStatus 테이블 seed 데이터 확인

---

## 📝 구현 노트

### 주요 결정 사항
- 관리자 해결 기능 (RESOLVED_BUYER, RESOLVED_SELLER, REJECTED)은 Phase 3 관리자 패널에서 구현
- 이번 Sprint에서는 신고 생성/조회/취소/증거첨부만 구현
- Escrow 상태 전환 시 DB 트랜잭션을 사용하여 원자성 보장

---

## 🔍 코드 리뷰 체크포인트

### 리뷰어 확인 사항
- [ ] Escrow 상태 전환이 트랜잭션 내에서 원자적으로 수행되는지
- [ ] 중복 신고 방지 로직이 정확한지
- [ ] 권한 검사 (구매자만 신고, 본인 신고만 조회/취소)
- [ ] 이미지 업로드 시 파일 타입/크기 검증
- [ ] 증거 첨부 최대 5건 제한 동작

---

## 📚 참고 자료

- 기존 TransactionRepository — 거래 조회/검증 로직
- 기존 FileUploadService — 이미지 업로드 로직
- 기존 PaymentService — Escrow 관련 로직
- DB 모델: Dispute.cs, DisputeEvidence.cs, DisputeType.cs, DisputeStatus.cs, Escrow.cs, EscrowStatus.cs

---

**리뷰어**: Backend Lead
**리뷰 완료일**: 2026-02-13
**상태**: 🔄 업데이트됨 (통합 테스트 완료 후 최종 완료)
