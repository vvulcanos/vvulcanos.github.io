---
title: "워크플로 해부 · WF2 pixel-char — PixelLab image-to-pixelart 호출의 실제"
date: 2026-04-22
category: comfyui
tags: [comfyui, workflow, pixellab, api, pixel-art, portrait-pipeline]
draft: false
---

4-워크플로 파이프라인의 두 번째 조각 `pixel-char.json`. [[2026-04-22-wf1-portrait-npc|WF1]] 이 만든 일러스트 PNG 를 받아 PixelLab 의 `image-to-pixelart` API 를 태워 64×64 픽셀 아트로 바꾸는 게 하는 일의 전부다. 노드는 네 개.

## WF2 가 맡는 좁은 역할

파이프라인 전체에서 보면 WF2 는 한 API 호출을 감싸는 얇은 레이어다.

```
WF1 (PixelLab illustration-attention-nobg) → 일러스트 PNG
    ↓
WF2 (PixelLab image-to-pixelart)           → 픽셀 PNG (64×64) ★ 이 글
    ↓
WF3 (rotate)                                → 4방향 픽셀 PNG 4장
    ↓
WF4a/b (배경 + 합성)                         → 최종 씬
```

이렇게 쪼개는 이유는 [[2026-04-22-workflow-json-구조와-설계|여기]] 에서 따로 정리했다.

## 그래프

