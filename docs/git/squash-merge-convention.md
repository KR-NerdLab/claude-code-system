# Git Squash Merge Convention

main 브랜치에 머지할 때 squash merge를 사용한다. feature 브랜치의 커밋들을 하나로 합쳐서 main 히스토리를 깔끔하게 유지한다.

## 메시지 형식

```
<type>[(<scope>)]: <subject> (#PR번호)

[<변경 내용 요약>]

[<footer>]
```

- **type**: 필수. 커밋 컨벤션과 동일
- **scope**: 선택. 변경 영역이 명확한 경우 사용
- **subject**: 필수. 기능 전체를 한 문장으로 요약 (한글, 명령형, 72자 이내)
- **변경 내용 요약**: bullet point로 주요 변경사항 나열
- **footer**: 선택. 이슈 참조 (예: `Refs: ISSUE-123`)

## Type

커밋 컨벤션과 동일: feat, fix, refactor, chore, docs, test, style, perf

## Scope (선택)

커밋 컨벤션과 동일. 변경 영역이 명확한 경우 모듈명 사용, 여러 모듈이면 생략.

## 예시

```
feat(auth): 소셜 로그인 구현 (#12)

- OAuth2 인증 플로우 구현
- 기존 인증 파이프라인과 통합
- 소셜 계정 연동 및 회원 매칭 처리

Refs: ISSUE-42
```

```
fix(api): 필수 파라미터 누락 시 500 에러 수정 (#15)

- 요청 유효성 검사 누락 수정
- 400 Bad Request로 적절한 에러 반환
```

```
refactor: 외부 API 어댑터 구조 개선 (#18)

- 어댑터별 클래스 분리
- 공통 인터페이스 통일
- 비동기 응답 지원 추가
```

## 작성 기준

- feature 브랜치의 개별 커밋을 나열하지 않는다
- "무엇을 왜 바꿨는지" 중심으로 요약한다
- main 히스토리만 보고도 변경 이력을 파악할 수 있어야 한다
