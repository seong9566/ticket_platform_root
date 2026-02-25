# TicketHub 작업 관리 가이드

## 📋 목차
- [작업 흐름](#작업-흐름)
- [이슈 관리](#이슈-관리)
- [브랜치 전략](#브랜치-전략)
- [커밋 규칙](#커밋-규칙)
- [코드 리뷰](#코드-리뷰)
- [작업 우선순위](#작업-우선순위)

---

## 작업 흐름

### 1. 이슈 생성
GitHub Issues에서 작업을 이슈로 생성합니다.
- Bug Report: 버그 신고
- Feature Request: 새 기능 제안
- Task: 개발/리팩토링/문서화 작업

### 2. 이슈 할당 및 우선순위 설정
- 담당자 할당 (Assignee)
- 라벨 추가 (feature, bug, enhancement 등)
- 우선순위 라벨 추가 (priority:high, priority:medium, priority:low)
- 마일스톤 설정 (Phase 1, Phase 2 등)

### 3. 브랜치 생성
```bash
# 기능 개발
git checkout -b feature/issue-123-add-payment-system

# 버그 수정
git checkout -b bugfix/issue-456-fix-chat-crash

# 긴급 수정
git checkout -b hotfix/issue-789-fix-security-issue
```

### 4. 개발 및 커밋
- 커밋 메시지 규칙 준수 (Conventional Commits)
- 작은 단위로 자주 커밋
- 테스트 작성 (가능한 경우)

### 5. Pull Request 생성
- PR 템플릿 작성
- 관련 이슈 링크 (Closes #이슈번호)
- 스크린샷 첨부 (UI 변경 시)
- 리뷰어 지정

### 6. 코드 리뷰
- 최소 1명의 승인 필요
- 리뷰 코멘트 반영
- CI/CD 테스트 통과 확인

### 7. 머지 및 배포
- Squash and merge (커밋 히스토리 정리)
- 브랜치 삭제
- 이슈 자동 종료

---

## 이슈 관리

### 이슈 라벨 시스템

#### 유형 (Type)
- `feature`: 새 기능 추가
- `bug`: 버그 수정
- `enhancement`: 기능 개선
- `refactor`: 코드 리팩토링
- `docs`: 문서 작업
- `test`: 테스트 추가/수정
- `chore`: 빌드, 설정 변경

#### 플랫폼 (Platform)
- `mobile`: 모바일 앱 작업
- `backend`: 백엔드 서버 작업
- `database`: 데이터베이스 관련
- `devops`: CI/CD, 인프라

#### 우선순위 (Priority)
- `priority:critical`: 서비스 중단, 즉시 처리 필요
- `priority:high`: 주요 기능 동작 불가, 빠른 처리 필요
- `priority:medium`: 일부 기능 동작 불가, 일정 내 처리
- `priority:low`: UI 오류, 마이너한 버그, 추후 처리 가능

#### 상태 (Status)
- `status:todo`: 작업 대기 중
- `status:in-progress`: 작업 진행 중
- `status:blocked`: 블로킹 이슈 있음
- `status:review`: 코드 리뷰 중
- `status:done`: 완료

#### 기타
- `good first issue`: 초보자에게 적합한 이슈
- `help wanted`: 도움 필요
- `duplicate`: 중복 이슈
- `wontfix`: 수정하지 않음
- `question`: 질문

### 이슈 작성 예시

#### Bug Report 예시
```markdown
## 버그 설명
채팅방에서 이미지를 전송하면 앱이 크래시됩니다.

## 재현 단계
1. 채팅방 진입
2. 이미지 첨부 버튼 클릭
3. 갤러리에서 이미지 선택
4. 앱 크래시

## 예상 동작
이미지가 채팅방에 전송되어야 합니다.

## 환경
- 플랫폼: Mobile (iOS)
- 버전: 1.0.0
- 기기: iPhone 15 Pro
- OS: iOS 17.2

## 우선순위
- [x] High (주요 기능 동작 불가)
```

#### Feature Request 예시
```markdown
## 기능 설명
찜 목록에서 티켓을 제거할 수 있는 기능

## 문제 또는 필요성
현재 찜 목록에 추가는 가능하지만 제거할 방법이 없습니다.

## 제안하는 솔루션
찜 목록 페이지에서 스와이프 또는 휴지통 아이콘으로 삭제

## 우선순위
- [x] Medium (추후 추가 가능)
```

---

## 브랜치 전략

### 메인 브랜치
- `main`: 프로덕션 배포 코드 (항상 배포 가능 상태)
- `develop`: 개발 브랜치 (다음 릴리즈 준비)

### 작업 브랜치
- `feature/*`: 새 기능 개발
  - 예: `feature/issue-123-add-payment-system`
  - `develop`에서 분기, `develop`으로 머지

- `bugfix/*`: 버그 수정
  - 예: `bugfix/issue-456-fix-chat-crash`
  - `develop`에서 분기, `develop`으로 머지

- `hotfix/*`: 긴급 수정
  - 예: `hotfix/issue-789-fix-security-issue`
  - `main`에서 분기, `main`과 `develop` 모두 머지

- `release/*`: 릴리즈 준비
  - 예: `release/v1.0.0`
  - `develop`에서 분기, `main`으로 머지 후 태그

### 브랜치 네이밍 규칙
```
<type>/<issue-number>-<short-description>

예시:
- feature/123-add-payment-system
- bugfix/456-fix-chat-crash
- hotfix/789-fix-security-issue
```

### 브랜치 생성 예시
```bash
# develop 브랜치 최신화
git checkout develop
git pull origin develop

# 기능 개발 브랜치 생성
git checkout -b feature/123-add-payment-system

# 작업 후 커밋
git add .
git commit -m "feat(payment): add payment request API"

# 원격 브랜치에 푸시
git push origin feature/123-add-payment-system

# GitHub에서 PR 생성
```

---

## 커밋 규칙

### Conventional Commits
```
<type>(<scope>): <subject>

<body>

<footer>
```

#### Type
- `feat`: 새 기능 추가
- `fix`: 버그 수정
- `docs`: 문서 변경
- `style`: 코드 포맷팅 (기능 변경 없음)
- `refactor`: 코드 리팩토링
- `test`: 테스트 추가/수정
- `chore`: 빌드, 설정 변경
- `perf`: 성능 개선

#### Scope (선택 사항)
- `auth`: 인증
- `chat`: 채팅
- `payment`: 결제
- `ticket`: 티켓
- `profile`: 프로필
- `mobile`: 모바일
- `backend`: 백엔드

#### Subject
- 명령형 동사로 시작 (add, fix, update, remove)
- 첫 글자는 소문자
- 마침표 없음
- 50자 이내

#### Body (선택 사항)
- 무엇을, 왜 변경했는지 설명
- 72자마다 줄바꿈

#### Footer (선택 사항)
- Breaking Changes 명시: `BREAKING CHANGE: ...`
- 이슈 참조: `Closes #123`, `Refs #456`

### 커밋 메시지 예시

#### 기본 예시
```bash
# 좋은 예
git commit -m "feat(chat): add image upload functionality"
git commit -m "fix(payment): resolve null reference exception"
git commit -m "docs(readme): update installation guide"

# 나쁜 예
git commit -m "update"
git commit -m "fix bug"
git commit -m "WIP"
```

#### Body 포함 예시
```bash
git commit -m "feat(auth): add JWT refresh token

Add refresh token endpoint to extend user sessions without
requiring re-login. Token expires after 7 days.

Closes #123"
```

#### Breaking Change 예시
```bash
git commit -m "refactor(api): change user endpoint response format

BREAKING CHANGE: User endpoint now returns nested object
instead of flat structure. Update mobile app accordingly."
```

---

## 코드 리뷰

### 리뷰 요청 전 체크리스트
- [ ] 자체 코드 리뷰 완료
- [ ] 테스트 작성 및 통과
- [ ] 린팅 에러 없음
- [ ] 빌드 성공
- [ ] PR 템플릿 작성 완료
- [ ] 스크린샷 첨부 (UI 변경 시)
- [ ] 문서 업데이트 (필요 시)

### 리뷰어 가이드

#### 리뷰 초점
1. **기능 정확성**: 요구사항을 충족하는가?
2. **코드 품질**: 가독성, 유지보수성
3. **성능**: 불필요한 반복, 비효율적인 쿼리
4. **보안**: SQL Injection, XSS, 인증/인가 누락
5. **테스트**: 충분한 테스트 커버리지
6. **아키텍처**: 프로젝트 구조 준수

#### 리뷰 코멘트 작성
- **명확하게**: 무엇을, 왜 수정해야 하는지 설명
- **친절하게**: 비판보다는 제안으로 표현
- **구체적으로**: 예시 코드 제공

#### 리뷰 코멘트 예시
```markdown
# 좋은 예
💡 Suggestion: `getUserById`를 `findUserById`로 변경하는 것이
Repository 네이밍 컨벤션과 일치합니다.

🐛 Bug: Line 45에서 null 체크가 누락되었습니다.
`user?.name ?? 'Unknown'`처럼 null-safe 연산자 사용을 추천합니다.

✨ Nice: 에러 처리가 깔끔하네요!

# 나쁜 예
이거 안 됩니다. 고치세요.
코드 스타일이 별로네요.
```

### 리뷰 승인 기준
- [ ] 모든 리뷰 코멘트 반영 또는 논의 완료
- [ ] CI/CD 테스트 통과
- [ ] 코딩 스타일 준수
- [ ] 보안 이슈 없음
- [ ] 성능 문제 없음

---

## 작업 우선순위

### Priority Matrix

| 우선순위 | 긴급도 | 중요도 | 예시 |
|---------|-------|-------|------|
| Critical | 높음 | 높음 | 서비스 중단, 보안 취약점 |
| High | 높음 | 중간 | 주요 기능 버그, MVP 기능 |
| Medium | 중간 | 중간 | 일부 기능 개선, UI 버그 |
| Low | 낮음 | 낮음 | 마이너 버그, 문서 업데이트 |

### 우선순위 결정 요소
1. **비즈니스 영향**: 매출, 사용자 경험에 미치는 영향
2. **기술적 의존성**: 다른 기능의 선행 조건 여부
3. **개발 시간**: ROI (투입 대비 효과)
4. **리스크**: 지연 시 발생할 문제

### 작업 순서 예시 (Phase 1)
1. 🔴 **Critical/High**: 티켓 상세/구매 플로우 (MVP 필수)
2. 🔴 **High**: 티켓 판매 등록 (MVP 필수)
3. 🔴 **High**: 결제 시스템 (MVP 필수)
4. 🟡 **Medium**: 프로필 업데이트 (개선)
5. 🟡 **Medium**: 찜 기능 (개선)
6. 🟢 **Low**: UI 디자인 개선 (추후)

---

## 팁

### 효율적인 작업 관리
1. **작은 단위로 분할**: 큰 작업은 여러 이슈로 나누기
2. **의존성 명시**: 블로킹 이슈 명확히 표시
3. **정기적인 동기화**: 스탠드업 미팅에서 진행 상황 공유
4. **문서화**: 중요한 결정사항은 이슈에 코멘트로 기록
5. **자동화**: CI/CD로 테스트 및 배포 자동화

### 일일 루틴
1. **오전**: GitHub Issues 확인, 우선순위 재점검
2. **개발**: 집중 시간 확보 (알림 끄기)
3. **오후**: PR 리뷰, 블로킹 이슈 해결
4. **퇴근 전**: 진행 상황 업데이트, 내일 계획

---

**문서 버전**: 1.0
**작성일**: 2026-02-09
**최종 수정일**: 2026-02-09
