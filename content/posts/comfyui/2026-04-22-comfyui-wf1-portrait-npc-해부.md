---
title: "ComfyUI · WF1 portrait-npc 해부 — 한 NPC 의 초상화 5종을 얼굴 일관성 유지하며 생성"
date: 2026-04-22
category: comfyui
tags: [comfyui, workflow, sdxl, ipadapter, controlnet, rembg, lora, portrait-pipeline]
summary: "체크포인트 3종 + 얼굴 임베딩 공유 구조로 한 번에 5장을 같은 얼굴로 떨어뜨리는 그래프"
draft: false
---

## 개요

**범위**: 포트레이트 파이프라인의 첫 워크플로 `portrait-npc.json` 을 그룹 단위로 해부. 한 NPC 에 대해 **얼굴이 같은 초상화 5종이 순차적으로 나오는** 그래프가 어떻게 설계됐고, 각 그룹이 왜 그 자리에 있는지.

**대상**: 같은 캐릭터를 여러 스타일·포즈·해상도로 뽑아야 하는 상황에 있는 사람 — 게임 NPC 아트, 이모티콘 시트, 캐릭터 포트폴리오 등. 체크포인트 여러 개를 한 그래프에 올리는 구조에 관심 있는 사람.

**전제**: [[posts/comfyui/2026-04-22-comfyui-환경-셋업|환경 셋업]] 완료, [[posts/comfyui/2026-04-22-comfyui-워크플로-설계-원칙|워크플로 설계 원칙]] 읽음 (이 글은 그 원칙들이 실제로 적용된 사례). IPAdapter · ControlNet · rembg 같은 SDXL 확장들의 이름은 들어봤음.

**산출물 5종** (모두 같은 `<CID>` 접두어):

- `<CID>_realistic-halfbody_00001.png` — 실사 반신상
- `<CID>_illustration-halfbody_00001.png` — 일러 반신상 (대화 씬용, 실사에 덧칠)
- `<CID>_illustration-halfbody-nobg_00001.png` — 위의 배경 제거본 (합성용)
- `<CID>_bust_00001.png` — 흉상 1024×1024 크롭 (아바타·프로필)
- `<CID>_illustration-attention-nobg_00001.png` — 차렷자세 전신 배경 제거 (WF2 로 이어져 픽셀 스프라이트가 됨)

`<CID>` 는 `slug_YYYYMMDD_rand6` 형식 문자열. WF1 뿐 아니라 WF2/3/4 결과물까지 같은 CID 접두어로 묶여서, 파이프라인 전체에서 한 NPC 의 자산이 파일명만 보고 찾아진다.

## 배경

게임에서 같은 NPC 가 등장하는 자리가 여러 군데다 — 대화 씬의 일러 반신상, 프로필 · 아바타의 흉상, 인게임 픽셀 스프라이트. 장면마다 스타일은 다른데 "같은 사람" 으로 보여야 한다. 얼굴이 조금만 어긋나면 플레이어가 다른 캐릭터로 느낀다.

그렇다고 "일러 반신상 + 흉상 + 픽셀 전신 + 차렷자세 + 실사 ..." 같은 요구를 프롬프트 한 줄로 쑤셔 넣으면 어느 쪽도 제대로 안 나온다. 그래서 **용도별로 렌더는 분리하되 얼굴 임베딩은 그래프 안에서 공유**하는 구조로 짰다.

**왜 다섯 장을 한 그래프에 몰아넣었나** — 일관성과 품질, 두 가지.

- **일관성**: 얼굴 임베딩이 G2 · G4 에서 같은 값으로 재사용돼야 얼굴이 안 흔들린다. 워크플로를 쪼개서 "일러 전용", "흉상 전용", "차렷자세 전용" 따로 돌리면 매 실행마다 얼굴 임베딩을 다시 만들어야 하고, 그 과정에서 얼굴 YOLO 가 잡는 영역이 미세하게 달라지면서 임베딩이 흔들린다. 같은 그래프 안에 두면 한 번 만든 임베딩이 그대로 공급된다.
- **품질**: 하나의 프롬프트 · 하나의 KSampler 에 "실사 / 일러 / 흉상 / 차렷자세" 네 요구를 섞으면 어느 쪽도 깔끔하게 안 찍힌다. G1(실사, 구도 잡기용) → G2(일러 img2img, 스타일 입힘) → G3(배경 제거) → G3-2(흉상 크롭) → G4(다른 포즈 · 같은 얼굴) 순으로 **단계마다 하나의 목표만** 주고 이어 붙여야 각 단계가 덜 힘들고 품질이 유지된다.

