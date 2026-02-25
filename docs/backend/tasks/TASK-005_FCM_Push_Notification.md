# TASK-005: FCM 푸시 알림 시스템

**작성일**: 2026-02-11
**작성자**: PM
**담당 팀**: Backend
**담당자**: Backend Agent
**상태**: 🟡 부분 완료 (핵심 기능 완료, 신고 알림 트리거 대기)
**우선순위**: 🔴 High

---

## 📋 작업 개요

### 작업 설명
Firebase Cloud Messaging(FCM) HTTP v1 API를 연동하여 푸시 알림 발송 시스템을 구현하라. 디바이스 토큰 관리, 알림 저장/조회, 읽음 처리, 미읽음 카운트 API를 제공하라.

### 목표
- FCM HTTP v1 API 연동을 통한 실시간 푸시 알림 발송
- 디바이스 토큰 등록/삭제 관리
- 알림 목록 조회 (cursor pagination)
- 읽음 처리 및 미읽음 카운트 제공

### 배경
Phase 2 핵심 기능으로, 채팅 메시지, 결제, 구매 확정, 신고 등 주요 이벤트 발생 시 사용자에게 푸시 알림을 전달해야 한다. 기존 DB에 Notification, NotificationToken, NotificationType, NotificationPlatform 테이블이 이미 존재한다.

---

## 🎯 완료 기준 (Acceptance Criteria)

- [x] `POST /api/notifications/token` — 디바이스 토큰 등록 API 동작
- [x] `DELETE /api/notifications/token` — 디바이스 토큰 삭제 API 동작
- [x] `GET /api/notifications` — 알림 목록 cursor pagination 동작
- [x] `PUT /api/notifications/{id}/read` — 개별 읽음 처리 동작
- [x] `PUT /api/notifications/read-all` — 전체 읽음 처리 동작
- [x] `GET /api/notifications/unread-count` — 미읽음 카운트 반환
- [x] FCM HTTP v1 API를 통한 푸시 알림 발송 동작
- [x] 알림 트리거 포인트 (채팅, 결제, 구매확정)에서 알림 발송 동작
- [ ] 알림 트리거 포인트 (신고 접수/해결)에서 알림 발송 동작
- [x] 에러 처리 및 로깅 완료

### 최신 점검 (2026-02-13)

- 구현 완료: Notification API 6종, FCM HTTP v1 발송, 토큰 upsert/delete, cursor pagination, 읽음 처리, 미읽음 카운트
- 트리거 완료: `CHAT_MESSAGE`, `PAYMENT_REQUEST`, `PAYMENT_SUCCESS`, `PURCHASE_CONFIRMED`
- 트리거 미완료: `DISPUTE_OPENED`, `DISPUTE_RESOLVED` (Dispute 서비스 구현 후 연동 예정)
- 점검 기준: `NotificationController`, `NotificationService`, `FcmService`, `NotificationRepository`, `NotificationTokenRepository`, `ChatService`, `PaymentService`, `Program.cs`

---

## 🔧 기술 스펙 (Backend)

### 알림 타입 정의

| Code | 설명 | 트리거 포인트 |
|------|------|---------------|
| `CHAT_MESSAGE` | 새 채팅 메시지 | 채팅 메시지 전송 시 |
| `PAYMENT_REQUEST` | 결제 요청 | 판매자가 결제 요청 생성 시 |
| `PAYMENT_SUCCESS` | 결제 완료 | 결제 성공 시 (구매자+판매자) |
| `PURCHASE_CONFIRMED` | 구매 확정 | 구매 확정 시 (판매자) |
| `DISPUTE_OPENED` | 신고 접수 | 신고 생성 시 (피신고자) |
| `DISPUTE_RESOLVED` | 신고 해결 | 신고 해결 시 (양측) |

### API 엔드포인트

#### 1. POST /api/notifications/token — 디바이스 토큰 등록

**요청 (Request)**
```json
{
  "deviceToken": "fcm_device_token_string",
  "platform": "ANDROID" | "IOS"
}
```

**성공 응답 (200 OK)**
```json
{
  "message": "토큰 등록 성공",
  "data": {
    "id": 1,
    "deviceToken": "fcm_device_token_string",
    "platform": "ANDROID"
  },
  "statusCode": 200
}
```

**에러 응답**
- `400 Bad Request` — 필수 필드 누락 또는 잘못된 platform 값
- `401 Unauthorized` — JWT 인증 실패
- `409 Conflict` — 동일 토큰 이미 등록됨 (upsert 처리하여 실제로는 발생하지 않음)

