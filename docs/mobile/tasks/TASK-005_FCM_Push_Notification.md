# TASK-005: FCM 푸시 알림 (모바일)

**작성일**: 2026-02-11
**작성자**: PM
**담당 팀**: Mobile
**담당자**: Mobile Agent
**상태**: 🚧 진행 중
**우선순위**: 🔴 High

---

## 📋 작업 개요

### 작업 설명
Firebase Cloud Messaging(FCM)을 연동하여 푸시 알림 수신, 표시, 딥링크 이동 기능을 구현하라. 알림 목록 화면과 하단 네비게이션 미읽음 뱃지를 추가하라.

### 목표
- FCM 토큰 획득 및 서버 등록
- 포어그라운드/백그라운드 알림 수신 처리
- 알림 탭 시 해당 화면으로 딥링크 이동
- 알림 목록 화면 구현
- 하단 네비게이션 바에 미읽음 뱃지 표시

### 배경
Phase 2 핵심 기능으로, 채팅 메시지, 결제, 구매 확정, 신고 등 주요 이벤트 알림을 사용자에게 실시간 전달해야 한다. Backend TASK-005에서 FCM 발송 시스템을 구현하며, 모바일에서는 수신/표시/관리 기능을 담당한다.

### ⚠️ 플랫폼 전략: Android-First
Apple 유료 개발자 계정 미확보 상태 (1-2개월 내 확보 예정). 다음 전략을 따르라:
- **Android**: 전체 기능 구현 + 테스트 + 검증
- **iOS**: Flutter 코드는 공유되므로 자동 포함. 단, iOS 전용 설정(APNs 키 등록, Push Notification capability)은 **보류**
- **iOS GoogleService-Info.plist**: Firebase Console에서 발급 가능하나, 푸시 알림 수신은 APNs 키 등록 전까지 동작하지 않음
- 계정 확보 후 iOS 일괄 검증 Sprint에서 테스트 진행

---

## 🎯 완료 기준 (Acceptance Criteria)

- [x] Firebase 프로젝트 설정 완료 (Android: google-services.json)
- [ ] ⏸️ iOS Firebase 설정 (GoogleService-Info.plist) — Apple Developer 계정 확보 후
- [ ] ⏸️ iOS APNs 인증 키 등록 — Apple Developer 계정 확보 후
- [x] 앱 시작 시 FCM 토큰 획득 → `POST /api/notifications/token` 서버 등록 (Android 검증)
- [x] 포어그라운드 알림: 인앱 배너로 표시
- [x] 백그라운드 알림: 시스템 알림 트레이에 표시
- [x] 알림 탭 시 해당 화면으로 이동 (딥링크)
- [x] 알림 목록 화면 구현 (cursor pagination, 무한 스크롤)
- [x] 개별/전체 읽음 처리 동작
- [x] 하단 네비게이션 바에 미읽음 뱃지 표시
- [x] 로그아웃 시 FCM 토큰 서버에서 삭제

---

## 🔧 기술 스펙 (Mobile)

### Firebase 프로젝트 설정

#### Android
- `android/app/google-services.json` 파일 추가
- `android/build.gradle`에 Google Services 플러그인 추가
- `android/app/build.gradle`에 Firebase 의존성 추가

#### iOS (⏸️ Apple Developer 계정 확보 후 진행)
- `ios/Runner/GoogleService-Info.plist` 파일 추가 — Firebase Console에서 발급 가능
- Xcode에서 Push Notifications capability 활성화 — **유료 계정 필요**
- APNs 인증 키를 Firebase Console에 등록 — **유료 계정 필요**
- iOS 푸시 알림은 APNs 키 없이는 동작하지 않음. Flutter 코드는 공유되므로 별도 분기 불필요

### FCM 토큰 관리 플로우

```
앱 시작
  ↓
FirebaseMessaging.instance.getToken()
  ↓
토큰 획득 성공
  ↓
POST /api/notifications/token (서버 등록)
  ↓
FirebaseMessaging.instance.onTokenRefresh 구독
  ↓
토큰 갱신 시 서버에 재등록
```

- 로그인 성공 시: 토큰 등록
- 로그아웃 시: `DELETE /api/notifications/token`으로 토큰 삭제
- 토큰 갱신 시: 자동 재등록

### 포어그라운드 알림

