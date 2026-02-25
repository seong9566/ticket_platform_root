# Auth API - /api/auth

**작성일**: 2026-02-10
**작성자**: Backend Team
**상태**: 🚧 초안
**버전**: 1.0

---

## 📋 개요

### 공통 응답 형식
```json
{
  "message": "메시지",
  "data": {},
  "statusCode": 200,
  "success": true
}
```

### 인증
- [ ] 필요 없음 (Public) — 회원가입/로그인/토큰갱신
- [x] JWT 토큰 필요 — 로그아웃

---

## 1) 회원가입

### 엔드포인트
```
POST /api/auth/sign
```

### 요청 바디
```json
{
  "email": "user@example.com",
  "password": "Pass123!",
  "phone": "01012345678",
  "role": "user",
  "provider": "email"
}
```

#### 필드 설명
| 필드 | 타입 | 필수 | 설명 | 제약사항 |
|------|------|------|------|----------|
| email | string | ✅ | 이메일 | Email 형식 |
| password | string | ✅ | 비밀번호 | 최소 8자 권장 |
| phone | string | ❌ | 전화번호 | - |
| role | string | ❌ | 역할 코드 | 기본값 `user` |
| provider | string | ✅ | 가입 유형 | `email` |

### 성공 응답 (200 OK)
```json
{
  "message": "회원가입 성공",
  "data": {
    "email": "user@example.com",
    "phone": "01012345678",
    "role": "user",
    "provider": "email"
  },
  "statusCode": 200,
  "success": true
}
```

---

## 2) 로그인

### 엔드포인트
```
POST /api/auth/login
```

### 요청 바디
```json
{
  "email": "user@example.com",
  "password": "Pass123!"
}
```

### 성공 응답 (200 OK)
```json
{
  "message": "로그인 성공",
  "data": {
    "id": 1,
    "email": "user@example.com",
    "phone": "01012345678",
    "role": "user",
    "provider": "email",
    "lastLoginAt": "2026-02-10T10:00:00Z",
    "accessToken": "<JWT>",
    "refreshToken": "<UUID>",
    "expiresIn": 604800,
    "tokenType": "Bearer",
    "expiresAt": "2026-02-17T10:00:00Z"
  },
  "statusCode": 200,
  "success": true
}
```

---

## 3) 토큰 갱신

### 엔드포인트
```
POST /api/auth/refresh
```

### 요청 바디
```json
{
  "refreshToken": "<UUID>"
}
```

### 성공 응답 (200 OK)
```json
{
  "message": "Token 갱신 성공",
  "data": {
    "accessToken": "<JWT>",
    "refreshToken": "<UUID>",
    "expiresIn": 604800,
    "tokenType": "Bearer",
    "expiresAt": "2026-02-17T10:00:00Z"
  },
  "statusCode": 200,
  "success": true
}
```

---

## 4) 로그아웃

### 엔드포인트
```
POST /api/auth/logout
```

### 요청 헤더
```http
Authorization: Bearer {JWT_TOKEN}
Content-Type: application/json
```

### 요청 바디
```json
{
  "refreshToken": "<UUID>"
}
```

### 성공 응답 (200 OK)
```json
{
  "message": "로그아웃 성공",
  "data": null,
  "statusCode": 200,
  "success": true
}
```

---

## 5) 소셜 로그인

### 엔드포인트
```
POST /api/auth/social/login
```

### 요청 바디
```json
{
  "provider": "google",
  "accessToken": "<PROVIDER_ACCESS_TOKEN>"
}
```

#### 필드 설명
| 필드 | 타입 | 필수 | 설명 | 제약사항 |
|------|------|------|------|----------|
| provider | string | ✅ | 소셜 제공자 | `google` 또는 `kakao` |
| accessToken | string | ✅ | 제공자 Access Token | 빈 값 불가 |

### 성공 응답 (200 OK)
```json
{
  "message": "로그인 성공",
  "data": {
    "userId": 123,
    "accessToken": "<JWT>",
    "refreshToken": "<UUID>",
    "isNewUser": false
  },
  "statusCode": 200,
  "success": true
}
```

### 동작 규칙
- 기존 사용자: `email` 기준으로 사용자 조회 후 로그인 처리
- 신규 사용자: 자동 회원가입 후 로그인 처리
- Kakao에서 이메일 제공이 없거나 미검증인 경우: 내부 식별 이메일(`kakao_<hash>@social.local`)로 계정 생성

### 주요 에러
- 400: 지원하지 않는 provider
- 401: 유효하지 않은 Access Token
- 500: 소셜 로그인 처리 중 내부 오류

---

## ❌ 공통 에러 응답
```json
{
  "message": "오류 메시지",
  "data": null,
  "statusCode": 400,
  "success": false
}
```

---

## 🔗 관련 문서
- `docs/backend/tasks/TASK-003_API_Route_Consistency.md`
