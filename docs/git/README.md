# Git Convention Docs

프로젝트 공통 Git 규칙 문서 모음. 새 프로젝트에 복사해서 사용한다.

## 문서 목록

| 문서 | 설명 |
|------|------|
| [commit-convention.md](commit-convention.md) | 커밋 메시지 형식, type, scope, 커밋 단위 |
| [branch-strategy.md](branch-strategy.md) | 브랜치 구조, 작업 흐름, 네이밍 규칙 |
| [pr-convention.md](pr-convention.md) | PR 제목, 본문 템플릿, 작성 기준 |
| [squash-merge-convention.md](squash-merge-convention.md) | main 머지 시 squash merge 메시지 형식 |

## 적용 방법

두 가지 레벨로 적용할 수 있다. 상황에 맞게 선택한다.

### 방법 1: 프로젝트 레벨 (팀 공유, 프로젝트별 커스터마이징)

규칙 문서를 프로젝트에 복사한다. git에 포함되므로 팀원과 공유되고, 프로젝트마다 다르게 수정할 수 있다.

```bash
# 프로젝트 루트에서
mkdir -p docs/git
cp -r <claude-code-system경로>/docs/git/*.md docs/git/
rm docs/git/README.md  # README는 프로젝트에 불필요
```

프로젝트 `CLAUDE.md`에 추가:

```markdown
## Git 규칙 (필수)

- **커밋**: 메시지 형식, type, 커밋 단위 — docs/git/commit-convention.md
- **브랜치**: 작업 흐름, 네이밍 규칙 — docs/git/branch-strategy.md
- **PR**: 제목/본문 템플릿 — docs/git/pr-convention.md
- **Squash 머지**: main 머지 메시지 형식 — docs/git/squash-merge-convention.md

### 작업 완료 후 워크플로우
1. develop에 머지 → 푸시 → 개발서버 확인
2. 사용자 확인 완료 후 진행
3. main rebase 후 PR 생성 (squash merge)
```

### 방법 2: 사용자 레벨 (개인 전체 프로젝트 공통 적용)

`~/.claude/` 아래에 규칙 문서를 복사하고 `~/.claude/CLAUDE.md`에서 참조하면 모든 프로젝트에 자동 적용된다.

```bash
# 규칙 문서 복사
mkdir -p ~/.claude/docs/git
cp <claude-code-system경로>/docs/git/*.md ~/.claude/docs/git/
rm ~/.claude/docs/git/README.md  # README는 불필요
```

```bash
# ~/.claude/CLAUDE.md 생성 (또는 기존 파일에 추가)
cat >> ~/.claude/CLAUDE.md << 'EOF'

## Git 규칙 (필수)

- **커밋**: 메시지 형식, type, 커밋 단위 — ~/.claude/docs/git/commit-convention.md
- **브랜치**: 작업 흐름, 네이밍 규칙 — ~/.claude/docs/git/branch-strategy.md
- **PR**: 제목/본문 템플릿 — ~/.claude/docs/git/pr-convention.md
- **Squash 머지**: main 머지 메시지 형식 — ~/.claude/docs/git/squash-merge-convention.md

### 작업 완료 후 워크플로우
1. develop에 머지 → 푸시 → 개발서버 확인
2. 사용자 확인 완료 후 진행
3. main rebase 후 PR 생성 (squash merge)
EOF
```

### 어떤 방법을 선택할까?

| | 프로젝트 레벨 | 사용자 레벨 |
|---|---|---|
| 팀 공유 | O (git에 포함) | X (내 머신만) |
| 프로젝트별 커스터마이징 | O | X (전체 동일) |
| 새 프로젝트 설정 필요 | O (매번 복사) | X (자동 적용) |
| 다른 머신 이동 | O (clone하면 됨) | X (머신마다 설정) |
| 규칙 업데이트 | 프로젝트마다 수동 | 한 곳만 수정 |

**권장**: 팀 프로젝트는 프로젝트 레벨, 개인 사이드 프로젝트는 사용자 레벨. 둘 다 설정하면 양쪽 모두 로드되므로 프로젝트 레벨이 우선한다 (더 구체적인 규칙이 적용됨).

## 커스터마이징

문서를 프로젝트에 복사한 후에는 프로젝트 상황에 맞게 수정해도 된다.

- develop 브랜치가 없는 프로젝트: branch-strategy.md에서 develop 관련 내용 제거
- 영문 커밋을 사용하는 프로젝트: commit-convention.md의 subject 언어 규칙 수정
- 모노레포가 아닌 프로젝트: scope 규칙 단순화

원본은 이 저장소에서 관리하므로, 공통 규칙이 변경되면 여기를 업데이트한다.
