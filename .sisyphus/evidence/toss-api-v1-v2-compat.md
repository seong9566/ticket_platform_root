# API 계약안 v1 호환 + v2 확장 (T7)

## 목표
- 기존 모바일 클라이언트를 깨지 않으면서 Toss 기반 검증 메타데이터를 확장한다.

## 현재 v1 계약

### Request
- `POST /api/bank-account/verify/request`
  - body 없음 (현재 사용자 컨텍스트 기반)
- `POST /api/bank-account/verify/confirm`
  - `{ "code": "1234" }`

### Response
- `verify/request`: `{ "expiresAt": "ISO", "message": "..." }`
- `verify/confirm`: `{ "verified": true|false, "message": "..." }`

근거:
- `TicketPlatFormServer/TicketPlatFormServer/DTO/BankAccount/RequestVerificationResponseDto.cs`
- `TicketPlatFormServer/TicketPlatFormServer/DTO/BankAccount/VerifyAccountResponseDto.cs`
- `ticket_platform_mobile/lib/features/bank_account/data/dto/response/request_verification_resp_dto.dart`
- `ticket_platform_mobile/lib/features/bank_account/data/dto/response/verify_account_resp_dto.dart`

## v2 확장안 (하위호환)

### 원칙
- 기존 필드 유지 (`expiresAt`, `message`, `verified`)
- 신규 필드는 optional 추가
- 기존 endpoint 경로 유지 가능, 또는 별도 `v2` endpoint 병행 제공

### verify/request 응답 확장
```json
{
  "expiresAt": "2026-02-25T03:00:00Z",
  "message": "계좌 인증 요청이 완료되었습니다.",
  "provider": "TOSS",
  "verificationStatus": "PENDING",
  "verificationTier": "TIER_0_NONE",
  "reasonCode": null
}
```

### verify/confirm 응답 확장
```json
{
  "verified": true,
  "message": "계좌 인증이 완료되었습니다.",
  "provider": "TOSS",
  "verificationStatus": "VERIFIED",
  "verificationTier": "TIER_3_REAL_NAME_MATCH",
  "reasonCode": null
}
```

## 모바일 영향도

| 필드 | 기존 모바일 사용 | 변경 필요 |
|---|---|---|
| `verified` | 사용함 | 없음 |
| `message` | 사용함(에러 표시 일부) | 없음 |
| `expiresAt` | 사용함(타이머) | 없음 |
| `provider` | 미사용 | 선택적 UI 개선 |
| `verificationStatus` | 미사용 | 선택적 UI 개선 |
| `verificationTier` | 미사용 | 선택적 UI 개선 |
| `reasonCode` | 미사용 | 실패 문구 매핑 시 필요 |

## 호환 기간
- 최소 1개 앱 릴리즈 주기 동안 v1 필드 보장
- 서버는 v2 필드 추가 후에도 v1-only 클라이언트 정상 동작 유지

## 금지사항
- `verified` 타입 변경 금지
- `expiresAt` 포맷 변경 금지(ISO-8601 유지)
- 기존 응답 래퍼 구조(`ApiResponse<T>`) 변경 금지
