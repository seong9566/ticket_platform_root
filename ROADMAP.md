# TicketHub 개발 로드맵

## 📅 전체 타임라인

```
Phase 1: MVP 핵심 기능 (2-3개월)
  └─> Phase 2: 검증 및 신뢰성 (1-2개월)
      └─> Phase 3: 정산 및 운영 (1개월)
          └─> Phase 4: 출시 준비 (1개월)
              └─> Launch 🚀
```

**예상 총 개발 기간**: 5-7개월
**현재 진행률**: ~85% (Phase 1 거의 완료, Phase 2 준비 중)

---

## Phase 1: MVP 핵심 기능 (2-3개월)

### 목표
사용자가 티켓을 등록하고, 검색하고, 채팅하고, 구매할 수 있는 최소 기능 제품 완성

### 1.1 인증 시스템 ✅ 완료
**기간**: ~2주 (완료)
**담당**: Backend + Mobile

- [x] 이메일 회원가입/로그인
- [x] JWT 토큰 발급 및 검증
- [x] 토큰 갱신 (Refresh Token)
- [x] 로그아웃

**결과물**:
- Backend: `AuthController`, `AuthService`, JWT 미들웨어
- Mobile: `auth` feature 모듈, 토큰 저장소

---

### 1.2 홈/이벤트 시스템 ✅ 완료
**기간**: ~1주 (완료)
**담당**: Backend + Mobile

- [x] 홈 화면 데이터 API
- [x] 카테고리별 이벤트 목록 API
- [x] 배너, 추천 티켓
- [x] 모바일 홈 UI 구현

**결과물**:
- Backend: `HomeController`, `EventController`
- Mobile: `home`, `events` feature 모듈

---

### 1.3 채팅 시스템 ✅ 완료
**기간**: ~2주 (완료)
**담당**: Backend + Mobile

- [x] SignalR Hub 구현
- [x] 채팅방 생성/조회/나가기 API
- [x] 실시간 메시지 송수신
- [x] 이미지 전송
- [x] 결제 요청/완료 메시지 카드
- [x] 구매 확정 메시지 처리
- [x] 채팅방 나가기 (스와이프 제스처)

**결과물**:
- Backend: `ChatHub`, `ChatController`, SignalR 설정
- Mobile: `chat` feature 모듈, SignalR 클라이언트

---

### 1.4 거래 내역 시스템 ✅ Backend 완료 / 🚧 Mobile 진행 중
**기간**: ~1주
**담당**: Backend + Mobile

- [x] 구매 내역 조회 API (Backend)
- [x] 판매 내역 조회 API (Backend)
- [x] 상태별/기간별 필터링
- [x] 커서 기반 페이지네이션
- [ ] 모바일 거래 내역 UI
- [ ] 상태별 탭 구현
- [ ] 상세 내역 모달

**결과물**:
- Backend: `TransactionController`, `TransactionService`, 복잡한 SQL 쿼리
- Mobile: `transaction_history` feature 모듈 (예정)

**참고 문서**: [구매_판매_내역_개발_계획서.md](./구매_판매_내역_개발_계획서.md)

---

### 1.5 프로필 업데이트 🚧 진행 중
**기간**: ~1주
**담당**: Backend + Mobile

- [ ] 프로필 조회 API
- [ ] 프로필 수정 API (닉네임, 이미지 등)
- [ ] 프로필 이미지 업로드 (Supabase Storage)
- [ ] 모바일 프로필 편집 UI
- [ ] 이미지 선택 및 업로드

**결과물**:
- Backend: `ProfileController`, 이미지 업로드 서비스
- Mobile: `profile` feature 개선

**참고 문서**:
- [PROFILE_UPDATE_IMPLEMENTATION_PLAN.md](./ticket_platform_mobile/docs/PROFILE_UPDATE_IMPLEMENTATION_PLAN.md)
- [BACKEND_REQUEST_PROFILE_UPDATE.md](./ticket_platform_mobile/docs/api/BACKEND_REQUEST_PROFILE_UPDATE.md)

---

### 1.6 티켓 상세/구매 플로우 ✅ 완료
**기간**: 2-3주 (완료)
**담당**: Backend + Mobile
**우선순위**: 🔴 High