**비즈니스 규칙**:
- 동일 deviceToken이 이미 존재하면 userId를 업데이트 (upsert)
- 한 사용자가 여러 디바이스 토큰을 가질 수 있음

---

#### 2. DELETE /api/notifications/token — 디바이스 토큰 삭제

**요청 (Request)**
```json
{
  "deviceToken": "fcm_device_token_string"
}
```

**성공 응답 (200 OK)**
```json
{
  "message": "토큰 삭제 성공",
  "statusCode": 200
}
```

**에러 응답**
- `400 Bad Request` — deviceToken 누락
- `401 Unauthorized` — JWT 인증 실패
- `404 Not Found` — 해당 토큰이 존재하지 않음

---

#### 3. GET /api/notifications — 알림 목록 (Cursor Pagination)

**요청 (Query Parameters)**
```
GET /api/notifications?cursor={cursor}&limit={limit}
```

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| cursor | string? | null | 마지막 알림 ID 기반 커서 |
| limit | int | 20 | 조회 개수 (최대 50) |

**성공 응답 (200 OK)**
```json
{
  "message": "알림 목록 조회 성공",
  "data": {
    "items": [
      {
        "id": 100,
        "typeCode": "CHAT_MESSAGE",
        "typeName": "채팅 메시지",
        "title": "새 메시지가 도착했습니다",
        "body": "홍길동님이 메시지를 보냈습니다",
        "data": "{\"roomId\": 42, \"senderId\": 5}",
        "readFlag": false,
        "readAt": null,
        "createdAt": "2026-02-11T10:30:00Z"
      }
    ],
    "nextCursor": "99",
    "hasMore": true
  },
  "statusCode": 200
}
```

**에러 응답**
- `401 Unauthorized` — JWT 인증 실패

---

#### 4. PUT /api/notifications/{id}/read — 읽음 처리

**성공 응답 (200 OK)**
```json
{
  "message": "읽음 처리 완료",
  "statusCode": 200
}
```

**에러 응답**
- `401 Unauthorized` — JWT 인증 실패
- `403 Forbidden` — 본인의 알림이 아님
- `404 Not Found` — 알림이 존재하지 않음

**비즈니스 규칙**:
- ReadFlag를 true로, ReadAt을 현재 시각으로 설정
- 이미 읽은 알림에 대해 재요청해도 200 반환 (멱등)

---

#### 5. PUT /api/notifications/read-all — 전체 읽음 처리

**성공 응답 (200 OK)**
```json
{
  "message": "전체 읽음 처리 완료",
  "data": {
    "updatedCount": 15
  },
  "statusCode": 200
}
```

**에러 응답**
- `401 Unauthorized` — JWT 인증 실패

**비즈니스 규칙**:
- 해당 사용자의 ReadFlag가 false인 모든 알림을 일괄 업데이트

---

#### 6. GET /api/notifications/unread-count — 미읽음 카운트

**성공 응답 (200 OK)**
```json
{
  "message": "미읽음 카운트 조회 성공",
  "data": {
    "unreadCount": 5
  },
  "statusCode": 200
}
```

**에러 응답**
- `401 Unauthorized` — JWT 인증 실패

---

### FCM HTTP v1 API 연동 요구사항

- Firebase Admin SDK 또는 HTTP v1 API 직접 호출 사용
- 서비스 계정 JSON 키 파일을 환경 설정으로 관리 (`appsettings.json`)
- FCM 프로젝트 ID를 환경별로 분리 (`appsettings.Development.json`, `appsettings.Production.json`)
- 알림 발송 실패 시 로그 기록 (재시도 없음, Phase 2 범위에서는 fire-and-forget)
- 만료된 토큰(FCM 응답 `UNREGISTERED`) 감지 시 NotificationToken 테이블에서 삭제

### 알림 발송 데이터 페이로드

```json
{
  "message": {
    "token": "device_token",
    "notification": {
      "title": "알림 제목",
      "body": "알림 내용"
    },
    "data": {
      "type": "CHAT_MESSAGE",
      "targetId": "42",
      "extra": "{}"
    }
  }
}
```

### 알림 트리거 포인트

| 트리거 | 수신자 | 타입 | data 예시 |
|--------|--------|------|-----------|
| 채팅 메시지 전송 | 상대방 | `CHAT_MESSAGE` | `{"roomId": 42}` |
| 결제 요청 생성 | 구매자 | `PAYMENT_REQUEST` | `{"transactionId": 10}` |
| 결제 성공 | 구매자 + 판매자 | `PAYMENT_SUCCESS` | `{"transactionId": 10}` |
| 구매 확정 | 판매자 | `PURCHASE_CONFIRMED` | `{"transactionId": 10}` |
| 신고 접수 | 피신고자 | `DISPUTE_OPENED` | `{"disputeId": 5}` |
| 신고 해결 | 신고자 + 피신고자 | `DISPUTE_RESOLVED` | `{"disputeId": 5}` |

