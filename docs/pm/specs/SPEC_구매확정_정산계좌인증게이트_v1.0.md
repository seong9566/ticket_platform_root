# SPEC_구매확정_정산계좌인증게이트_v1.0

**작성일**: 2026-02-24  
**작성자**: PM/개발 협업 초안  
**담당 팀**: Backend + Mobile  
**상태**: 🚧 진행중 (기능 기획 확정)  
**버전**: 1.0

---

## 1) 개요

구매자가 `구매 확정`을 눌렀을 때 거래 자체는 정상 완료하되, 판매자 계좌 인증이 끝나지 않았다면 **정산 지급만 보류**하는 정책을 도입한다.

핵심 목표:
- 구매자 UX를 막지 않는다 (구매 확정 성공 유지)
- 판매자 미인증 계좌로 정산이 나가지 않도록 한다
- 계좌 인증 완료 시 보류 정산이 자동으로 재개되도록 한다

---

## 2) 배경 및 현재 동작 (AS-IS)

현재 서버 흐름:
1. 구매 확정 시 `ChatService.ConfirmPurchase`에서 `PaymentService.ReleaseEscrowAsync(transactionId)`를 호출한다.
2. `ReleaseEscrowAsync`에서 인증된 판매자 계좌(`bank_account.verified = true`)를 조회한다.
3. 계좌가 없어도 에러로 막지 않고 `settlements`를 `pending`으로 생성하며 `bank_account_id = null`일 수 있다.

확인된 근거 파일:
- `TicketPlatFormServer/TicketPlatFormServer/Services/Chat/ChatService.cs`
- `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs`
- `TicketPlatFormServer/TicketPlatFormServer/DBModel/BankAccount.cs`
- `TicketPlatFormServer/TicketPlatFormServer/DBModel/UserVerification.cs`

모바일 현황:
- 프로필에 `계좌 인증` 메뉴 항목은 있으나 실사용 인증 플로우는 미구현 상태.
- `ticket_platform_mobile/lib/features/profile/presentation/views/profile_view.dart`

---

## 3) 정책 결정 (확정)

사용자 결정 정책:
- **구매확정 허용 + 정산보류**

즉, 판매자 계좌 미인증이어도 구매자 `구매 확정`은 성공해야 하며,
정산 지급 상태만 `계좌 인증 대기`로 보류한다.

---

## 4) 목표 상태 (TO-BE)

### 4.1 구매 확정 시 동작

- **Case A: 판매자 인증 계좌 존재**
  - 기존처럼 정산 생성 (`pending`)
  - `bank_account_id` 세팅
  - 스케줄 시점 기준 정산 처리 진행

- **Case B: 판매자 인증 계좌 없음**
  - 거래는 `confirmed`로 완료
  - 정산은 `on_hold_account_verification`(신규)로 생성
  - `bank_account_id = null`, `scheduled_at = null` (또는 처리 제외 값)
  - 판매자에게 계좌 인증 유도 알림 발송

### 4.2 계좌 인증 완료 시 동작

- 판매자 계좌 인증 완료 이벤트 발생 시:
  - 해당 판매자의 `on_hold_account_verification` 정산건 조회
  - 기본 인증 계좌 연결
  - `pending`으로 전환
  - `scheduled_at` 재설정(권장: 인증 완료 + 24시간)

---

## 5) 상태 머신 제안

### 5.1 정산 상태 (`settlement_statuses`)

기존:
- `pending`, `processing`, `completed`, `failed`

신규 추가:
- `on_hold_account_verification` = 계좌 인증 대기(지급 보류)

전이 규칙:
1. 구매확정 + 계좌 인증됨 -> `pending`
2. 구매확정 + 계좌 미인증 -> `on_hold_account_verification`
3. 계좌 인증 완료 -> `on_hold_account_verification` -> `pending`
4. 배치/정산잡 처리 -> `pending` -> `processing` -> `completed|failed`

---

## 6) 상세 요구사항

### 6.1 Backend