- `FirebaseMessaging.onMessage` 스트림 구독
- 알림 수신 시 화면 상단에 인앱 배너(SnackBar 또는 OverlayEntry)로 표시
- 배너 탭 시 해당 화면으로 이동
- 현재 채팅방에 있는 경우, 해당 채팅방의 CHAT_MESSAGE 알림은 배너 미표시

### 백그라운드 알림

- `FirebaseMessaging.onBackgroundMessage` 핸들러 등록
- 시스템 알림 트레이에 자동 표시 (FCM notification payload 활용)
- 알림 탭하여 앱 진입 시 `FirebaseMessaging.onMessageOpenedApp` 처리

### 알림 탭 시 화면 이동 (딥링크)

| 알림 타입 | data 필드 | 이동 화면 |
|-----------|-----------|-----------|
| `CHAT_MESSAGE` | `roomId` | 채팅방 상세 |
| `PAYMENT_REQUEST` | `transactionId` | 거래 상세 (결제 진행) |
| `PAYMENT_SUCCESS` | `transactionId` | 거래 상세 |
| `PURCHASE_CONFIRMED` | `transactionId` | 거래 상세 |
| `DISPUTE_OPENED` | `disputeId` | 신고 상세 |
| `DISPUTE_RESOLVED` | `disputeId` | 신고 상세 |

### 알림 목록 화면

- 프로필 탭에서 접근 또는 앱바 알림 아이콘으로 접근
- `GET /api/notifications?cursor={cursor}&limit=20` 호출
- 무한 스크롤 (cursor pagination)
- 각 알림 항목: 아이콘 + 제목 + 내용 + 시간 + 읽음 상태
- 미읽음 알림은 배경색 구분
- 알림 탭 시 읽음 처리 (`PUT /api/notifications/{id}/read`) + 화면 이동
- 상단에 "전체 읽음" 버튼 (`PUT /api/notifications/read-all`)

### 하단 네비게이션 미읽음 뱃지

- `GET /api/notifications/unread-count` 주기적 호출 (앱 포어그라운드 진입 시)
- 푸시 수신 시 카운트 로컬 증가
- 읽음 처리 시 카운트 로컬 감소
- 뱃지에 숫자 표시 (99+)

### API 연동

| 엔드포인트 | 용도 |
|------------|------|
| `POST /api/notifications/token` | FCM 토큰 등록 |
| `DELETE /api/notifications/token` | FCM 토큰 삭제 |
| `GET /api/notifications` | 알림 목록 조회 |
| `PUT /api/notifications/{id}/read` | 개별 읽음 처리 |
| `PUT /api/notifications/read-all` | 전체 읽음 처리 |
| `GET /api/notifications/unread-count` | 미읽음 카운트 |

---

## 📱 화면 구조

### 알림 목록 화면
```
┌──────────────────────────┐
│  ← 알림            전체 읽음 │
├──────────────────────────┤
│ ┌──────────────────────┐ │
│ │ 💬 새 메시지           │ │  ← 미읽음 (배경색)
│ │ 홍길동님이 메시지를...   │ │
│ │                 2분 전 │ │
│ └──────────────────────┘ │
│ ┌──────────────────────┐ │
│ │ 💰 결제 완료           │ │  ← 읽음 (기본 배경)
│ │ 150,000원 결제가...    │ │
│ │               1시간 전 │ │
│ └──────────────────────┘ │
│ ┌──────────────────────┐ │
│ │ ⚠️ 신고 접수           │ │
│ │ 거래 #10에 대한 신고... │ │
│ │                 어제   │ │
│ └──────────────────────┘ │
│         ...무한 스크롤     │
├──────────────────────────┤
│  🏠   🔍   💬   👤      │  ← 하단 네비게이션 (뱃지)
└──────────────────────────┘
```

### 인앱 배너 (포어그라운드)
```
┌──────────────────────────┐
│ 💬 홍길동님이 메시지를 보냈습니다  │  ← 상단 배너 (3초 후 자동 사라짐)
└──────────────────────────┘
```

---

## 📂 파일 구조

