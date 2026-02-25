# Provider 전환 롤백 Runbook

**문서 버전:** 1.0  
**작성일:** 2026-02-25  
**대상 환경:** TicketHub 운영 (ASP.NET Core 9 + MySQL)

---

## 전환 절차 (Custom → Hybrid → Toss)

### Step 1: Hybrid 전환 (Phase 1 — Canary)

**사전 조건:**
- DB 마이그레이션 적용 완료 (`verification_provider`, `verification_status` 컬럼 존재)
- 기존 `verified=true` 계좌 backfill 완료
- Toss 테스트 API 키 유효성 확인

**전환 절차:**

```bash
# 방법 A: 환경변수 (권장 — 재배포 없이 즉시 적용)
export TossPayments__BankVerificationProvider=Hybrid
export TossPayments__BankVerificationFallbackEnabled=true

# 방법 B: appsettings.json 수정
# "BankVerificationProvider": "Hybrid"
# "BankVerificationFallbackEnabled": true
```

2. 서버 재시작 (또는 환경변수 hot-reload 지원 시 생략)
3. 헬스체크: `GET /health` → 200 OK 확인
4. Smoke test:
   ```
   GET /api/bank-account/me
   → 응답 200 OK, verification_status 필드 존재 확인
   ```
5. 모니터링 (30분): fallback_count, toss_error_rate, on_hold 계좌 수

---

### Step 2: Toss 전환 (Phase 3 — Full)

**사전 조건:**
- Phase 1 (Hybrid) 30분 모니터링 이상 없음
- Toss API 실패율 < 5%

**전환 절차:**

```bash
# 환경변수 갱신
export TossPayments__BankVerificationProvider=Toss
export TossPayments__BankVerificationFallbackEnabled=false  # 선택적
```

2. 서버 재시작
3. Smoke test:
   ```
   POST /api/bank-account/verify/request
   → 응답 확인 (Toss API 호출 여부 로그 확인)
   로그: [TossBankVerificationProvider] 계좌 사전 검증 완료.
   ```
4. 모니터링 (30분): `verify/request` 응답 시간 < 3초, `verify/confirm` 성공률 > 95%

---

## 롤백 절차 (즉시 실행 가능 — 목표: 10분 이내)

### 즉시 롤백 (Toss/Hybrid → Custom)

| 단계 | 작업 | 예상 소요 |
|------|------|-----------|
| 1 | 설정 변경 | 1분 |
| 2 | 서버 재시작 | 2분 |
| 3 | Smoke test | 2분 |
| 4 | 데이터 무결성 확인 | 3분 |
| **합계** | | **8분** |

**Step 1: 설정 즉시 전환**

```bash
# 환경변수 방식 (가장 빠름)
export TossPayments__BankVerificationProvider=Custom
export TossPayments__BankVerificationFallbackEnabled=true
```

**Step 2: 서버 재시작**

```bash
# systemd 환경
sudo systemctl restart tickethub-api

# Docker 환경
docker restart tickethub-api

# dotnet run 직접 실행 환경
# 기존 프로세스 종료 후 재시작
dotnet run --project TicketPlatFormServer/TicketPlatFormServer.csproj
```

**Step 3: Smoke test**

```bash
# 헬스체크
curl -s http://localhost:5224/health | jq .

# 계좌 조회
curl -s -H "Authorization: Bearer {TOKEN}" \
  http://localhost:5224/api/bank-account/me | jq .

# 로그에서 Custom Provider 동작 확인
# 기대 로그: [CustomBankVerificationProvider] 인증 코드 발급.
```

**Step 4: 데이터 무결성 체크**

```sql
-- 롤백 후 Provider별 인증 상태 분포 확인
SELECT 
    verification_provider, 
    verification_status, 
    COUNT(*) AS cnt
FROM bank_account 
GROUP BY verification_provider, verification_status;

-- 정산 보류 계좌 확인 (on_hold 비정상 증가 여부)
SELECT 
    ba.id, 
    ba.account_number,
    ba.verification_status, 
    s.status AS settlement_status,
    s.created_at
FROM bank_account ba
JOIN settlement s ON s.bank_account_id = ba.id
WHERE s.status = 'on_hold'
ORDER BY s.created_at DESC
LIMIT 50;

-- 최근 1시간 인증 요청 실패율 확인
SELECT 
    DATE_FORMAT(created_at, '%Y-%m-%d %H:%i') AS minute_bucket,
    COUNT(*) AS total,
    SUM(CASE WHEN verification_status = 'FAILED' THEN 1 ELSE 0 END) AS failed
FROM bank_account
WHERE created_at >= NOW() - INTERVAL 1 HOUR
GROUP BY minute_bucket
ORDER BY minute_bucket DESC;
```

---

## 부분 롤백 (Toss → Hybrid)

Toss Provider 장애이나 Custom으로 완전 롤백하기 전에 Hybrid로 먼저 후퇴 가능:

```bash
export TossPayments__BankVerificationProvider=Hybrid
export TossPayments__BankVerificationFallbackEnabled=true
```

→ Toss 실패 시 자동으로 Custom fallback 처리됨.

---

## 알림 및 에스컬레이션

| 지표 | 임계치 | 조치 |
|------|--------|------|
| `verify/request` 응답 시간 | > 3초 | 즉시 Hybrid 전환 |
| `verify/confirm` 성공률 | < 95% | 즉시 롤백 |
| `on_hold` 계좌 수 | 비정상 급증 | 즉시 롤백 + DBA 확인 |
| Toss API 오류율 | > 5% | 즉시 Hybrid/Custom 전환 |

---

## 참조

- 설정 파일: `TicketPlatFormServer/TicketPlatFormServer/appsettings.json` line 42
- Provider Factory: `Services/BankAccount/BankAccountVerificationProviderFactory.cs`
- 설정 클래스: `Config/TossPaymentsSettings.cs` line 60
- 기존 런북: `.sisyphus/evidence/toss-rollout-rollback-runbook.md`
