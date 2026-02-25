# TASK-002: 소셜 로그인 구현 (Google, Kakao)

**작성일**: 2026-02-10
**작성자**: PM
**담당**: Mobile Agent
**상태**: ✅ 완료 (Google/Kakao 구현 및 테스트 완료, 2026-02-11)
**우선순위**: 🔴 High

---

## 📋 Agent 작업 지시

### 목표
Google과 Kakao OAuth SDK를 연동하여 소셜 로그인 기능을 구현하라. 사용자가 버튼을 클릭하면 각 플랫폼의 로그인을 수행하고, Access Token을 백엔드로 전송하여 JWT 토큰을 받아 저장해야 함.

### 구현 위치
- Feature: `lib/features/auth/`
- 화면: 기존 `login_view.dart` 수정 (버튼 로직 연결)

---

## 🎯 완료 기준 (Acceptance Criteria)

다음 체크리스트를 모두 완료해야 함:

### Google 로그인
- [x] `google_sign_in` 패키지 추가
- [x] GoogleOAuthDataSource 구현
- [x] Google Access Token 획득
- [x] 백엔드 API 호출 (`POST /api/auth/social/login`)
- [x] JWT 토큰 Secure Storage 저장
- [x] 로그인 성공 시 홈 화면 이동
- [x] 에러 처리 (로그인 취소, 네트워크 오류)

### Kakao 로그인
- [x] `kakao_flutter_sdk` 패키지 추가
- [x] KakaoOAuthDataSource 구현
- [x] Kakao Access Token 획득
- [x] 백엔드 API 호출
- [x] JWT 토큰 저장
- [x] 로그인 성공 시 홈 화면 이동
- [x] 에러 처리

### 플랫폼 설정
- [x] Android 설정 (AndroidManifest.xml, build.gradle)
- [x] iOS 설정 (Info.plist)
- [x] Clean Architecture 구조 준수

---

## 🔧 API 명세

### 엔드포인트
```
POST /api/auth/social/login
```

### 요청
```json
{
  "provider": "google" | "kakao",
  "accessToken": "string"
}
```

### 성공 응답 (200 OK)
```json
{
  "message": "로그인 성공",
  "data": {
    "userId": 123,
    "accessToken": "jwt_access_token",
    "refreshToken": "jwt_refresh_token",
    "isNewUser": false
  },
  "statusCode": 200
}
```

### 에러 응답
- 400: `INVALID_PROVIDER`
- 401: `INVALID_ACCESS_TOKEN`
- 500: `OAUTH_API_ERROR`

---

## 📂 파일 구조

### 신규 생성
```
lib/features/auth/
├── data/
│   ├── datasources/
│   │   ├── google_oauth_datasource.dart
│   │   └── kakao_oauth_datasource.dart
│   └── dto/
│       └── social_login_req_dto.dart
├── domain/
│   ├── repositories/
│   │   └── auth_repository.dart (수정: socialLogin 메서드 추가)
│   └── usecases/
│       ├── google_sign_in_usecase.dart
│       └── kakao_sign_in_usecase.dart
└── presentation/
    ├── viewmodels/
    │   └── login_viewmodel.dart (수정: 소셜 로그인 메서드 추가)
    └── views/
        └── login_view.dart (수정: 버튼 로직 연결)
```

### 수정 필요
```
pubspec.yaml (패키지 추가)
android/app/build.gradle
android/app/src/main/AndroidManifest.xml
ios/Runner/Info.plist
```

---

## 🔨 구현 지시사항

### Step 1: 패키지 추가

**파일**: `pubspec.yaml`

추가할 패키지:
```yaml
dependencies:
  google_sign_in: ^6.2.1
  kakao_flutter_sdk: ^1.9.5
```

설치 명령:
```bash
flutter pub get
```

### Step 2: DTO 생성

**파일**: `lib/features/auth/data/dto/social_login_req_dto.dart`

요구사항:
- Freezed 사용
- `provider` 필드 (String: "google" | "kakao")
- `accessToken` 필드 (String)
- `toMap()` 메서드 구현

