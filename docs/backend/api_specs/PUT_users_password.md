# PUT /api/users/password - 비밀번호 변경

**작성일**: 2026-02-10
**작성자**: Backend Team
**상태**: ✅ 구현됨
**버전**: 1.0

---

## 📋 개요

### 엔드포인트
```
PUT /api/users/password
```

### 설명
로그인된 사용자가 현재 비밀번호를 입력하여 본인 확인 후 새 비밀번호로 변경합니다. 소셜 로그인 사용자는 비밀번호 변경이 불가능합니다.

### 인증
- [ ] 필요 없음 (Public)
- [x] JWT 토큰 필요
- [ ] API Key 필요

---

## 📥 요청 (Request)

### HTTP 메서드
```
PUT
```

### URL
```
https://api.tickethub.com/api/users/password
```

### 경로 파라미터 (Path Parameters)
없음

### 쿼리 파라미터 (Query Parameters)
없음

### 요청 헤더 (Request Headers)
```http
Authorization: Bearer {JWT_TOKEN}
Content-Type: application/json
```

### 요청 바디 (Request Body)
```json
{
  "currentPassword": "OldPass123!",
  "newPassword": "NewPass456!"
}
```

#### 필드 설명
| 필드 | 타입 | 필수 | 설명 | 제약사항 |
|------|------|------|------|----------|
| currentPassword | string | ✅ | 현재 비밀번호 | 비어있지 않아야 함 |
| newPassword | string | ✅ | 새 비밀번호 | 8자 이상, 영문 대소문자/숫자/특수문자 중 3가지 이상 조합 |

### 요청 예시 (cURL)
```bash
curl -X PUT 'https://api.tickethub.com/api/users/password' \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...' \
  -H 'Content-Type: application/json' \
  -d '{
    "currentPassword": "OldPass123!",
    "newPassword": "NewPass456!"
  }'
```

---

## 📤 응답 (Response)

### 성공 응답 (200 OK)

#### 응답 헤더
```http
Content-Type: application/json
```

#### 응답 바디
```json
{
  "message": "비밀번호 변경 성공",
  "data": null,
  "statusCode": 200,
  "success": true
}
```

#### 필드 설명
| 필드 | 타입 | 설명 |
|------|------|------|
| message | string | 성공 메시지 |
| data | null | 응답 데이터 (비밀번호 변경은 별도 데이터 없음) |
| statusCode | integer | HTTP 상태 코드 (200) |
| success | boolean | 성공 여부 |

---

## ❌ 에러 응답

### 400 Bad Request - 약한 비밀번호 (길이 부족)
```json
{
  "message": "비밀번호는 8자 이상이어야 하며, 영문 대소문자, 숫자, 특수문자 중 3가지 이상을 조합해야 합니다",
  "data": null,
  "statusCode": 400,
  "success": false
}
```

**발생 조건**:
- newPassword가 8자 미만
- 영문 대소문자/숫자/특수문자 중 3가지 이상 조합을 만족하지 않음

---

### 400 Bad Request - 현재 비밀번호와 동일
```json
{
  "message": "새 비밀번호는 현재 비밀번호와 달라야 합니다",
  "data": null,
  "statusCode": 400,
  "success": false
}
```

**발생 조건**:
- newPassword가 currentPassword와 동일함

---

### 401 Unauthorized - 인증 실패
```json
{
  "message": "사용자 인증 정보가 유효하지 않습니다.",
  "data": null,
  "statusCode": 401,
  "success": false
}
```

**발생 조건**:
- JWT 토큰 누락
- 만료된 토큰
- 잘못된 토큰

---

### 401 Unauthorized - 현재 비밀번호 불일치
```json
{
  "message": "현재 비밀번호가 일치하지 않습니다",
  "data": null,
  "statusCode": 401,
  "success": false
}
```

**발생 조건**:
- currentPassword가 DB에 저장된 비밀번호와 일치하지 않음 (BCrypt 검증 실패)

---

### 403 Forbidden - 소셜 로그인 사용자
```json
{
  "message": "소셜 로그인 사용자는 비밀번호를 변경할 수 없습니다",
  "data": null,
  "statusCode": 403,
  "success": false
}
```

**발생 조건**:
- User.PasswordHash가 null인 경우 (Google, Kakao, Apple 로그인 사용자)

---

### 404 Not Found - 사용자 없음
```json
{
  "message": "사용자를 찾을 수 없습니다.",
  "data": null,
  "statusCode": 404,
  "success": false
}
```

**발생 조건**:
- JWT에서 추출한 userId에 해당하는 사용자가 DB에 없음

---

### 500 Internal Server Error
```json
{
  "message": "서버 내부 오류 발생",
  "data": null,
  "statusCode": 500,
  "success": false
}
```

**발생 조건**:
- 데이터베이스 오류
- BCrypt 해싱 오류
- 예상치 못한 서버 오류

---

## 📊 에러 코드 목록

현재 응답은 `code` 필드를 제공하지 않으며, `message`와 `statusCode`로 구분합니다.

---

## 🔐 보안

### 인증 방식
```
Bearer Token (JWT)
```

