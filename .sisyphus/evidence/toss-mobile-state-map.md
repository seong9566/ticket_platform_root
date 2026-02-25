# 모바일 상태/문구/전환 맵 (T10)

## 상태 머신

| 상태 | 진입 조건 | UI 액션 |
|---|---|---|
| `idle` | 계좌 로드 완료, 미인증 | 인증 요청 버튼 노출 |
| `requesting` | 인증 요청 API 호출 중 | 로딩 표시 |
| `codeInput` | `expiresAt` 수신 성공 | 코드 입력 + 타이머 |
| `verified` | 인증 성공 | 인증완료 상태/상세 화면 |
| `error` | 요청/확인 실패 | 오류 텍스트 + 재시도 |

근거: `ticket_platform_mobile/lib/features/bank_account/presentation/viewmodels/bank_account_viewmodel.dart`

## 프로필 진입 상태 문구

| 조건 | trailingText | 색상 |
|---|---|---|
| 계좌 없음 | `등록하기` | secondary |
| 계좌 있음 + verified | `인증 완료` | success |
| 계좌 있음 + 미인증 | `인증하기` | warning |

근거: `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart`

## 판매 대시보드 가이드

| 조건(statusCode) | 표시 텍스트 | CTA |
|---|---|---|
| `settlement_on_hold` 또는 `on_hold` | `정산 보류` + `계좌 인증 필요` | `인증하러가기` |

근거:
- `ticket_platform_mobile/lib/features/sales_dashboard/presentation/ui_models/event_ticket_ui_model.dart`
- `ticket_platform_mobile/lib/features/sales_dashboard/presentation/widgets/event_ticket_item.dart`

## reasonCode 문구 매핑(v2 확장용)

| reasonCode | 사용자 메시지 |
|---|---|
| `INVALID_INPUT` | 입력 정보를 다시 확인해주세요. |
| `ACCOUNT_MISMATCH` | 계좌/예금주 정보가 일치하지 않습니다. |
| `IDENTITY_MISMATCH` | 본인(사업자) 정보가 일치하지 않습니다. |
| `UPSTREAM_TEMPORARY` | 은행 점검 중입니다. 잠시 후 다시 시도해주세요. |
| `PROVIDER_AUTH_ERROR` | 인증 서비스 설정 문제입니다. 잠시 후 다시 시도해주세요. |
