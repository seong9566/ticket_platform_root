# Toss 계좌인증 API 계약 정리 (T2)

기준: https://docs.tosspayments.com/reference/additional#계좌인증

## 공통 규칙
- `bankCode`: 2자리 또는 3자리 코드
- `accountNumber`: 하이픈 없이 숫자, 최대 14자리
- 테스트 환경: 실제 인증 절차 없이 Mock 데이터 응답 가능
- 실패 응답: HTTP 상태코드 + 에러 객체

## Endpoint 계약

### 1) 계좌번호로 예금주명 조회
- Method/Path: `POST /v2/bank-accounts/lookup-holder-name`
- Request
  - `bankCode` (required)
  - `accountNumber` (required)
- Success Response
  - `holderName`
- 주요 용도
  - 계좌 입력 시 예금주명 사전 확인

### 2) 계좌번호 유효성 확인
- Method/Path: `POST /v2/bank-accounts/validate`
- Request
  - `bankCode` (required)
  - `accountNumber` (required)
- Success Response
  - `isValid` (boolean)

### 3) 계좌번호-식별자 일치 확인
- Method/Path: `POST /v2/bank-accounts/verify-identifier`
- Request
  - `bankCode` (required)
  - `accountNumber` (required)
  - `identityNumber` (required, 개인: 생년월일 YYMMDD / 법인: 사업자번호)
- Success Response
  - `isValid` (boolean)

### 4) 계좌번호-예금주명 일치 확인
- Method/Path: `POST /v2/bank-accounts/verify-holder-name`
- Request
  - `bankCode` (required)
  - `accountNumber` (required)
  - `holderName` (required, 최대 50자)
- Success Response
  - `isValid` (boolean)

### 5) 계좌번호-예금주명-식별자 일치 확인
- Method/Path: `POST /v2/bank-accounts/verify-holder-real-name`
- Request
  - `bankCode` (required)
  - `accountNumber` (required)
  - `holderName` (required)
  - `identityNumber` (required)
- Success Response
  - `isValid` (boolean)

## 재시도/에러 분류 가이드 (계약 기반)

아래는 구현 정책이 아니라, 계약 적용 시 안전한 기본 분류다.

### Retryable
- 네트워크 타임아웃, 연결 오류
- HTTP `429` (rate limit)
- HTTP `5xx`

### Non-retryable (Terminal)
- 요청 포맷 오류/누락 (HTTP `400` 계열)
- 인증 불일치(`isValid=false`) 등 비즈니스 검증 실패

### 조건부
- 인증/권한 문제(설정 오류 가능성) -> 즉시 재시도보다 설정 점검 우선

## 현재 앱 계약과의 차이
- 현재 백엔드 검증 응답: `verified`, `message` 중심
  - `TicketPlatFormServer/TicketPlatFormServer/DTO/BankAccount/VerifyAccountResponseDto.cs`
- 현재 모바일 소비 필드도 동일
  - `ticket_platform_mobile/lib/features/bank_account/data/dto/response/verify_account_resp_dto.dart`
- Toss 계좌인증 연동 시 내부 표준 응답으로 매핑 레이어가 필요
  - 권장 내부 확장 필드: `provider`, `verificationTier`, `reasonCode`
