# 정산 게이트 연계 시나리오 (T9)

## 기준 로직
- 보류 생성: `PaymentService.ReleaseEscrowAsync` (`on_hold`)
- 보류 해제: `PaymentService.ResumeHeldSettlementsAsync` (`pending`)

## 상태 전이 매트릭스

| 계좌 검증 상태 | 구매확정 처리 | 정산 상태 | 비고 |
|---|---|---|---|
| 미등록/미인증 | 허용 | `on_hold` | 계좌 미인증으로 지급 보류 |
| 인증 진행중 | 허용 | `on_hold` | 만료/실패 시 유지 |
| 인증 성공 | 허용 | `pending` | 배치 처리 대상 |
| 인증 실패 | 허용 | `on_hold` | 재검증 전까지 유지 |

## 핵심 시나리오
1. 구매확정 시점에 인증 계좌 없음 -> `on_hold` 생성
2. 인증 완료 이벤트 -> 동일 판매자의 `on_hold` 일괄 `pending` 전환
3. `pending`은 `SettlementProcessingService`가 주기 처리

## 금지 전이
- `UNVERIFIED -> COMPLETED` 직접 전이 금지
- `FAILED -> pending` 직접 전이 금지 (재검증 성공 이벤트 필요)

## 검증 체크 포인트
- `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs:527`
- `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs:577`
- `TicketPlatFormServer/TicketPlatFormServer/Services/BackgroundServices/SettlementProcessingService.cs`
