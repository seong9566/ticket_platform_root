# 공통 문서

이 폴더는 백엔드/모바일 팀 모두가 참고하는 공통 문서들이 저장됩니다.

---

## 📂 하위 폴더

### 🏗️ architecture/ - 아키텍처 문서
시스템 전체 아키텍처 설계 문서

**포함 내용**:
- 시스템 아키텍처 다이어그램
- 데이터 플로우
- 통신 프로토콜 (REST, SignalR)
- 보안 설계
- 인프라 구조
- 확장성 고려사항

**파일명 예시**:
- `시스템아키텍처.md`
- `데이터플로우.md`
- `보안설계.md`

---

### 🔄 workflow/ - 워크플로우
개발 프로세스 및 협업 방식

**포함 내용**:
- Git 브랜치 전략
- 코드 리뷰 프로세스
- 배포 프로세스
- 이슈 관리 방법
- CI/CD 파이프라인

**파일명 예시**:
- `Git브랜치전략.md`
- `코드리뷰프로세스.md`
- `배포프로세스.md`

---

### 📄 templates/ - 문서 템플릿
각종 문서 작성을 위한 템플릿

**제공 템플릿**:
- 기능 명세서 템플릿
- API 명세서 템플릿
- 작업 명세서 템플릿
- 회의록 템플릿
- 릴리즈 노트 템플릿
- 버그 리포트 템플릿

**파일명 예시**:
- `TEMPLATE_기능명세서.md`
- `TEMPLATE_API명세서.md`
- `TEMPLATE_작업명세서.md`

---

## 🔗 중요 공통 문서

### 필독 문서
- [PM_README.md](../../PM_README.md) - 프로젝트 전체 개요
- [ROADMAP.md](../../ROADMAP.md) - 개발 로드맵
- [WORK_MANAGEMENT.md](../../WORK_MANAGEMENT.md) - 작업 관리 가이드
- [IMPLEMENTATION_STATUS.md](../../IMPLEMENTATION_STATUS.md) - 구현 현황

### 기술 문서
- Backend: [AGENTS.md](../../TicketPlatFormServer/AGENTS.md)
- Mobile: [claude.md](../../ticket_platform_mobile/claude.md)

---

## 📝 문서 활용 가이드

### 새로운 기능 개발 시
1. `shared/templates/` 에서 적절한 템플릿 복사
2. PM이 초안 작성
3. 해당 팀 폴더(`backend/` or `mobile/`)에 저장
4. 팀 리뷰 후 승인

### 아키텍처 변경 시
1. `shared/architecture/` 문서 업데이트
2. 양쪽 팀 검토
3. 영향 받는 문서들 함께 업데이트

### 워크플로우 변경 시
1. `shared/workflow/` 문서 업데이트
2. 팀 전체 공지
3. 충분한 전환 기간 제공
