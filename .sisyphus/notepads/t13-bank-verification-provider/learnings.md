
## T13: 백엔드 계좌인증 Provider 추상화 (2026-02-25)

### 구현 완료 파일
- `Services/BankAccount/IBankAccountVerificationProvider.cs` — 인터페이스 + 4개 record (Input/Result × 2)
- `Services/BankAccount/CustomBankVerificationProvider.cs` — 4자리 코드 방식
- `Services/BankAccount/TossBankVerificationProvider.cs` — Toss API 사전 검증 방식
- `Services/BankAccount/HybridBankVerificationProvider.cs` — Toss 우선 + Custom fallback
- `Services/BankAccount/BankAccountVerificationProviderFactory.cs` — Factory 인터페이스 + 구현
- `DBModel/BankAccount.cs` — 5개 신규 필드 추가
- `database_history/migrations/20260225_003_add_verification_provider_fields.sql`
- `Services/BankAccount/BankAccountService.cs` — Provider Factory 사용, 신규 필드 갱신
- `Program.cs` — 4개 DI 등록 추가
- `DTO/BankAccount/RequestVerificationResponseDto.cs` — ExpiresAt DateTime → DateTime? (Toss는 null)

### 핵심 설계 결정
1. **Provider 분리**: IMemoryCache → CustomBankVerificationProvider, ITossPaymentsService → TossBankVerificationProvider
2. **BankAccountService 의존성 제거**: IMemoryCache, ITossPaymentsService 제거; IBankAccountVerificationProviderFactory 추가
3. **Hybrid.ConfirmAsync 라우팅**: input.VerificationCode가 null이면 Toss 경로(항상 verified), 있으면 Custom 경로
4. **VerificationStatus 체크**: hasPendingRequest = VerificationStatus=="PENDING" OR VerificationCode!=null (하위 호환)
5. **ExpiresAt nullable**: Toss 경로에서 null이 정상 (코드 없음), DTO도 DateTime? 로 변경
6. **MAX_ATTEMPTS_EXCEEDED**: Provider가 ReasonCode 반환 → Service가 DB 정리 후 AppException 발생

### 빌드 결과
- 경고 0, 오류 0
