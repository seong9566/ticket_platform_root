# Agent 작업 전달: Phase 2 Sprint 2-A (FCM 푸시 알림 + 신고/분쟁 시스템)

**작성일**: 2026-02-11
**작성자**: PM
**대상**: Backend Agent, Mobile Agent
**목적**: Phase 2 Sprint 2-A 기능 구현 지시

---

## 📋 Sprint 컨텍스트

### Phase 1 완료 상태
| TASK | 내용 | 상태 |
|------|------|------|
| TASK-001 | TossPayments 환경 설정 분리 (Backend) | ✅ 완료 |
| TASK-001 | AppConfig 환경 설정 (Mobile) | ✅ 완료 |
| TASK-002 | OAuth Google/Kakao (Backend + Mobile) | ✅ 완료 |
| TASK-003 | API Route 통일 (Backend) | ✅ 완료 |
| TASK-004 | 비밀번호 변경 (Backend + Mobile) | ✅ 완료 |
| TASK-003 | Release Build (Mobile) | ⏳ 대기 |

Apple Sign-In은 추후 진행. Phase 2로 진입한다.

### ⚠️ 플랫폼 제약: Apple Developer 계정 미확보

Apple 유료 개발자 계정($99/년)이 현재 없으며, **1-2개월 내 확보 예정**이다.

**영향 범위**:
- iOS APNs 인증 키 등록 불가 → iOS FCM 푸시 수신 불가
- iOS Release Build (IPA/TestFlight/App Store) 불가
- Apple Sign-In 구현 불가

**대응 전략: Android-First**:
- 모든 모바일 기능을 Android에서 우선 구현/테스트/검증
- Flutter 코드는 양 플랫폼 공유이므로 iOS용 별도 코드 작성 불필요
- iOS 전용 설정(APNs, capability, 인증서)만 보류
- 계정 확보 후 **iOS 일괄 검증 Sprint** 진행

### Sprint 2-A 범위

| TASK | 내용 | 담당 |
|------|------|------|
| TASK-005 | FCM 푸시 알림 | Backend + Mobile |
| TASK-006 | 신고/분쟁 시스템 | Backend + Mobile |

---

## 🚀 Agent 실행 명령

### Backend Agent — TASK-005 (FCM 푸시 알림)

```bash
# Backend Agent Session
cd TicketPlatFormServer/

# 작업 문서 읽기
Read: docs/backend/tasks/TASK-005_FCM_Push_Notification.md

# 구현 시작
"TASK-005 FCM 푸시 알림 시스템을 구현하라. 디바이스 토큰 관리, 알림 CRUD, FCM HTTP v1 발송, 알림 트리거를 구현하라."
```

**산출물**:
- `POST /api/notifications/token` — 토큰 등록
- `DELETE /api/notifications/token` — 토큰 삭제
- `GET /api/notifications` — 알림 목록
- `PUT /api/notifications/{id}/read` — 읽음 처리
- `PUT /api/notifications/read-all` — 전체 읽음
- `GET /api/notifications/unread-count` — 미읽음 카운트
- FCM HTTP v1 발송 서비스
- 기존 서비스 트리거 추가 (ChatService, PaymentService 등)

**예상 시간**: 15시간 (약 2일)

---

### Backend Agent — TASK-006 (신고/분쟁 시스템)

```bash
# Backend Agent Session
cd TicketPlatFormServer/

# 작업 문서 읽기
Read: docs/backend/tasks/TASK-006_Dispute_System.md

# 구현 시작
"TASK-006 신고/분쟁 시스템을 구현하라. 신고 생성/조회/취소, 증거 첨부, Escrow 상태 전환을 구현하라."
```

**산출물**:
- `POST /api/disputes` — 신고 생성
- `GET /api/disputes` — 내 신고 목록
- `GET /api/disputes/{id}` — 신고 상세
- `POST /api/disputes/{id}/evidence` — 증거 첨부
- `PUT /api/disputes/{id}/cancel` — 신고 취소
- Escrow 상태 전환 (HOLD ↔ FROZEN)

**예상 시간**: 14시간 (약 2일)

---

### Mobile Agent — TASK-005 (FCM 푸시 알림)

