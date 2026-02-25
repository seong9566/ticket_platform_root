# 통합 QA 시나리오 카탈로그 (T12)

## 시나리오 목록

### Happy Path (3)
1. 계좌 등록 -> 인증요청 -> 인증성공 -> `verified=true`
2. 미인증 상태 구매확정 -> `on_hold` 생성 확인
3. 인증 완료 후 `on_hold -> pending` 전환 확인

### Error Path (3)
4. 잘못된 인증코드 입력 -> 실패 메시지/재시도 가능
5. 코드 만료 후 확인 요청 -> `EXPIRED` 처리
6. 외부 provider 일시 실패 -> retry 후 fallback(Custom) 동작

## 증적 파일 규칙
- 성공: `.sisyphus/evidence/task-12-happy-{N}.txt|png`
- 실패: `.sisyphus/evidence/task-12-error-{N}.txt|png`

## 판정 기준
- PASS: 상태 전이/응답 필드/로그 규칙 모두 충족
- FAIL: 상태 전이 누락, 계약 필드 누락, 민감정보 로그 노출

## 자동 검증 원칙
- 수동 확인 문구 금지
- 각 시나리오는 재실행 가능한 커맨드 또는 UI 스텝으로 기록

## 최소 증적 항목
- API 응답 캡처(JSON)
- DB 상태 확인 결과(SQL 출력)
- 모바일 화면 캡처(해당되는 경우)
