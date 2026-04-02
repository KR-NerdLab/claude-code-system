# gstack + superpowers 하이브리드 개발 가이드

> gstack으로 기획/설계/검증/배포를 주도하고, superpowers로 코딩을 맡기는 하이브리드 전략

## 목차

1. [소개](#1-소개)
2. [각 도구의 역할 비교](#2-각-도구의-역할-비교)
3. [plan-exec: 구현 단계 진입점](#3-plan-exec-구현-단계-진입점)
4. [설치 및 설정](#4-설치-및-설정)
5. [권장 프로젝트 구성](#5-권장-프로젝트-구성)
6. [세션 운영 방식](#6-세션-운영-방식)
7. [상황별 워크플로우](#7-상황별-워크플로우)
8. [검증 원칙](#8-검증-원칙)
9. [충돌 영역과 해결법](#9-충돌-영역과-해결법)
10. [CLAUDE.md 템플릿](#10-claudemd-템플릿)
11. [트러블슈팅 / FAQ](#11-트러블슈팅--faq)

---

## 1. 소개

### 하이브리드 전략

gstack과 superpowers는 개발 사이클의 서로 다른 영역을 담당한다.

- **gstack**: 기획, 설계, 리뷰, QA, 보안, 배포 — 코딩을 제외한 전 단계
- **superpowers**: TDD 기반 코딩, 디버깅, 코드 검증 — 코딩 단계

> "Superpowers owns the implementation loop, GStack owns everything before and after it."
> — [Particula Tech](https://particula.tech/blog/superpowers-vs-gstack-ai-coding-skill-packs)

gstack만 쓰면 코딩 품질 관리가 약하고, superpowers만 쓰면 제품 기획과 QA가 빠진다. 둘을 병행하면 아이디어부터 배포까지 전체 사이클을 커버할 수 있다.

### 핵심 원칙

1. **모든 단계를 명시적 명령어로 전환한다** — gstack 명령어(`/autoplan`, `/ship` 등)와 `/plan-exec`로 단계를 진행
2. **구현 구간에서는 superpowers가 코딩 품질을 자동 관리한다** — `/plan-exec`가 superpowers를 호출하면, TDD, 리뷰, 검증이 자동 진행
3. **세션 간 연결고리는 문서다** — 컨텍스트 윈도우 관리를 위해 단계별 세션 클리어, 문서가 다음 세션의 입력이 된다

---

## 2. 각 도구의 역할 비교

### 트리거 방식

| | gstack | superpowers |
|---|---|---|
| **호출 방식** | 슬래시 명령어 (`/ship`, `/qa` 등) | AI 자동 판단 |
| **사용자 제어** | 완전한 제어 (명시적 호출) | 간접 제어 (구현 시작 시 자동 발동) |
| **비활성화** | `proactive false` 설정 가능 | 비활성화 불가 (항상 자동) |
| **설계 철학** | 사용자가 단계를 주도 | AI가 코딩 규율을 강제 |

### 담당 영역

| 개발 단계 | 담당 | 주요 스킬/명령어 |
|----------|------|----------------|
| 아이디어 검증 | gstack | `/office-hours` |
| 제품 방향 리뷰 | gstack | `/plan-ceo-review` |
| 디자인 리뷰 | gstack | `/plan-design-review` |
| 아키텍처 리뷰 | gstack | `/plan-eng-review` |
| 자동 플랜 (위 3개 통합) | gstack | `/autoplan` |
| **구현 실행 (브릿지)** | **plan-exec** | **`/plan-exec`** |
| 구현 계획 (작업 목록) | superpowers | `writing-plans` (plan-exec가 호출) |
| TDD 기반 코딩 | superpowers | `test-driven-development` (자동) |
| 디버깅 | superpowers | `systematic-debugging` (자동) |
| 커밋 전 검증 | superpowers | `verification-before-completion` (자동) |
| 코드 리뷰 | gstack | `/review` |
| 시각적 QA | gstack | `/qa` |
| 보안 점검 | gstack | `/cso` |
| 배포 | gstack | `/ship` |

---

## 3. plan-exec: 구현 단계 진입점

### 용도

superpowers는 AI가 상황을 판단하여 자동으로 스킬을 호출하는 구조다. v5.0부터 슬래시 명령어가 deprecated되어, 사용자가 superpowers를 명시적으로 실행할 방법이 없다.

`/plan-exec`는 이 문제를 해결하는 커스텀 스킬이다. superpowers의 코딩 관련 기능을 수동 커맨드로 실행한다. gstack 기획이 끝난 후 이 명령어 하나로 구현 단계에 진입할 수 있다.

```
/autoplan → /plan-exec → /review → /ship
 (기획)      (구현)       (검증)    (배포)
```

### 사용법

```
/plan-exec                              # 최신 계획 문서를 자동 탐색
/plan-exec docs/gstack/plans/파일명.md  # 특정 문서 지정
```

### 실행 시 동작하는 superpowers 스킬

`/plan-exec`는 아래 superpowers 스킬을 순차적으로 호출한다.

**1단계: `writing-plans`**

gstack 계획 문서를 읽고 TDD 기반 구현 계획(작업 목록)을 생성한다. 각 태스크는 "테스트 작성 → 실패 확인 → 구현 → 통과 확인 → 커밋" 사이클로 구성되며, 정확한 파일 경로와 실행 코드가 포함된다. 생성된 계획은 `docs/superpowers/plans/`에 저장된다.

**2단계: `subagent-driven-development`**

생성된 작업 목록을 태스크 단위로 실행한다. 태스크마다 3개의 독립 서브에이전트가 투입된다:

| 서브에이전트 | 역할 |
|------------|------|
| 구현 에이전트 | TDD로 코딩, 테스트 작성, 커밋 |
| 스펙 리뷰 에이전트 | 계획대로 구현했는지 확인 |
| 코드 품질 리뷰 에이전트 | 코드 품질, 구조적 문제 확인 |

리뷰에서 문제가 발견되면 구현 에이전트가 자동으로 수정하고 재리뷰를 받는다. 모든 태스크 완료 후 전체 코드에 대한 최종 리뷰가 진행된다.

각 서브에이전트는 독립된 컨텍스트에서 작동하므로 컨텍스트 오염 없이 작업이 진행된다.

### 차단하는 superpowers 스킬

| 스킬 | 차단 이유 |
|------|----------|
| `brainstorming` | gstack에서 기획/설계를 이미 완료했으므로 불필요 |
| `finishing-a-development-branch` | 배포는 gstack `/ship`이 담당 |

---

## 4. 설치 및 설정

### 4.1 gstack 설치

```bash
curl -fsSL https://raw.githubusercontent.com/garrytan/gstack/main/install.sh | bash
```

### 4.2 superpowers 설치

Claude Code 내에서:
```
/install-plugin superpowers
```

### 4.3 plan-exec 스킬 설치

gstack + superpowers 하이브리드 전략의 구현 단계 진입점이다. superpowers는 명시적 호출 수단이 없으므로, `/plan-exec`가 유일한 명시적 구현 시작 명령어 역할을 한다.

```bash
cp -r <claude-code-system>/skills/commands/plan-exec/ .claude/skills/plan-exec/
```

**스킬 설정:**
- `model: sonnet` — Sonnet 모델로 동작 (별칭 사용 시 항상 최신 버전 자동 적용)
- `disable-model-invocation: true` — AI 자동 호출 차단, `/plan-exec` 명령어로만 실행 가능

### 4.4 gstack proactive 비활성화

gstack의 자동 추천 기능을 끄고, 슬래시 명령어 전용으로 사용한다. superpowers의 자동 트리거와 충돌을 방지한다.

```bash
~/.claude/skills/gstack/bin/gstack-config set proactive false
```

### 4.5 문서 저장 경로 통일

**문제**: gstack은 `~/.gstack/projects/{SLUG}/`(전역)에, superpowers는 `docs/superpowers/plans/`(프로젝트 내부)에 문서를 저장한다. 세션 클리어 후 연결이 끊길 수 있다.

**해결**: CLAUDE.md에서 gstack 문서를 프로젝트 내부에 저장하도록 규칙을 추가한다. (섹션 9의 CLAUDE.md 템플릿 참조)

> **테스트 결과 (2026-04-02):** 세 가지 방법을 테스트했다:
>
> | 방법 | 프로젝트 내부 저장 | 신뢰도 |
> |------|:----------------:|:------:|
> | 프로젝트 레벨 설치 (`--local`) | X | - |
> | `GSTACK_HOME` 환경변수 | 부분적 (30곳 중 4곳) | 낮음 |
> | CLAUDE.md 규칙 | O | 중간 |
>
> gstack 스킬 코드는 `~/.gstack/`을 하드코딩하고 있어, 프로젝트 레벨 설치나 환경변수로는 해결되지 않는다. CLAUDE.md에 "프로젝트 내부에 저장하라"는 규칙을 추가하면 AI가 이를 따르는 것을 확인했다. 100% 보장은 아니지만 실용적으로 작동한다. 전역 경로에 저장된 경우 수동 복사로 대응 가능. (섹션 10 FAQ 참조)

### 4.6 설정 확인

```bash
# gstack 설치 확인
ls ~/.claude/skills/gstack/

# superpowers 설치 확인
ls ~/.claude/plugins/cache/claude-plugins-official/superpowers/

# gstack proactive 상태 확인
~/.claude/skills/gstack/bin/gstack-config get proactive
```

---

## 5. 권장 프로젝트 구성

### 폴더 구조

```
project-root/
├── CLAUDE.md                          # 매 세션 로드 (간결하게)
├── .claude/
│   └── skills/
│       └── plan-exec/                 # 하이브리드 전략 브릿지 스킬
├── docs/
│   ├── gstack/                        # gstack 산출물
│   │   ├── design/                    # /office-hours 디자인 문서
│   │   └── plans/                     # /autoplan 계획 문서
│   ├── superpowers/                   # superpowers 산출물
│   │   └── plans/                     # writing-plans 구현 계획
│   └── rules/                         # CLAUDE.md 참조 규칙 문서
│       ├── coding-rules.md            # 필수: 아키텍처 + 컨벤션 + 금지사항
│       └── (프로젝트 유형별 추가)
```

### CLAUDE.md와 rules/ 의 관계

CLAUDE.md는 매 세션 시스템 프롬프트로 로드되므로 토큰 비용이 발생한다. 매 세션 필요한 정보(Role, Project, Tech Stack, Commands, Gotchas, Hybrid Strategy)만 CLAUDE.md에 유지하고, 코딩 시에만 필요한 상세 규칙은 `docs/rules/`에 분리하여 참조 링크로 연결한다.

```markdown
# CLAUDE.md에서의 참조 방식
코딩 시 반드시 참조: @docs/rules/coding-rules.md
```

### rules/ 문서 구성

#### 필수 (모든 프로젝트)

| 문서 | 내용 |
|------|------|
| `coding-rules.md` | 아키텍처 구조 + 코드 컨벤션 + 금지사항을 하나로 통합 |

코딩할 때 매번 확인해야 하는 내용이므로 하나의 파일로 통합한다. 3개 파일로 분리하면 매번 3개를 열어야 하는 비효율이 생긴다.

#### 백엔드 프로젝트 (선택)

| 문서 | 내용 |
|------|------|
| `api-rules.md` | API 응답 형식, 에러 코드 체계, 페이지네이션, 인증 규칙 |
| `database-rules.md` | 스키마 네이밍 규칙, 마이그레이션 절차, 인덱싱 기준 |
| `testing-rules.md` | 단위/통합 테스트 전략, 모킹 규칙, 커버리지 기준 |

#### 프론트엔드 프로젝트 (선택)

| 문서 | 내용 |
|------|------|
| `component-rules.md` | 컴포넌트 구조, 상태관리 전략, 스타일링 규칙 |
| `testing-rules.md` | 컴포넌트 테스트, E2E 전략, 접근성 테스트 |

#### 풀스택 프로젝트

백엔드 + 프론트엔드에서 해당하는 것을 선택하여 추가한다.

> **참고:** 가이드에서는 구조와 목록만 제시한다. 각 문서의 실제 내용은 프로젝트의 기술 스택과 요구사항에 따라 작성한다.

---

## 6. 세션 운영 방식

### 단계별 세션 분리

컨텍스트 윈도우는 유한하다. 기획부터 배포까지 한 세션에서 진행하면 컨텍스트가 넘치거나 품질이 떨어진다. 단계별로 세션을 클리어하고, 문서로 다음 세션에 연결한다.

### 세션 간 연결 흐름

```
[세션 1] /office-hours
    ↓ 문서 생성
    ↓ 세션 클리어
[세션 2] /autoplan (문서 참조)
    ↓ 계획 문서 생성
    ↓ 세션 클리어
[세션 3] /plan-exec
    ↓ gstack 계획 문서 자동 탐색
    ↓ writing-plans → subagent-driven-development (TDD 구현)
    ↓ 세션 클리어
[세션 4] /review → /qa → /ship
```

### 문서 저장 위치

| 단계 | 생성 문서 | 저장 위치 | 파일명 패턴 |
|------|----------|----------|------------|
| office-hours | 디자인 문서 | `docs/gstack/design/` | `{user}-{branch}-design-{datetime}.md` |
| autoplan | 계획 문서 | `docs/gstack/plans/` | `{user}-{branch}-plan-{datetime}.md` |
| plan-ceo-review | CEO 리뷰 | `docs/gstack/plans/` | `{date}-{feature-slug}.md` |
| plan-eng-review | 엔지니어링 리뷰 | `docs/gstack/plans/` | 계획 문서에 리뷰 섹션 추가 |
| writing-plans | 구현 작업 목록 | `docs/superpowers/plans/` | `YYYY-MM-DD-{feature-name}.md` |

> **참고:** gstack의 기본 저장 경로는 `~/.gstack/projects/{SLUG}/`(전역)이다. CLAUDE.md 규칙으로 프로젝트 내부 저장을 유도하지만, AI 판단에 의존하므로 간혹 전역 경로에 저장될 수 있다. 이 경우 프로젝트 내부로 복사한다.

### 구현 시작하기

세션 클리어 후, 새 세션에서 `/plan-exec`로 구현을 시작한다:

```
# 최신 계획 문서를 자동 탐색하여 구현 시작
/plan-exec

# 특정 문서를 지정할 수도 있음
/plan-exec docs/gstack/plans/tiredman-main-plan-20260402-141600.md
```

`/plan-exec`는 `docs/gstack/plans/` → `docs/gstack/design/` → `~/.gstack/projects/` 순서로 계획 문서를 자동 탐색한다.

---

## 7. 상황별 워크플로우

모든 작업이 풀 코스를 거칠 필요는 없다. 작업 유형에 따라 진입점이 달라진다.

### 상황 판단

```
새 아이디어인가? ─── Yes ──→ 상황 1: 풀 코스
       │
       No
       │
제품 방향은 정해져 있나? ─── No ──→ 상황 1: 풀 코스
       │
       Yes
       │
화면/UX가 바뀌나? ─── Yes ──→ 상황 2: 디자인 개선
       │
       No
       │
아키텍처/기술이 바뀌나? ─── Yes ──→ 상황 3: 기술 변경
       │
       No
       │
버그/오류인가? ─── Yes ──→ 상황 4: 버그 수정
       │
       No ──→ 상황 5: 간단한 수정
```

---

### 상황 1: 새 기능 / 제품 개발

아이디어 단계부터 시작하는 풀 코스.

**기획 (gstack)**
```
[세션 1] /office-hours
  → 아이디어 검증, 6가지 강제 질문
  → 디자인 문서 생성
  → 세션 클리어

[세션 2] /autoplan
  → CEO → Design → Eng 리뷰 자동 진행
  → 계획 문서 생성
  → 세션 클리어
```

**구현 (/plan-exec)**
```
[세션 3] /plan-exec
  → gstack 계획 문서 자동 탐색
  → writing-plans (TDD 작업 목록 생성)
  → subagent-driven-development (태스크별: 구현 → 스펙 리뷰 → 코드 리뷰)
  → (규모에 따라 여러 세션으로 분할 가능)
  → 세션 클리어
```

**검증 & 배포 (gstack)**
```
[세션 4] /review → /qa → /cso (해당시) → /ship
```

---

### 상황 2: 디자인 개선

제품 방향은 정해져 있고, UI/UX만 변경하는 경우.

**기획 (gstack)**
```
[세션 1] /plan-design-review → /plan-eng-review
  → 디자인 + 기술 리뷰만 진행
  → 문서 생성 → 세션 클리어
```

**구현 (/plan-exec)**
```
[세션 2] /plan-exec
  → 계획 문서 자동 탐색 → writing-plans → TDD 구현
  → 세션 클리어
```

**검증 (gstack)**
```
[세션 3] /review → /qa → /ship
  → 화면이 바뀌므로 /qa 필수
```

---

### 상황 3: 기술 스택 / 아키텍처 변경 / 리팩토링

내부 기술 변경. 디자인 리뷰 불필요.

**기획 (gstack)**
```
[세션 1] /plan-eng-review
  → 아키텍처, 기술 판단만 진행
  → 문서 생성 → 세션 클리어
```

**구현 (/plan-exec)**
```
[세션 2] /plan-exec
  → 계획 문서 자동 탐색 → writing-plans → TDD 구현
  → 세션 클리어
```

**검증 (gstack)**
```
[세션 3] /review → /cso (외부 노출면 변경 시) → /ship
  → 화면 안 바뀌면 /qa 불필요
```

---

### 상황 4: 버그 / 오류 수정

gstack 기획 단계 불필요. superpowers만 사용.

```
[세션 1] 버그 설명
  → superpowers systematic-debugging 자동 발동
  → 근본 원인 조사 → TDD로 수정 → verification
  → (규모에 따라 /review 추가)
```

---

### 상황 5: 간단한 수정

오타, 설정 변경, 사소한 수정. 스킬 불필요.

```
바로 수정 → 커밋
```

---

## 8. 검증 원칙

### 4가지 검증 레이어

| 레이어 | 도구 | 검증 대상 |
|--------|------|----------|
| 단위 테스트 | superpowers TDD | 함수/모듈이 스펙대로 동작하는가 |
| 코드 레벨 검증 | superpowers `verification` | 테스트 통과, 빌드 성공 확인 |
| 코드 리뷰 | gstack `/review` | diff에 보안/구조적 문제 없는가 |
| 시각적 QA | gstack `/qa` | 사용자 화면이 정상인가 |

### 기본 원칙

- **코드가 바뀌면** → `verification` (항상)
- **diff가 크면** → `/review` (추가)
- **화면이 바뀌면** → `/qa` (추가)
- **외부 노출면이 바뀌면** → `/cso` (추가)

### 상황별 적용

| 상황 | verification | /review | /qa | /cso |
|------|:-----------:|:-------:|:---:|:----:|
| 새 기능/제품 | O | O | O | 해당시 |
| 디자인 개선 | O | O | O | - |
| 기술 스택 변경 | O | O | - | 해당시 |
| 버그 수정 | O | 규모에 따라 | UI 버그시 | - |
| 간단한 수정 | - | - | - | - |

### /cso 실행 기준

매번 실행하는 것은 비효율적이다. 외부 노출면이 바뀔 때만 실행한다:

- 새 API 엔드포인트 추가
- 인증/권한 로직 변경
- 새 의존성 패키지 추가
- 환경 변수/시크릿 변경
- 배포 파이프라인 수정

---

## 9. 충돌 영역과 해결법

gstack과 superpowers에는 5개의 기능 중복 영역이 있다.

### 8.1 디버깅: `/investigate` vs `systematic-debugging`

| | gstack `/investigate` | superpowers `systematic-debugging` |
|---|---|---|
| 방식 | 명시적 호출 | 자동 발동 |
| 접근법 | 4단계 근본 원인 분석 | 4단계 근본 원인 분석 (동일) |

**규칙**: superpowers가 자동 처리. `/investigate`는 사용하지 않는다.

### 8.2 브레인스토밍: `/office-hours` vs `brainstorming`

| | gstack `/office-hours` | superpowers `brainstorming` |
|---|---|---|
| 용도 | 아이디어 검증, 시장 적합성 | 구현 전 설계 탐색 |
| 시점 | 프로젝트 시작 단계 | 코딩 시작 직전 |

**규칙**: 기획 단계에서 `/office-hours` 사용. superpowers `brainstorming`은 gstack 계획 문서가 이미 대체하므로 구현 진입 시 불필요.

### 8.3 코드 리뷰: `/review` vs `requesting-code-review`

| | gstack `/review` | superpowers `requesting-code-review` |
|---|---|---|
| 방식 | diff 기반 구조적/보안 분석 | 서브에이전트 독립 리뷰 |
| 시점 | 배포 전 최종 리뷰 | 구현 중 태스크 완료 시 |

**규칙**: 보완적. superpowers가 구현 중 자동 리뷰, gstack `/review`는 배포 전 최종 리뷰.

### 8.4 배포: `/ship` vs `finishing-a-development-branch`

| | gstack `/ship` | superpowers `finishing-a-development-branch` |
|---|---|---|
| 방식 | 자동 (VERSION 범프, CHANGELOG, PR) | 4가지 옵션 제시 |

**규칙**: gstack `/ship` 사용. superpowers 배포 스킬은 무시.

### 8.5 계획: `/plan-eng-review` vs `writing-plans`

| | gstack `/plan-eng-review` | superpowers `writing-plans` |
|---|---|---|
| 레벨 | 전략적 (무엇을, 왜) | 전술적 (어떤 파일을, 어떤 순서로) |

**규칙**: 보완적. gstack이 전략 계획 → superpowers가 실행 계획으로 변환.

---

## 10. CLAUDE.md 템플릿

프로젝트 루트에 `CLAUDE.md`를 생성하고 아래 내용을 추가한다. 프로젝트 기본 정보와 하이브리드 전략 규칙을 모두 포함한다.

> **테스트 검증 완료 (2026-04-02):** 아래 템플릿의 하이브리드 전략 섹션으로 테스트한 결과, gstack 문서가 프로젝트 내부에 저장되고, superpowers brainstorming이 기획 단계에서 끼어들지 않으며, writing-plans가 gstack 문서를 정상적으로 읽고 TDD 작업 목록을 생성하는 것을 확인했다.

> **작성 원칙:** CLAUDE.md는 매 세션 시스템 프롬프트로 로드되므로 모든 토큰이 비용이다. 매 세션 필요한 정보만 포함하고, 가끔 필요한 정보(API 규격, DB 스키마, 배포 절차 등)는 `docs/`에 분리하여 참조만 남긴다.

```markdown
# {프로젝트명}

## Role

{기술 스택 기반 역할 정의}
예: Next.js 14 App Router + TypeScript 기반 풀스택 개발

## Project

{프로젝트 1줄 설명}
예: TaskFlow - 소규모 팀을 위한 프로젝트 관리 SaaS

## Tech Stack

{핵심 기술 + 버전. 버전에 따라 API가 달라지는 것만 명시}
예:
- Next.js 14.2 (App Router)
- TypeScript 5.4
- Prisma 6.1 + PostgreSQL 16

## Commands

{자주 쓰는 명령어}
예:
- `npm run dev`: 개발 서버 (port 3000)
- `npm run test`: Jest 단위 테스트
- `npm run lint`: ESLint

## Rules

코딩 시 반드시 참조: @docs/rules/coding-rules.md
{프로젝트 유형에 따라 추가 규칙 문서 참조}
예:
- API 규칙: @docs/rules/api-rules.md
- DB 규칙: @docs/rules/database-rules.md

## Gotchas

{이 프로젝트만의 특수한 주의사항}
예:
- Stripe webhook은 시그니처 검증 필수
- 인증 흐름: @docs/authentication.md 참조

## Hybrid Strategy (gstack + superpowers)

### 스킬 역할 분리

**gstack (명시적 호출만)**
- 기획: /office-hours, /autoplan, /plan-ceo-review, /plan-design-review, /plan-eng-review
- 검증: /review, /qa, /cso
- 배포: /ship, /land-and-deploy
- gstack proactive는 false로 설정되어 있음. 자동 추천하지 말 것.

**superpowers (구현 단계에서 자동)**
- 구현 계획: writing-plans
- TDD 기반 코딩: test-driven-development
- 디버깅: systematic-debugging
- 코드 검증: verification-before-completion

### 충돌 방지 규칙

1. gstack 슬래시 명령어 실행 중에 superpowers 스킬을 자동 호출하지 말 것
2. 디버깅은 superpowers systematic-debugging만 사용. /investigate 사용하지 말 것
3. /ship 실행 시 finishing-a-development-branch 자동 호출하지 말 것
4. gstack 기획 단계에서 superpowers brainstorming 자동 호출하지 말 것

### 문서 저장 규칙

모든 기획/설계 문서는 프로젝트 내부에 저장할 것:
- gstack 문서: docs/gstack/ 하위에 저장
- superpowers 문서: docs/superpowers/ 하위에 저장
- 전역 경로(~/.gstack/)가 아닌 프로젝트 내부 경로를 우선 사용

### 세션 운영

- 각 단계는 독립 세션으로 진행
- 세션 간 연결은 docs/ 내 문서로 수행
- 새 세션 시작 시 이전 단계 문서를 먼저 읽을 것
- 상황별 워크플로우: @docs/gstack-superpowers-hybrid.md 참조
```

---

## 11. 트러블슈팅 / FAQ

### Q: superpowers가 gstack 기획 단계에서 자동 실행된다

CLAUDE.md에 "gstack 슬래시 명령어 실행 중에 superpowers 스킬을 자동 호출하지 말 것"이 있는지 확인한다. 그래도 발생하면 해당 스킬을 무시하고 gstack 워크플로우를 따른다.

### Q: 세션 클리어 후 이전 문서를 찾지 못한다

`/plan-exec` 명령어를 사용한다. 자동으로 `docs/gstack/plans/` → `docs/gstack/design/` → `~/.gstack/projects/` 순서로 계획 문서를 탐색한다. 특정 파일을 지정할 수도 있다:

```
/plan-exec docs/gstack/plans/파일명.md
```

### Q: 구현 시작 시 writing-plans가 발동하지 않는다

`/plan-exec` 명령어를 사용한다. gstack 계획 문서를 자동으로 찾아서 superpowers `writing-plans`를 호출하고, `subagent-driven-development`로 TDD 구현까지 자동 진행한다.

### Q: 두 스킬이 동시에 뜰 때 어느 쪽을 따를까

- **기획/리뷰/배포 단계**: gstack을 따른다
- **구현/디버깅 단계**: superpowers를 따른다
- 현재 어떤 단계인지에 따라 담당 도구가 달라진다

### Q: 검증을 어디까지 해야 할지 모르겠다

섹션 7의 검증 원칙을 참조한다:
- 코드 변경 → verification (항상)
- diff 큼 → /review (추가)
- 화면 변경 → /qa (추가)
- 외부 노출면 변경 → /cso (추가)

### Q: gstack 문서가 전역 경로(~/.gstack/)에만 생성된다

gstack 스킬 코드는 `~/.gstack/projects/{SLUG}/`를 하드코딩하고 있어, CLAUDE.md 규칙에도 불구하고 전역 경로에 저장될 수 있다. 이 경우:

1. 전역 경로에서 프로젝트 내부로 복사한다:
```bash
# gstack 프로젝트 SLUG 확인
ls ~/.gstack/projects/

# 문서 복사
cp ~/.gstack/projects/{SLUG}/*-design-*.md docs/gstack/design/
cp ~/.gstack/projects/{SLUG}/*-plan-*.md docs/gstack/plans/
```
2. 또는 `/plan-exec`에 전역 경로를 직접 지정한다:
```
/plan-exec ~/.gstack/projects/{SLUG}/최신문서.md
```

---

## 테스트 검증 결과

2026-04-02에 "마크다운 TODO 추출 CLI 도구"를 대상으로 전체 워크플로우를 테스트했다. 각 단계는 독립 서브에이전트(= 새 세션)에서 실행하여 세션 클리어 환경을 시뮬레이션했다.

### 검증 항목

| 항목 | 결과 | 비고 |
|------|:----:|------|
| gstack 문서 프로젝트 내부 저장 | O | CLAUDE.md 규칙을 AI가 따름 (100% 보장은 아님) |
| superpowers가 gstack 문서 이해 | O | 667줄의 계획 문서를 전체 파싱 |
| writing-plans 스킬 발동 | O | Skill 도구로 정상 호출됨 |
| TDD 형식 작업 목록 생성 | O | failing test → implementation → passing test → commit |
| brainstorming 차단 | O | CLAUDE.md 충돌 방지 규칙이 효과적 |
| 문서 형식 호환 | O | gstack 마크다운 → superpowers 입력으로 직접 사용 |

### 생성된 문서 구조

```
docs/
├── gstack/
│   ├── design/
│   │   └── tiredman-main-design-20260402-143000.md    ← /office-hours
│   └── plans/
│       └── tiredman-main-plan-20260402-141600.md      ← /autoplan
└── superpowers/
    └── plans/
        └── 2026-04-02-md-todo.md                      ← writing-plans
```

### 알려진 제한사항

1. **문서 경로 보장 안 됨**: gstack 스킬 코드는 `~/.gstack/`을 하드코딩. CLAUDE.md 규칙은 AI 판단에 의존하므로 간혹 전역 경로에 저장될 수 있다. 수동 복사로 대응.
2. **superpowers 자동 트리거 제어 한계**: CLAUDE.md 규칙으로 대부분 제어 가능하나, 100% 보장은 아니다.
3. **프로젝트 레벨 설치로도 경로 문제 미해결**: `setup --local`은 스킬 코드만 이동. 문서 저장 경로는 변하지 않는다.
4. **`GSTACK_HOME` 환경변수 부분 지원**: office-hours 스킬 내 30곳 중 4곳만 반영. 환경변수만으로는 완전한 경로 전환 불가.

---

*최종 업데이트: 2026-04-02*