### Mobile
```
lib/features/notification/
├── data/
│   ├── datasources/
│   │   └── notification_remote_datasource.dart  — API 호출
│   ├── dto/
│   │   ├── register_token_req_dto.dart          — 토큰 등록 요청
│   │   ├── notification_list_resp_dto.dart      — 알림 목록 응답
│   │   └── unread_count_resp_dto.dart           — 미읽음 카운트 응답
│   └── repositories/
│       └── notification_repository_impl.dart    — Repository 구현
├── domain/
│   ├── entities/
│   │   └── notification_entity.dart             — 알림 엔티티
│   ├── repositories/
│   │   └── notification_repository.dart         — Repository 인터페이스
│   └── usecases/
│       ├── get_notifications_usecase.dart       — 알림 목록 조회
│       ├── mark_read_usecase.dart               — 읽음 처리
│       └── get_unread_count_usecase.dart        — 미읽음 카운트
└── presentation/
    ├── viewmodels/
    │   ├── notification_list_viewmodel.dart     — 목록 상태 관리
    │   └── unread_badge_viewmodel.dart          — 뱃지 카운트 관리
    ├── views/
    │   └── notification_list_view.dart          — 알림 목록 화면
    └── widgets/
        ├── notification_item_widget.dart        — 알림 항목
        └── notification_badge_widget.dart       — 뱃지 위젯

lib/core/
├── services/
│   └── fcm_service.dart                         — FCM 초기화/토큰 관리/메시지 핸들링
└── navigation/
    └── deep_link_handler.dart                   — 알림 딥링크 처리
```

### 수정 필요
```
pubspec.yaml                  — firebase_messaging, firebase_core 패키지 추가
android/app/google-services.json  — Firebase 설정 파일 추가
ios/Runner/GoogleService-Info.plist — Firebase 설정 파일 추가
lib/main.dart                 — Firebase 초기화 추가
lib/core/navigation/          — 기존 라우터에 딥링크 핸들링 추가
lib/features/*/presentation/  — 하단 네비게이션에 뱃지 추가
```

---

## ✅ 작업 체크리스트

### 개발
- [x] Firebase 프로젝트 설정 (Android)
- [ ] ⏸️ Firebase 프로젝트 설정 (iOS) — Apple Developer 계정 확보 후
- [x] FCM 토큰 획득/등록/갱신 구현
- [x] 포어그라운드 알림 처리 (인앱 배너)
- [x] 백그라운드 알림 처리
- [x] 알림 탭 딥링크 이동
- [x] 알림 목록 화면 구현
- [x] 읽음 처리 (개별/전체)
- [x] 미읽음 뱃지 구현
- [x] 로그아웃 시 토큰 삭제

### 테스트 (Android)
- [ ] FCM 토큰 등록/갱신 테스트
- [ ] 포어그라운드/백그라운드 알림 수신 테스트
- [ ] 딥링크 이동 테스트
- [ ] 알림 목록 pagination 테스트
- [ ] 뱃지 카운트 테스트
- [ ] Android 수동 테스트 완료

### 테스트 (⏸️ iOS — Apple Developer 계정 확보 후)
- [ ] iOS 푸시 알림 수신 테스트 (실기기)
- [ ] iOS 백그라운드 알림 테스트
- [ ] iOS 알림 권한 요청 다이얼로그 테스트

### 코드 품질
- [x] Clean Architecture 구조 준수
- [x] 린팅 에러 없음
- [ ] 코딩 컨벤션 준수
- [ ] 자체 코드 리뷰 완료

---

## 🧪 테스트 시나리오

### 시나리오 1: FCM 토큰 등록 (앱 시작)
```
동작:
1. 앱 시작 (로그인 상태)
2. FCM 토큰 자동 획득

예상 결과:
- POST /api/notifications/token 호출
- 서버에 토큰 등록 성공
- 콘솔에 "FCM 토큰 등록 완료" 로그
```

### 시나리오 2: 포어그라운드 알림 수신 (인앱 배너)
```
전제: 앱이 포어그라운드에 있음

동작:
1. 다른 사용자가 채팅 메시지 전송
2. FCM 알림 수신

예상 결과:
- 화면 상단에 인앱 배너 표시 (제목 + 내용)
- 3초 후 자동 사라짐
- 배너 탭 시 채팅방으로 이동
- 현재 해당 채팅방에 있는 경우 배너 미표시
```

### 시나리오 3: 백그라운드 알림 → 딥링크 이동
```
전제: 앱이 백그라운드 또는 종료 상태

동작:
1. 결제 성공 알림 수신
2. 시스템 알림 트레이에서 알림 탭

예상 결과:
- 앱 실행/포어그라운드 전환
- 거래 상세 화면으로 자동 이동
```

### 시나리오 4: 알림 목록 무한 스크롤
```
전제: 알림 30건 존재

동작:
1. 알림 목록 화면 진입
2. 스크롤하여 하단 도달

예상 결과:
- 처음 20건 표시
- 하단 도달 시 로딩 인디케이터 표시
- 추가 10건 로드
- 미읽음 알림은 배경색 구분
```

