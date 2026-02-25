# 백엔드 개발 문서

이 폴더는 백엔드 개발팀을 위한 문서들이 저장됩니다.

---

## 📂 하위 폴더

### 🔌 api_specs/ - API 명세서
RESTful API 엔드포인트 상세 명세

**포함 내용**:
- HTTP 메서드 및 URL
- 요청 헤더
- 요청 바디 (예시 포함)
- 응답 바디 (예시 포함)
- 에러 코드 및 메시지
- 인증 요구사항

**파일명 규칙**:
```
[METHOD]_[endpoint_path].md

예시:
- POST_api_payment_request.md
- GET_api_tickets_detail.md
- DELETE_api_sell_tickets.md
```

---

### 🗄️ database/ - 데이터베이스 문서
DB 스키마 및 관련 문서

**포함 내용**:
- 테이블 스키마 정의
- ERD 다이어그램
- 인덱스 설계
- 마이그레이션 스크립트
- 시드 데이터

**파일명 예시**:
- `ERD_전체시스템.md`
- `SCHEMA_transactions.md`
- `MIGRATION_2026-02-09_add_settlement.sql`

---

### 📖 guides/ - 개발 가이드
백엔드 개발 가이드라인

**포함 내용**:
- 코딩 컨벤션
- 아키텍처 패턴
- 테스트 작성 가이드
- 배포 프로세스
- 트러블슈팅

**파일명 예시**:
- `코딩컨벤션.md`
- `테스트가이드.md`
- `배포가이드.md`

---

### ✅ tasks/ - 작업 명세
백엔드 작업 상세 명세서

**포함 내용**:
- 작업 개요
- 기술 요구사항
- 구현 계획
- 테스트 계획
- 예상 소요 시간

**파일명 규칙**:
```
TASK-[번호]_[작업명].md

예시:
- TASK-001_티켓검증API구현.md
- TASK-002_정산스케줄러구현.md
```

---

## 🔗 관련 문서

- [TicketPlatFormServer/AGENTS.md](../../TicketPlatFormServer/AGENTS.md) - 백엔드 개발 가이드
- [TicketPlatFormServer/ReadMe.md](../../TicketPlatFormServer/ReadMe.md) - 서버 셋업 가이드

---

## 📝 문서 작성 시 참고

1. **API 명세서**: OpenAPI/Swagger 형식 권장
2. **DB 스키마**: Markdown 테이블 또는 SQL 스크립트
3. **코드 예시**: 실행 가능한 예시 포함
4. **에러 처리**: 모든 가능한 에러 케이스 명시
