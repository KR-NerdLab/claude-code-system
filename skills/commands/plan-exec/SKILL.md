---
name: plan-exec
description: gstack(기획/설계) + superpowers(구현) 하이브리드 전략에서 구현 단계 진입점. gstack 계획 문서를 자동 탐색하여 superpowers의 writing-plans → subagent-driven-development를 순차 호출한다. superpowers는 v5.0부터 명시적 호출 수단이 없으므로, 이 스킬이 /autoplan → /plan-exec → /review → /ship 흐름에서 gstack과 superpowers를 연결하는 브릿지 역할을 한다.
disable-model-invocation: true
model: sonnet
---

# plan-exec: gstack 계획 → superpowers 구현 실행

## 용도

gstack + superpowers 하이브리드 전략에서 사용하는 브릿지 스킬이다.

gstack은 기획/설계/리뷰/배포를 슬래시 명령어(`/autoplan`, `/review`, `/ship`)로 제어하고, superpowers는 TDD 기반 구현을 AI 자동 판단으로 관리한다. 하지만 superpowers는 v5.0부터 슬래시 명령어가 deprecated되어, gstack 기획이 끝난 후 구현으로 넘어가는 전환점에서 명시적 호출 수단이 없다.

`/plan-exec`는 이 전환점을 담당한다. gstack이 만든 계획 문서를 자동으로 찾아서, superpowers의 구현 파이프라인을 실행하고, 불필요한 스킬(brainstorming 등)은 차단한다.

```
/autoplan → /plan-exec → /review → /ship
 (기획)      (구현)       (검증)    (배포)
```

## 사용법

```
/plan-exec                              # 자동으로 최신 계획 문서 탐색
/plan-exec docs/gstack/plans/파일명.md  # 특정 문서 지정
```

## 실행 흐름

```
/plan-exec [문서경로]
    ↓
1. gstack 계획 문서 탐색
    ↓
2. superpowers:writing-plans → TDD 작업 목록 생성
    ↓
3. superpowers:subagent-driven-development → 자동 실행
   (태스크별: 구현 → 스펙 리뷰 → 코드 품질 리뷰)
    ↓
4. 전체 완료 후 안내
```

## 절차

### 0단계: 작업 브랜치 생성

구현을 시작하기 전에 반드시 main 브랜치에서 새 브랜치를 만든다.

1. `git checkout main && git pull` 으로 최신 상태 확인
2. 계획 문서 제목이나 핵심 키워드를 기반으로 브랜치명 생성 (예: `feat/타이머-설정-기능`)
3. `git checkout -b <브랜치명>` 으로 새 브랜치 생성

**main 브랜치에서 직접 구현하지 않는다.** 머지는 `/ship`이 담당한다.

### 1단계: gstack 계획 문서 탐색

사용자가 `$ARGUMENTS`로 문서 경로를 지정하면 해당 파일을 사용한다.

경로가 없으면 아래 순서로 자동 탐색:

1. `docs/gstack/plans/` 에서 가장 최신 파일
2. `docs/gstack/design/` 에서 가장 최신 파일
3. `~/.gstack/projects/` 전역 경로에서 탐색

어디에서도 찾지 못하면 사용자에게 경로를 물어본다.

문서를 찾으면 전체 내용을 읽고 다음 단계로 진행한다.

### 2단계: superpowers writing-plans 실행

찾은 gstack 계획 문서를 기반으로 `superpowers:writing-plans` 스킬을 호출한다.

**반드시 지킬 것:**
- `superpowers:brainstorming` 스킬은 호출하지 않는다. gstack에서 이미 기획/설계를 완료했다.
- 구현 계획은 `docs/superpowers/plans/` 에 저장한다.

### 3단계: subagent-driven-development 실행

writing-plans가 완료되면 바로 `superpowers:subagent-driven-development` 스킬을 호출하여 구현을 시작한다.

사용자에게 실행 여부를 묻지 않고 바로 진행한다. 태스크마다 스펙 리뷰와 코드 품질 리뷰가 자동으로 수행되므로 별도 확인이 불필요하다.

**실행 방식:**
- 태스크별 독립 서브에이전트로 구현
- 태스크 완료 후 스펙 리뷰 서브에이전트 → 코드 품질 리뷰 서브에이전트 자동 실행
- 리뷰에서 문제 발견 시 자동 수정 → 재리뷰

### 4단계: 완료 안내

모든 태스크가 완료되면 사용자에게 다음 단계를 안내한다:

```
구현이 완료되었습니다.

다음 단계:
- /review : 코드 리뷰 (diff 기반 구조적/보안 분석)
- /qa     : 시각적 QA (화면 변경이 있는 경우)
- /cso    : 보안 점검 (외부 노출면 변경이 있는 경우)
- /ship   : 배포
```

## 규칙

- **main 브랜치에서 직접 구현하지 않는다** — 반드시 0단계에서 새 브랜치를 만든 후 작업한다
- **커밋 메시지는 한국어로 작성한다** (예: `feat: 타이머 설정 기능 추가`, `test: useTimer 훅 단위 테스트`)
- `superpowers:brainstorming`은 절대 호출하지 않는다
- `superpowers:finishing-a-development-branch`는 호출하지 않는다 (배포는 gstack `/ship`이 담당)
- `superpowers:using-git-worktrees`는 호출하지 않는다 (브랜치는 0단계에서 직접 생성)
- gstack 슬래시 명령어(`/review`, `/qa`, `/ship` 등)는 이 스킬 내에서 실행하지 않는다
- 이 스킬은 구현(코딩)에만 집중한다
