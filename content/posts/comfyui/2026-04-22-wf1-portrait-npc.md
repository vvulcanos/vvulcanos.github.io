---
title: "워크플로 해부 · WF1 portrait-npc — 한 캐릭터의 초상화 5종을 순차 생성"
date: 2026-04-22
category: comfyui
tags: [comfyui, workflow, sdxl, ipadapter, controlnet, rembg, lora, portrait-pipeline]
draft: false
---

`portrait-npc.json` 은 파이프라인의 첫 번째 워크플로. 여기서 한 NPC 에 대해 **얼굴이 같은 초상화 5종이 순차적으로 나온다**.

왜 다섯 장씩이나 필요한지부터 짚고 가자. 게임에서 같은 NPC 가 등장하는 자리가 여러 군데다 — 대화 씬의 일러 반신상, 프로필·아바타의 흉상, 인게임 픽셀 스프라이트. 장면마다 스타일은 다른데 "같은 사람"으로 보여야 한다. 얼굴이 조금만 어긋나면 플레이어가 다른 캐릭터로 느낀다. 그렇다고 "일러 반신상 + 흉상 + 픽셀 전신 + 차렷자세 + 실사 ..." 같은 요구를 프롬프트 한 줄로 쑤셔 넣으면 어느 쪽도 제대로 안 나온다. 그래서 **용도별로 렌더는 분리하되, 얼굴 임베딩은 그래프 안에서 공유**하는 구조로 짰다. 한 번 돌리면 다음 5장이 같은 얼굴로 떨어진다.

- `<CID>_realistic-halfbody_00001.png` — 실사 반신상
- `<CID>_illustration-halfbody_00001.png` — 일러 반신상 (대화 씬용, 위 실사에 덧칠)
- `<CID>_illustration-halfbody-nobg_00001.png` — 위의 배경 제거본 (합성용)
- `<CID>_bust_00001.png` — 흉상 1024×1024 크롭 (아바타·프로필)
- `<CID>_illustration-attention-nobg_00001.png` — 차렷자세 전신 배경 제거 (WF2 로 이어져 픽셀 스프라이트가 된다)

`<CID>` 는 사용자가 직접 넣는 `slug_YYYYMMDD_rand6` 형식 문자열. 이 다섯 장뿐 아니라 뒤에 붙는 WF2/3/4 결과물까지 전부 같은 CID 접두어로 묶여서, 파이프라인 전체에서 한 NPC 의 자산이 파일명만 보고도 찾아진다.

## 그래프 구조

ComfyUI 안에서 6개 그룹(G0~G3-2)에 G4가 덧붙어 있다. 굳이 그룹 박스로 쪼개 놓은 건, 실행 순서가 머릿속에서 선형으로 따라가지게 하려는 목적이 크다.

```
G0    로더 (체크포인트 3개, LoRA, ControlNet, IPAdapter, VAE)
G0.5  얼굴 YOLO → 얼굴 crop → IPAdapter 얼굴 임베딩
G1    RealVis 로 실사 반신상 (768×1024)
G2    G1 결과를 Juggernaut + pixel-art-xl 로 img2img (denoise 0.55)
G3    G2 일러 결과를 rembg 로 배경 제거
G3-2  흉상 1024×1024 크롭
G4    Animagine + OpenPose + IPAdapter(G2 얼굴 재사용) → 차렷자세 → rembg
```

G1이 먼저 돌아야 G0.5의 얼굴 임베딩이 G2/G4로 흘러갈 수 있고, G3는 G2 결과를 받고, G3-2는 G3 결과를 받는다. 그래서 선형.

## G0 — 체크포인트 3개를 한 그래프에 올려놓는 이유

`realvisxl_v4`(실사), `animagine-xl-3.1`(일러), `juggernaut_xl_v9`(img2img 베이스) — 세 개가 동시에 로드된다. 여기에 `pixel-art-xl` LoRA가 Juggernaut 위에 얹힌다.

RX 9070 XT 16GB에서 이건 빡빡하다. ComfyUI가 그룹 단위로 모델을 알아서 언로드·재로드하면서 도는데, 세 체크포인트가 한꺼번에 VRAM에 올라가는 순간은 없다. 첫 실행에 MIOpen 튜닝까지 겹치면 10~20분 나오기도 한다 ([[2026-04-21-comfyui-소개-amd-zluda-설치|ZLUDA 설치 편]] 참고). 캐시가 달궈진 뒤에는 1~3분 정도.

## G0.5 — 얼굴을 따로 뽑는 이유

`face_yolov8m.pt` 로 G1 실사 반신상에서 얼굴 박스를 찾아 crop 하고, 그걸 `IPAdapterEncoder` 에 넣어 얼굴 임베딩을 만든다. 이 임베딩을 **G2와 G4가 둘 다 재사용**한다.

