# 계좌인증 Tier/상태 모델 설계 (T3)

## 목표
- 기존 `Verified` 단일 boolean을 보존하면서도, 인증의 신뢰수준과 출처를 분리 표현한다.
- 정산 정책(`on_hold -> pending`)을 깨지 않게 단계적으로 Provider를 교체한다.

## 제안 모델

### Verification Provider
- `CUSTOM`: 내부 4자리 코드 검증
- `TOSS`: 토스 계좌인증 API 검증
- `HYBRID`: Toss 우선 + 정책 기반 fallback(Custom)

### Verification Tier
- `TIER_0_NONE`: 미인증
- `TIER_1_CONTROL_PROOF`: 사용자 입력 코드 기반 통제 증명(현행 CUSTOM)
- `TIER_2_ACCOUNT_VALID`: 계좌 유효성/예금주 단순 검증
- `TIER_3_REAL_NAME_MATCH`: 계좌+예금주(+식별자) 실명 일치

### Verification Status
- `UNVERIFIED`
- `PENDING`
- `VERIFIED`
- `FAILED`
- `EXPIRED`

## 상태 전이

| From | Trigger | To | 비고 |
|---|---|---|---|
| `UNVERIFIED` | 인증 요청 | `PENDING` | 코드 발급 또는 Toss 검증 실행 |
| `PENDING` | 검증 성공 | `VERIFIED` | Tier 갱신 |
| `PENDING` | 코드 만료 | `EXPIRED` | 재요청 필요 |
| `PENDING` | 검증 실패 | `FAILED` | 재시도 가능 |
| `FAILED`/`EXPIRED` | 재요청 | `PENDING` | 재검증 루프 |

`UNVERIFIED -> COMPLETED` 직접 전이는 금지한다.

## 정산 게이트 매트릭스

| 검증 상태 | Tier | 구매확정 허용 | 정산 상태 |
|---|---|---|---|
| `UNVERIFIED` | `TIER_0_NONE` | 허용 | `on_hold` |
| `PENDING` | `TIER_0/1` | 허용 | `on_hold` |
| `VERIFIED` | `TIER_1+` | 허용 | `pending` |
| `FAILED`/`EXPIRED` | any | 허용 | `on_hold` |

근거 코드:
- 보류 생성: `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs`
- 보류 재개: `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs`

## 하위호환 원칙
- 기존 `Verified`는 유지하고, 새 필드는 부가 메타데이터로 추가한다.
- 기존 모바일/백엔드 계약은 파괴하지 않고 optional 필드로 확장한다.

## 최소 저장 필드 제안
- `verification_provider` (`CUSTOM|TOSS|HYBRID`)
- `verification_tier` (`TIER_0..3`)
- `verification_status` (`UNVERIFIED|PENDING|VERIFIED|FAILED|EXPIRED`)
- `last_verification_failure_code` (nullable)
- `last_verified_at` (기존 `verified_at` 재사용 가능)
