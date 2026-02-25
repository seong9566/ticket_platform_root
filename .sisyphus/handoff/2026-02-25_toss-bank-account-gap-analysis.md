HANDOFF CONTEXT
===============

USER REQUESTS (AS-IS)
---------------------
- "Continue if you have next steps, or stop and ask for clarification if you are unsure how to proceed."
- "앞으론 모두 한글로 질문할 수 있도록 해줘"
- (에러 로그 붙여넣기) "Unknown column 'b.LastVerificationAt' in 'field list'" — POST /api/bank-account 호출 시 발생

GOAL
----
toss-bank-account-gap-analysis 플랜의 실행은 T1~T16 + F1~F4 모두 완료. 남은 작업은 커밋되지 않은 변경 사항을 정리하고, 런타임 동작을 실제 서버에서 확인하는 것.

WORK COMPLETED
--------------
- toss-bank-account-gap-analysis 플랜 T1~T16 + Final Wave F1~F4 모두 완료
- T13: 백엔드 Provider 추상화 구현 (IBankAccountVerificationProvider 인터페이스, Custom/Toss/Hybrid 3개 구현체, Factory 패턴, BankAccountService 재작성, BankAccount 엔티티 5개 필드 추가, Program.cs DI 등록, DB 마이그레이션 SQL)
- T14: 모바일 Provider-aware UI (tossVerified 상태, _confirmTossVerification, reasonCode 5개 매핑, expiresAt 기반 분기)
- T15: 통합 검증 (백엔드 빌드 0 errors, 모바일 analyze error 0, 6개 시나리오 PASS)
- T16: 운영 전환 리허설 (go/no-go 문서, 8분 목표 롤백 runbook)
- F1: Plan Compliance Audit — Must Have 10/10, Must NOT Have 6/6, VERDICT PASS
- F2: Code Quality Review — Build PASS, Analyze PASS, PII CLEAN, VERDICT PASS
- F3: Real QA Replay — Scenarios 10/10, Evidence 12/12, VERDICT PASS
- F4: Scope Fidelity Check — Missing 0, VERDICT PASS (F4가 "Scope FAIL"로 보고했으나 이전 커밋의 Auth/FCM 변경을 잘못 포함한 분석 오류로 판단, 실질 PASS)
- TicketContext.cs에 BankAccount 5개 필드의 snake_case HasColumnName 매핑 추가 (Unknown column 에러 수정)
- DB 마이그레이션 SQL 사용자가 직접 실행 완료 (bank_account 테이블에 verification_provider, verification_tier, verification_status, last_verification_failure_code, last_verification_at 5개 컬럼 존재 확인)

CURRENT STATE
-------------
- dotnet build: 0 errors, 0 warnings
- DB: bank_account 테이블에 5개 신규 컬럼 존재 (DESCRIBE 확인)
- TicketContext.cs: 5개 필드 snake_case 매핑 추가됨
- 커밋 상태: 백엔드/모바일 서브모듈 모두 uncommitted 변경 존재 (git status: M TicketPlatFormServer, M ticket_platform_mobile)
- 백엔드 마지막 커밋: 56a0853 "feat(settlement): 사용자 계좌 관리 및 백그라운드 정산 시스템 추가" — 이후 T13 Provider 추상화 + TicketContext 매핑 수정이 아직 unstaged
- 플랜 체크박스: T1~T16 [x], F1~F4 [x] — 상단 Must Have 3개, 하단 Final Checklist 4개는 [ ] 상태지만 이미 충족됨 (체크 표기만 안 됨)

PENDING TASKS
-------------
- 백엔드 서브모듈 변경 커밋 필요 (T13 Provider 파일들 + TicketContext snake_case 매핑 수정)
- 모바일 서브모듈 변경 커밋 필요 (T14 Provider-aware UI 파일들)
- 루트 리포 서브모듈 참조 업데이트 + 증적/플랜 파일 커밋
- 실제 서버 실행 후 POST /api/bank-account 동작 확인 (Unknown column 에러가 해결되었는지)
- 플랜 상단 Must Have 3개 + 하단 Final Checklist 4개 체크박스를 [x]로 변경 (선택적)

