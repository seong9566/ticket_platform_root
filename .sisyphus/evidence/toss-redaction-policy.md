# 계좌인증/정산 연동 보안 및 로그 레드액션 정책 (T4)

## 목적
- 토스 계좌인증 도입 시 PII 노출 위험을 줄이고, 운영 추적성은 유지한다.

## 데이터 분류

### 민감정보(원문 로그 금지)
- `accountNumber`
- `holderName`
- `identityNumber`
- Toss 원문 response body (`RawResponse` 전체)

### 제한 로그 허용(진단 목적)
- 내부 `requestId`/`correlationId`
- 결과 상태(`VERIFIED`, `FAILED` 등)
- 표준화된 `reasonCode`
- HTTP status code

## 저장/전송 정책
- 저장 최소화: 검증 결과와 사유코드 중심으로 저장
- 전송 최소화: 필요한 필드만 Toss API로 전달
- UI 응답 최소화: 사용자에게는 reasonCode 기반 메시지 제공, 민감 원문 미노출

## 로깅 정책

### MUST NOT
- 예외 로그에 민감 필드를 문자열 보간으로 직접 출력하지 않는다.
- `RawResponse`를 영구 저장하지 않는다.

### MUST
- 민감 필드는 마스킹 후 출력한다.
  - 계좌번호: `********1234`
  - 예금주명: `홍*동`
  - 식별자: `******` (전면 마스킹)
- 실패 원인은 표준 코드로 정규화한다.

## 현재 코드 기준 리스크 포인트
- `TransferResponseDto.RawResponse`에 응답 원문 저장 중  
  - 참조: `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/TossPaymentsService.cs`
- 글로벌 예외 로깅이 `InnerException.Message`를 그대로 남길 수 있음  
  - 참조: `TicketPlatFormServer/TicketPlatFormServer/Common/Exception/GlobalExceptionMiddleware.cs`

## 개선 체크리스트
- [ ] Raw response 저장 정책 제거 또는 암호화+단기 TTL 저장으로 제한
- [ ] 민감 필드 레드액션 유틸 도입(서비스 공통)
- [ ] 로그 스키마 표준화(`event`, `reasonCode`, `status`, `requestId`)
- [ ] 에러 응답 메시지와 내부 로그 메시지 분리

## 샘플 로그(허용)
```text
[BankVerification] status=FAILED reasonCode=ACCOUNT_MISMATCH provider=TOSS requestId=abc123 httpStatus=400
```

## 샘플 로그(금지)
```text
[BankVerification] holderName=홍길동 accountNumber=12345678901234 identityNumber=900101
```