덕분에 G1 실사 / G2 일러 / G4 차렷자세 — 세 장이 체크포인트도 프롬프트도 포즈도 다 다른데 얼굴은 같은 사람으로 찍힌다. 5장 사이의 동일성이 여기서 나온다.

## G1 / G2 / G4 샘플러 설정

| 그룹 | Steps | CFG | Sampler | Scheduler | Denoise |
|---|---|---|---|---|---|
| G1 실사 | 30 | 7.0 | dpmpp_2m_sde | karras | 1.0 |
| G2 일러 img2img | 28 | 6.5 | euler_ancestral | normal | **0.55** |
| G4 차렷자세 | 30 | 7.0 | euler_ancestral | normal | 1.0 |

G2의 `denoise=0.55` 가 핵심. G1 실사 반신상의 VAEDecode 결과를 다시 VAEEncode + ImageScale 거쳐 latent로 바꿔 넣고, 그걸 기반으로 55%만 재생성한다. 그래서 구도·옷·자세는 유지되고 스타일만 일러로 바뀐다.

순수 일러로 바로 가도 되는 거 아닌가 싶긴 한데, 일러 체크포인트에 직접 "한복 입은 조선 여인 정면 반신상" 같은 프롬프트 넣으면 옷 구조나 얼굴 비율이 매번 흔들려서 G1 실사 단계를 구조 고정용으로 끼워 넣은 거다.

## Advisory #15 — 프롬프트를 네 토막으로 쪼갬

G1·G2·G4 모두 positive/negative 를 각각 두 조각씩으로 더 쪼개서 넣는다.

- `char_*` — 캐릭터 고유 (매번 수정). 예: `1girl, silver hair, stern expression, blue eyes`
- `comp_*` — 구도·스타일·품질 (거의 안 건드림). 예: `masterpiece, best quality, solo, upper body, centered, looking at viewer`

두 조각을 `ConditioningConcat` 으로 합쳐서 KSampler.positive/negative 로 보낸다. 이래서 CLIPTextEncode가 워크플로에 12개, ConditioningConcat이 6개나 박혀 있다.

분리한 이유는 단순하다 — 새 NPC 만들 때 `char_*` 두 군데만 건드리면 5장 톤이 저절로 일관된다. 프로젝트 공통 스타일(`comp_*`)은 한 번 정해 놓고 안 건드리는 게 이상적.

## G4 — 차렷자세 전신, 그리고 픽셀 사이드로 이어지는 접점

실사 반신상·일러 반신상·흉상까지는 "화면 안에서 보여주는 용도"고, G4 는 여기서 조금 결이 다르다. 전신 차렷자세(정면, 팔 차렷) 로 뽑아 배경까지 지워 놓는다. 이 포즈·구도가 WF2 의 `/image-to-pixelart` 로 들어갈 때 실루엣이 가장 깔끔하게 픽셀화된다 — 팔·다리가 몸통에 붙어 있는 자세라 64×64 그리드에서 형태가 뭉치지 않는다.

G4 블록의 노드 ID를 보면 33–43 + 86–89 로 두 덩이에 나뉘어 있다. 왜 중간에 숫자가 끊기냐면, 처음에 G4를 설계할 때 ID 33–42 로 열 개를 예약해 뒀는데, 그 뒤에 Advisory #15 적용하면서 CLIPTextEncode 두 개 + ConditioningConcat 두 개가 더 필요해졌다. 그 시점에는 이미 그래프 다른 쪽에 79–85 같은 번호가 점유된 상태라, 비어 있던 86–89 를 땡겨 썼다. 편집 스크립트로 워크플로 건드릴 때 ID 충돌 안 나게 하려고 빈 구간을 먼저 검사하는 단계가 Plan 0 Task 1 에 들어 있다.

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