#### 백엔드 작업 ✅
- [x] 티켓 상세 조회 API (`GET /api/tickets/detail`)
- [x] 거래 생성 API (채팅방 생성과 함께)
- [x] 티켓 상태 관리
- [x] 찜 상태 포함 조회

#### 모바일 작업 ✅
- [x] 티켓 상세 페이지 UI (18개 위젯)
  - ticket_detail_performance_header
  - ticket_detail_seller_info
  - ticket_summary_section
  - ticket_detail_feature_section
  - ticket_detail_transaction_features
  - ticket_detail_trade_section
  - ticket_detail_warning_section
  - ticket_detail_bottom_action
  - 기타 리스트, 필터 위젯
- [x] 좌석 정보, 가격, 할인율 표시
- [x] 판매자 프로필 표시
- [x] 구매하기 버튼
- [x] 채팅 시작하기 버튼
- [x] 찜 토글

**결과물** ✅:
- Backend: `TicketController.cs` - 티켓 상세 조회
- Mobile: `lib/features/ticketing/` - 티켓 상세 UI 완성

---

### 1.7 티켓 판매 등록 ✅ 완료
**기간**: 2주 (완료)
**담당**: Backend + Mobile
**우선순위**: 🔴 High

#### 백엔드 작업 ✅
- [x] 카테고리 목록 API (`GET /api/sell/categories`)
- [x] 공연 목록 API (페이징) (`GET /api/sell/events`)
- [x] 공연 일정 API (`GET /api/sell/events/schedules`)
- [x] 좌석 옵션 API (`GET /api/sell/events/seat-options`)
- [x] 정가 조회 API (`GET /api/sell/events/original-price`)
- [x] 티켓 등록 API (`POST /api/sell/tickets`, multipart/form-data)
- [x] 티켓 이미지 업로드 (Supabase Storage)
- [x] 원가 검증 (판매가 ≤ 원가)
- [x] 내 판매 티켓 목록 (`GET /api/sell/my-tickets`)
- [x] 티켓 판매 취소 (`DELETE /api/sell/tickets`)
- [x] 이미지 URL 재발급 (`GET /api/sell/tickets/images/refresh`)
- [x] 특이사항 목록 (`GET /api/sell/features`)
- [x] 거래 방식 목록 (`GET /api/sell/trade-methods`)

#### 모바일 작업 ✅
- [x] 7단계 플로우 (총 2,357줄)
  1. sell_shell_view - 메인 셸
  2. sell_ticket_category_view - 카테고리 선택
  3. sell_event_selection_view - 공연 선택
  4. sell_date_time_selection_view - 날짜/시간
  5. sell_seat_info_view - 좌석 정보
  6. sell_price_view - 가격 설정
  7. sell_additional_info_view - 추가 정보
- [x] 이미지 다중 선택 (최대 5장)
- [x] 원가/판매가 입력 및 검증
- [x] 할인율 자동 계산
- [x] 단계별 유효성 검증
- [x] 미리보기 기능

**기술 스펙** ✅:
- 이미지: 최대 5장, JPG/PNG
- 가격 검증: 판매가 ≤ 원가
- 할인율: 자동 계산

**결과물** ✅:
- Backend: `SellController.cs` (229줄)
- Mobile: `lib/features/sell/` - 7개 화면 완성

---

### 1.8 결제 시스템 (토스 페이먼츠) ✅ 완료
**기간**: 2-3주 (완료)
**담당**: Backend + Mobile
**우선순위**: 🔴 High

#### 백엔드 작업 ✅
- [x] 토스 페이먼츠 API 연동
  - `POST /api/payment/request` - 결제 요청 (OrderId 생성)
  - `POST /api/payment/confirm` - 결제 승인
  - `POST /api/payment/cancel` - 결제 취소 (환불)
  - `GET /api/payment/order/{orderId}` - 결제 조회
  - `POST /api/payment/webhook` - 웹훅 수신
- [x] `Payment`, `Escrow` 엔티티 및 테이블
- [x] 결제 웹훅 처리 (IP 화이트리스트 검증)
- [x] 에스크로 보관금 상태 관리 (HOLD, RELEASED, FROZEN, REFUNDED)
- [x] Settlement(정산) 자동 생성 (구매확정 시)
- [x] 티켓 소유권 이전 (RemainingQuantity 감소)

