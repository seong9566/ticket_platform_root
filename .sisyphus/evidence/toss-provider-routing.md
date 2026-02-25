# Provider 라우팅/DI 설계 (T5)

## 목표
- `IBankAccountService` 시그니처를 유지하면서 내부 검증 Provider만 교체 가능하게 만든다.

## 인터페이스 제안

```csharp
public interface IBankAccountVerificationProvider
{
    string Name { get; }

    Task<VerificationRequestResult> RequestAsync(VerificationRequestInput input, CancellationToken ct = default);
    Task<VerificationConfirmResult> ConfirmAsync(VerificationConfirmInput input, CancellationToken ct = default);
}
```

## Provider 구성
- `CustomBankVerificationProvider`
  - 현행 4자리 코드 발급/확인 로직 담당
- `TossBankVerificationProvider`
  - 토스 계좌인증 5개 API 호출/매핑 담당
- `HybridBankVerificationProvider`
  - 기본 `Toss` 시도, 정책상 retryable 오류일 때 `Custom` fallback

## 설정값 제안
- `TossPaymentsSettings` 확장
  - `BankVerificationProvider`: `Custom|Toss|Hybrid`
  - `BankVerificationTimeoutSeconds`
  - `BankVerificationFallbackEnabled`

## 라우팅 결정표

| 설정값 | RequestVerification | ConfirmVerification | fallback |
|---|---|---|---|
| `Custom` | Custom | Custom | 없음 |
| `Toss` | Toss | Toss | 없음 |
| `Hybrid` | Toss 우선 | Toss 우선 | retryable 오류 시 Custom |

## DI 등록 위치
- `TicketPlatFormServer/TicketPlatFormServer/Program.cs`

예시:
```csharp
builder.Services.AddScoped<CustomBankVerificationProvider>();
builder.Services.AddScoped<TossBankVerificationProvider>();
builder.Services.AddScoped<HybridBankVerificationProvider>();
builder.Services.AddScoped<IBankAccountVerificationProviderFactory, BankAccountVerificationProviderFactory>();
```

## Unknown 설정 처리
- `Unknown/empty` 입력 시 안전 기본값은 `Custom`
- 동시에 경고 로그 1회 출력

## 하위호환 원칙
- `BankAccountController` endpoint/요청 바디는 v1 그대로 유지
- 내부 서비스 구현에서만 Provider 호출로 분기