`attention.png` 는 [open-pose-editor](https://zhuyu1997.github.io/open-pose-editor/) 에서 만든 정면 차렷자세 맵. 그룹 박스 제목 옆에 편집기 URL 도 주석으로 박아 뒀으니, 포즈 바꿀 일 있으면 거기서 고쳐서 교체하면 된다.

## CID 파일명 주입 트릭

5개 SaveImage 모두 `filename_prefix` 위젯을 Convert Widget to Input 으로 뒤집어 놓고, 그 앞에 `StringConcatenate` 노드가 하나씩 붙어 있다.

```
PrimitiveNode(CID) ─┬─→ StringConcat("_realistic-halfbody")         → SaveImage#12
                    ├─→ StringConcat("_illustration-halfbody")       → SaveImage#28
                    ├─→ StringConcat("_illustration-halfbody-nobg")  → SaveImage#30
                    ├─→ StringConcat("_bust")                        → SaveImage#32
                    └─→ StringConcat("_illustration-attention-nobg") → SaveImage#43
```

CID 하나만 바꾸면 다섯 장의 파일명이 동시에 갱신된다. 이 StringConcatenate 노드는 ComfyUI-Impact-Pack 에서 제공하는 것을 쓴다.

## 재현 준비물

- 체크포인트 3종 — `realvisxl_v4`, `animagine-xl-3.1`, `juggernaut_xl_v9`
- LoRA — `pixel-art-xl.safetensors`
- ControlNet — `xinsir-openpose-sdxl.safetensors`
- IPAdapter — `ip-adapter-plus-face_sdxl_vit-h.safetensors` + CLIPVision ViT-H
- 얼굴 감지 — `bbox/face_yolov8m.pt`
- 커스텀 노드 — `comfyui_ipadapter_plus`, `rembg-comfyui-node`, `ComfyUI-Impact-Pack`
- 포즈 — `input/poses/attention.png`

`extra_model_paths.yaml` 에 `D:/AI/models/*` 만 잡혀 있으면 체크포인트·컨트롤넷·LoRA는 알아서 따라온다.

## 왜 다섯 장을 한 그래프에 몰아넣었나

일관성과 품질, 두 가지 다.

**일관성** — G0.5 의 얼굴 임베딩이 G2 와 G4 에서 같은 값으로 재사용돼야 얼굴이 안 흔들린다. 워크플로를 쪼개서 "일러 전용", "흉상 전용", "차렷자세 전용" 따로 돌리면 매 실행마다 얼굴 임베딩을 다시 만들어야 하고, 그 과정에서 얼굴 YOLO 가 잡는 영역이 미세하게 달라지면서 임베딩 값이 흔들린다. 같은 그래프 안에 두면 한 번 만든 임베딩이 그대로 공급된다.

**품질** — 하나의 프롬프트·하나의 KSampler 에 "실사/일러/흉상/차렷자세" 네 요구를 섞으면 어느 쪽도 깔끔하게 안 찍힌다. 그래서 G1(실사, 구도 잡기용) → G2(일러 img2img, 스타일 입힘) → G3(배경 제거) → G3-2(흉상 크롭) → G4(다른 포즈·같은 얼굴) 순으로 **단계마다 하나의 목표만** 주고 이어 붙였다. 각 단계가 이전 단계의 결과를 재료 삼아 얹는 구조라 매 단계가 덜 힘들고 품질이 유지된다.

이걸 파이프라인 전체 규모로 뒤집어 보면, WF1 → WF2 → WF3 → WF4 로 쪼개 놓은 이유도 같다. 일러 → 픽셀 → 4방향 → 애니메이션 을 한 그래프에 욱여넣으면 API 호출 순서·재시도·중간 결과 확인이 지옥이 되고, 특히 픽셀화·회전·애니메이션은 PixelLab API 를 여러 번 호출해야 하는데 실패가 겹칠 때 복구가 불가능해진다. 단계별 파일로 산출물을 박제하면서 이어가는 쪽이 안정적이다. 이 쪼개는 기준 얘기는 [[2026-04-22-workflow-json-구조와-설계|여기]] 에서 따로 정리했다.

## 다음 단계

WF1 한 바퀴 돌리고 나면 `<CID>_illustration-attention-nobg_00001.png` 가 WF2 의 입력으로 자동으로 사용할 수 있는 파일명 규약에 맞춰 저장돼 있다. WF2 에서 같은 CID 를 적어 넣으면 이어진다. 캐릭터 자체가 마음에 안 들면 (얼굴 인상·옷 색이 NPC 설정과 어긋나는 경우) CID 를 새로 발급해서 WF1 부터 다시 — 이 경우 5장이 전부 재생성된다.

한 가지 열려 있는 확인 항목. G4 산출물이 768×1024 RGBA 라, base64 인코딩해서 PixelLab API 로 업로드할 때 4MB 한도 안에 들어가는지 실측이 필요하다. 여유 있는지는 Plan 0 Task 7 의 probe 스크립트로 잰다. 초과하면 1순위 대응이 latent 를 512×768 로 축소, 그다음이 PNG optimize + 256색 양자화.

## 관련 글

- [[2026-04-22-wf2-pixel-char|WF2 · pixel-char — PixelLab image-to-pixelart 호출의 실제]]
- [[2026-04-22-workflow-json-구조와-설계|워크플로 JSON 구조와 설계]]
- [[2026-04-21-comfyui-소개-amd-zluda-설치|ComfyUI 소개 & AMD + ZLUDA 설치]]