```bash
# Mobile Agent Session
cd ticket_platform/

# 작업 문서 읽기
Read: docs/mobile/tasks/TASK-005_FCM_Push_Notification.md

# 구현 시작
"TASK-005 FCM 푸시 알림 클라이언트를 구현하라. 토큰 관리, 포어그라운드/백그라운드 알림, 딥링크, 알림 목록 화면, 뱃지를 구현하라."
```

**산출물**:
- FCM 토큰 획득/등록/갱신
- 포어그라운드 인앱 배너
- 백그라운드 시스템 알림
- 알림 탭 딥링크 이동
- 알림 목록 화면 (무한 스크롤)
- 미읽음 뱃지

**예상 시간**: 18시간 (약 2.5일)

---

### Mobile Agent — TASK-006 (신고/분쟁 시스템)

```bash
# Mobile Agent Session
cd ticket_platform/

# 작업 문서 읽기
Read: docs/mobile/tasks/TASK-006_Dispute_System.md

# 구현 시작
"TASK-006 신고/분쟁 시스템 클라이언트를 구현하라. 신고 생성 플로우, 증거 업로드, 신고 목록/상세 화면을 구현하라."
```

**산출물**:
- 신고 생성 3단계 플로우
- 증거 이미지 업로드
- 신고 목록/상세 화면
- 기존 화면에 신고 진입점 추가
- 신고 취소 기능

**예상 시간**: 17시간 (약 2.5일)

---

## 🔗 의존성 다이어그램

```
          TASK-005 (FCM)              TASK-006 (신고)
     ┌─────────────────┐        ┌─────────────────┐
     │                 │        │                 │
     │  Backend FCM    │        │  Backend 신고    │
     │  (15시간)       │        │  (14시간)       │
     │                 │        │                 │
     └────────┬────────┘        └────────┬────────┘
              │                          │
              │  ←── 병렬 개발 가능 ──→    │
              │                          │
     ┌────────┴────────┐        ┌────────┴────────┐
     │                 │        │                 │
     │  Mobile FCM     │        │  Mobile 신고     │
     │  (17시간)       │        │  (17시간)       │
     │  Android only   │        │  Android only   │
     │                 │        │                 │
     └────────┬────────┘        └────────┴────────┘
              │                          │
              └─────── 후반부 연동 ────────┘
                  (신고 알림 발송 + 딥링크)
```

### 의존성 규칙
1. **TASK-005와 TASK-006은 서로 독립** → 병렬 개발 가능
2. **Backend → Mobile**: 각 Mobile 작업은 해당 Backend API 완료 후 API 연동 단계 진행
3. **후반부 연동**: TASK-006의 신고 알림(DISPUTE_OPENED/RESOLVED)은 TASK-005 완료 후 연동
4. **Mobile UI 선행 개발**: API 명세가 확정되어 있으므로 Backend 완료 전 Mock 데이터로 UI 구현 가능

### 권장 실행 순서

```
Day 1-2:
├── Backend Agent: TASK-005 FCM 구현 (병렬)
├── Backend Agent: TASK-006 신고 구현 (병렬)
├── Mobile Agent: TASK-005 UI 선행 개발 (Mock)
└── Mobile Agent: TASK-006 UI 선행 개발 (Mock)

Day 3:
├── Mobile Agent: TASK-005 API 연동
├── Mobile Agent: TASK-006 API 연동
└── Backend Agent: TASK-005 ↔ TASK-006 알림 트리거 연동

Day 4:
├── 통합 테스트 (Android)
└── 버그 수정

[Apple Developer 계정 확보 후]:
├── iOS 일괄 검증 Sprint
│   ├── APNs 인증 키 등록
│   ├── iOS Push Notification capability 설정
│   ├── iOS 푸시 알림 수신 테스트 (실기기)
│   ├── iOS Release Build (IPA)
│   └── Apple Sign-In 구현
```

---

## 🧪 통합 테스트 시나리오 (End-to-End, Android)

### E2E 시나리오 1: 채팅 메시지 알림 플로우
```
1. 사용자 A가 앱 시작 → FCM 토큰 서버 등록
2. 사용자 B가 A에게 채팅 메시지 전송
3. Backend가 CHAT_MESSAGE 알림 생성 + FCM 발송
4. A의 디바이스에 푸시 알림 수신
   - 포어그라운드: 인앱 배너 표시
   - 백그라운드: 시스템 알림
5. A가 알림 탭 → 채팅방으로 이동
6. A가 알림 목록 확인 → 해당 알림이 읽음 처리됨
```