#### 모바일 작업 ✅
- [x] 결제 페이지 UI
  - payment_view.dart
  - payment_final_view.dart (최종 확인)
  - payment_success_widget.dart
  - payment_failure_view.dart
- [x] 토스 페이먼츠 SDK 연동
- [x] 결제 성공/실패 처리
- [x] 채팅방에 PAYMENT_SUCCESS 메시지 자동 생성

**기술 스펙** ✅:
- PG사: 토스 페이먼츠
- 플랫폼 수수료: 5% 미만
- 에스크로 완성: 결제 → HOLD → 구매확정 → RELEASED → 정산(D+1)

**결과물** ✅:
- Backend: `PaymentController.cs` (142줄)
- Mobile: `lib/features/payment/` - 결제 UI 완성

---

### 1.9 찜 기능 ✅ 완료
**기간**: 3일 (완료)
**담당**: Backend + Mobile
**우선순위**: 🟡 Medium

#### 백엔드 작업 ✅
- [x] 찜 토글 API (`POST /api/favorites/tickets`)
- [x] 찜 목록 조회 API (`GET /api/favorites/tickets`)
- [x] `FavoriteTicket` 테이블

#### 모바일 작업 ✅
- [x] 하트 아이콘 토글 (티켓 상세, 목록)
- [x] 찜 목록 페이지 UI (wishlist_view.dart)
- [x] 찜 해제 기능

**결과물** ✅:
- Backend: `FavoriteController.cs` (72줄)
- Mobile: `lib/features/wishlist/` - 찜 기능 완성

---

### Phase 1 완료 기준 (85% 완료)
- [x] 사용자가 회원가입 → 티켓 검색 → 판매자와 채팅 → 결제 → 구매 확정까지 전체 플로우 완료
- [x] 판매자가 티켓 등록 → 구매자 채팅 → 티켓 전달 → 대금 수령 플로우 완료
- [x] 기본적인 에러 처리 및 로딩 UI 구현
- [ ] 주요 API 응답 시간 < 500ms (95 percentile) - 성능 테스트 필요
- [x] 거래 내역 조회 (구매/판매)
- [x] Settlement(정산) 레코드 자동 생성

**남은 작업**:
- 🚧 프로필 이미지 업로드 (모바일) - 1-2일
- ⏳ 통합 테스트 및 성능 최적화

---

## Phase 2: 검증 및 신뢰성 (1-2개월)

### 목표
티켓 진위 확인, 본인 인증, 신고/분쟁 처리로 플랫폼 신뢰도 향상

### 2.1 티켓 검증 시스템 ⏳ 예정
**기간**: ~2-3주
**담당**: Backend + Mobile
**우선순위**: 🔴 High

#### 검증 방법
1. **QR 코드 검증** (우선순위 1)
   - [ ] QR 코드 스캔 (모바일 카메라)
   - [ ] 해시 기반 검증 (AES 암호화)
   - [ ] 검증 결과 저장 및 알림

2. **OCR 검증** (우선순위 2)
   - [ ] 티켓 이미지 OCR 처리
   - [ ] 티켓 번호, 공연명, 날짜 추출
   - [ ] 추출 데이터와 등록 정보 대조

3. **티켓 번호 수동 입력** (우선순위 3)
   - [ ] 티켓 번호 입력 폼
   - [ ] 서버 검증 API

4. **이미지 업로드 검증** (우선순위 4)
   - [ ] 실물 티켓 사진 업로드
   - [ ] 관리자 수동 검증 (초기 단계)

**기술 스펙**:
- QR: AES 암호화, SHA-256 해시
- OCR: Google ML Kit 또는 Tesseract
- 검증 상태: `pending`, `verified`, `failed`, `manual_review`

**결과물**:
- Backend: `VerificationController`, OCR 서비스
- Mobile: QR 스캐너, 검증 UI

---

### 2.2 본인 인증 ⏳ 예정
**기간**: ~2주
**담당**: Backend + Mobile
**우선순위**: 🟡 Medium

#### 인증 종류
- [ ] 실명 인증 (본인 인증 API 연동)
- [ ] 휴대폰 인증 (SMS 인증)
- [ ] 계좌 인증 (1원 송금 또는 오픈뱅킹)

