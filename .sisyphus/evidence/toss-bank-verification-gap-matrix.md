# Toss 계좌인증 갭 매트릭스 (T1)

기준 문서: https://docs.tosspayments.com/reference/additional#계좌인증  
분석 기준 시점: 2026-02-25

## 현재 내부 구현 요약
- 공개 API: `POST /api/bank-account/verify/request`, `POST /api/bank-account/verify/confirm` (`TicketPlatFormServer/TicketPlatFormServer/Controllers/BankAccountController.cs`)
- 인증 방식: 서버가 4자리 코드 발급/만료/시도 제한 후 사용자 입력 검증 (`TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs`)
- 정산 연계: 미인증 계좌는 `on_hold`, 인증 성공 시 `pending` 전환 (`TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs`)
- 토스 연동 현황: 지급대행 `/v2/payouts`만 구현, 토스 계좌인증 5종 미구현 (`TicketPlatFormServer/TicketPlatFormServer/Services/Payment/TossPaymentsService.cs`)

## 1:1 매핑 표

| Toss 공식 기능 | Toss Endpoint | 현재 구현 매핑 | 구현 상태 | 근거 |
|---|---|---|---|---|
| 계좌번호로 예금주명 조회 | `POST /v2/bank-accounts/lookup-holder-name` | 내부 API에 동등 기능 없음 | 미구현 | `TicketPlatFormServer/TicketPlatFormServer/Controllers/BankAccountController.cs` |
| 계좌번호 유효성 확인 | `POST /v2/bank-accounts/validate` | 내부 API에 동등 기능 없음 | 미구현 | `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs` |
| 계좌번호-식별자 일치 | `POST /v2/bank-accounts/verify-identifier` | 내부 API는 식별자 필드 자체 없음 | 미구현 | `TicketPlatFormServer/TicketPlatFormServer/DTO/BankAccount/VerifyAccountRequestDto.cs` |
| 계좌번호-예금주명 일치 | `POST /v2/bank-accounts/verify-holder-name` | 내부는 예금주명 외부 검증 없이 4자리 코드만 검증 | 부분구현(의도는 유사, 검증 수단 상이) | `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs` |
| 계좌번호-예금주명-식별자 일치 | `POST /v2/bank-accounts/verify-holder-real-name` | 내부 API에 복합실명 검증 없음 | 미구현 | `TicketPlatFormServer/TicketPlatFormServer/DTO/BankAccount/RegisterBankAccountRequestDto.cs` |

## 추가 관찰 (모바일/정산)
- 모바일 DTO는 `verified`, `message`, `expiresAt` 중심이며 provider/tier/reasonCode 미포함  
  - `ticket_platform_mobile/lib/features/bank_account/data/dto/response/verify_account_resp_dto.dart`
  - `ticket_platform_mobile/lib/features/bank_account/data/dto/response/request_verification_resp_dto.dart`
- 프로필 진입점은 현재 항상 등록 화면으로 라우팅(상세/인증 분기 주석 처리)  
  - `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart`
- 판매 대시보드는 `on_hold`/`settlement_on_hold` 안내는 이미 표시 가능  
  - `ticket_platform_mobile/lib/features/sales_dashboard/presentation/ui_models/event_ticket_ui_model.dart`

## 결론
- Toss 계좌인증 관점에서 현재 백엔드는 **5개 중 4개 미구현, 1개 부분구현**이다.
- 단, 정산 보류/재개 정책(`on_hold -> pending`)은 이미 운영 가능한 상태이므로, 인증 Provider 교체를 점진적으로 적용할 수 있다.
