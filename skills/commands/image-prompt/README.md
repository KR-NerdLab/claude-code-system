# /image-prompt

Nano Banana Pro용 이미지 생성 프롬프트를 만들어주는 Claude Code Skill.

## 기본 사용법

```
/image-prompt [만들고 싶은 이미지 설명]
```

간단하게 써도 되고, 상세하게 써도 된다.

```
/image-prompt 로고
/image-prompt 사주 운세 앱 로고, 둥근 원형, 보라색 그라데이션, 배경 투명
```

## 스타일 지정

키워드를 포함하면 자동으로 해당 스타일이 반영된다.

| 키워드 | 스타일 |
|--------|--------|
| `지브리` / `ghibli` | 지브리 애니메이션 스타일 |
| `픽셀` / `pixel` | 레트로 픽셀아트 |
| `수채화` / `watercolor` | 수채화 |
| `미니멀` / `minimal` | 미니멀 플랫 디자인 |
| `3D` | 3D 렌더링 |
| `사실적` / `realistic` | 포토리얼리스틱 |

```
/image-prompt 지브리 스타일로 숲속 배경 일러스트
/image-prompt 픽셀 아트 게임 아이콘, 검과 방패
```

프리셋에 없는 스타일도 자유롭게 요청 가능:

```
/image-prompt 반 고흐 화풍으로 별이 빛나는 밤하늘
```

## 이미지 to 이미지

기존 이미지를 참조해서 새 이미지를 만들고 싶을 때 `ref:` 를 사용한다.

```
/image-prompt 지브리 스타일로 변환 ref:assets/drafts/app-logo-v1.png
/image-prompt 색감을 따뜻하게 변경 ref:assets/drafts/background-v1.png
```

참조 이미지는 Antigravity AI 채팅에 프롬프트와 함께 직접 첨부해야 한다.

## 저장 경로

| 단계 | 경로 |
|------|------|
| 작업/검토 | `assets/drafts/` |
| 최종 반영 | `frontend/static/` |

- 파일명은 자동으로 제안된다 (예: `app-logo-v1.png`)
- 같은 이름이 있으면 버전이 올라간다 (v1 → v2 → v3)

## 워크플로우

1. `/image-prompt`로 프롬프트 생성
2. 출력된 영문 프롬프트를 복사
3. Antigravity AI 채팅에 붙여넣기 (ref 이미지가 있으면 함께 첨부)
4. 생성된 이미지를 제안된 경로에 저장
5. 마음에 들면 `frontend/static/`으로 이동
