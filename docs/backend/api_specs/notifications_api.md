# Notifications API

**작성일**: 2026-02-11
**작성자**: Backend Team
**상태**: 🚧 구현중

---

## 1) POST /api/notifications/token

디바이스 토큰 등록/업데이트(upsert)

요청
```json
{
  "deviceToken": "fcm_device_token_string",
  "platform": "ANDROID"
}
```

응답 (200)
```json
{
  "message": "토큰 등록 성공",
  "data": {
    "id": 1,
    "deviceToken": "fcm_device_token_string",
    "platform": "ANDROID"
  },
  "statusCode": 200,
  "success": true
}
```

---

## 2) DELETE /api/notifications/token

디바이스 토큰 삭제

요청
```json
{
  "deviceToken": "fcm_device_token_string"
}
```

응답 (200)
```json
{
  "message": "토큰 삭제 성공",
  "data": null,
  "statusCode": 200,
  "success": true
}
```

---

## 3) GET /api/notifications

알림 목록 cursor pagination

요청 예시
```text
GET /api/notifications?cursor=100&limit=20
```

응답 (200)
```json
{
  "message": "알림 목록 조회 성공",
  "data": {
    "items": [
      {
        "id": 100,
        "typeCode": "CHAT_MESSAGE",
        "typeName": "채팅 메시지",
        "title": "지킬 앤 하이드 10주년",
        "body": "안녕하세요, 아직 구매 가능할까요?",
        "data": "{\"type\":\"CHAT_MESSAGE\",\"title\":\"지킬 앤 하이드 10주년\",\"body\":\"안녕하세요, 아직 구매 가능할까요?\",\"roomId\":\"42\",\"message\":\"안녕하세요, 아직 구매 가능할까요?\",\"messageType\":\"TEXT\",\"ticketTitle\":\"지킬 앤 하이드 10주년\",\"ticketImageUrl\":\"https://...\",\"senderId\":\"5\",\"messageId\":\"987\"}",
        "readFlag": false,
        "readAt": null,
        "createdAt": "2026-02-11T10:30:00Z"
      }
    ],
    "nextCursor": "99",
    "hasMore": true
  },
  "statusCode": 200,
  "success": true
}
```

### CHAT_MESSAGE payload 필드

`typeCode = CHAT_MESSAGE`인 경우 `data`에 아래 필드를 포함합니다.

| 필드 | 타입 | 설명 |
|------|------|------|
| type | string | 알림 타입 코드 (`CHAT_MESSAGE`) |
| title | string | 푸시 알림 제목 (티켓/공연명) |
| body | string | 푸시 알림 본문 (메시지 미리보기) |
| roomId | string | 채팅방 ID |
| message | string | 실제 메시지 본문 또는 이미지 프리뷰 |
| messageType | string | `TEXT` 또는 `IMAGE` |
| ticketTitle | string | 티켓(공연) 제목 |
| ticketImageUrl | string | 티켓 대표 이미지 URL (없으면 빈 문자열) |
| senderId | string | 발신자 사용자 ID |
| messageId | string | 채팅 메시지 ID |

`messageType = IMAGE`인 경우 `message` 값은 `[이미지]`로 전달됩니다.

---

## 4) PUT /api/notifications/{id}/read

개별 읽음 처리 (멱등)

응답 (200)
```json
{
  "message": "읽음 처리 완료",
  "data": null,
  "statusCode": 200,
  "success": true
}
```

---

## 5) PUT /api/notifications/read-all

전체 읽음 처리

응답 (200)
```json
{
  "message": "전체 읽음 처리 완료",
  "data": {
    "updatedCount": 15
  },
  "statusCode": 200,
  "success": true
}
```

---

## 6) GET /api/notifications/unread-count

미읽음 카운트 조회

응답 (200)
```json
{
  "message": "미읽음 카운트 조회 성공",
  "data": {
    "unreadCount": 5
  },
  "statusCode": 200,
  "success": true
}
```

---

## 에러 응답 공통

```json
{
  "message": "오류 메시지",
  "data": null,
  "statusCode": 400,
  "success": false
}
```

주요 상태코드
- 400: 요청 파라미터/바디 오류
- 401: 인증 실패
- 403: 본인 알림 아님
- 404: 리소스 없음