**정책**:
- 조회 시: 선택 사항
- 거래 시: 필수 (구매자 + 판매자)

**기술 스펙**:
- 본인 인증: NICE 평가정보 또는 PASS
- SMS: NHN Cloud SMS 또는 Solapi
- 계좌 인증: 토스 오픈뱅킹

**결과물**:
- Backend: `VerificationController`, 외부 API 연동
- Mobile: 인증 플로우 UI

---

### 2.3 신고/분쟁 시스템 ⏳ 예정
**기간**: ~2주
**담당**: Backend + Mobile
**우선순위**: 🟡 Medium

#### 참여자 역할
- **구매자/판매자**: 문제 제기
- **관리자**: 심사 및 판결

#### 프로세스
1. **신고 접수**
   - [ ] 신고 사유 선택 (사기, 잘못된 티켓, 태도 불량 등)
   - [ ] 증거 첨부 (채팅 로그, 이미지)

2. **심사**
   - [ ] 관리자 대시보드에서 신고 목록 조회
   - [ ] 증거 검토 (채팅 로그, 이미지)
   - [ ] 양측 소명 확인

3. **처리**
   - [ ] 승인: 환불 처리 + 페널티 부과
   - [ ] 거절: 거래 진행
   - [ ] 보류: 추가 증거 요청

**기술 스펙**:
- 신고 상태: `pending`, `in_review`, `resolved`, `rejected`
- 에스크로 상태 변경: `HOLD` → `FROZEN` (신고 접수 시)
- 페널티: 경고, 계정 정지, 영구 차단

**결과물**:
- Backend: `DisputeController`, 관리자 API
- Mobile: 신고 UI, 관리자 대시보드 (웹)

---

### 2.4 평판 시스템 ⏳ 예정
**기간**: ~1주
**담당**: Backend + Mobile
**우선순위**: 🟢 Low

- [ ] 거래 완료 후 상대방 평가 (별점 1-5)
- [ ] 평가 내용 입력 (선택)
- [ ] 사용자 프로필에 평점 표시
- [ ] 거래 횟수 표시

**결과물**:
- Backend: `RatingController`
- Mobile: 평가 모달, 프로필에 평점 표시

---

### Phase 2 완료 기준
- [ ] QR 코드 검증 기능 동작
- [ ] 본인 인증 완료 후에만 결제 가능
- [ ] 신고 접수 및 관리자 처리 가능
- [ ] 평판 시스템 정상 동작

---

## Phase 3: 정산 및 운영 (1개월)

### 목표
자동 정산 시스템 구현, 관리자 도구 개발, 운영 안정성 확보

### 3.1 에스크로 정산 시스템 ⏳ 예정
**기간**: ~2주
**담당**: Backend
**우선순위**: 🔴 High

#### 정산 정책
- 정산 주기: **D+3** (분쟁 대비)
- 필수 조건: 판매자 실명 인증 + 계좌 인증 완료
- 재시도 로직: 정산 실패 시 3-5회 자동 재시도

#### 백엔드 작업
- [ ] `Settlement` 엔티티 및 테이블 설계
- [ ] 정산 스케줄러 (Quartz.NET 또는 BackgroundService)
- [ ] 정산 API (수동 정산, 재시도)
- [ ] 정산 내역 조회 API
- [ ] 에스크로 상태 변경: `RELEASED` → 정산 완료

#### 환불 사유 처리
- [ ] QR 검증 실패
- [ ] 잘못된 티켓 전달
- [ ] 공연/경기 취소
- [ ] 사기 의심 신고 (관리자 승인 시)

**결과물**:
- Backend: `SettlementService`, 스케줄러, 정산 API

---

### 3.2 관리자 대시보드 ⏳ 예정
**기간**: ~2주
**담당**: Backend + Web (React/Vue 추가 또는 Blazor)
**우선순위**: 🟡 Medium

#### 기능
- [ ] 사용자 관리 (조회, 정지, 삭제)
- [ ] 신고/분쟁 관리 (조회, 심사, 처리)
- [ ] 거래 모니터링 (실시간 거래 현황)
- [ ] 티켓 관리 (승인, 삭제)
- [ ] 정산 관리 (수동 정산, 정산 내역)
- [ ] 통계 대시보드 (매출, 거래량, 사용자 증가)