```text
┌─────────────────────────────────────┐
│ PrimitiveNode                       │  ← Character ID (CID) 만 담음
│ "paste_CID_here_20260421_000001"    │
└─────────────────────────────────────┘
       (UI 메모용, 실제 그래프 연결 없음)

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

노드는 PrimitiveNode, LoadImage, PixelLabImageToPixelart, SaveImage 네 개. 실제 데이터 흐름은 3노드 직렬이고 PrimitiveNode 는 CID 를 복사-붙여넣기 하는 메모용이라 어디에도 link 가 안 걸려 있다.

## 커스텀 노드 PixelLabImageToPixelart

ComfyUI 기본 노드가 아니라 이 프로젝트에서 추가한 커스텀 노드다. 소스는 `custom_nodes/ComfyUI-PixelLab/pixellab_nodes/` 에 있고, 내부에서 PixelLab API 의 `/image-to-pixelart` 엔드포인트를 호출한다. 디렉터리 이름이 `nodes/` 가 아니라 `pixellab_nodes/` 인 이유는 shadow 버그 때문인데 그 얘기는 [[2026-04-21-custom-node-shadow-collision|여기]] 에.

위젯은 네 개다. JSON 에 `widgets_values: [64, 8.0, 0, ""]` 로 박혀 있다.

| 위젯 | 값 | 의미 |
|---|---|---|
| `target_size` | 64 | 출력 PNG 한 변 픽셀 수 (정사각) |
| `text_guidance_scale` | 8.0 | PixelLab 가중치 (API 스펙 범위 1~20) |
| `seed` | 0 | 0 = 랜덤 (API 내부 생성) |
| `prompt` | `""` | 선택적 추가 텍스트 (비우면 이미지 기반) |

64 를 고른 이유는 게임 에셋에 바로 넣을 수 있는 해상도라서. 32 는 얼굴 디테일이 뭉개지고 128 이상은 픽셀감이 사라진다. Stardew Valley / Dave the Diver 계열 캐릭터 초상화와 대략 같은 스케일이다.

`text_guidance_scale` 은 API 스펙상 1~20 범위인데 기본값 8.0 을 그대로 썼다. 낮추면(1~5) 입력 이미지의 색·형태를 더 충실히 따라가고, 올리면(15~20) 픽셀아트 스타일이 강하게 덧칠된다. 8.0 은 "스타일은 입히되 원본 실루엣은 유지" 쪽 기본값. 품질이 마음에 안 들 때 가장 먼저 돌리는 손잡이다.

## 실측 (1차)

실제 한 번 돌려본 결과는 커밋 `5ebdbd7` 에 박아 뒀다.

- 입력: WF1 산출 일러스트 PNG (512×512, 배경 제거됨)
- 출력: `<CID>_pixel-south_64_00001.png`
- 실행 시간: 14.7초 — 대부분 API 왕복, 로컬 연산은 무시할 만한 수준
- 해상도: 64×64 (스펙 그대로)
- 색 수: 1095 (PIL `Image.getcolors(maxcolors=65536)` 기준)
- 알파 채널: 투명 픽셀 0

둘이 눈에 걸린다.

먼저 색 수 1095. 의도한 팔레트(16~32색)에 비하면 거의 50배다. API 가 픽셀 그리드는 맞춰 주지만 색 수 제한은 해 주지 않는다. 게임에 넣으려면 후처리로 팔레트 양자화(k-means / median-cut)가 한 번 더 필요한데, 지금 WF2 에는 이 단계가 빠져 있다.

다음 알파. WF1 결과가 `illustration-attention-nobg` 라 투명 배경이었는데 WF2 출력은 완전히 불투명하다. API 응답이 RGB 라서 알파가 날아간 결과. 복원하려면 원본 알파 마스크를 따로 저장해 뒀다가 출력에 다시 합성하거나, 단색 배경을 후처리로 잘라내야 한다.

두 가지 다 현재 "품질 이슈" 로 열려 있고, 다음 반복에서 결정할 몫이다.

## WF1 과의 계약

WF2 가 성립하려면 WF1 이 몇 가지 약속을 지켜야 한다. 파일명에 CID 접두어가 포함돼야 하고(`LoadImage` 위젯이 `<CID>_illustration-attention-nobg_00001.png` 를 전제), 투명 배경 PNG 여야 한다(WF2 에서 알파가 날아가더라도 실루엣 정확도는 WF1 배경 제거 품질에 달려 있음). 해상도는 512×512 정사각형 권장 — target_size=64 로 줄일 때 비율 왜곡이 없다.

이 contract 는 `docs/superpowers/specs/2026-04-21-portrait-pipeline-4workflow-design.md` 에 문서로 박혀 있고, JSON 의 `extra.spec_version: "2026-04-21-4workflow"` 로 스펙 버전까지 찍혀 있다.

## 재현

1. ComfyUI 가 돌고 있다고 가정 (설치는 [[2026-04-21-comfyui-소개-amd-zluda-설치]]).
2. `custom_nodes/ComfyUI-PixelLab/` 가 로드돼 있어야 한다 — 로그에 `PixelLabImageToPixelart` 노드 등록 찍힘.
3. PixelLab API 키는 환경변수 `PIXELLAB_API_KEY` 로. 레포·워크플로 JSON 에 키 박지 말 것.
4. WF1 에서 뽑힌 PNG 를 `ComfyUI/input/` 에 복사.
5. Web UI 에서 `pixel-char.json` 로드, `LoadImage` 의 파일명을 실제 파일명으로 교체.
6. PrimitiveNode 의 문자열도 같은 CID 로 (SaveImage 파일명 접두어가 여기서 유래).
7. Queue Prompt.

결과는 `ComfyUI/output/<CID>_pixel-south_64_00001.png`.

---

노드 네 개에 링크 두 개, 실행 15초. 구조 자체는 단순한데 뽑아낸 픽셀 아트를 그대로 게임에 넣기 전에 팔레트 축소와 알파 복원이 한 번 더 붙어야 한다. 이건 WF2.1 쯤에 얹힐 과제로 남겨 뒀다.

## 관련 글

- [[2026-04-22-wf1-portrait-npc|WF1 · portrait-npc — 한 캐릭터의 초상화 5종 순차 생성]]
- [[2026-04-22-workflow-json-구조와-설계|워크플로 JSON 구조와 설계]]
- [[2026-04-21-custom-node-shadow-collision|커스텀 노드 shadow collision — nodes/ 서브패키지 함정]]
