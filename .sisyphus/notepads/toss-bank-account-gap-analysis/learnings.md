## 2026-02-25 T15 통합 검증
- 백엔드 빌드: 오류 0, 경고 0로 통과.
- 모바일 analyze: error 0 확인(기존 warning/info 23건 존재).
- 마이그레이션 SQL: information_schema 기반 idempotent ADD COLUMN 패턴 확인, DROP/DELETE 문 없음.
- Provider Factory: Custom/Toss/Hybrid 분기와 Unknown -> Custom fallback + LogWarning 확인.
- 정산 연계: ConfirmVerificationAsync에서 VERIFIED/LastVerificationAt 설정 후 ResumeHeldSettlementsAsync 호출 확인.
- 모바일 Provider-aware: tossVerified 상태, _confirmTossVerification, reasonCode 5개 매핑 확인.

## T16: 운영 전환 리허설 (2026-02-25)

### 설정 파일 위치 확인
- `appsettings.json` line 42: `BankVerificationProvider: "Custom"` (운영 기본값)
- `appsettings.Development.json` line 32: `BankVerificationProvider: "Hybrid"` (개발 환경 오버라이드)
- `appsettings.TossPayments.json` line 17: `BankVerificationProvider: "Hybrid"` (별도 설정 파일)
- `Config/TossPaymentsSettings.cs` line 60: C# 기본값 `"Custom"`
- ASP.NET Core 환경변수 주입: `TossPayments__BankVerificationProvider=Toss` (더블언더스코어)

### Provider Factory 구조
- 3분기 스위치 구현: `custom | toss | hybrid` → 알 수 없는 값은 Custom fallback
- DI: Program.cs lines 330-332에 Scoped 등록
- 사용 위치: BankAccountService.cs lines 84, 138

### dotnet build 결과
- PASS: 0 errors, 0 warnings, 경과 0.88초

### 전환 핵심 발견사항
- 환경변수 방식이 재배포 없이 가장 빠른 전환 방법
- Hybrid 전환 시 `BankVerificationFallbackEnabled=true` 필수
- 목표 롤백 시간 8분 (10분 이내 달성 가능)

## F4: Scope Fidelity Check (2026-02-25)
- 검사 범위(`HEAD~10..HEAD`) 변경 파일 53개 중 계획 범위 내 파일은 1개(`Program.cs`)만 확인됨.
- 계획 범위 외 변경 52개가 Auth/Notification/FCM/OAuth/문서/IDE 설정에 집중되어 스코프 크립으로 판정.
- 금지 영역(`Settlement*.cs`, `settlement_*.dart`, `*.g.dart`, `*.freezed.dart`) 변경은 없음.
- Success Criteria 4개(5개 endpoint 매핑, Provider 전략, on_hold->pending 연계, 롤백 runbook)는 증적 파일로 모두 충족 확인.