파이프라인 전체(WF1 → WF2 → WF3 → WF4)를 쪼개 놓은 이유도 같은 논리다. 일러 → 픽셀 → 4방향 → 애니메이션을 한 그래프에 욱여넣으면 외부 API 호출 순서 · 재시도 · 중간 결과 확인이 지옥이 되고, 특히 픽셀화 · 회전 · 애니메이션은 PixelLab API 를 여러 번 호출하는 구조라 실패 겹칠 때 복구가 불가능해진다. 단계별 파일로 산출물을 박제하면서 이어가는 쪽이 안정적. 쪼개는 기준 상세는 [[posts/comfyui/2026-04-22-comfyui-워크플로-설계-원칙|워크플로 설계 원칙]] 에.

## 그래프 구조 분해

ComfyUI 안에서 6 개 그룹(G0 ~ G3-2) 에 G4 가 덧붙는 구조. 그룹 박스로 쪼개 놓은 건 실행 순서가 머릿속에서 선형으로 따라가지게 하려는 목적.

```
G0    로더 (체크포인트 3개, LoRA, ControlNet, IPAdapter, VAE)
G0.5  얼굴 YOLO → 얼굴 crop → IPAdapter 얼굴 임베딩
G1    RealVis 로 실사 반신상 (768×1024)
G2    G1 결과를 Juggernaut + pixel-art-xl 로 img2img (denoise 0.55)
G3    G2 일러 결과를 rembg 로 배경 제거
G3-2  흉상 1024×1024 크롭
G4    Animagine + OpenPose + IPAdapter(G2 얼굴 재사용) → 차렷자세 → rembg
```

G1 이 먼저 돌아야 G0.5 의 얼굴 임베딩이 G2 / G4 로 흘러갈 수 있고, G3 는 G2 결과를 받고, G3-2 는 G3 결과를 받는다. 그래서 선형.

### G0 — 체크포인트 3 개를 한 그래프에 올려놓는 이유

`realvisxl_v4`(실사) · `animagine-xl-3.1`(일러) · `juggernaut_xl_v9`(img2img 베이스) 세 개가 동시에 로드된다. 여기에 `pixel-art-xl` LoRA 가 Juggernaut 위에 얹힌다.

RX 9070 XT 16GB 에서 이건 빡빡하다. ComfyUI 가 그룹 단위로 모델을 알아서 언로드 · 재로드하면서 돌려서 **세 체크포인트가 한꺼번에 VRAM 에 올라가는 순간은 없다**. 첫 실행에 MIOpen 튜닝까지 겹치면 10~20 분 나오기도 한다 ([[posts/comfyui/2026-04-22-comfyui-환경-셋업|환경 셋업]] 의 MIOpen 항목 참고). 캐시가 달궈진 뒤에는 1~3 분.

*왜 하나로 묶었나*: 체크포인트마다 파이프라인을 쪼개면 그룹 경계에서 얼굴 임베딩을 다시 만들어야 해서 얼굴 일관성이 무너진다. 한 그래프 안에서 돌려야 G0.5 임베딩이 G2 · G4 로 같은 값으로 흘러간다.

### G0.5 — 얼굴을 따로 뽑는 이유

`face_yolov8m.pt` 로 G1 실사 반신상에서 얼굴 박스를 찾아 crop, `IPAdapterEncoder` 에 넣어 얼굴 임베딩을 만든다. 이 임베딩을 **G2 와 G4 가 둘 다 재사용**한다.

덕분에 G1 실사 / G2 일러 / G4 차렷자세 — 세 장이 체크포인트도 프롬프트도 포즈도 다 다른데 얼굴은 같은 사람으로 찍힌다. 5장 사이의 동일성이 여기서 나온다.

### G1 / G2 / G4 샘플러 설정

