---
name: image-prompt
description: Google Nano Banana Pro용 이미지 생성 프롬프트를 작성한다
disable-model-invocation: true
model: sonnet
---

## 역할

사용자의 요청을 기반으로 Google Nano Banana Pro에 최적화된 이미지 생성 프롬프트를 작성한다.

## 프롬프트 작성 규칙

- 영문으로 작성 (Nano Banana Pro 최적화)
- 스타일, 색상, 구도, 분위기, 해상도를 구체적으로 명시
- 모호한 표현 대신 시각적으로 명확한 디스크립션 사용
- 원하지 않는 요소가 있으면 네거티브 프롬프트도 포함

## 스타일 프리셋

사용자가 아래 키워드를 포함하면 해당 스타일을 프롬프트에 반영한다:

- `지브리` / `ghibli` → Studio Ghibli anime style, soft watercolor textures, warm lighting, whimsical atmosphere
- `픽셀` / `pixel` → Pixel art style, retro 8-bit/16-bit aesthetic, clean pixel edges
- `수채화` / `watercolor` → Traditional watercolor painting, soft blended edges, paper texture
- `미니멀` / `minimal` → Minimalist flat design, clean vector style, simple geometric shapes
- `3D` → 3D rendered, smooth surfaces, soft studio lighting, subtle shadows
- `사실적` / `realistic` → Photorealistic, high detail, natural lighting

사용자가 별도 스타일을 지정하면 프리셋보다 우선한다.

## 이미지 to 이미지

사용자가 `ref:파일경로` 형식으로 참조 이미지를 지정하면:

1. 해당 파일이 존재하는지 확인
2. 프롬프트 출력 시 "참조 이미지를 Antigravity AI 채팅에 함께 첨부하세요"라는 안내 포함
3. 참조 이미지의 어떤 요소를 유지/변경할지 프롬프트에 명시

## 저장 경로

- 작업/검토용: `assets/drafts/` 디렉토리에 저장
- 파일명: 요청 내용을 반영한 케밥케이스 (예: `app-logo-v1.png`, `background-illustration-v1.png`)
- 동일 파일명이 이미 존재하면 버전 번호를 올린다 (v1 → v2 → v3)
- 프롬프트 끝에 저장 경로 지시문을 한국어로 포함 (절대경로)

## 출력 형식

모든 정보를 하나의 코드블록 안에 담아서 한 번에 복사 가능하게 출력한다.
코드블록 바깥에는 아무것도 쓰지 않는다 (참조 이미지 안내만 예외).

아래 예시처럼 출력한다:

```
[Prompt]
A beautiful landscape, soft lighting, warm colors, 4K resolution

[Negative]
blurry, low quality, dark atmosphere

[Save]
/절대경로/assets/drafts/파일명-v1.png
```

- `[Prompt]`: 포지티브 프롬프트 (필수)
- `[Negative]`: 네거티브 프롬프트 (해당 시에만, 없으면 섹션 자체 생략)
- `[Save]`: 저장할 절대경로 (필수)
- 참조 이미지가 있는 경우에만 코드블록 아래에 안내 문구 추가

## 요청

$ARGUMENTS