### 데이터베이스

기존 테이블 활용 (신규 테이블 생성 불필요):

- **Notification** — 알림 저장 (UserId, TypeId, Title, Body, Data, ReadFlag, ReadAt, CreatedAt)
- **NotificationToken** — 디바이스 토큰 (UserId, DeviceToken, PlatformId, CreatedAt)
- **NotificationType** — 알림 유형 코드 (Code, NameKo, IsActive, SortOrder)
- **NotificationPlatform** — 플랫폼 코드 (Code: ANDROID/IOS)

NotificationType 테이블에 위 6개 알림 타입 코드가 등록되어 있는지 확인하고, 없으면 seed 데이터를 추가하라.

---

## 📂 파일 구조

### Backend
```
TicketPlatFormServer/
├── Controllers/
│   └── NotificationController.cs          — 알림 API 컨트롤러
├── Services/
│   └── Notification/
│       ├── INotificationService.cs        — 알림 서비스 인터페이스
│       ├── NotificationService.cs         — 알림 CRUD + 발송 로직
│       ├── IFcmService.cs                 — FCM 발송 인터페이스
│       └── FcmService.cs                  — FCM HTTP v1 API 연동
├── Repository/
│   └── Notification/
│       ├── INotificationRepository.cs     — 알림 저장소 인터페이스
│       ├── NotificationRepository.cs      — 알림 쿼리/저장
│       ├── INotificationTokenRepository.cs — 토큰 저장소 인터페이스
│       └── NotificationTokenRepository.cs — 토큰 CRUD
└── DTO/
    └── Notification/
        ├── RegisterTokenReqDto.cs         — 토큰 등록 요청
        ├── DeleteTokenReqDto.cs           — 토큰 삭제 요청
        ├── NotificationListRespDto.cs     — 알림 목록 응답
        └── UnreadCountRespDto.cs          — 미읽음 카운트 응답
```

---

## ✅ 작업 체크리스트

### 개발
- [x] NotificationController 구현
- [x] NotificationService 구현
- [x] FcmService 구현 (FCM HTTP v1)
- [x] NotificationRepository 구현
- [x] NotificationTokenRepository 구현
- [x] DTO 생성
- [x] 기존 서비스에 알림 트리거 추가 (ChatService, PaymentService)
- [ ] 신고 서비스 트리거 추가 (DISPUTE_OPENED, DISPUTE_RESOLVED)
- [x] DI 등록 (Program.cs)
- [x] FCM 설정 연결 (Development: ProjectId + ServiceAccountJsonPath)

### 테스트
- [ ] 토큰 등록/삭제 테스트
- [ ] 알림 목록 pagination 테스트
- [ ] 읽음 처리 테스트
- [ ] FCM 발송 테스트
- [ ] 수동 테스트 완료

### 문서화
- [x] API 명세서 업데이트
- [ ] Swagger 문서 확인

### 코드 품질
- [ ] 린팅 에러 없음
- [ ] 코딩 컨벤션 준수
- [ ] 자체 코드 리뷰 완료

---

## 🧪 테스트 시나리오

### 시나리오 1: 디바이스 토큰 등록 성공
```
입력:
- POST /api/notifications/token
- Body: { "deviceToken": "valid_fcm_token", "platform": "ANDROID" }
- Header: Authorization: Bearer {jwt}

예상 결과:
- 200 OK
- NotificationToken 테이블에 레코드 생성
- 동일 토큰 재등록 시 upsert (200 OK)
```

### 시나리오 2: 알림 목록 Cursor Pagination
```
전제: 사용자에게 알림 30건 존재

입력:
- GET /api/notifications?limit=20

예상 결과:
- 200 OK
- items: 20건 (최신순)
- hasMore: true
- nextCursor: 11번째 알림의 ID

후속:
- GET /api/notifications?cursor={nextCursor}&limit=20
- items: 10건
- hasMore: false
```

### 시나리오 3: 개별 읽음 처리
```
전제: 미읽음 알림 (id=100) 존재

입력:
- PUT /api/notifications/100/read

예상 결과:
- 200 OK
- Notification.ReadFlag = true
- Notification.ReadAt = 현재 시각
```

### 시나리오 4: 전체 읽음 처리
```
전제: 미읽음 알림 15건 존재

입력:
- PUT /api/notifications/read-all

예상 결과:
- 200 OK
- updatedCount: 15
- 모든 알림의 ReadFlag = true
```

