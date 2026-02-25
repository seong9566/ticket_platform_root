# TASK-[번호] [작업명]

**작성일**: YYYY-MM-DD
**작성자**: PM
**담당 팀**: Backend / Mobile / Both
**담당자**: [이름]
**상태**: ⏳ 대기 / 🚧 진행중 / ✅ 완료
**우선순위**: 🔴 High / 🟡 Medium / 🟢 Low

---

## 📋 작업 개요

### 작업 설명
[이 작업이 무엇을 하는지 간단히 설명]

### 목표
[이 작업을 통해 달성하고자 하는 목표]

### 배경
[왜 이 작업이 필요한지, 어떤 문제를 해결하는지]

---

## 🎯 완료 기준

작업이 완료되었다고 판단할 수 있는 기준:

- [ ] 기준 1: [구체적이고 측정 가능한 기준]
- [ ] 기준 2: [예: 단위 테스트 작성 및 통과]
- [ ] 기준 3: [예: 코드 리뷰 승인]
- [ ] 기준 4: [예: 문서 업데이트]

---

## 🔧 기술 스펙 (Backend)

### API 엔드포인트
- **URL**: `[METHOD] /api/[path]`
- **설명**: [엔드포인트 설명]
- **명세 문서**: [링크]

### 데이터베이스 변경
- [ ] 테이블 추가/수정
- [ ] 마이그레이션 스크립트 작성
- [ ] 인덱스 추가

```sql
-- 변경 사항 SQL
```

### 비즈니스 로직
```csharp
// 주요 로직 의사 코드
public async Task<Result> DoSomethingAsync()
{
    // 1. 유효성 검증
    // 2. 데이터 처리
    // 3. 결과 반환
}
```

### 외부 연동
- [ ] 외부 API: [API명]
- [ ] 라이브러리: [라이브러리명]

---

## 📱 기술 스펙 (Mobile)

### 화면 변경
- [ ] 새 화면 추가: [화면명]
- [ ] 기존 화면 수정: [화면명]

### 위젯 구조
```
FeatureView
├── FeatureAppBar
├── FeatureBody
│   ├── FeatureHeader
│   ├── FeatureContent
│   └── FeatureFooter
└── FeatureBottomAction
```

### 상태 관리
```dart
@riverpod
class FeatureViewModel extends _$FeatureViewModel {
  @override
  FutureOr<FeatureState> build() async {
    // 초기화 로직
  }

  Future<void> doAction() async {
    // 액션 처리
  }
}
```

### API 연동
- **엔드포인트**: `[METHOD] /api/[path]`
- **DTO**: [DTO 클래스명]
- **Entity**: [Entity 클래스명]

---

## 📂 파일 구조

### Backend
```
TicketPlatFormServer/
├── Controllers/
│   └── FeatureController.cs
├── Services/
│   └── Feature/
│       ├── IFeatureService.cs
│       └── FeatureService.cs
├── Repository/
│   └── Feature/
│       ├── IFeatureRepository.cs
│       └── FeatureRepository.cs
└── DTO/
    └── Feature/
        ├── FeatureReqDto.cs
        └── FeatureRespDto.cs
```

### Mobile
```
lib/features/feature/
├── data/
│   ├── datasources/
│   ├── dto/
│   └── repositories/
├── domain/
│   ├── entities/
│   ├── repositories/
│   └── usecases/
└── presentation/
    ├── viewmodels/
    ├── views/
    └── widgets/
```

---

## ✅ 작업 체크리스트

### 개발
- [ ] 기능 구현
- [ ] 에러 처리
- [ ] 로깅 추가
- [ ] 주석 작성

### 테스트
- [ ] 단위 테스트 작성
- [ ] 통합 테스트 작성 (필요 시)
- [ ] 수동 테스트 완료
- [ ] 엣지 케이스 확인

### 문서화
- [ ] API 명세서 업데이트 (Backend)
- [ ] 코드 주석 추가
- [ ] README 업데이트 (필요 시)

### 코드 품질
- [ ] 린팅 에러 없음
- [ ] 코딩 컨벤션 준수
- [ ] 자체 코드 리뷰 완료
- [ ] PR 생성

---

## 🧪 테스트 계획

### 테스트 시나리오
1. **정상 케이스**
   - 입력: [정상 데이터]
   - 예상 결과: [성공 응답]

2. **에러 케이스 1**
   - 입력: [잘못된 데이터]
   - 예상 결과: [에러 메시지]

3. **에지 케이스**
   - 입력: [경계값]
   - 예상 결과: [정상 처리]

### 테스트 데이터
```json
{
  "test_case_1": {
    "input": {},
    "expected": {}
  }
}
```

---

## 🔗 의존성

### 선행 작업
- [ ] TASK-XXX: [작업명]
- [ ] TASK-YYY: [작업명]

### 관련 이슈
- GitHub Issue: #123
- 기능 명세서: [링크]

### 후속 작업
- [ ] TASK-ZZZ: [작업명]

---

## ⏱️ 예상 소요 시간

| 항목 | 시간 |
|------|------|
| 설계 | [X시간] |
| 구현 | [X시간] |
| 테스트 | [X시간] |
| 리뷰 및 수정 | [X시간] |
| **총 예상 시간** | **[X시간]** |

---

## 📅 일정

- **시작일**: YYYY-MM-DD
- **목표 완료일**: YYYY-MM-DD
- **실제 완료일**: YYYY-MM-DD

---

## 🚨 리스크 및 고려사항

### 기술적 리스크
- [리스크 1]: [설명 및 대응 방안]
- [리스크 2]: [설명 및 대응 방안]

### 블로커
- [블로커 1]: [해결 방법]

---

## 📝 구현 노트

### 주요 결정 사항
- [결정 1]: [이유]
- [결정 2]: [이유]

### 트러블슈팅
- [문제 1]: [해결 방법]
- [문제 2]: [해결 방법]

---

## 🔍 코드 리뷰 체크포인트

### 리뷰어 확인 사항
- [ ] 비즈니스 로직 정확성
- [ ] 에러 처리 적절성
- [ ] 코드 가독성
- [ ] 테스트 커버리지
- [ ] 성능 고려
- [ ] 보안 고려

---

## 📚 참고 자료

- 기능 명세서: [링크]
- API 명세서: [링크]
- 관련 문서: [링크]

---

## 💬 진행 상황 업데이트

| 날짜 | 진행률 | 내용 |
|------|--------|------|
| YYYY-MM-DD | 0% | 작업 시작 |
| YYYY-MM-DD | 50% | 구현 완료 |
| YYYY-MM-DD | 100% | 테스트 및 리뷰 완료 |

---

**리뷰어**: [이름]
**리뷰 완료일**: YYYY-MM-DD
**상태**: ✅ 승인 / 🚧 수정 필요