### 권한 확인
- JWT 토큰에서 추출한 userId로 본인만 비밀번호 변경 가능
- 다른 사용자의 비밀번호 변경 불가

### 비밀번호 정책
- **최소 길이**: 8자 이상
- **복잡도**: 영문 대소문자, 숫자, 특수문자 중 3가지 이상 조합
  - 영문 대문자: [A-Z]
  - 영문 소문자: [a-z]
  - 숫자: [0-9]
  - 특수문자: [^a-zA-Z0-9]

### 비밀번호 저장
- BCrypt 해싱 사용 (BCrypt.Net-Next 4.0.3)
- Salt 자동 생성
- 평문 비밀번호 저장 금지

### Rate Limiting
- 현재 미구현 (향후 추가 예정)
- 권장: 최대 5회/분 (브루트포스 방지)

---

## 📈 성능

### 응답 시간
- 목표: < 1000ms (95 percentile)
- 최대: < 3000ms
- BCrypt 해싱으로 인해 일반 API보다 느림

### 데이터 크기
- 최대 요청 크기: 1KB
- 최대 응답 크기: 500B

---

## 🧪 테스트

### 테스트 케이스

1. **정상 비밀번호 변경**
   - 입력: `{ "currentPassword": "OldPass123!", "newPassword": "NewPass456!" }`
   - 전제: currentPassword가 DB 값과 일치
   - 예상 출력: 200 OK

2. **잘못된 현재 비밀번호**
   - 입력: `{ "currentPassword": "WrongPass!", "newPassword": "NewPass456!" }`
   - 예상 출력: 401 Unauthorized (INVALID_CURRENT_PASSWORD)

3. **약한 새 비밀번호 (8자 미만)**
   - 입력: `{ "currentPassword": "OldPass123!", "newPassword": "Abc12!" }`
   - 예상 출력: 400 Bad Request

4. **약한 새 비밀번호 (조합 부족)**
   - 입력: `{ "currentPassword": "OldPass123!", "newPassword": "abcdefgh" }`
   - 예상 출력: 400 Bad Request

5. **현재 비밀번호와 동일**
   - 입력: `{ "currentPassword": "OldPass123!", "newPassword": "OldPass123!" }`
   - 예상 출력: 400 Bad Request

6. **소셜 로그인 사용자**
   - 전제: user.PasswordHash == null
   - 입력: `{ "currentPassword": "any", "newPassword": "NewPass456!" }`
   - 예상 출력: 403 Forbidden

7. **인증되지 않은 요청**
   - 전제: Authorization 헤더 없음
   - 예상 출력: 401 Unauthorized

---

## 📝 예시 코드

### Flutter (Dio)
```dart
import 'package:dio/dio.dart';

final dio = Dio();

Future<void> changePassword(String token, String currentPassword, String newPassword) async {
  try {
    final response = await dio.put(
      '/api/users/password',
      data: {
        'currentPassword': currentPassword,
        'newPassword': newPassword,
      },
      options: Options(
        headers: {
          'Authorization': 'Bearer $token',
        },
      ),
    );

    if (response.statusCode == 200) {
      print('비밀번호 변경 성공: ${response.data['message']}');
    }
  } on DioException catch (e) {
    if (e.response != null) {
      final statusCode = e.response!.statusCode;
      final errorMessage = e.response!.data['message'] ?? '오류가 발생했습니다.';
      
      print('에러 발생 ($statusCode): $errorMessage');
      
      if (statusCode == 401) {
        print('현재 비밀번호가 틀렸거나 인증이 만료되었습니다.');
      } else if (statusCode == 400) {
        print('비밀번호 정책을 만족하지 않습니다.');
      } else if (statusCode == 403) {
        print('소셜 로그인 사용자는 비밀번호를 변경할 수 없습니다.');
      }
    }
  }
}
```

### C# (HttpClient)
```csharp
using System.Net.Http;
using System.Net.Http.Json;
using System.Text.Json;

var client = new HttpClient();
client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", token);

var requestBody = new
{
    currentPassword = "OldPass123!",
    newPassword = "NewPass456!"
};

var content = new StringContent(
    JsonSerializer.Serialize(requestBody),
    Encoding.UTF8,
    "application/json"
);

try
{
    var response = await client.PutAsync("/api/users/password", content);
    
    if (response.IsSuccessStatusCode)
    {
        var result = await response.Content.ReadFromJsonAsync<ApiResponse>();
        Console.WriteLine(result.Message);
    }
    else
    {
        var error = await response.Content.ReadFromJsonAsync<ErrorResponse>();
        Console.WriteLine($"에러: {error.Error.Code} - {error.Error.Message}");
    }
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"네트워크 오류: {ex.Message}");
}
```

---

## 🔗 관련 문서

- 기능 명세서: `docs/backend/tasks/TASK-004_Password_Change.md`
- 사용자 모델 스키마: `TicketPlatFormServer/DBModel/User.cs`
- 인증 가이드: `docs/backend/authentication.md` (미작성)

---

## 📅 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|-----------|--------|
| 1.0 | 2026-02-10 | 초안 작성 및 구현 완료 | Backend Agent (Sisyphus) |

---

**검토**: [ ] Backend Lead / [ ] Mobile Lead
**승인 날짜**: 2026-02-10
