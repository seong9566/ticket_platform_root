# DB 확장안 초안 (T6)

목표: 기존 `bank_account.verified` 흐름을 깨지 않고 Provider/Tier/실패코드 추적 필드를 추가한다.

## 변경 원칙
- 기존 컬럼(`verified`, `verification_code`, `verification_expires_at`, `verified_at`) 유지
- 신규 컬럼은 nullable 또는 안전 기본값 사용
- 파괴적 변경(`DROP COLUMN`, 데이터 삭제) 금지

## 마이그레이션 초안 (MySQL 9.x 호환 조건부 패턴)

```sql
-- 1) verification_provider
SET @c := (
    SELECT COUNT(*) FROM information_schema.columns
    WHERE table_schema = DATABASE()
      AND table_name = 'bank_account'
      AND column_name = 'verification_provider'
);
SET @q := IF(
    @c = 0,
    'ALTER TABLE bank_account ADD COLUMN verification_provider VARCHAR(20) NULL COMMENT ''CUSTOM|TOSS|HYBRID'' AFTER verified',
    'SELECT 1'
);
PREPARE s FROM @q; EXECUTE s; DEALLOCATE PREPARE s;

-- 2) verification_tier
SET @c := (
    SELECT COUNT(*) FROM information_schema.columns
    WHERE table_schema = DATABASE()
      AND table_name = 'bank_account'
      AND column_name = 'verification_tier'
);
SET @q := IF(
    @c = 0,
    'ALTER TABLE bank_account ADD COLUMN verification_tier VARCHAR(20) NULL COMMENT ''TIER_0..3'' AFTER verification_provider',
    'SELECT 1'
);
PREPARE s FROM @q; EXECUTE s; DEALLOCATE PREPARE s;

-- 3) verification_status
SET @c := (
    SELECT COUNT(*) FROM information_schema.columns
    WHERE table_schema = DATABASE()
      AND table_name = 'bank_account'
      AND column_name = 'verification_status'
);
SET @q := IF(
    @c = 0,
    'ALTER TABLE bank_account ADD COLUMN verification_status VARCHAR(20) NULL COMMENT ''UNVERIFIED|PENDING|VERIFIED|FAILED|EXPIRED'' AFTER verification_tier',
    'SELECT 1'
);
PREPARE s FROM @q; EXECUTE s; DEALLOCATE PREPARE s;

-- 4) last_verification_failure_code
SET @c := (
    SELECT COUNT(*) FROM information_schema.columns
    WHERE table_schema = DATABASE()
      AND table_name = 'bank_account'
      AND column_name = 'last_verification_failure_code'
);
SET @q := IF(
    @c = 0,
    'ALTER TABLE bank_account ADD COLUMN last_verification_failure_code VARCHAR(100) NULL COMMENT ''최근 검증 실패 코드'' AFTER verification_status',
    'SELECT 1'
);
PREPARE s FROM @q; EXECUTE s; DEALLOCATE PREPARE s;

-- 5) last_verification_at
SET @c := (
    SELECT COUNT(*) FROM information_schema.columns
    WHERE table_schema = DATABASE()
      AND table_name = 'bank_account'
      AND column_name = 'last_verification_at'
);
SET @q := IF(
    @c = 0,
    'ALTER TABLE bank_account ADD COLUMN last_verification_at DATETIME NULL COMMENT ''최근 검증 시각'' AFTER last_verification_failure_code',
    'SELECT 1'
);
PREPARE s FROM @q; EXECUTE s; DEALLOCATE PREPARE s;

-- 6) Backfill (비파괴)
UPDATE bank_account
SET verification_provider = COALESCE(verification_provider, 'CUSTOM'),
    verification_tier = COALESCE(verification_tier, CASE WHEN verified = 1 THEN 'TIER_1_CONTROL_PROOF' ELSE 'TIER_0_NONE' END),
    verification_status = COALESCE(verification_status, CASE WHEN verified = 1 THEN 'VERIFIED' ELSE 'UNVERIFIED' END),
    last_verification_at = COALESCE(last_verification_at, verified_at)
WHERE verification_provider IS NULL
   OR verification_tier IS NULL
   OR verification_status IS NULL;
```

## 롤백 전략 (DB 스키마 롤백 없이)
- 애플리케이션 설정으로 `BankVerificationProvider=Custom` 전환
- 신규 컬럼은 미사용 상태로 남겨도 기존 로직 영향 없음

## 검증 체크
- 기존 `verified=true` 계정이 그대로 인증 계정으로 동작
- `ReleaseEscrowAsync`의 `Verified == true` 분기와 충돌 없음 (`TicketPlatFormServer/TicketPlatFormServer/Services/Payment/PaymentService.cs`)