| 그룹 | Steps | CFG | Sampler | Scheduler | Denoise |
|---|---|---|---|---|---|
| G1 실사 | 30 | 7.0 | dpmpp_2m_sde | karras | 1.0 |
| G2 일러 img2img | 28 | 6.5 | euler_ancestral | normal | **0.55** |
| G4 차렷자세 | 30 | 7.0 | euler_ancestral | normal | 1.0 |

G2 의 `denoise=0.55` 가 핵심. G1 실사 반신상의 VAEDecode 결과를 다시 VAEEncode + ImageScale 거쳐 latent 로 바꿔 넣고, 그걸 기반으로 55% 만 재생성. 구도 · 옷 · 자세는 유지되고 스타일만 일러로 바뀐다.

순수 일러로 바로 가도 되는 거 아닌가 싶지만, 일러 체크포인트에 "한복 입은 조선 여인 정면 반신상" 같은 프롬프트 넣으면 옷 구조나 얼굴 비율이 매번 흔들려서 G1 실사 단계를 **구조 고정용**으로 끼워 넣은 것.

### 프롬프트를 네 토막으로 쪼갬

G1 · G2 · G4 모두 positive / negative 를 각각 두 조각씩으로 더 쪼개서 넣는다.

- `char_*` — 캐릭터 고유 (매번 수정). 예: `1girl, silver hair, stern expression, blue eyes`
- `comp_*` — 구도 · 스타일 · 품질 (거의 안 건드림). 예: `masterpiece, best quality, solo, upper body, centered, looking at viewer`

두 조각을 `ConditioningConcat` 으로 합쳐 `KSampler.positive/negative` 로 보낸다. 이래서 워크플로에 `CLIPTextEncode` 가 12 개, `ConditioningConcat` 이 6 개 박혀 있다.

*왜 쪼갰나*: 새 NPC 만들 때 `char_*` 두 군데만 건드리면 5 장 톤이 저절로 일관된다. 프로젝트 공통 스타일(`comp_*`) 은 한 번 정해 놓고 안 건드리는 게 이상적. 이 축 분리 규약의 배경은 [[posts/comfyui/2026-04-22-comfyui-워크플로-설계-원칙|설계 원칙]] §5 에.

### G4 — 차렷자세 전신, 픽셀 사이드로 이어지는 접점

실사 반신상 · 일러 반신상 · 흉상까지는 "화면 안에서 보여주는 용도" 고, G4 는 결이 다르다. **전신 차렷자세(정면, 팔 차렷)로 뽑아 배경까지 지워 놓는다**. 이 포즈 · 구도가 WF2 의 `/image-to-pixelart` 로 들어갈 때 실루엣이 가장 깔끔하게 픽셀화된다 — 팔 · 다리가 몸통에 붙어 있는 자세라 64×64 그리드에서 형태가 뭉치지 않는다.

G4 블록의 노드 ID 를 보면 33–43 + 86–89 로 두 덩이에 나뉘어 있다. 처음 설계할 때 ID 33–42 로 열 개를 예약해 뒀는데, 이후 프롬프트 4 토막 쪼개기를 적용하면서 `CLIPTextEncode` 두 개 + `ConditioningConcat` 두 개가 더 필요해졌다. 그 시점엔 이미 그래프 다른 쪽에 79–85 가 점유된 상태라 비어 있던 86–89 를 땡겨 썼다.

| ID | 역할 |
|---|---|
| 33 | PrimitiveNode — Character ID |
| 34 | LoadImage — `input/poses/attention.png` |
| 35 | EmptyLatentImage 768×1024 |
| 36 | IPAdapterEmbeds (G2 얼굴 임베딩 재사용, weight 0.7) |
| 37 | ControlNetApplyAdvanced (OpenPose strength 0.8) |
| 38 | CLIPTextEncode `char_pos` ★ |
| 86 | CLIPTextEncode `comp_pos` |
| 87 | CLIPTextEncode `char_neg` ★ |
| 39 | CLIPTextEncode `comp_neg` |
| 88 | ConditioningConcat pos |
| 89 | ConditioningConcat neg |
| 40 | KSampler (Animagine, 30/7, euler_a) |
| 41 | VAEDecode |
| 42 | rembg u2net |
| 43 | SaveImage `<CID>_illustration-attention-nobg` |