### E2E 시나리오 2: 결제 → 신고 → 알림 플로우
```
1. 구매자가 티켓 결제 완료
2. Backend가 PAYMENT_SUCCESS 알림 발송 (구매자 + 판매자)
3. 양측 디바이스에 푸시 알림 수신
4. 구매자가 거래 상세에서 "신고하기" 진입
5. 신고 생성 (사유: FAKE_TICKET, 설명 입력, 증거 2장)
6. Backend가 Escrow HOLD → FROZEN 전환
7. Backend가 DISPUTE_OPENED 알림 발송 (판매자)
8. 판매자가 알림 수신 → 탭하여 신고 상세 확인
```

### E2E 시나리오 3: 신고 취소 플로우
```
1. 구매자가 PENDING 상태 신고 상세 진입
2. "신고 취소" 탭 → 확인 다이얼로그 → 취소 실행
3. Backend가 Escrow FROZEN → HOLD 복원
4. 신고 상태가 CANCELLED로 변경
5. 신고 목록에서 상태 업데이트 확인
```

### E2E 시나리오 4: 알림 목록 + 읽음 처리
```
1. 사용자에게 미읽음 알림 10건 존재
2. 하단 네비게이션에 뱃지 "10" 표시
3. 알림 목록 진입 → 미읽음 항목 배경색 구분
4. 알림 1건 탭 → 읽음 처리 → 뱃지 "9"로 감소
5. "전체 읽음" 탭 → 모든 알림 읽음 처리 → 뱃지 사라짐
```

---

## 📊 작업 추적 테이블

### Backend Agent 진행 상황

| TASK | 단계 | 상태 | 시작 | 완료 |
|------|------|------|------|------|
| TASK-005 | DTO 생성 | ⏳ | | |
| TASK-005 | Repository 구현 | ⏳ | | |
| TASK-005 | NotificationService 구현 | ⏳ | | |
| TASK-005 | FcmService 구현 | ⏳ | | |
| TASK-005 | Controller 구현 | ⏳ | | |
| TASK-005 | 트리거 추가 | ⏳ | | |
| TASK-005 | 테스트 | ⏳ | | |
| TASK-006 | DTO 생성 | ⏳ | | |
| TASK-006 | Repository 구현 | ⏳ | | |
| TASK-006 | DisputeService 구현 | ⏳ | | |
| TASK-006 | Controller 구현 | ⏳ | | |
| TASK-006 | 이미지 업로드 연동 | ⏳ | | |
| TASK-006 | 테스트 | ⏳ | | |

### Mobile Agent 진행 상황

| TASK | 단계 | 상태 | 시작 | 완료 |
|------|------|------|------|------|
| TASK-005 | Firebase 설정 | ⏳ | | |
| TASK-005 | FCM 토큰 관리 | ⏳ | | |
| TASK-005 | 알림 수신 처리 | ⏳ | | |
| TASK-005 | 딥링크 핸들러 | ⏳ | | |
| TASK-005 | 알림 목록 화면 | ⏳ | | |
| TASK-005 | 읽음 처리 + 뱃지 | ⏳ | | |
| TASK-005 | 테스트 | ⏳ | | |
| TASK-006 | DTO/Repository 생성 | ⏳ | | |
| TASK-006 | 신고 생성 플로우 | ⏳ | | |
| TASK-006 | 신고 목록 화면 | ⏳ | | |
| TASK-006 | 신고 상세 화면 | ⏳ | | |
| TASK-006 | 증거 업로드 | ⏳ | | |
| TASK-006 | 기존 화면 수정 | ⏳ | | |
| TASK-006 | 테스트 | ⏳ | | |

**범례**: ⏳ 대기 | 🚧 진행중 | ✅ 완료

---

## ⚠️ 리스크 대응