**기술 스펙**:
- 관리자 전용 API (역할 기반 접근 제어)
- 웹 프론트엔드: React/Vue 또는 ASP.NET Core MVC

**결과물**:
- Backend: 관리자 API
- Web: 관리자 대시보드

---

### 3.3 알림 시스템 ⏳ 예정
**기간**: ~1주
**담당**: Backend + Mobile
**우선순위**: 🟡 Medium

#### 발송 시점
- [ ] 구매 요청 도착 (판매자에게)
- [ ] 결제 완료 (구매자, 판매자)
- [ ] 검증 요청 (판매자에게)
- [ ] 구매 확정 (판매자에게)
- [ ] 정산 완료 (판매자에게)
- [ ] 신고 접수 (양측)
- [ ] 채팅 메시지 수신 (앱이 백그라운드일 때)

#### 알림 채널
- [ ] 앱 푸시 알림 (FCM)
- [ ] 이메일 알림 (선택)
- [ ] SMS 알림 (중요한 경우만)

**기술 스펙**:
- 푸시: Firebase Cloud Messaging (FCM)
- 이메일: SendGrid 또는 AWS SES
- SMS: NHN Cloud SMS

**결과물**:
- Backend: `NotificationService`, FCM 연동
- Mobile: FCM 설정, 알림 처리

---

### 3.4 모니터링 및 로깅 ⏳ 예정
**기간**: ~1주
**담당**: Backend
**우선순위**: 🟡 Medium

- [ ] 구조화된 로그 (Serilog)
- [ ] 로그 수집 (ELK Stack 또는 Seq)
- [ ] 성능 메트릭 (Application Insights 또는 Prometheus)
- [ ] 에러 추적 (Sentry)
- [ ] 알림 설정 (에러율, 응답시간 초과)

**결과물**:
- 로그 인프라 구축
- 대시보드 설정

---

### Phase 3 완료 기준
- [ ] D+3 자동 정산 시스템 동작
- [ ] 관리자가 신고/분쟁 처리 가능
- [ ] 주요 이벤트에 푸시 알림 발송
- [ ] 로그 및 메트릭 모니터링 가능

---

## Phase 4: 출시 준비 (1개월)

### 목표
성능 최적화, 보안 강화, 테스트 완료, 앱스토어 심사 준비

### 4.1 성능 최적화 ⏳ 예정
**기간**: ~1주
**담당**: Backend + Mobile

#### 백엔드 최적화
- [ ] 데이터베이스 인덱싱 최적화
- [ ] 쿼리 성능 개선 (EXPLAIN 분석)
- [ ] Redis 캐싱 (사용자 프로필, 티켓 목록)
- [ ] API 응답 시간 목표: 95 percentile < 500ms

#### 모바일 최적화
- [ ] 이미지 레이지 로딩 및 캐싱
- [ ] 네트워크 요청 최적화 (배칭, 재시도)
- [ ] 앱 시작 시간 단축
- [ ] 메모리 누수 체크

**결과물**:
- 성능 테스트 보고서
- 최적화 전후 비교

---

### 4.2 보안 강화 ⏳ 예정
**기간**: ~1주
**담당**: Backend + Mobile

- [ ] 보안 취약점 스캔 (OWASP Top 10)
- [ ] SQL Injection 방지 (파라미터화 쿼리)
- [ ] XSS 방지
- [ ] CSRF 방지
- [ ] API Rate Limiting (Polly)
- [ ] 민감 정보 암호화 (PII, 결제 정보)
- [ ] HTTPS 강제
- [ ] SSL Pinning (모바일)

**결과물**:
- 보안 감사 보고서

---

### 4.3 테스트 ⏳ 예정
**기간**: ~2주
**담당**: Backend + Mobile + QA

#### 백엔드 테스트
- [ ] 단위 테스트 (xUnit, Moq)
  - Service Layer
  - Repository Layer
- [ ] 통합 테스트 (WebApplicationFactory)
  - API 엔드포인트
  - 데이터베이스 연동
- [ ] 부하 테스트 (JMeter, K6)
  - 목표: 100 concurrent users, 200 req/sec

