---
title: "워크플로 해부 · WF2 pixel-char — PixelLab image-to-pixelart 호출의 실제"
date: 2026-04-22
category: comfyui
tags: [comfyui, workflow, pixellab, api, pixel-art, portrait-pipeline]
draft: false
---

이 글은 4-워크플로 리디자인 파이프라인에서 두 번째 조각에 해당하는 **WF2(`pixel-char.json`)** 의 해부 기록이다. WF1이 만든 일러스트 PNG를 입력으로 받아, PixelLab의 `image-to-pixelart` API를 호출해서 **64×64 픽셀 아트**로 변환하는 것이 전부다. 단 4개 노드로 구성되어 있다.

## WF2가 담당하는 좁은 역할

4-워크플로 설계에서 각 워크플로는 **한 API 호출**만 책임진다.

```
WF1 (PixelLab illustration-attention-nobg) → 일러스트 PNG
    ↓
WF2 (PixelLab image-to-pixelart)           → 픽셀 PNG (64×64) ★ 이 글
    ↓
WF3 (rotate)                                → 4방향 픽셀 PNG 4장
    ↓
WF4a/b (배경 + 합성)                         → 최종 씬
```

이렇게 쪼개는 이유는 앞서 쓴 [[2026-04-22-workflow-json-구조와-설계|워크플로 JSON 구조와 설계]] 글을 참고.

## 그래프 전체

```text
┌─────────────────────────────────────┐
│ PrimitiveNode                       │  ← Character ID (CID) 만 들어있음
│ "paste_CID_here_20260421_000001"    │
└─────────────────────────────────────┘
       (UI용, 실제 그래프 연결 안 됨)

┌──────────────────────────────┐   IMAGE   ┌───────────────────────┐
│ LoadImage                    │ ────────→ │ PixelLabImageToPixel- │
│ WF1 attention-nobg PNG 로드   │           │ art                   │
└──────────────────────────────┘           │ size=64, tgs=8.0,     │
                                           │ seed=0 (random)       │
                                           └──────────┬────────────┘
                                                      │ IMAGE
                                                      ▼
                                           ┌───────────────────────┐
                                           │ SaveImage             │
                                           │ <CID>_pixel-south_64  │
                                           └───────────────────────┘
```

노드는 **PrimitiveNode, LoadImage, PixelLabImageToPixelart, SaveImage** 4개. 실제 데이터 흐름은 3노드 직렬이고 PrimitiveNode는 "CID를 복사-붙여넣기 하는 UI용 메모" 역할이다 — 어디에도 link가 연결되어 있지 않다.

## 커스텀 노드: PixelLabImageToPixelart

ComfyUI 기본 노드가 아니라 이 프로젝트에서 추가한 커스텀 노드다. 소스는 `custom_nodes/pixellab/pixellab_nodes/` 에 있고, 내부적으로 PixelLab API의 `/image-to-pixelart` 엔드포인트를 호출한다. `nodes/` 서브패키지 이름 충돌 버그 때문에 디렉터리 이름을 `pixellab_nodes/` 로 강제로 바꿔야 했다 — 그 이야기는 [[2026-04-21-custom-node-shadow-collision|커스텀 노드 shadow collision]] 글에 정리해뒀다.

### 노드 입력 위젯 4개

JSON에 `widgets_values: [64, 8.0, 0, ""]` 로 박혀있다. 순서대로:

| 위젯 | 값 | 의미 |
|---|---|---|
| `target_size` | 64 | 출력 PNG 한 변 픽셀 수 (정사각형) |
| `text_guidance_scale` | 8.0 | PixelLab 프롬프트 가중치 (API 스펙 범위 1~20) |
| `seed` | 0 | 0 = 랜덤 (API가 내부에서 생성) |
| `prompt` | `""` | 선택적 추가 텍스트 프롬프트 (비어있음 = 이미지만) |

### 왜 target_size = 64 인가

이 값이 게임 에셋에 바로 쓸 수 있는 해상도다. 32는 너무 작아서 얼굴 디테일이 뭉개지고, 128 이상은 픽셀 아트의 "픽셀감"이 사라진다. 64×64는 Stardew Valley / Dave the Diver 계열 게임의 캐릭터 초상화와 거의 같은 스케일이다.

### 왜 text_guidance_scale = 8.0 인가

API 스펙상 1~20 범위인데 기본값 8.0을 그대로 썼다. 낮으면(1~5) 입력 이미지의 색/형태를 더 충실히 따라가고, 높으면(15~20) 픽셀아트 스타일이 더 강하게 덧칠된다. 8.0은 "스타일 적용은 하되 원본 실루엣은 유지" 쪽의 기본값이다. 품질 불만이 있으면 이 값을 돌리는 게 첫 번째 레버.