### 리스크 1: Apple Developer 계정 미확보 (확정)
**영향**: iOS 전체 — 푸시 알림, Release Build, Apple Sign-In 불가
**확률**: 확정 (1-2개월 내 확보 예정)
**대응**:
- **Android-First 전략** 채택: 모든 기능을 Android에서 우선 구현/검증
- Flutter 코드는 양 플랫폼 공유 → iOS용 별도 코드 작성 불필요
- iOS 전용 설정(APNs 키, capability, 인증서)만 보류
- 계정 확보 후 **iOS 일괄 검증 Sprint** 실행 (예상 3-5일)
- iOS 검증 Sprint 범위: APNs 키 등록, Push Notification capability, Release Build, Apple Sign-In

### 리스크 2: Firebase Android 설정 파일 미확보
**영향**: Android FCM 기능 전체 구현 불가
**확률**: 중
**대응**:
- Firebase Console에서 즉시 프로젝트 생성 및 google-services.json 발급
- iOS GoogleService-Info.plist는 보류 (APNs 키 없이 푸시 미동작)

### 리스크 3: FCM 서비스 계정 키 미확보
**영향**: Backend FCM 발송 기능 구현 불가
**확률**: 중
**대응**:
- Firebase Console → 프로젝트 설정 → 서비스 계정에서 JSON 키 발급
- appsettings에 경로 설정

### 리스크 4: Escrow 상태 전환 동시성 이슈
**영향**: 신고 생성과 구매 확정이 동시 발생 시 데이터 불일치
**확률**: 낮
**대응**:
- DB 트랜잭션 격리 수준 설정
- 낙관적 잠금 또는 비관적 잠금 적용
- 상태 전환 전 현재 상태 재확인

### 리스크 5: Backend API 지연으로 Mobile 작업 블록
**영향**: Mobile API 연동 지연
**확률**: 낮
**대응**:
- API 명세가 확정되어 있으므로 Mock 데이터로 UI 선행 개발
- Backend 완료 후 실제 API로 교체

---

## 📞 Agent 간 커뮤니케이션

Agent는 직접 소통할 수 없으므로 다음 방식으로 조율:

### 문제 발생 시
1. Agent가 PM Session에 보고
2. PM이 다른 Agent에게 수정 지시
3. 문서 업데이트 후 재전달

### API 명세 변경 시
1. Backend Agent가 구현 중 명세 변경 필요 시 보고
2. PM이 명세서 업데이트
3. Mobile Agent에게 변경 사항 전달

### 통합 테스트
1. Backend + Mobile 각각 구현 완료
2. PM이 E2E 테스트 시나리오 전달
3. 통합 테스트 결과 보고

---

## 📊 Sprint 전체 현황

| 항목 | 수치 |
|------|------|
| 총 작업 문서 | 4건 (Backend 2 + Mobile 2) |
| 총 API 엔드포인트 | 11개 (FCM 6 + 신고 5) |
| 총 예상 시간 (Android-First) | 63시간 (약 8일) |
| Backend 총 시간 | 29시간 |
| Mobile 총 시간 (Android) | 34시간 |
| ⏸️ iOS 일괄 검증 Sprint (계정 확보 후) | 별도 3-5일 |
| 테스트 시나리오 | 24건 (각 6건 × 4문서) |
| E2E 테스트 시나리오 | 4건 (Android) |

---

## 📝 PM 확인 필요 사항

### 즉시 확인
- [ ] Firebase 프로젝트 생성 여부
- [ ] FCM 서비스 계정 키 발급
- [ ] Android google-services.json 확보
- [ ] NotificationType, DisputeType, DisputeStatus, EscrowStatus seed 데이터 등록 상태

### 구현 후 확인
- [ ] Escrow 상태 전환 동시성 처리 검증
- [ ] FCM 발송 성공률 모니터링 (Android)
- [ ] 관리자 신고 해결 기능 Phase 3 일정

### Apple Developer 계정 확보 후 (1-2개월 내)
- [ ] APNs 인증 키 Firebase Console 등록
- [ ] iOS GoogleService-Info.plist 적용
- [ ] Xcode Push Notifications capability 설정
- [ ] iOS Release Build (IPA/TestFlight)
- [ ] Apple Sign-In 구현
- [ ] iOS 전체 기능 일괄 검증

---

**작성자**: PM
**작성일**: 2026-02-11
**상태**: 📤 전달 준비 완료