### 시나리오 5: 알림 읽음 처리 + 뱃지 업데이트
```
전제: 미읽음 알림 5건, 뱃지에 5 표시

동작:
1. 알림 목록에서 알림 1건 탭

예상 결과:
- PUT /api/notifications/{id}/read 호출
- 해당 알림 배경색 변경 (읽음 처리)
- 뱃지 카운트 4로 감소
- 해당 화면으로 이동
```

### 시나리오 6: 로그아웃 시 토큰 삭제
```
동작:
1. 설정에서 로그아웃 실행

예상 결과:
- DELETE /api/notifications/token 호출
- 서버에서 토큰 삭제
- 로그아웃 후 푸시 알림 미수신
```

---

## 🔗 의존성

### 선행 작업
- [ ] Backend TASK-005: FCM 푸시 알림 API 구현 완료
- [ ] Firebase 프로젝트 생성 및 Android 설정 파일 확보 (google-services.json)

### 필요한 패키지
- `firebase_core`
- `firebase_messaging`

### 후속 작업
- [ ] TASK-006: 신고 시스템 (DISPUTE_OPENED/RESOLVED 딥링크 연동)
- [ ] Phase 2 후속: QR 인증, 정산, 평점 알림 화면 연동
- [ ] ⏸️ iOS 일괄 검증 Sprint: Apple Developer 계정 확보 후 iOS 푸시/빌드/배포 일괄 검증

---

## ⏱️ 예상 소요 시간

| 단계 | 시간 |
|------|------|
| Firebase 프로젝트 설정 (Android) | 1시간 |
| FCM 토큰 관리 구현 | 2시간 |
| 포어그라운드/백그라운드 알림 처리 | 3시간 |
| 딥링크 핸들러 구현 | 2시간 |
| 알림 목록 화면 (DTO/Repository/UseCase/View) | 4시간 |
| 읽음 처리 + 뱃지 구현 | 2시간 |
| 테스트 | 2시간 |
| 버그 수정 | 1시간 |
| **총 예상 시간 (Android)** | **17시간 (약 2.5일)** |
| ⏸️ iOS 검증 (계정 확보 후) | 3시간 |

---

## 📅 일정

- **시작일**: 2026-02-12
- **목표 완료일**: 2026-02-15
- **실제 완료일**: -

---

## 🚨 리스크 및 고려사항

### 기술적 리스크
- **Firebase 설정 파일 미확보**: Android google-services.json이 없으면 빌드 실패 → Firebase Console에서 즉시 발급
- **iOS 푸시 테스트 불가 (현재)**: Apple 유료 개발자 계정 미확보로 APNs 키 등록 불가 → Android에서 우선 검증, 계정 확보 후 iOS 일괄 검증
- **백그라운드 메시지 핸들러**: Flutter에서 top-level function이어야 함 → 설계 시 주의

### 블로커
- Firebase 프로젝트 생성 및 Android 설정 파일 확보
- ⏸️ APNs 인증 키 (iOS 푸시용) — Apple Developer 계정 확보 후

---

## ⚠️ 주의사항

### Firebase Messaging
- `FirebaseMessaging.onBackgroundMessage`는 top-level 함수 또는 static 메서드여야 함
- Android 13+ 에서도 알림 권한 요청 필요 (`POST_NOTIFICATIONS`)
- iOS는 사용자 알림 권한 요청 필요 (`requestPermission()`) — Apple Developer 계정 확보 후 테스트
- iOS 관련 코드(권한 요청 등)는 작성하되, 동작 검증은 계정 확보 후 진행

### Clean Architecture
- FCM 초기화 로직은 `lib/core/services/`에 배치
- 알림 데이터 매핑은 Data 레이어에서 처리
- ViewModel은 UseCase만 호출

### 성능
- 알림 목록은 cursor pagination으로 대량 데이터 대응
- 미읽음 카운트는 로컬 캐시 후 서버 동기화

---

## 📚 참고 자료

- Backend TASK-005: `docs/backend/tasks/TASK-005_FCM_Push_Notification.md`
- 기존 ChatService — SignalR 실시간 통신 패턴 참고
- 기존 TransactionHistory — cursor pagination 패턴 참고

---

**리뷰어**: Mobile Lead
**리뷰 완료일**: -
**상태**: 🚧 진행 중
