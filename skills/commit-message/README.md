# /commit-message

스테이지된 변경사항을 분석하여 Conventional Commits 형식의 커밋 메시지를 생성하는 Claude Code Skill.

## 기본 사용법

```
/commit-message
```

스테이지된 diff를 자동 분석하여 커밋 메시지를 생성한다. 힌트 없이 호출하면 한국어로 작성된다.

```
/commit-message 로그인 관련 수정
/commit-message add login validation
```

힌트를 추가하면 메시지 작성에 반영된다. 영문 힌트를 주면 영문 메시지가 생성된다.

## 사전 조건

커밋할 변경사항을 먼저 스테이지해야 한다.

```bash
git add <파일>
```

스테이지된 변경이 없으면 안내 메시지가 출력된다.

## 커밋 메시지 형식

[Conventional Commits](https://www.conventionalcommits.org/) 규칙을 따른다.

```
<type>(<scope>): <subject>

<body>
```

### 지원하는 type

| type     | 용도                                      |
| -------- | ----------------------------------------- |
| feat     | 새로운 기능 추가                          |
| fix      | 버그 수정                                 |
| refactor | 기능 변경 없는 코드 구조 개선             |
| perf     | 성능 개선                                 |
| docs     | 문서 변경                                 |
| test     | 테스트 추가/수정                          |
| build    | 빌드 시스템, 의존성 변경                  |
| ci       | CI 설정 변경                              |
| chore    | 코드/기능에 영향 없는 잡무                |
| style    | 코드 포맷팅 (로직 변경 없음)              |

## 출력 예시

코드블록 하나로 출력되어 한 번에 복사 가능하다.

```
feat(auth): 로그인 폼 유효성 검사 추가
```

변경이 복잡한 경우 body가 포함된다.

```
refactor(api): 에러 핸들링 미들웨어로 분리

- 각 라우터의 중복 try-catch 로직을 공통 미들웨어로 추출
- 에러 응답 포맷 통일
```

## 워크플로우

1. `git add`로 커밋할 파일 스테이지
2. `/commit-message`로 메시지 생성
3. 출력된 메시지를 복사
4. `git commit -m "복사한 메시지"` 실행
