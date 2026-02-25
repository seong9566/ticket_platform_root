# 에러 매핑/재시도/서킷브레이커 정책 (T8)

## 정책 목적
- Toss 계좌인증 호출 실패를 일관되게 처리하고, 장애 시 서비스 연속성을 확보한다.

## 에러 분류

### Retryable
- 네트워크 타임아웃
- HTTP `429`
- HTTP `5xx`
- Toss `COMMON_ERROR`, `BANK_ACCOUNT_VERIFICATION_ERROR`

### Terminal
- HTTP `400` 입력오류
- `INVALID_REQUEST`, `INVALID_ACCOUNT_NUMBER`
- 실명/계좌 불일치(`isValid=false`, `BANK_ACCOUNT_VERIFICATION_FAIL`)

### Conditional
- HTTP `401/403` -> 재시도보다 설정/권한 점검 우선

## 재시도 규칙
- 최대 재시도: 2회 (총 시도 3회)
- 백오프: 1s -> 3s
- jitter: +-300ms
- 재시도 대상: Retryable만

## fallback 규칙 (Hybrid)
- Toss 호출이 Retryable 오류로 3회 실패하면 Custom provider로 fallback
- Terminal 오류는 fallback하지 않고 사용자 입력 수정 유도

## 서킷브레이커 규칙
- 관찰창: 30초
- 실패율 임계치: 50%
- 최소 샘플: 20건
- 오픈 시간: 60초
- 오픈 중 동작: Toss 호출 생략 + Custom fallback(설정 허용 시)

## ReasonCode 표준화

| 원인 | reasonCode | 사용자 메시지 방향 |
|---|---|---|
| 입력 포맷 오류 | `INVALID_INPUT` | 입력값 확인 요청 |
| 계좌 불일치 | `ACCOUNT_MISMATCH` | 계좌/예금주 정보 재확인 |
| 식별자 불일치 | `IDENTITY_MISMATCH` | 본인/사업자 정보 재확인 |
| 외부 시스템 지연 | `UPSTREAM_TEMPORARY` | 잠시 후 재시도 |
| 권한/설정 오류 | `PROVIDER_AUTH_ERROR` | 운영자 점검 필요 |

## 운영 가드레일
- 모든 요청에 `requestId`/`correlationId` 부여
- 민감정보는 로그에서 마스킹 (`toss-redaction-policy.md` 준수)
- fallback 발생률을 지표화하여 운영 대시보드에 노출

## 적용 위치
- `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/TossPaymentsService.cs`
- (추가 예정) `TossBankVerificationProvider` 구현부
- 설정: `TicketPlatFormServer/TicketPlatFormServer/Config/TossPaymentsSettings.cs`