`attention.png` 는 [open-pose-editor](https://zhuyu1997.github.io/open-pose-editor/) 에서 만든 정면 차렷자세 맵. 그룹 박스 제목 옆에 편집기 URL 을 주석으로 박아 뒀으니, 포즈 바꿀 일 있으면 거기서 고쳐서 교체.

### CID 파일명 주입

5 개 `SaveImage` 모두 `filename_prefix` 위젯을 Convert Widget to Input 으로 뒤집고, 앞에 `StringConcatenate` 노드가 하나씩 붙어 있다.

```
PrimitiveNode(CID) ─┬─→ StringConcat("_realistic-halfbody")         → SaveImage#12
                    ├─→ StringConcat("_illustration-halfbody")       → SaveImage#28
                    ├─→ StringConcat("_illustration-halfbody-nobg")  → SaveImage#30
                    ├─→ StringConcat("_bust")                        → SaveImage#32
                    └─→ StringConcat("_illustration-attention-nobg") → SaveImage#43
```

CID 하나만 바꾸면 다섯 장의 파일명이 동시에 갱신된다. `StringConcatenate` 노드는 `ComfyUI-Impact-Pack` 에서 제공.

### 재현 준비물

- **체크포인트 3 종** — `realvisxl_v4`, `animagine-xl-3.1`, `juggernaut_xl_v9`
- **LoRA** — `pixel-art-xl.safetensors`
- **ControlNet** — `xinsir-openpose-sdxl.safetensors`
- **IPAdapter** — `ip-adapter-plus-face_sdxl_vit-h.safetensors` + CLIPVision ViT-H
- **얼굴 감지** — `bbox/face_yolov8m.pt`
- **커스텀 노드** — `comfyui_ipadapter_plus`, `rembg-comfyui-node`, `ComfyUI-Impact-Pack`
- **포즈** — `input/poses/attention.png`

`extra_model_paths.yaml` 에 `D:/AI/models/*` 가 잡혀 있으면 체크포인트 · 컨트롤넷 · LoRA 는 자동으로 따라온다.

## 자주 틀리는 지점

### 증상 — 5 장 사이에 얼굴이 살짝씩 다르다

동일 NPC 인데 G1 실사 · G2 일러 · G4 차렷자세 세 장의 얼굴 인상이 미묘하게 어긋남.

**원인**: G0.5 의 얼굴 임베딩이 G2 · G4 로 제대로 재사용되지 않음. 가장 흔한 건 임베딩 출력 슬롯을 잘못 연결한 경우, 또는 그룹을 쪼개 따로 돌리면서 매 실행마다 임베딩이 새로 계산되는 경우.

**해결**:
1. G2 · G4 각각의 `IPAdapterApply` (또는 `IPAdapterEmbeds`) 에 G0.5 가 만든 얼굴 임베딩 노드 하나의 출력이 **공유**되고 있는지 확인. 두 군데 다 같은 노드 ID 참조.
2. 5 장을 한 그래프 안에서 한 번에 돌린다 — 그룹을 외부 파일로 쪼개지 말 것.

**재발 방지 · 확인 방법**: 실행 직후 `<CID>_realistic-halfbody`, `<CID>_illustration-halfbody`, `<CID>_illustration-attention-nobg` 세 장을 나란히 열어 눈으로 비교. 얼굴이 "어긋난다" 싶으면 바로 임베딩 공유 점검.

### 증상 — G2 일러 결과가 일러가 아니라 실사 느낌이 유지된다

스타일 바뀐 기색이 약하고 G1 과 거의 같아 보임.

**원인**: G2 의 `denoise` 값이 너무 낮음 (예: 0.3). img2img 는 `denoise` 가 재생성 비율이라, 너무 낮으면 원본 유지 쪽으로 가서 스타일 변환이 거의 일어나지 않는다.

**해결**: `denoise` 를 0.5 ~ 0.6 범위로. 이 WF1 의 기본값은 0.55. 구도는 유지되면서 스타일은 확실히 바뀌는 지점.

**재발 방지 · 확인 방법**: 새 LoRA / 새 일러 체크포인트로 바꿀 때는 `denoise` 를 0.45 / 0.55 / 0.65 세 값으로 돌려 톤을 먼저 찍어 보고 값 고정.

### 증상 — G4 차렷자세 결과의 배경 제거가 깔끔하지 않음 / 팔·다리 일부가 잘림

`illustration-attention-nobg` 에서 실루엣 외곽이 너저분하거나 몸 일부가 투명으로 뚫림.

**원인**: `attention.png` 포즈 맵이 프레임 경계에 너무 가까움. OpenPose 가 기대하는 비율(전신이 화면 중앙 60~80% 차지)을 벗어나면 생성 이미지에서 몸이 잘리고, rembg 가 잘린 경계를 배경으로 오인.

**해결**:
1. [open-pose-editor](https://zhuyu1997.github.io/open-pose-editor/) 에서 `attention.png` 를 다시 만들 때 **캔버스 여백을 충분히** (전신이 화면 60% 이하 차지하도록)
2. `EmptyLatentImage` 를 768×1024 가 아니라 1024×1280 등 세로 더 긴 해상도로 바꿔서 머리 위 · 발 아래 여유

**재발 방지 · 확인 방법**: 새 포즈 만들 때 저장 전에 편집기에서 "경계에 닿는 뼈 있는지" 한 번 확인.

### 증상 — 새 NPC 만들 때마다 5 장 톤이 제각각으로 흔들린다

같은 사람으로는 보이지만 실사 · 일러 · 차렷자세의 전체 분위기(조명 · 카메라 각도 · 배경 질감)가 NPC 마다 다르다.

**원인**: 프롬프트를 `char_*` / `comp_*` 로 쪼개지 않고 한 줄로 합쳐 놓음. 그래서 NPC 바꿀 때 같이 `masterpiece, best quality, solo, upper body` 같은 공통 태그도 건드리게 되고, 미묘한 차이가 누적.

**해결**: 프롬프트를 `char_*` / `comp_*` 두 축으로 분리 (이 WF1 의 구조). `comp_*` 는 프로젝트 초기에 한 번 정해 놓고 안 건드리기.

**재발 방지 · 확인 방법**: 새 NPC 만들 때 `char_positive` · `char_negative` **두 자리만 수정**해서 바로 실행. 결과가 이상하면 `comp_*` 도 봤는지 자문.

### 증상 — G4 산출물(768×1024 RGBA) 이 PixelLab API 업로드 시 413 Payload Too Large

WF2 로 넘어가 PixelLab `/image-to-pixelart` 호출에서 413 리턴.

**원인**: PNG base64 인코딩이 PixelLab API 의 4MB 한도 초과. 768×1024 RGBA 는 투명 영역이 많으면 압축되지만 패턴이 복잡하면 4MB 근처로 간다.

**해결** (우선순위):
1. `EmptyLatentImage` 해상도를 512×768 로 축소해 재생성
2. 원본 PNG 를 `pngquant --quality 65-85` 로 양자화해 크기 절반 이하로
3. RGBA 의 투명 영역을 clean-up (알파 0 인 픽셀의 RGB 값을 0 으로 통일)

**재발 방지 · 확인 방법**: WF2 진입 전 `<CID>_illustration-attention-nobg_00001.png` 파일 크기가 3MB 이하인지 미리 확인. 넘으면 재생성.

## 관련

- [[posts/comfyui/2026-04-22-comfyui-환경-셋업|환경 셋업 편]] (1편) — MIOpen · VRAM 관련 참조
- [[posts/comfyui/2026-04-22-comfyui-커스텀-노드-작성|커스텀 노드 작성 편]] (2편) — `StringConcatenate` 같은 유틸 노드가 어떻게 만들어지는지
- [[posts/comfyui/2026-04-22-comfyui-워크플로-설계-원칙|워크플로 설계 원칙 편]] (3편) — 이 그래프가 따르는 설계 규약의 원전
- [open-pose-editor](https://zhuyu1997.github.io/open-pose-editor/) — 차렷자세 포즈 맵 만들기
- [ComfyUI-Impact-Pack](https://github.com/ltdrdata/ComfyUI-Impact-Pack) — `StringConcatenate` 등 유틸 노드 제공
- [comfyui_ipadapter_plus](https://github.com/cubiq/ComfyUI_IPAdapter_plus) — 얼굴 임베딩 재사용을 담당