#### 모바일 테스트
- [ ] 위젯 테스트 (Flutter)
- [ ] 통합 테스트 (Flutter)
- [ ] E2E 테스트 (Patrol 또는 Maestro)

#### QA 테스트
- [ ] 시나리오 테스트 (회원가입 → 구매 → 판매)
- [ ] 버그 리포트 및 수정

**결과물**:
- 테스트 커버리지 > 70%
- QA 테스트 보고서

---

### 4.4 앱스토어 심사 준비 ⏳ 예정
**기간**: ~1주
**담당**: Mobile

#### iOS 앱스토어
- [ ] 앱 아이콘, 스크린샷 준비
- [ ] 앱 설명, 키워드 작성
- [ ] 개인정보 처리방침 페이지 작성
- [ ] 서비스 이용약관 페이지 작성
- [ ] App Store Connect 등록
- [ ] TestFlight 베타 테스트

#### Google Play Store
- [ ] 앱 아이콘, 스크린샷 준비
- [ ] 앱 설명 작성
- [ ] 개인정보 처리방침 페이지 작성
- [ ] 서비스 이용약관 페이지 작성
- [ ] Google Play Console 등록
- [ ] 내부 테스트 트랙 배포

**결과물**:
- 앱스토어 등록 완료
- 베타 테스터 피드백 수집

---

### Phase 4 완료 기준
- [ ] 모든 테스트 통과
- [ ] 보안 감사 완료
- [ ] 성능 목표 달성
- [ ] 앱스토어 심사 통과 준비 완료

---

## Launch 🚀

### 출시 계획
1. **소프트 런치** (Soft Launch)
   - 제한된 사용자 대상 (초대제)
   - 피드백 수집 및 버그 수정
   - 기간: 2-4주

2. **정식 출시** (Public Launch)
   - 앱스토어 공개
   - 마케팅 캠페인 시작
   - 사용자 증가 모니터링

---

## 우선순위 요약

### 🔴 High (즉시 착수)
1. 티켓 상세/구매 플로우
2. 티켓 판매 등록
3. 결제 시스템 (토스 페이먼츠)
4. 티켓 검증 시스템
5. 에스크로 정산 시스템

### 🟡 Medium (Phase 2-3)
1. 프로필 업데이트
2. 찜 기능
3. 본인 인증
4. 신고/분쟁 시스템
5. 관리자 대시보드
6. 알림 시스템

### 🟢 Low (Phase 3-4)
1. 평판 시스템
2. 모니터링 및 로깅
3. 성능 최적화
4. 보안 강화

---

## 마일스톤

| 마일스톤 | 예상 완료일 | 상태 |
|---------|------------|------|
| Phase 1 완료 (MVP) | 2026-02-10 (예상) | 🚧 85% 완료 (거의 완료) |
| Phase 2 시작 (검증 및 신뢰성) | 2026-02-15 (예상) | ⏳ 예정 |
| Phase 2 완료 | TBD | ⏳ 예정 |
| Phase 3 완료 (정산 및 운영) | TBD | ⏳ 예정 |
| Phase 4 완료 (출시 준비) | TBD | ⏳ 예정 |
| 소프트 런치 | TBD | ⏳ 예정 |
| 정식 출시 | TBD | ⏳ 예정 |

---

## 리스크 및 대응 방안

### 리스크 1: 결제 연동 복잡도
- **리스크**: 토스 페이먼츠 연동 및 에스크로 처리가 예상보다 복잡
- **대응**: 초기 단계에 POC (Proof of Concept) 진행, 토스 페이먼츠 기술 지원 요청

### 리스크 2: 티켓 검증 정확도
- **리스크**: QR/OCR 검증 실패율 높음
- **대응**: 초기에는 수동 검증 병행, 점진적으로 자동화 개선

### 리스크 3: 동시성 이슈
- **리스크**: 여러 사용자가 동시에 같은 티켓 구매 시도
- **대응**: DB 락 또는 Redis 분산 락 구현, 부하 테스트로 검증

### 리스크 4: 개발 일정 지연
- **리스크**: 예상 개발 기간 초과
- **대응**: 우선순위 재조정, MVP 기능 축소, 일부 기능 Phase 2로 이동

---

**문서 버전**: 1.0
**작성일**: 2026-02-09
**최종 수정일**: 2026-02-09