코드 생성:
```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

### Step 3: Google OAuth DataSource

**파일**: `lib/features/auth/data/datasources/google_oauth_datasource.dart`

구현 로직:
1. GoogleSignIn 인스턴스 생성:
   ```dart
   final GoogleSignIn _googleSignIn = GoogleSignIn(
     scopes: ['email', 'profile'],
   );
   ```
2. `signIn()` 메서드 구현:
   - `_googleSignIn.signIn()` 호출
   - `account.authentication` 획득
   - `auth.accessToken` 반환
   - 취소 시 null 반환
   - 에러 시 Exception throw
3. `signOut()` 메서드 구현 (선택)

### Step 4: Kakao OAuth DataSource

**파일**: `lib/features/auth/data/datasources/kakao_oauth_datasource.dart`

구현 로직:
1. Kakao SDK 초기화 (앱 시작 시 - `main.dart`):
   ```dart
   KakaoSdk.init(nativeAppKey: 'YOUR_NATIVE_APP_KEY');
   ```
2. `signIn()` 메서드 구현:
   - KakaoTalk 설치 여부 확인:
     ```dart
     if (await isKakaoTalkInstalled()) {
       token = await UserApi.instance.loginWithKakaoTalk();
     } else {
       token = await UserApi.instance.loginWithKakaoAccount();
     }
     ```
   - `token.accessToken` 반환
   - 취소 시 null 반환
   - 에러 시 Exception throw

### Step 5: Repository 수정

**파일**: `lib/features/auth/domain/repositories/auth_repository.dart` (인터페이스)

추가할 메서드:
```dart
Future<Either<Failure, AuthEntity>> socialLogin(
  String provider,
  String accessToken,
);
```

**파일**: `lib/features/auth/data/repositories/auth_repository_impl.dart` (구현)

구현 로직:
1. `SocialLoginReqDto` 생성
2. `_remoteDataSource.socialLogin(dto)` 호출
3. 성공 시 `AuthEntity` 반환 (userId, accessToken, refreshToken 포함)
4. 실패 시 `Failure` 반환

**파일**: `lib/features/auth/data/datasources/auth_remote_datasource.dart`

추가할 메서드:
```dart
Future<AuthRespDto> socialLogin(SocialLoginReqDto dto) async {
  return safeApiCall(() async {
    final response = await _dio.post(
      ApiEndpoints.socialLogin,  // '/api/auth/social/login'
      data: dto.toMap(),
    );
    return AuthRespDto.fromJson(response.data['data']);
  });
}
```

### Step 6: UseCase 생성

**파일**: `lib/features/auth/domain/usecases/google_sign_in_usecase.dart`

요구사항:
- `@riverpod` 사용
- `execute()` 메서드:
  1. `GoogleOAuthDataSource().signIn()` 호출
  2. accessToken이 null이면 `Left(Failure('로그인 취소'))` 반환
  3. `authRepository.socialLogin('google', accessToken)` 호출
  4. 결과 반환

**파일**: `lib/features/auth/domain/usecases/kakao_sign_in_usecase.dart`

Google과 동일하게 구현 (provider만 'kakao'로)

### Step 7: ViewModel 수정

**파일**: `lib/features/auth/presentation/viewmodels/login_viewmodel.dart`

추가할 메서드:
```dart
Future<void> signInWithGoogle() async {
  state = const AsyncLoading();

  final result = await ref.read(googleSignInUsecaseProvider.notifier).execute();

  result.fold(
    (failure) {
      state = AsyncError(failure, StackTrace.current);
      // 에러 메시지 표시 (SnackBar 등)
    },
    (auth) {
      // JWT 토큰 저장
      ref.read(authTokenStorageProvider).saveTokens(
        accessToken: auth.accessToken,
        refreshToken: auth.refreshToken,
      );
      state = AsyncData(auth);
      // 홈 화면으로 이동
    },
  );
}

Future<void> signInWithKakao() async {
  // Google과 동일 로직, kakaoSignInUsecaseProvider 사용
}
```

### Step 8: LoginView 수정

**파일**: `lib/features/auth/presentation/views/login_view.dart`

Google 버튼:
```dart
ElevatedButton(
  onPressed: () {
    ref.read(loginViewmodelProvider.notifier).signInWithGoogle();
  },
  child: Row(
    children: [
      Image.asset('assets/google_logo.png', height: 24),
      SizedBox(width: 8),
      Text('Google로 로그인'),
    ],
  ),
)
```

Kakao 버튼:
```dart
ElevatedButton(
  onPressed: () {
    ref.read(loginViewmodelProvider.notifier).signInWithKakao();
  },
  style: ElevatedButton.styleFrom(
    backgroundColor: Color(0xFFFEE500),  // Kakao yellow
  ),
  child: Row(
    children: [
      Image.asset('assets/kakao_logo.png', height: 24),
      SizedBox(width: 8),
      Text('카카오로 로그인', style: TextStyle(color: Colors.black)),
    ],
  ),
)
```

### Step 9: Android 설정

**파일**: `android/app/build.gradle`

추가:
```gradle
android {
    defaultConfig {
        ...
        minSdkVersion 21  // Google Sign In 요구사항
    }
}
```

**파일**: `android/app/src/main/AndroidManifest.xml`

Google (이미 설정되어 있을 수 있음):
```xml
<meta-data
    android:name="com.google.android.gms.version"
    android:value="@integer/google_play_services_version" />
```

Kakao:
```xml
<meta-data
    android:name="com.kakao.sdk.AppKey"
    android:value="YOUR_KAKAO_NATIVE_APP_KEY" />

