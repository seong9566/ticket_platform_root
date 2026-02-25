# F4 Scope Fidelity Check
Date: 2026-02-25

## Scope Creep Analysis

검사 기준: `git diff --name-only HEAD~10 HEAD` (fallback 포함) + `git log --oneline -20`

### 계획 범위 내 변경 파일
| 파일 | 상태 | 비고 |
|------|------|------|
| TicketPlatFormServer/Program.cs | ✅ 계획 내 | DI 관련 변경으로 감지됨 (경로 표기는 히스토리 기준) |

### 계획 범위 외 변경 파일 (스코프 크립)
최근 10커밋 변경 파일 53개 중 52개가 계획 범위 외 파일이다.

| 파일 | 변경 내용 | 심각도 |
|------|---------|--------|
| TicketPlatFormServer/Controllers/AuthController.cs | 소셜 로그인/인증 도메인 변경 | 높음 |
| TicketPlatFormServer/Controllers/NotificationController.cs | 알림 API 도메인 변경 | 높음 |
| TicketPlatFormServer/Controllers/UserController.cs | 사용자 도메인 변경 | 중간 |
| TicketPlatFormServer/Services/Notification/NotificationService.cs | 알림 비즈니스 로직 변경 | 높음 |
| TicketPlatFormServer/Services/Notification/FcmService.cs | FCM 연동 로직 추가/변경 | 높음 |
| TicketPlatFormServer/Services/Auth/GoogleOAuthService.cs | OAuth 연동 변경 | 높음 |
| TicketPlatFormServer/Repository/Notification/NotificationRepository.cs | 알림 저장소 변경 | 높음 |
| TicketPlatFormServer/DTO/Notification/RegisterTokenReqDto.cs | 알림 DTO 변경 | 중간 |
| TicketPlatFormServer/api_spec/signalr_realtime_notification_api.md | 계획 외 API 문서 변경 | 낮음 |
| .idea/.idea.TicketPlatFormServer/.idea/vcs.xml | IDE 설정 파일 변경 | 낮음 |

추가 점검:
- 금지 영역(`Settlement*.cs`, `settlement_*.dart`) 변경: 없음
- 금지 영역(`*.g.dart`, `*.freezed.dart`) 변경: 없음

## Missing Items Analysis [0/4]
| # | 항목 | 충족 여부 | 근거 |
|---|------|---------|------|
| 1 | 5개 endpoint 매핑 | ✅ | `.sisyphus/evidence/toss-bank-verification-gap-matrix.md`에 `lookup-holder-name`, `validate`, `verify-identifier`, `verify-holder-name`, `verify-holder-real-name` 5개 모두 매핑 |
| 2 | Provider 전환 전략(Custom/Toss/Hybrid) 문서화 | ✅ | `.sisyphus/evidence/toss-provider-routing.md`에 인터페이스/라우팅/DI/Unknown fallback 명시 |
| 3 | 정산 `on_hold -> pending` 연계 시나리오 검증 | ✅ | `.sisyphus/evidence/task-15-settlement-gate.txt` + `TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs`의 `ResumeHeldSettlementsAsync` 구현 |
| 4 | 롤백 절차(설정 전환 + 데이터 무결성) 확인 | ✅ | `.sisyphus/evidence/task-16-rollback-runbook.md`에 Provider 설정 전환, 10분 내 롤백 목표, SQL 무결성 체크 포함 |

## Scope: FAIL
## Missing: 0
## VERDICT: FAIL
- Reason: 누락 항목은 0건이지만, 최근 10커밋 기준 변경 파일의 대다수(52/53)가 `toss-bank-account-gap-analysis` 계획 정의 범위 외(Auth/Notification/FCM/OAuth/IDE 설정 등)로 확인되어 스코프 크립이 발생했다.