#### 비즈니스 규칙
- 구매확정 API의 성공/실패 기준에서 계좌 미인증은 실패 사유가 아니다.
- 정산 지급 가능 조건:
  - `bank_account.verified = true`
  - (권장) `user_verification.account_verified = true` 동기화
- 보류 정산은 일반 정산 배치 대상에서 제외한다.

#### 데이터/마이그레이션
- `settlement_statuses`에 `on_hold_account_verification` 추가.
- 기존 데이터 백필:
  - `settlements.status_id = pending_status_id AND bank_account_id IS NULL` -> `on_hold_account_verification` 전환.

#### API/서비스 변경 범위
- `ReleaseEscrowAsync`에서 계좌 유무에 따라 `pending`/`on_hold_account_verification` 분기.
- 계좌 인증 완료 후 보류 정산 재개 서비스(도메인 서비스 또는 배치 트리거) 추가.
- 판매자 알림 이벤트 추가:
  - 타입 예시: `ACCOUNT_VERIFICATION_REQUIRED_FOR_SETTLEMENT`

### 6.2 Mobile

#### 화면/UX
- 프로필 `계좌 인증` 메뉴를 실제 인증 플로우로 연결.
- 판매자에게 보류 상태를 명확히 표시:
  - 문구: `정산 보류 (계좌 인증 필요)`
  - CTA: `계좌 인증하러 가기`
- 구매자 화면에는 영향 없음(구매확정 성공 UX 유지).

#### 판매자 정산/판매내역 표시
- 상태 배지 추가:
  - `계좌 인증 대기` (`on_hold_account_verification`)
- 인증 완료 후 상태가 `정산 대기`로 바뀌는 것을 즉시 반영.

---

## 7) 수용 기준 (Acceptance Criteria)

- [ ] 판매자 계좌 미인증 상태에서도 구매확정 API가 성공한다.
- [ ] 위 케이스에서 정산 레코드가 `on_hold_account_verification`으로 생성된다.
- [ ] 계좌 인증 완료 시 보류 정산이 `pending`으로 전환된다.
- [ ] 보류 정산은 정산 실행 배치에서 제외된다.
- [ ] 판매자에게 계좌 인증 필요 알림이 발송된다.
- [ ] 모바일에서 `계좌 인증 대기` 상태와 인증 진입 CTA가 표시된다.
- [ ] 기존 인증 완료 판매자 플로우는 회귀 없이 정상 동작한다.

---

## 8) 예외/엣지 케이스

- 다중 계좌 보유: 기본 정산 계좌 선택 기준 필요 (최신 인증 계좌 또는 대표 계좌 플래그).
- 인증 해제/무효화: `pending`/`processing` 진입 전 계좌 재검증 필요.
- 구매확정 중복 호출: 정산 생성 idempotency 보장 필요.
- 과거 데이터 정합성: 이미 `pending + bank_account_id null`인 건 일괄 보류 전환.

---

## 9) 운영/모니터링 지표

- `on_hold_account_verification` 건수 (일/주)
- 보류 -> pending 전환 평균 소요시간
- 구매확정 성공률(정책 도입 전/후)
- 판매자 인증 완료율(알림 수신 후 24h/72h)

---

## 10) 구현 범위 (1차 릴리즈)

### In Scope
- 정산 보류 상태 도입 및 서버 분기 로직
- 보류 정산 재개 로직
- 모바일 계좌 인증 유도 UX + 보류 상태 노출
- 기본 알림 연동

### Out of Scope
- 고도화된 외부 실계좌 인증 벤더 연동 (오픈뱅킹/1원인증) 완전 자동화
- 운영자 백오피스 상세 워크플로우

---

## 11) 후속 결정 필요 항목

1. 계좌 인증 방식 (수동 승인 vs 1원 인증 vs 외부 실계좌 API)
2. 복수 인증 계좌일 때 기본 정산 계좌 선정 규칙
3. 보류 정산 재스케줄 기준 시각 (`인증 시점 + 24h` 유지 여부)

---

## 12) 관련 문서

- `docs/backend/tasks/TASK-006_Dispute_System.md`
- `TicketPlatFormServer/TicketPlatFormServer/api_spec/sell-sales-dashboard.md`
- `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs`
