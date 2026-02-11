# claude-code-system

nerdlab 조직에서 사용하는 Claude Code 리소스 모음.

새 프로젝트 시작 시 기술 스택에 맞는 스킬, 아키텍처 가이드, 템플릿을 가져다 바로 사용할 수 있도록 표준화한 저장소다.

## 왜 필요한가

nerdlab은 소/중규모 웹앱 서비스를 빠르게 만드는 팀이다.
정해진 개발 방식으로 많은 서비스를 찍어내는 게 핵심이기 때문에, 프로젝트마다 Claude Code 설정을 처음부터 만드는 건 비효율적이다.

기술 스택별로 검증된 아키텍처와 Claude Code 설정을 한 곳에 모아두고, 새 프로젝트에 필요한 것만 골라 적용한다.

## 구조

```
claude-code-system/
├── skills/            # Claude Code 슬래시 커맨드 스킬
├── guides/            # 역할별/기술별 가이드 (CLAUDE.md 구성 등)
├── agents/            # 에이전트 설정 및 워크플로우
└── templates/         # 프로젝트 초기 설정 템플릿
```

### Skills

반복되는 작업을 슬래시 커맨드로 자동화한다.

| 스킬 | 설명 | 사용법 |
|------|------|--------|
| [commit-message](skills/commit-message/) | Conventional Commits 형식 커밋 메시지 생성 | `/commit-message [힌트]` |
| [image-prompt](skills/image-prompt/) | 이미지 생성 프롬프트 작성 | `/image-prompt [설명]` |

### Guides

기술 스택별 아키텍처 가이드와 CLAUDE.md 구성 예시. *(준비 중)*

### Agents

특정 워크플로우를 자동화하는 에이전트 설정. *(준비 중)*

### Templates

새 프로젝트 시작 시 바로 복사해서 쓸 수 있는 초기 설정 템플릿. *(준비 중)*

## 사용 방법

### 스킬 추가

프로젝트의 `.claude/skills/` 디렉토리에 원하는 스킬을 복사한다.

```bash
# 예시: commit-message 스킬 추가
cp -r skills/commit-message/ <프로젝트>/.claude/skills/commit-message/
```

### 가이드 적용

가이드의 내용을 프로젝트 `CLAUDE.md`에 맞게 조합하여 사용한다.

## 기여

새 스킬이나 가이드를 추가할 때는 아래 구조를 따른다.

```
skills/<스킬명>/
├── SKILL.md    # 스킬 정의 (Claude Code가 읽는 파일)
└── README.md   # 사용법 문서 (사람이 읽는 파일)
```

## 라이선스

nerdlab 내부용
