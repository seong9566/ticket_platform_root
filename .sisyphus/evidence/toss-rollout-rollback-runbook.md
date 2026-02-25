# 점진 롤아웃 + 롤백 런북 (T11)

## 기본 전략
- Provider 설정 기반 점진 전환: `Custom -> Hybrid -> Toss`
- 신규 계좌부터 우선 적용, 기존 계좌는 재인증 이벤트 시 전환

## 단계별 롤아웃

### Phase 0
- `BankVerificationProvider=Custom`
- 신규 지표 수집만 활성화

### Phase 1 (Canary)
- `BankVerificationProvider=Hybrid`
- 내부/베타 계정 대상(소량) 적용
- 모니터링: 실패율, fallback 비율, on_hold 잔류건수

### Phase 2
- Hybrid 적용 범위 확장
- 장애 없으면 `Toss` 중심 전환 준비

### Phase 3
- `BankVerificationProvider=Toss`
- fallback은 운영 플래그로 유지

## 롤백 절차 (목표 10분 내)
1. 설정 즉시 전환: `BankVerificationProvider=Custom`
2. 애플리케이션 재시작/구성 반영
3. 헬스체크 + 스모크 테스트 실행
4. 미처리 검증 요청건은 Custom 경로로 재처리

## 확인 커맨드
```bash
dotnet build --project TicketPlatFormServer/TicketPlatFormServer.sln
```

## 무결성 체크 예시
```sql
SELECT code, COUNT(*)
FROM settlements s
JOIN settlement_statuses ss ON ss.id = s.status_id
GROUP BY code;
```

## Go/No-Go 기준
- Go:
  - 검증 성공률 목표치 충족
  - fallback 비율 임계치 이하
  - `on_hold` 비정상 급증 없음
- No-Go:
  - 인증 오류 급증
  - provider auth 오류 지속
  - 사용자 불만/문의 급증
