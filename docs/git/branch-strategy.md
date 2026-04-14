# Git Branch Strategy

## 브랜치 구조

```
main        운영 배포 브랜치
develop     개발 서버 확인용 브랜치
feat/xxx    기능 개발 브랜치
fix/xxx     버그 수정 브랜치
refactor/xxx 리팩터링 브랜치
chore/xxx   설정/빌드 작업 브랜치
```

## 작업 흐름

```
1. main 최신화
   git checkout main
   git pull origin main

2. main에서 feature 브랜치 생성
   git checkout -b feat/기능명

3. feature 브랜치에서 작업 및 커밋
   (커밋 규칙: commit-convention.md)

4. 작업 완료 → develop에 머지, 푸시
   git checkout develop
   git pull origin develop
   git merge feat/기능명
   git push origin develop
   → 개발서버에서 확인

5. 확인 완료 → PR 전 main 변경사항 동기화
   git checkout main
   git pull origin main
   git checkout feat/기능명
   git rebase main
   → 충돌 발생 시 해결 후 진행

6. main에 PR 생성 (squash merge)
   (PR 규칙: pr-convention.md)
   (squash 머지 규칙: squash-merge-convention.md)
```

## 브랜치별 규칙

| 브랜치 | 직접 커밋 | 직접 머지 | PR 필수 | 비고 |
|--------|----------|----------|---------|------|
| main | 금지 | 금지 | 필수 (squash merge) | 운영 브랜치 |
| develop | 금지 | 허용 (feature → develop) | 불필요 | 개발 확인용 |
| feat/xxx | 허용 | - | - | 작업 브랜치 |

## 브랜치 네이밍

```
feat/기능명         새 기능 (예: feat/social-login)
fix/버그명          버그 수정 (예: fix/null-pointer-error)
refactor/대상명     구조 개선 (예: refactor/auth-module)
chore/작업명        설정/빌드 (예: chore/docker-config)
docs/문서명         문서 작업 (예: docs/api-guide)
```

## 금지 사항

- main에 직접 커밋하지 않는다
- main에 직접 머지하지 않는다 (반드시 PR)
- develop을 거치지 않고 main에 PR하지 않는다
- feature 브랜치를 다른 feature 브랜치에서 생성하지 않는다 (항상 main에서)
