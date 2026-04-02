# plan-exec

gstack(기획/설계) + superpowers(구현) 하이브리드 전략에서 구현 단계 진입점.

gstack 계획 문서를 자동 탐색하여 superpowers의 TDD 구현 파이프라인을 실행하는 브릿지 스킬이다.

```
/autoplan → /plan-exec → /review → /ship
 (기획)      (구현)       (검증)    (배포)
```

## 전제 조건

- **gstack** 설치 및 `proactive false` 설정
- **superpowers** 플러그인 설치
- gstack 기획 단계 (`/office-hours`, `/autoplan` 등) 완료

## 사용법

```bash
# 자동으로 최신 계획 문서 탐색
/plan-exec

# 특정 문서 지정
/plan-exec docs/gstack/plans/파일명.md
```

## 워크플로우에서의 위치

```
[gstack 기획]          [구현]                    [gstack 검증/배포]
/office-hours    →    /plan-exec           →    /review
/autoplan              (이 스킬)                 /qa
                                                 /ship
```

## 내부 동작

1. gstack 계획 문서 탐색 (인자 또는 자동)
2. `superpowers:writing-plans` → TDD 작업 목록 생성
3. `superpowers:subagent-driven-development` → 태스크별 자동 실행
   - 구현 서브에이전트 (TDD 코딩)
   - 스펙 리뷰 서브에이전트 (계획대로 만들었나 확인)
   - 코드 품질 리뷰 서브에이전트 (코드 품질 확인)
4. 완료 후 다음 단계 안내 (`/review`, `/qa`, `/ship`)

## 설치

프로젝트의 `.claude/skills/` 디렉토리에 복사:

```bash
cp -r plan-exec/ <프로젝트>/.claude/skills/plan-exec/
```

## 참고

- [gstack + superpowers 병행 사용 가이드](../../docs/gstack-superpowers-guide.md)
