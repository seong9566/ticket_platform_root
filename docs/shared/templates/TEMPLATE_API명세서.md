# [METHOD] /api/[endpoint] - [기능명]

**작성일**: YYYY-MM-DD
**작성자**: Backend Team
**상태**: 🚧 초안 / ✅ 구현됨
**버전**: 1.0

---

## 📋 개요

### 엔드포인트
```
[METHOD] /api/[path]
```

### 설명
[이 API가 무엇을 하는지 간단히 설명]

### 인증
- [ ] 필요 없음 (Public)
- [ ] JWT 토큰 필요
- [ ] API Key 필요

---

## 📥 요청 (Request)

### HTTP 메서드
```
GET / POST / PUT / DELETE / PATCH
```

### URL
```
https://api.example.com/api/[endpoint]
```

### 경로 파라미터 (Path Parameters)
| 파라미터 | 타입 | 필수 | 설명 | 예시 |
|---------|------|------|------|------|
| id | integer | ✅ | 리소스 ID | 123 |

### 쿼리 파라미터 (Query Parameters)
| 파라미터 | 타입 | 필수 | 기본값 | 설명 | 예시 |
|---------|------|------|--------|------|------|
| page | integer | ❌ | 1 | 페이지 번호 | 1 |
| limit | integer | ❌ | 20 | 페이지 크기 | 20 |
| sort | string | ❌ | created_at | 정렬 기준 | name, created_at |

### 요청 헤더 (Request Headers)
```http
Authorization: Bearer {token}
Content-Type: application/json
```

### 요청 바디 (Request Body)
```json
{
  "field1": "value1",
  "field2": 123,
  "field3": {
    "nested": "value"
  },
  "field4": ["item1", "item2"]
}
```

#### 필드 설명
| 필드 | 타입 | 필수 | 설명 | 제약사항 |
|------|------|------|------|----------|
| field1 | string | ✅ | 설명 | 최대 255자 |
| field2 | integer | ✅ | 설명 | > 0 |
| field3 | object | ❌ | 설명 | - |
| field4 | array | ❌ | 설명 | 최대 10개 |

### 요청 예시 (cURL)
```bash
curl -X POST 'https://api.example.com/api/[endpoint]' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "field1": "value1",
    "field2": 123
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
  "message": "성공 메시지",
  "data": {
    "id": 123,
    "name": "example",
    "created_at": "2026-02-09T10:00:00Z"
  },
  "statusCode": 200
}
```

#### 필드 설명
| 필드 | 타입 | 설명 |
|------|------|------|
| message | string | 성공 메시지 |
| data | object | 응답 데이터 |
| data.id | integer | 리소스 ID |
| data.name | string | 리소스 이름 |
| data.created_at | string | 생성 시각 (ISO 8601) |
| statusCode | integer | HTTP 상태 코드 |

---

## ❌ 에러 응답

### 400 Bad Request
```json
{
  "error": {
    "code": "INVALID_PARAMETER",
    "message": "잘못된 파라미터입니다",
    "details": "field1은 필수입니다"
  },
  "statusCode": 400
}
```

**발생 조건**:
- 필수 파라미터 누락
- 잘못된 파라미터 타입
- 제약사항 위반

---

### 401 Unauthorized
```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "인증이 필요합니다"
  },
  "statusCode": 401
}
```

**발생 조건**:
- JWT 토큰 누락
- 만료된 토큰
- 잘못된 토큰

---

### 403 Forbidden
```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "권한이 없습니다"
  },
  "statusCode": 403
}
```

**발생 조건**:
- 리소스 접근 권한 없음
- 소유자가 아님

---

### 404 Not Found
```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "리소스를 찾을 수 없습니다"
  },
  "statusCode": 404
}
```

**발생 조건**:
- 존재하지 않는 ID
- 삭제된 리소스

---

### 500 Internal Server Error
```json
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "서버 오류가 발생했습니다"
  },
  "statusCode": 500
}
```

**발생 조건**:
- 데이터베이스 오류
- 예상치 못한 서버 오류

---

## 📊 에러 코드 목록

| 코드 | HTTP 상태 | 설명 |
|------|-----------|------|
| INVALID_PARAMETER | 400 | 잘못된 파라미터 |
| UNAUTHORIZED | 401 | 인증 필요 |
| FORBIDDEN | 403 | 권한 없음 |
| NOT_FOUND | 404 | 리소스 없음 |
| INTERNAL_ERROR | 500 | 서버 오류 |

---

## 🔐 보안

### 인증 방식
```
Bearer Token (JWT)
```

### 권한 확인
- [어떤 권한이 필요한지]
- 예: 리소스 소유자만 수정 가능

### Rate Limiting
- 최대 요청: 100회/분
- 초과 시: 429 Too Many Requests

---

## 📈 성능

### 응답 시간
- 목표: < 500ms (95 percentile)
- 최대: < 2000ms

### 데이터 크기
- 최대 응답 크기: 5MB
- 페이지네이션 권장: limit ≤ 50

---

## 🧪 테스트

### 테스트 케이스
1. **정상 요청**
   - 입력: [정상 데이터]
   - 예상 출력: 200 OK

2. **필수 파라미터 누락**
   - 입력: field1 누락
   - 예상 출력: 400 Bad Request

3. **권한 없음**
   - 입력: 다른 사용자의 리소스 요청
   - 예상 출력: 403 Forbidden

---

## 📝 예시 코드

### Flutter (Dio)
```dart
final dio = Dio();
final response = await dio.post(
  '/api/[endpoint]',
  data: {
    'field1': 'value1',
    'field2': 123,
  },
  options: Options(
    headers: {
      'Authorization': 'Bearer $token',
    },
  ),
);

if (response.statusCode == 200) {
  final data = response.data['data'];
  print(data['id']);
}
```

### C# (HttpClient)
```csharp
var client = new HttpClient();
client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", token);

var content = new StringContent(
    JsonSerializer.Serialize(new { field1 = "value1", field2 = 123 }),
    Encoding.UTF8,
    "application/json"
);

var response = await client.PostAsync("/api/[endpoint]", content);
var result = await response.Content.ReadFromJsonAsync<ApiResponse>();
```

---

## 🔗 관련 문서

- 기능 명세서: [링크]
- 데이터베이스 스키마: [링크]
- 모바일 연동 가이드: [링크]

---

## 📅 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|-----------|--------|
| 1.0 | YYYY-MM-DD | 초안 작성 | [이름] |

---

**검토**: [ ] Backend Lead / [ ] Mobile Lead
**승인 날짜**: YYYY-MM-DD
