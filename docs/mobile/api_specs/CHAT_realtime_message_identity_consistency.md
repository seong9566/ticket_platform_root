# 채팅 실시간 메시지 소유자 판별 일관성 요청 (Mobile -> Backend)

**작성일**: 2026-02-12  
**작성자**: Mobile Team  
**상태**: ✅ 반영 완료 (Backend)  
**버전**: 1.0

---

## 1) 배경

현재 모바일에서 동일한 메시지가
- 실시간(SignalR) 수신 직후에는 상대방 메시지처럼 보이고,
- 화면 재진입(REST 재조회) 후에는 내 메시지로 보정되는 케이스가 있습니다.

원인: 실시간 이벤트와 REST 응답의 소유자 판별 정보 일관성 부족.

---

## 2) 요청 목적

`CHAT_MESSAGE`의 소유자 판별(내 메시지/상대 메시지)을
실시간/재조회 경로 모두에서 동일하게 보장하기 위함.

---

## 3) 요청 항목

### 3.1 SignalR `ReceiveMessage` payload 필드 보강

아래 필드를 항상 포함 요청드립니다.

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| messageId | int | ✅ | 서버 저장 메시지 ID |
| roomId | int | ✅ | 채팅방 ID |
| senderId | int | ✅ | 발신자 사용자 ID |
| senderNickname | string | ✅ | 발신자 닉네임 |
| message | string? | ✅ | 메시지 본문 (이미지만 전송 시 null 가능) |
| type | string | ✅ | `TEXT`, `IMAGE`, ... |
| createdAt | string(ISO8601) | ✅ | 생성 시각 |
| isMyMessage | bool | 권장 | 현재 수신 사용자 기준 내 메시지 여부 |
| clientMessageId | string | 권장 | 클라이언트 임시 메시지 식별자 (reconcile 용) |

### 3.2 `POST /api/chat/messages` 응답 필드 보강

전송 응답에도 `clientMessageId` echo를 요청드립니다.

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| messageId | int | ✅ | 서버 저장 메시지 ID |
| clientMessageId | string | 권장 | 요청에서 받은 clientMessageId를 그대로 반환 |

### 3.3 REST/SignalR 일관성 보장

- 동일 `messageId`에 대해 아래 값이 일관되어야 합니다.
  - `senderId`
  - `message`
  - `type`
  - `createdAt`
  - `isMyMessage` (제공 시)

---

## 4) 예시

### 4.1 SignalR `ReceiveMessage` 예시
```json
{
  "messageId": 1287,
  "roomId": 42,
  "senderId": 17,
  "senderNickname": "판매자A",
  "senderProfileImage": "https://cdn.tickethub.com/profiles/17.jpg",
  "message": "네, 아직 가능합니다.",
  "type": "TEXT",
  "createdAt": "2026-02-12T10:21:32Z",
  "isMyMessage": false,
  "clientMessageId": "a3dc8578-1f30-4f6c-bf01-0dc0f88e41b4"
}
```

### 4.2 `POST /api/chat/messages` 응답 예시
```json
{
  "message": "메시지 전송 성공",
  "data": {
    "messageId": 1287,
    "roomId": 42,
    "senderId": 17,
    "senderNickname": "판매자A",
    "message": "네, 아직 가능합니다.",
    "createdAt": "2026-02-12T10:21:32Z",
    "success": true,
    "clientMessageId": "a3dc8578-1f30-4f6c-bf01-0dc0f88e41b4"
  },
  "statusCode": 200,
  "success": true
}
```

---

## 5) 수용 기준 (Acceptance Criteria)

1. 동일 메시지가 실시간 수신 직후/재진입 후에도 동일한 좌우 정렬(내 메시지/상대 메시지)로 표시된다.
2. `senderId`와 `isMyMessage`(제공 시)가 실시간/REST에서 모순되지 않는다.
3. `clientMessageId` echo 제공 시, 모바일은 중복 append 없이 임시 메시지를 안정적으로 치환할 수 있다.

---

## 6) 참고

- 관련 모바일 이슈: 전송 직후 메시지 방향 반전 후 재진입 시 복구
- 관련 문서: `docs/mobile/api_specs/CHAT_MESSAGE_payload_request.md`

---

## 7) 반영 결과 (Backend)

- `SendMessageReqDto`에 `clientMessageId` 추가
- `POST /api/chat/messages` 응답(`SendMessageRespDto`)에 `clientMessageId` echo 추가
- SignalR `ReceiveMessage` payload(`NewMessageSignalDto`)에 아래 필드 반영
  - `senderId` (기존 유지)
  - `isMyMessage` (추가)
  - `clientMessageId` (추가)
- `ChatController.SendMessage`에서 동일 payload를 room 브로드캐스트하지 않고
  발신자/수신자 user 그룹으로 각각 전송하여 `isMyMessage` 값 일관성 보장