## E2E 실행 결과(1차 검증)

실제 돌려본 기록은 커밋 `5ebdbd7` 에 있다.

- 입력: WF1에서 생성된 일러스트 PNG (512×512, 배경 제거됨)
- 출력: `<CID>_pixel-south_64_00001.png`
- 실행 시간: **14.7초** (대부분 API 왕복, 로컬 연산은 무시 수준)
- 결과 해상도: 64×64 (spec 그대로)
- 색 수: **1095 colors** (PIL `Image.getcolors(maxcolors=65536)` 기준)
- 알파 채널: **0개 투명 픽셀**

두 가지가 바로 눈에 들어온다.

### 1. 1095 색은 "픽셀 아트"치고 많다

의도된 팔레트(16~32 색)에 비하면 거의 50배다. PixelLab API가 픽셀 그리드는 맞추지만 **색 수 제한은 하지 않는다**. 게임에 넣으려면 후처리로 팔레트 양자화(k-means / median-cut)가 한 번 더 필요하다 — 아직 WF2에 포함되지 않은 단계.

### 2. 알파가 사라진다

입력(WF1 결과)은 `illustration-attention-nobg` 라 투명 배경인데, WF2 출력은 **완전히 불투명**이다. API 응답이 RGB로 돌아온다(RGBA 아님). 이것도 후처리에서 복원해야 한다 — 원본 알파 마스크를 저장해뒀다가 WF2 출력에 다시 합성하거나, WF2 출력의 주변 흰색/단색을 후처리로 잘라내는 식.

이 두 가지는 현재 "품질 불만 기록" 으로 메모리에 남아있고, 다음 반복에서 결정될 이슈다.

## WF1과의 계약(contract)

이 워크플로가 성립하려면 WF1이 몇 가지 약속을 지켜야 한다.

- 파일명에 **CID 접두어** 포함 — WF2의 `LoadImage` 위젯 값이 `<CID>_illustration-attention-nobg_00001.png` 형식을 전제
- **투명 배경 PNG** (RGBA) — 비록 WF2 단계에서 알파가 사라지지만, 실루엣 정확도는 WF1의 배경 제거 품질에 의존
- **512×512 정사각형 권장** — target_size=64로 줄일 때 왜곡이 없는 비율

이 contract 를 명시적으로 문서화한 곳은 `docs/superpowers/specs/2026-04-21-portrait-pipeline-4workflow-design.md` 이고, JSON 안에도 `extra.spec_version: "2026-04-21-4workflow"` 로 스펙 버전이 박혀있다.

## 재현 방법

1. ComfyUI가 돌고 있다고 가정한다 (설치법은 [[2026-04-21-comfyui-소개-amd-zluda-설치]]).
2. `custom_nodes/pixellab/` 가 로드되어 있어야 한다 — 로그에 `PixelLabImageToPixelart` 노드가 등록되었다고 찍혀야 함.
3. PixelLab API 키가 환경 변수 `PIXELLAB_API_KEY` 로 설정되어 있어야 한다 (커스텀 노드가 여기서 읽음). **레포/워크플로 JSON에 키를 박지 말 것.**
4. WF1에서 생성된 PNG를 `ComfyUI/input/` 에 복사.
5. Web UI에서 `pixel-char.json` 을 로드하고 LoadImage 의 파일명을 실제 파일명으로 바꾼다.
6. PrimitiveNode 의 문자열도 같은 CID 로 수정 (SaveImage 파일명 접두어가 여기서 유래).
7. Queue Prompt.

결과물은 `ComfyUI/output/<CID>_pixel-south_64_00001.png`.

## 정리

WF2는 **"한 API 호출 감싸기"** 의 교과서적인 예시다. 노드 4개, 링크 2개, 실행 15초. 하지만 뽑아낸 픽셀 아트를 게임에 바로 넣으려면 팔레트 축소와 알파 복원이 추가로 필요하다 — 이게 다음 반복(WF2.1) 의 과제다.

## 관련 글

- [[2026-04-22-wf1-pixel-mystery-character-v10|WF1 · pixel-mystery-character v10 — 초상화 + 4방향 turnaround 한 번에]]
- [[2026-04-22-workflow-json-구조와-설계|워크플로 JSON 구조와 설계]]
- [[2026-04-21-custom-node-shadow-collision|커스텀 노드 shadow collision — nodes/ 서브패키지 함정]]