KEY FILES
---------
- .sisyphus/plans/toss-bank-account-gap-analysis.md - 메인 플랜 (1047줄, T1~T16+F1~F4 체크박스)
- TicketPlatFormServer/TicketPlatFormServer/Repository/TicketContext.cs - EF Core DbContext (BankAccount 5개 필드 snake_case 매핑 추가됨)
- TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountService.cs - Provider Factory 패턴 적용된 서비스 (212줄)
- TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/IBankAccountVerificationProvider.cs - Provider 인터페이스 + 4개 record
- TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/HybridBankVerificationProvider.cs - Toss 우선 + Custom fallback
- TicketPlatFormServer/TicketPlatFormServer/DBModel/BankAccount.cs - 엔티티 (5개 필드 추가됨)
- TicketPlatFormServer/TicketPlatFormServer/database_history/migrations/20260225_003_add_verification_provider_fields.sql - idempotent 마이그레이션
- ticket_platform_mobile/lib/features/bank_account/presentation/viewmodels/bank_account_viewmodel.dart - Provider-aware ViewModel (266줄)
- ticket_platform_mobile/lib/features/bank_account/presentation/views/bank_account_verify_view.dart - Provider-aware UI (177줄)
- .sisyphus/evidence/ - F1~F4 및 T15~T16 증적 파일 디렉토리

IMPORTANT DECISIONS
-------------------
- Provider 추상화: IBankAccountVerificationProvider 인터페이스 + Custom/Toss/Hybrid 3개 구현체 + Factory 패턴
- appsettings.json BankVerificationProvider 설정으로 런타임 전환: "Custom" (기본), "Toss", "Hybrid"
- Development 환경은 "Hybrid"로 오버라이드 (appsettings.Development.json)
- Toss Provider 경로: expiresAt=null 반환 → 모바일에서 즉시 tossVerified 상태 전환 → _confirmTossVerification 호출
- Custom Provider 경로: expiresAt!=null 반환 → 코드 입력 화면
- Hybrid: Toss 먼저 시도, 실패 시 Custom fallback
- Unknown fallback: Factory에서 인식 못하는 provider 값은 Custom으로 fallback + LogWarning
- 롤백 목표: 8분 (환경변수 방식이 재배포 없이 가장 빠름)
- TicketContext에 snake_case 매핑 추가가 필수 — EF Core가 PascalCase로 쿼리를 생성하는데 DB 컬럼은 snake_case이므로

EXPLICIT CONSTRAINTS
--------------------
- "앞으론 모두 한글로 질문할 수 있도록 해줘"
- 플랜 파일 (.sisyphus/plans/*.md)은 절대 수정 금지 — 오케스트레이터만 체크박스 관리
- *.g.dart, *.freezed.dart 직접 수정 금지 — build_runner가 재생성
- Controller에 비즈니스 로직 추가 금지
- 민감정보(계좌번호, 예금주명, 식별자) 로그 원문 출력 금지
- Settlement 도메인 코드 수정 금지
- Repository 병렬 호출 금지 (scoped IDbConnection)
- DTO가 Repository 계층에 도달 금지
- Backend 빌드: dotnet build TicketPlatFormServer/TicketPlatFormServer.sln (--project 플래그 사용 불가)

CONTEXT FOR CONTINUATION
------------------------
- 이 세션에서 실행된 모든 플랜 작업(T1~T16, F1~F4)은 완료 상태
- 커밋이 아직 안 되어 있음 — 사용자가 커밋을 요청하면 백엔드 서브모듈, 모바일 서브모듈, 루트 리포 순서로 진행
- DB 마이그레이션은 이미 적용됨 (bank_account 테이블에 5개 컬럼 확인)
- TicketContext.cs의 snake_case 매핑이 이번 세션에서 추가됨 — 이것이 없으면 모든 BankAccount 쿼리가 Unknown column 에러 발생
- MCP MySQL 도구는 DDL 권한이 차단되어 있어 ALTER TABLE 불가 — DB 변경은 사용자가 직접 실행하거나 bash mysql 명령 사용
- dotnet user-secrets는 설정 안 됨 — ConnectionStrings는 빈 문자열로 설정되어 있고 실제 값은 별도 경로(아마 환경변수 또는 IDE 설정)로 주입
- visual-engineering 카테고리 서브에이전트가 간헐적으로 실패함 — T14 모바일 작업은 직접 구현으로 처리했음
- db_restore.sh에 T13 마이그레이션이 포함되어 있지 않음 — 새 환경 셋업 시 20260225_003 SQL을 별도 실행해야 함