### 시나리오 5: FCM 푸시 발송 — 채팅 메시지
```
전제:
- 사용자 A가 사용자 B에게 채팅 메시지 전송
- 사용자 B의 FCM 토큰이 등록되어 있음

예상 결과:
- Notification 테이블에 레코드 생성 (userId=B, type=CHAT_MESSAGE)
- FCM HTTP v1 API 호출로 B의 디바이스에 푸시 발송
- 발송 실패 시 로그 기록 (에러 throw 없음)
```

### 시나리오 6: 만료 토큰 자동 정리
```
전제:
- 사용자 B의 FCM 토큰이 만료됨

동작:
- 알림 발송 시 FCM 응답에서 UNREGISTERED 수신

예상 결과:
- NotificationToken 테이블에서 해당 토큰 삭제
- 로그에 "만료 토큰 정리" 기록
- 알림 발송 프로세스는 정상 종료 (에러 없음)
```

---

## 🔗 의존성

### 선행 작업
- 없음 (기존 DB 테이블 활용, 즉시 구현 가능)

### 관련 이슈
- Firebase 프로젝트 설정 및 서비스 계정 키 파일 필요
- NotificationType seed 데이터 확인 필요

### 후속 작업
- [ ] Mobile TASK-005: FCM 푸시 알림 (클라이언트)
- [ ] TASK-006: 신고 시스템 (DISPUTE_OPENED/RESOLVED 알림 연동)
- [ ] Phase 2 후속: QR 인증, 정산, 평점 알림 추가

---

## ⏱️ 예상 소요 시간

| 항목 | 시간 |
|------|------|
| DTO + Repository 생성 | 2시간 |
| NotificationService 구현 | 3시간 |
| FcmService 구현 (HTTP v1) | 3시간 |
| NotificationController 구현 | 2시간 |
| 기존 서비스 트리거 추가 | 2시간 |
| 테스트 | 2시간 |
| 버그 수정 | 1시간 |
| **총 예상 시간** | **15시간 (약 2일)** |

---

## 📅 일정

- **시작일**: 2026-02-12
- **목표 완료일**: 2026-02-14
- **실제 완료일**: 2026-02-13 (부분 완료)

---

## 🚨 리스크 및 고려사항

### 기술적 리스크
- **FCM 서비스 계정 키 미확보**: Firebase Console에서 서비스 계정 JSON 키를 발급받아야 함. 미확보 시 FCM 발송 기능 구현 불가 → 키 발급 즉시 진행
- **FCM 토큰 만료**: 클라이언트에서 토큰 갱신이 제대로 되지 않으면 발송 실패 → UNREGISTERED 응답 시 토큰 삭제로 대응

### 블로커
- Firebase 프로젝트 생성 및 서비스 계정 키 파일 확보

---

## 📝 구현 노트

### 주요 결정 사항
- FCM 발송 실패 시 재시도 없음 (fire-and-forget) — Phase 2 범위
- 알림 저장은 FCM 발송과 독립적 (발송 실패해도 알림은 DB에 저장)
- Cursor pagination은 기존 TransactionHistory 패턴 참고
- 현재 트리거 반영 범위: CHAT_MESSAGE, PAYMENT_REQUEST, PAYMENT_SUCCESS, PURCHASE_CONFIRMED
- DISPUTE_OPENED, DISPUTE_RESOLVED는 Dispute 서비스/플로우 구현 후 연동 예정
- NotificationPlatform(ANDROID/IOS), NotificationType(6종)은 코드에서 누락 시 자동 seed

---

## 🔍 코드 리뷰 체크포인트

### 리뷰어 확인 사항
- [ ] FCM 서비스 계정 키가 소스코드에 포함되지 않았는지
- [ ] 알림 조회 시 본인 알림만 반환하는지 (userId 필터)
- [ ] Cursor pagination이 정확하게 동작하는지
- [ ] 알림 트리거가 기존 서비스 로직에 부작용 없이 추가되었는지
- [ ] 에러 처리 적절성 (FCM 발송 실패가 메인 로직에 영향 없는지)

---

## 📚 참고 자료

- 기존 TransactionController — cursor pagination 패턴
- 기존 ChatService — 메시지 전송 로직 (트리거 추가 지점)
- 기존 PaymentService — 결제 성공 로직 (트리거 추가 지점)
- DB 모델: Notification.cs, NotificationToken.cs, NotificationType.cs, NotificationPlatform.cs

---

**리뷰어**: Backend Lead
**리뷰 완료일**: 2026-02-13
**상태**: 🔄 업데이트됨 (Dispute 트리거 연동 후 최종 완료)
