# Provider 전환 Go/No-Go 기준

**문서 버전:** 1.0  
**작성일:** 2026-02-25  
**적용 전환:** Custom → Hybrid → Toss (단계별)

---

## Phase 1 전환 (Custom → Hybrid) Go/No-Go

### GO 조건 (모두 충족 시 전환 진행)

- [ ] `dotnet build TicketPlatFormServer/TicketPlatFormServer.sln` — **0 errors, 0 warnings**
- [ ] `flutter analyze ticket_platform_mobile` — **0 errors** (warnings 허용)
- [ ] DB 마이그레이션 적용 완료
  ```sql
  -- 확인 쿼리
  SHOW COLUMNS FROM bank_account LIKE 'verification_provider';
  SHOW COLUMNS FROM bank_account LIKE 'verification_status';
  ```
- [ ] 기존 `verified=true` 계좌 backfill 완료
  ```sql
  -- backfill 미완료 계좌 수 = 0 확인
  SELECT COUNT(*) FROM bank_account 
  WHERE is_verified = true AND verification_status IS NULL;
  ```
- [ ] `TossPayments:BankVerificationFallbackEnabled = true` 설정 확인
- [ ] Toss 테스트 API 키로 사전 검증 호출 1회 성공 (테스트 환경)
- [ ] 모니터링 대시보드/로그 수집 준비 완료

### NO-GO 조건 (하나라도 해당 시 전환 중단)

- 빌드 오류(error) 1개 이상 존재
- DB 마이그레이션 미적용 (`verification_provider` 컬럼 없음)
- backfill 미완료 계좌 존재
- `BankVerificationFallbackEnabled = false` (Hybrid 전환 시 반드시 true)
- Toss API 테스트 호출 실패 (연결 오류, 인증 오류)

---

## Phase 3 전환 (Hybrid → Toss Full) Go/No-Go

### GO 조건 (모두 충족 시 전환 진행)

- [ ] Phase 1(Hybrid) 운영 후 **30분 이상** 아래 지표 모두 정상
  - `verify/request` 평균 응답 시간 < **3초**
  - `verify/confirm` 성공률 > **95%**
  - Toss API fallback 비율 < **10%**
  - `on_hold` 계좌 수 기준치 이하 (전환 전 대비 +5% 이내)
- [ ] Toss 운영 API 키 검증 완료 (실제 키, 비밀번호 관리 시스템에서 주입)
- [ ] `verify/request` Smoke test 성공 — 로그에 `[TossBankVerificationProvider]` 확인

### NO-GO 조건 (하나라도 해당 시 전환 중단 또는 Hybrid 유지)

- Toss API 호출 실패율 > **5%**
- `on_hold` 계좌 비정상 급증 (기준치 +10% 초과)
- `verify/confirm` 성공률 < **95%**
- 사용자 인증 오류 관련 CS 문의 급증 (정상 대비 2배 초과)
- Toss 운영 API 키 미준비

---

## 모니터링 지표 (전환 후 30분 관찰)

| 지표 | 정상 기준 | 경고 임계치 | 롤백 임계치 |
|------|-----------|-------------|-------------|
| `verify/request` 응답 시간 | < 2초 | 2~3초 | > 3초 |
| `verify/confirm` 성공률 | > 98% | 95~98% | < 95% |
| Toss API fallback 비율 | < 5% | 5~10% | > 10% |
| `on_hold` 계좌 증가율 | 기준 ±2% | ±2~5% | > +5% |
| `verify/request` 5xx 오류율 | 0% | < 1% | ≥ 1% |

---

## 확인 쿼리 (전환 전/후 기준 수립)

```sql
-- [전환 전 기준 수집] Provider별 현황
SELECT 
    verification_provider,
    verification_status,
    COUNT(*) AS cnt
FROM bank_account
GROUP BY verification_provider, verification_status;

-- [전환 전 기준 수집] 정산 보류 계좌 수
SELECT COUNT(*) AS on_hold_count
FROM settlement
WHERE status = 'on_hold';

-- [전환 후 모니터링] 최근 30분 인증 성공/실패
SELECT 
    verification_status,
    COUNT(*) AS cnt
FROM bank_account
WHERE updated_at >= NOW() - INTERVAL 30 MINUTE
GROUP BY verification_status;

-- [전환 후 모니터링] Toss fallback 건수 (로그에서도 확인 가능)
-- 로그 키워드: "[HybridBankVerificationProvider] Toss 사전 검증 실패, Custom으로 fallback."
```

---

## 롤백 트리거 자동화 권장 (미래 과제)

| 조건 | 권장 자동화 |
|------|-------------|
| 5xx 오류율 ≥ 1% (5분 이상) | 자동 Hybrid 전환 알림 |
| `on_hold` 급증 감지 | Slack 알림 + 수동 롤백 |

---

## 현재 Smoke Test 결과 요약 (T16 기준)

| 항목 | 결과 |
|------|------|
| `dotnet build` | **PASS** (0 errors, 0 warnings) |
| Provider Factory 3분기 구현 | **확인** (Custom / Toss / Hybrid) |
| DI 등록 상태 | **완료** (Program.cs lines 330-332) |
| 설정 전환 가능 여부 | **가능** (환경변수 또는 appsettings 수정) |
| `appsettings.json` 현재 Provider | **Custom** (운영 기본값) |
| `appsettings.Development.json` | **Hybrid** (개발 환경) |

**현재 판정: Custom Provider 빌드 상태 GO ✓**  
(DB 마이그레이션, Toss API 키, backfill 완료 후 Hybrid 전환 GO 판정 가능)

---

## 참조

- Rollback Runbook: `.sisyphus/evidence/task-16-rollback-runbook.md`
- Smoke Test 결과: `.sisyphus/evidence/task-16-cutover-rehearsal.txt`
- 기존 설계 런북: `.sisyphus/evidence/toss-rollout-rollback-runbook.md`
- Provider Factory: `TicketPlatFormServer/TicketPlatFormServer/Services/BankAccount/BankAccountVerificationProviderFactory.cs`
