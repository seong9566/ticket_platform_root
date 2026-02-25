# 모바일 개발 문서

이 폴더는 모바일(Flutter) 개발팀을 위한 문서들이 저장됩니다.

---

## 📂 하위 폴더

### 🎯 features/ - 기능별 명세
기능별 상세 구현 명세서

**포함 내용**:
- 기능 개요
- 화면 플로우
- 상태 관리 설계
- API 연동 계획
- 위젯 트리 구조
- 에러 처리

**파일명 규칙**:
```
FEATURE_[기능명].md

예시:
- FEATURE_티켓상세페이지.md
- FEATURE_결제플로우.md
- FEATURE_채팅시스템.md
```

---

### 🎨 ui_specs/ - UI/UX 명세
화면 디자인 및 인터랙션 명세

**포함 내용**:
- 화면 레이아웃
- 위젯 컴포넌트
- 색상/폰트/간격
- 사용자 인터랙션
- 애니메이션
- 로딩/에러 상태 UI

**파일명 예시**:
- `UI_티켓상세화면.md`
- `UI_결제화면.md`
- `COMPONENT_티켓카드.md`

---

### 📖 guides/ - 개발 가이드
Flutter 개발 가이드라인

**포함 내용**:
- Flutter 코딩 컨벤션
- Riverpod 사용 가이드
- Clean Architecture 패턴
- 테스트 작성 가이드
- 빌드 및 배포

**파일명 예시**:
- `Flutter코딩컨벤션.md`
- `Riverpod가이드.md`
- `테스트가이드.md`

---

### ✅ tasks/ - 작업 명세
모바일 작업 상세 명세서

**포함 내용**:
- 작업 개요
- 구현 계획
- 위젯 설계
- API 연동 계획
- 테스트 계획
- 예상 소요 시간

**파일명 규칙**:
```
TASK-[번호]_[작업명].md

예시:
- TASK-101_QR스캐너구현.md
- TASK-102_프로필이미지업로드.md
```

---

## 🔗 관련 문서

- [ticket_platform_mobile/claude.md](../../ticket_platform_mobile/claude.md) - 모바일 개발 가이드
- [ticket_platform_mobile/docs/](../../ticket_platform_mobile/docs/) - 기존 모바일 문서

---

## 📝 문서 작성 시 참고

1. **화면 플로우**: Mermaid 다이어그램 사용 권장
2. **위젯 트리**: 들여쓰기로 구조 표현
3. **상태 관리**: Riverpod Provider 타입 명시
4. **API 연동**: 백엔드 API 명세 링크 포함
5. **UI 스펙**: AppColors, AppTextStyles, AppSpacing 사용 명시
