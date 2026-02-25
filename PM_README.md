# TicketHub - PM 관리 가이드

## 📋 목차
- [프로젝트 개요](#프로젝트-개요)
- [프로젝트 구조](#프로젝트-구조)
- [현재 상태](#현재-상태)
- [개발 로드맵](#개발-로드맵)
- [작업 관리](#작업-관리)
- [릴리즈 프로세스](#릴리즈-프로세스)
- [팀 협업](#팀-협업)
- [주요 링크](#주요-링크)

---

## 프로젝트 개요

### 기본 정보
- **프로젝트명**: TicketHub
- **목적**: 중고 티켓을 원가 이하로만 판매 가능한 티켓 중고 거래 플랫폼
- **플랫폼**: Android / iOS / Web Backend
- **개발 시작**: 2024년 12월
- **현재 단계**: MVP 개발 중

### 비즈니스 모델
- 티켓 판매 시 원가 이하로만 판매 가능 (가격 상한제)
- 플랫폼 수수료: 5% 미만
- 에스크로 보관금 정책: HOLD → RELEASED → 정산 (D+3)
- 결제: 토스 페이먼츠 연동 예정

### 핵심 기능
1. **회원 인증**: 이메일, 소셜 로그인 (Google, Kakao, Apple)
2. **본인 인증**: 실명 인증, 휴대폰 인증, 계좌 인증 (거래 시 필수)
3. **티켓 검증**: QR 코드, OCR, 티켓 번호, 이미지 업로드
4. **1:1 채팅**: SignalR 기반 실시간 채팅, 이미지 전송
5. **결제 시스템**: 에스크로, 자동 정산 (D+3)
6. **신고/분쟁**: 관리자 개입 시스템

---

## 프로젝트 구조

```
ticket_platform/
├── ticket_platform_mobile/          # Flutter 모바일 앱
│   ├── lib/
│   │   ├── core/                   # 공통 유틸, 테마, 네트워크
│   │   ├── features/               # 기능별 모듈 (Clean MVVM)
│   │   │   ├── auth/              # 인증
│   │   │   ├── home/              # 홈
│   │   │   ├── events/            # 공연/이벤트
│   │   │   ├── ticketing/         # 티켓 상세/구매
│   │   │   ├── sell/              # 티켓 판매
│   │   │   ├── chat/              # 채팅
│   │   │   ├── wishlist/          # 찜
│   │   │   └── profile/           # 마이페이지
│   │   └── shared/                # 공유 위젯
│   └── docs/                       # 문서
│       ├── api/                    # API 명세 (백엔드 요청/응답)
│       ├── plans/                  # 개발 계획서
│       ├── PROFILE_UPDATE_IMPLEMENTATION_PLAN.md
│       └── TRANSACTION_HISTORY_INTEGRATION_COMPLETE.md
│
└── TicketPlatFormServer/            # ASP.NET Core 9 백엔드
    ├── Controllers/                # HTTP API 엔드포인트
    ├── Services/                   # 비즈니스 로직
    ├── Repository/                 # 데이터 접근 (EF Core + Dapper)
    ├── DBModel/                    # 데이터베이스 엔티티
    ├── DTO/                        # API 요청/응답 DTO
    ├── Hubs/                       # SignalR 채팅
    ├── Common/                     # 미들웨어, 예외 처리
    ├── Config/                     # 설정
    ├── Enum/                       # 열거형
    ├── api_spec/                   # API 명세 문서
    ├── database_history/           # DB 덤프 및 마이그레이션
    ├── AGENTS.md                   # 개발 가이드
    ├── ReadMe.md                   # 서버 셋업 가이드
    └── 구매_판매_내역_개발_계획서.md
```

### 기술 스택

#### Frontend (Mobile)
- **Framework**: Flutter 3.35.4 / Dart 3.9.2
- **State Management**: Riverpod 3.x (code generation)
- **Routing**: GoRouter
- **Networking**: Dio
- **Model**: Freezed + JSON Serializable
- **Real-time**: SignalR (채팅)

#### Backend (Server)
- **Framework**: ASP.NET Core 9 (C#)
- **Database**: MySQL 9.0
- **ORM**: EF Core 9 + Dapper 2.1.66
- **Authentication**: JWT Bearer
- **Real-time**: SignalR
- **Storage**: Supabase Storage (AWS S3)
- **Payment**: Toss Payments (예정)
- **API Docs**: Swagger/OpenAPI

---

## 현재 상태

### ✅ 완료된 기능 (2026-02-09 기준)

**진행률**: ~85% (MVP 거의 완료)

#### Phase 1: MVP 핵심 기능 ✅

##### 백엔드 서버
1. **인증 API** ✅
   - 회원가입, 로그인, JWT 발급, 토큰 갱신

2. **홈/이벤트 API** ✅
   - 홈 데이터 조회
   - 카테고리별 이벤트 목록

3. **티켓 API** ✅
   - 티켓 상세 조회
   - 찜 상태 포함

4. **판매 등록 API** ✅
   - 카테고리, 공연, 일정, 좌석 조회
   - 티켓 판매 등록 (이미지 업로드 포함)
   - 내 판매 티켓 목록
   - 판매 취소
   - 원가 검증, 할인율 계산

5. **결제 API** ✅ (토스페이먼츠)
   - 결제 요청, 승인, 취소
   - 에스크로 보관금 시스템
   - 웹훅 처리

6. **채팅 API & SignalR Hub** ✅
   - 채팅방 생성/조회/나가기
   - 실시간 메시지 송수신
   - 결제 요청/완료 메시지
   - 구매 확정 메시지

7. **찜 API** ✅
   - 찜 토글
   - 찜한 티켓 목록

8. **거래 내역 API** ✅
   - 구매 내역 조회
   - 판매 내역 조회
   - 상태별/기간별 필터링
   - 커서 기반 페이지네이션

9. **구매 확정 API** ✅
   - Settlement(정산) 자동 생성
   - 티켓 소유권 이전
   - SignalR 실시간 알림

##### 모바일 앱
1. **인증 시스템** ✅
   - 로그인/회원가입
   - JWT 토큰 관리

2. **홈 화면** ✅
   - 카테고리별 이벤트 목록
   - 배너, 추천 티켓

3. **티켓 상세/구매** ✅
   - 티켓 상세 페이지 (18개 위젯)
   - 판매자 정보, 좌석 정보, 거래 방식
   - 찜 토글
   - 구매하기 버튼

4. **판매 등록** ✅
   - 7단계 플로우 (2,357줄)
   - 카테고리 → 공연 → 날짜 → 좌석 → 가격 → 추가정보
   - 이미지 다중 선택 (최대 5장)
   - 원가 검증, 할인율 계산

5. **결제 시스템** ✅
   - 토스페이먼츠 SDK 연동
   - 결제 페이지, 최종 확인, 성공/실패 화면

6. **채팅 시스템** ✅
   - SignalR 실시간 채팅
   - 이미지 전송
   - 결제 요청/완료/구매확정 카드 UI
   - 채팅방 나가기 (스와이프)

7. **찜 기능** ✅
   - 찜 토글
   - 찜한 티켓 목록

8. **거래 내역** ✅
   - 구매/판매 내역 조회
   - 상태/기간/정렬 필터
   - 무한 스크롤
   - Pull to Refresh

### 🚧 진행 중인 작업
1. **프로필 이미지 업로드** (모바일) - 예상: 1-2일

### ❌ 미완료 기능 (Phase 2-4)

#### Phase 2: 검증 및 신뢰성
1. **티켓 검증 시스템** (QR, OCR, 수동 입력)
2. **본인 인증** (실명, 휴대폰, 계좌)
3. **신고/분쟁 시스템**
4. **평판 시스템**

#### Phase 3: 정산 및 운영
1. **정산 스케줄러** (D+3 자동 정산)
2. **관리자 대시보드**
3. **알림 시스템** (FCM)
4. **모니터링 및 로깅**

#### Phase 4: 출시 준비
1. **성능 최적화**
2. **보안 강화**
3. **테스트** (단위, 통합, E2E)
4. **앱스토어 심사 준비**

---

## 개발 로드맵

자세한 로드맵은 [ROADMAP.md](./ROADMAP.md) 참고

### Phase 1: MVP 핵심 기능 (2-3개월)
- [x] 인증 시스템
- [x] 홈/이벤트 목록
- [x] 채팅 시스템
- [x] 거래 내역 API
- [ ] 티켓 상세/구매 플로우
- [ ] 티켓 판매 등록
- [ ] 결제 시스템 (토스 페이먼츠)
- [ ] 찜 기능

### Phase 2: 검증 및 신뢰성 (1-2개월)
- [ ] 티켓 검증 시스템 (QR, OCR)
- [ ] 본인 인증 (실명, 휴대폰, 계좌)
- [ ] 신고/분쟁 시스템
- [ ] 평판 시스템

### Phase 3: 정산 및 운영 (1개월)
- [ ] 에스크로 정산 시스템
- [ ] 관리자 대시보드
- [ ] 알림 시스템
- [ ] 모니터링 및 로깅

### Phase 4: 출시 준비 (1개월)
- [ ] 성능 최적화
- [ ] 보안 강화
- [ ] 테스트 (단위, 통합, E2E)
- [ ] 앱스토어 심사 준비

---

## 작업 관리

### 이슈 관리
- GitHub Issues 사용
- 라벨 시스템:
  - `feature`: 새 기능
  - `bug`: 버그 수정
  - `enhancement`: 기능 개선
  - `docs`: 문서 작업
  - `backend`: 백엔드 작업
  - `mobile`: 모바일 작업
  - `priority:high`: 높은 우선순위
  - `priority:medium`: 중간 우선순위
  - `priority:low`: 낮은 우선순위

### 브랜치 전략
- `main`: 프로덕션 코드 (항상 배포 가능)
- `develop`: 개발 브랜치 (다음 릴리즈 준비)
- `feature/*`: 기능 개발 브랜치
- `bugfix/*`: 버그 수정 브랜치
- `hotfix/*`: 긴급 수정 브랜치

### 커밋 메시지 규칙
Conventional Commits 사용:
```
feat: 새 기능 추가
fix: 버그 수정
chore: 빌드, 설정 변경
docs: 문서 업데이트
refactor: 코드 리팩토링
test: 테스트 추가/수정
style: 코드 포맷팅
```

예시:
- `feat(mobile): add chat room leave functionality`
- `fix(backend): resolve payment success message parsing`
- `docs(pm): add project management guide`

---

## 릴리즈 프로세스

### 버전 관리
- Semantic Versioning (MAJOR.MINOR.PATCH)
- 예: `1.0.0`, `1.1.0`, `1.1.1`

### 릴리즈 체크리스트
1. [ ] 기능 개발 완료
2. [ ] 코드 리뷰 완료
3. [ ] 테스트 통과 (단위, 통합)
4. [ ] API 문서 업데이트
5. [ ] 마이그레이션 스크립트 준비 (DB 변경 시)
6. [ ] CHANGELOG 업데이트
7. [ ] 버전 태그 생성

### 배포 프로세스
1. **개발 환경 배포** (develop → staging)
2. **QA 테스트**
3. **프로덕션 배포** (main → production)
4. **배포 후 모니터링**

---

## 팀 협업

### 코드 리뷰 가이드
1. PR 생성 시 명확한 설명 작성
2. 스크린샷 첨부 (UI 변경 시)
3. 테스트 결과 공유
4. 최소 1명의 승인 필요

### 커뮤니케이션
- **일일 스탠드업**: 진행 상황, 블로커 공유
- **주간 회의**: 로드맵 리뷰, 우선순위 조정
- **문서화**: 중요한 결정사항은 문서로 기록

### 문서 관리
- **기술 문서**: `AGENTS.md`, `claude.md`, `architecture.md`
- **API 명세**: `api_spec/`, `docs/api/`
- **개발 계획**: `docs/plans/`, `*_IMPLEMENTATION_PLAN.md`
- **인수인계**: `hand_off.md`

---

## 주요 링크

### 문서
- [서버 README](./TicketPlatFormServer/ReadMe.md) - 서버 셋업 가이드
- [서버 AGENTS](./TicketPlatFormServer/AGENTS.md) - 백엔드 개발 가이드
- [모바일 CLAUDE](./ticket_platform_mobile/claude.md) - 모바일 개발 가이드
- [구매/판매 내역 계획서](./구매_판매_내역_개발_계획서.md) - 거래 내역 API 명세

### 개발 환경
- **서버**: http://localhost:5224
- **Swagger UI**: http://localhost:5224/swagger
- **데이터베이스**: MySQL 9.0 (localhost:3306)

### 외부 서비스
- **Storage**: Supabase Storage (S3 호환)
- **Payment**: Toss Payments (연동 예정)

---

## 다음 단계

### 즉시 처리 필요
1. [ ] Git 초기 커밋 (현재 커밋 없음)
2. [ ] 이슈 템플릿 생성 (`.github/ISSUE_TEMPLATE/`)
3. [ ] PR 템플릿 생성 (`.github/PULL_REQUEST_TEMPLATE.md`)
4. [ ] 로드맵 상세 문서 작성 (`ROADMAP.md`)

### 단기 목표 (1-2주)
1. [ ] 프로필 업데이트 기능 완료
2. [ ] 거래 내역 UI 연동 완료
3. [ ] 티켓 상세 페이지 개발 시작

### 중기 목표 (1개월)
1. [ ] 티켓 구매 플로우 완료
2. [ ] 티켓 판매 등록 완료
3. [ ] 토스 페이먼츠 연동 완료

---

**문서 버전**: 1.0
**작성일**: 2026-02-09
**작성자**: PM
**최종 수정일**: 2026-02-09