<activity
    android:name="com.kakao.sdk.auth.AuthCodeHandlerActivity"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="kakaoYOUR_KAKAO_NATIVE_APP_KEY" />
    </intent-filter>
</activity>
```

### Step 10: iOS 설정

**파일**: `ios/Runner/Info.plist`

Google:
```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>com.googleusercontent.apps.YOUR_IOS_CLIENT_ID</string>
        </array>
    </dict>
</array>
```

Kakao:
```xml
<key>KAKAO_APP_KEY</key>
<string>YOUR_KAKAO_NATIVE_APP_KEY</string>

<key>LSApplicationQueriesSchemes</key>
<array>
    <string>kakaokompassauth</string>
    <string>kakaolink</string>
</array>

<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>kakaoYOUR_KAKAO_NATIVE_APP_KEY</string>
        </array>
    </dict>
</array>
```

---

## 🧪 테스트 시나리오

Agent는 다음 시나리오를 수동으로 테스트해야 함:

### 시나리오 1: Google 로그인 성공
```
동작:
1. Google 버튼 클릭
2. Google 계정 선택
3. 권한 동의

예상 결과:
- 로딩 표시
- JWT 토큰 저장
- 홈 화면으로 이동
```

### 시나리오 2: Google 로그인 취소
```
동작:
1. Google 버튼 클릭
2. 뒤로 가기 (취소)

예상 결과:
- 에러 없음
- 로그인 화면 유지
```

### 시나리오 3: Kakao 로그인 (KakaoTalk 설치됨)
```
동작:
1. Kakao 버튼 클릭
2. KakaoTalk 앱으로 이동
3. 동의하고 계속하기

예상 결과:
- 앱으로 복귀
- 로그인 성공
- 홈 화면 이동
```

### 시나리오 4: Kakao 로그인 (KakaoTalk 미설치)
```
동작:
1. Kakao 버튼 클릭
2. 웹 브라우저로 로그인

예상 결과:
- 브라우저 로그인
- 앱 복귀
- 홈 화면 이동
```

### 시나리오 5: 네트워크 오류
```
전제: 오프라인 상태

동작:
1. Google 또는 Kakao 로그인 시도

예상 결과:
- SnackBar: "네트워크 연결을 확인해주세요"
```

### 시나리오 6: 백엔드 에러 (401)
```
전제: 백엔드에서 401 반환

예상 결과:
- SnackBar: "유효하지 않은 Access Token입니다"
```

---

## 🔗 의존성

### 선행 작업
- [ ] 백엔드: TASK-002 OAuth API 구현 완료

### 필요한 정보
- Google: OAuth Client ID (iOS/Android)
- Kakao: Native App Key

### 후속 작업
- [ ] Apple Sign In 구현 (Phase 2, 별도 Task)

---

## ⏱️ 예상 소요 시간

| 단계 | 시간 |
|------|------|
| 패키지 설치 및 설정 | 1시간 |
| DTO 생성 | 30분 |
| DataSource 구현 | 2시간 |
| Repository/UseCase 구현 | 2시간 |
| ViewModel 수정 | 1시간 |
| UI 연결 | 1시간 |
| Android/iOS 설정 | 2시간 |
| 테스트 | 3시간 |
| 버그 수정 | 2시간 |
| **총 예상 시간** | **14.5시간 (약 2일)** |

---

## ⚠️ 주의사항

### Google Sign In
- iOS Client ID와 Android Client ID가 다름
- Firebase Console에서 발급 필요
- SHA-1 지문 등록 필요 (Android)

### Kakao SDK
- Native App Key 필요 (Kakao Developers에서 발급)
- `KakaoSdk.init()`은 `main()` 함수에서 호출
- KakaoTalk 설치 여부에 따라 플로우 다름

### Clean Architecture
- Data → Domain → Presentation 의존성 방향 엄수
- ViewModel은 UseCase만 호출
- UI는 ViewModel만 참조

### 에러 처리
- 사용자 취소는 에러가 아님 (조용히 처리)
- 네트워크 오류는 재시도 안내
- 백엔드 에러는 사용자 친화적 메시지

---

## ✅ 완료 확인

구현 완료 후 다음을 확인:

- [x] 코드가 컴파일됨
- [x] `flutter pub run build_runner build` 실행됨
- [ ] 모든 테스트 시나리오 통과
- [x] Google 로그인 (Android/iOS) 실기기 검증
- [x] Kakao 로그인 (Android/iOS) 실기기 검증
- [x] Clean Architecture 구조 준수
- [x] 에러 메시지가 사용자 친화적임
- [x] JWT 토큰 저장 확인
- [x] 홈 화면 이동 확인

---

**작성자**: PM
**검토자**: Mobile Lead
**상태**: ✅ 완료 (Google/Kakao 구현 및 테스트 완료, 2026-02-11)
**비고**: Apple Sign-In은 Apple Developer 계정 확보 후 추후 진행
