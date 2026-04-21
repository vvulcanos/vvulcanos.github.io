---
title: "워크플로 해부 · pixel-mystery-character v10 — 초상화 + 4방향 turnaround 한 번에"
date: 2026-04-22
category: comfyui
tags: [comfyui, workflow, stable-diffusion, rembg, turnaround, portrait-pipeline]
draft: false
---

이 글은 현재 볼트에 저장되어 있는 `pixel-mystery-character.json` 워크플로를 뜯어본 기록이다. 하나의 그래프 안에서 **초상화 하나 + 4방향 캐릭터 시트**를 동시에 뽑는 v10 버전으로, 32개 노드로 구성되어 있다. 이후 리디자인된 4-워크플로 파이프라인(WF1~WF4b)과 비교할 때 "이전 세대"에 해당한다.

## 목표

NPC 하나당 다음 5장이 나오는 것이 목표다.

- `portrait_XXXXX.png` — 512×512 전신 상반신 초상화 (배경 제거)
- `char_front_XXXXX.png` / `char_right_XXXXX.png` / `char_left_XXXXX.png` / `char_back_XXXXX.png` — 각 64×128 픽셀 타일 (4방향, 배경 제거)

추가로 디버그용 중간 결과물 `refsheet_preview_XXXXX.png`(1024×512 캐릭터 시트 원본)도 함께 저장된다.

## 두 개의 병렬 스테이지

구조는 단순하다. 같은 체크포인트(`animagine-xl-3.1.safetensors`)를 공유하는 **두 개의 독립 KSampler 분기**가 병렬로 돈다.

### Stage 1 — 초상화 파이프라인

```text
CheckpointLoader → CLIPTextEncode(+)→ KSampler(seed=42, steps=30, cfg=7.0, euler/normal)
                → CLIPTextEncode(-)↗            ↓
EmptyLatent(512×512) ────────────────────────→ VAEDecode → rembg → SaveImage("portrait_")
```

핵심 설정:

- **프롬프트 주제:** `1girl, joseon dynasty noblewoman, white hanbok, gold hairpin, ..., stardew valley portrait style, warm candlelight, painterly`
- **Negative:** `lowres, bad anatomy, blurry, 3d, realistic, photo, multiple characters, text, frame, border`
- **샘플러/스케줄러:** `euler` + `normal`, cfg 7.0, 30 steps — SDXL 캐릭터 초상화의 비교적 얌전한 프리셋

### Stage 2 — 4방향 turnaround 파이프라인

```text
KSampler(seed=42, steps=40, cfg=10.0, euler_a/normal)
   ↑
EmptyLatent(1024×512)
   ↓
VAEDecode ──→ SaveImage("refsheet_preview_")
          ├→ ImageCrop(256×256, x=0)   → rembg → ImageScaleBy(nearest-exact, 0.1875 → ???)
          ├→ ImageCrop(256×256, x=256) → rembg → ImageScaleBy
          ├→ ImageCrop(256×256, x=512) → rembg → ImageScaleBy
          └→ ImageCrop(256×256, x=768) → rembg → ImageScaleBy
                                                   → SaveImage("char_{front,right,left,back}_")
```

프롬프트가 Stage 1과 전혀 다르다.

- **Positive:** `multiple views, character sheet, turnaround, ..., chibi, full body, (facing viewer:1.4), (from side:1.3), (from behind:1.4), 4 views in a horizontal row, each character centered in square panel, simple background, grey background, flat color, flat lighting`
- **Negative:** `..., gradient background, shadow, drop shadow, ground shadow, extra limbs, deformed, (3/4 view:1.2), (diagonal:1.2), multiple people`

특히 negative 쪽의 `(3/4 view:1.2)` 와 `(diagonal:1.2)` 가 의도적이다 — 정면/좌/우/후면 네 방향이 정확히 떨어지도록 대각 뷰를 억제한다. positive 쪽의 `(from side:1.3)` / `(from behind:1.4)` 가중치도 방향 포커싱을 강제한다.

샘플러 설정도 Stage 1과 다르다:

- `euler_a` + `normal`, cfg **10.0**, 40 steps — 구조적 일관성을 더 요구하는 구성(방향 배치를 지키려면 더 강한 가이던스가 필요)
- seed는 Stage 1과 동일한 42지만, `control_after_generate: randomize` 로 매 실행마다 자동 랜덤화된다

## 크롭 좌표의 규칙성

1024×512 이미지에서 4방향을 잘라내는 `ImageCrop` 노드 4개의 좌표는 아래와 같이 정확히 256-pixel 간격으로 배치되어 있다.

| 방향 | 노드 ID | 크기 | x | y |
|---|---|---|---|---|
| front | 17 | 256×256 | 0 | 0 |
| right | 18 | 256×256 | 256 | 0 |
| left | 19 | 256×256 | 512 | 0 |
| back | 20 | 256×256 | 768 | 0 |

노트 노드(id=1)에 있는 주석과 정확히 일치한다. **순서가 `front → right → left → back`** 이라는 것이 중요한데, 이는 positive 프롬프트 안에서 네 방향 키워드를 나열한 순서이기도 하다 (확률적으로 생성된 시트가 이 순서대로 나오도록 유도한 프롬프팅).

## 배경 제거(rembg) 다섯 번

워크플로 안에 rembg 노드가 **총 5개** 들어있다. 초상화 1개 + 4방향 타일 각각 1개씩. 이게 이 워크플로의 가장 무거운 지점이다 — rembg는 CPU 기반이고 매 호출마다 u2net 모델을 메모리에 올린다(로컬 포크 기준). 5번 호출 = 5번 모델 로딩. 로컬 포크에서 모델을 전역 캐시하는 패치가 없다면 이 부분이 전체 시간의 절반을 먹는다.

## 다운스케일 비율의 의미

`ImageScaleBy` 의 비율이 `0.1875` 로 고정되어 있다. 256×256 이미지를 0.1875 배 하면 **48×48** 이 된다 — 그런데 노트에는 "각 256x512" → "0.25 스케일 → 64x128" 이라고 쓰여있다. 노트 주석과 실제 파일 값 사이에 불일치가 있다 (노트 = 설계 의도, 실제 값 = 조정 후 현재 상태). 이 부분은 다음 리뷰 때 반드시 검증할 지점이다.

최종 출력이 64×128 이어야 게임 안에서 캐릭터 타일로 바로 쓸 수 있으니, 파이프라인이 실제로 원하는 크기를 내고 있는지는 실행 후 PNG 해상도를 직접 확인해야 한다.

## 동시에 두 KSampler를 돌리는 비용

Stage 1과 Stage 2가 **완전히 독립적인 분기**이지만, 그래프가 직렬화되면서 ComfyUI는 이들을 순차적으로 실행한다 (VRAM 관리 때문). 체감 성능:

- Stage 1 (512×512, 30 steps, cfg 7.0) — AMD RX 9070 XT + ZLUDA 환경에서 빠름
- Stage 2 (1024×512, 40 steps, cfg 10.0) — latent 면적 2배 + steps 1.33배 + cfg 덕분에 한 번 더 무거움
- rembg 5회 — 외부 요인(u2net 로딩 반복)

한 번 실행에 1~2분대. MIOpen 캐시 웜업이 끝난 뒤의 값이다.

## 왜 이 워크플로가 "이전 세대" 인가

같은 볼트의 `pixel-char.json`(WF2)과 함께 두면 대조가 명확해진다.

| 항목 | pixel-mystery-character v10 | 새 파이프라인 WF1~WF4b |
|---|---|---|
| 워크플로 개수 | 1개 (monolithic) | 4개 (WF1~WF4b) |
| 베이스 모델 | Stable Diffusion (animagine-xl-3.1) | PixelLab API (illustration-attention-nobg) |
| 방향 생성 | 한 장에 4방향을 그림 (프롬프트 가중치) | WF3에서 rotate API 별도 호출 |
| 픽셀화 | 직접 다운스케일 (nearest-exact) | WF2에서 image-to-pixelart 전용 API |
| 제어 가능성 | 프롬프트 / seed / cfg | 각 API 엔드포인트 파라미터 (더 세밀) |
| 배경 제거 | rembg × 5 | WF1에서 API 응답에 이미 포함 |
| 재현성 | seed에 의존 | CID(Character ID) 기반 영속 |

요약하면, 이 워크플로는 "SDXL만으로 최대한 짜낸 버전" 이고, 다음 세대는 아예 **단계별 API 분리**로 접근이 바뀐다. 하지만 이 워크플로는 여전히 참고값 — 같은 체크포인트로 풀 로컬 실행이 가능하다는 장점이 있고, 새 파이프라인의 품질 비교 기준선으로 쓰기 좋다.

## 재현하려면 필요한 것

- **체크포인트:** `animagine-xl-3.1.safetensors` — `extra_model_paths.yaml` 에 매핑된 `checkpoints` 경로 밑
- **커스텀 노드:** `rembg-comfyui-node` — ComfyUI-Manager에서 설치 (로컬 포크는 u2net 전용)
- **기본 노드:** `CheckpointLoaderSimple`, `CLIPTextEncode`, `KSampler`, `EmptyLatentImage`, `VAEDecode`, `ImageCrop`, `ImageScaleBy`, `SaveImage`

파이프라인 워크플로 파일 자체는 `pixel-mystery-character.json` 이름 그대로 `ComfyUI/user/default/workflows/` 에 들어있다.

## 관련 글

- [[2026-04-22-workflow-json-구조와-설계|워크플로 JSON 구조와 설계]] — 왜 하나로 묶지 않고 잘게 나누는지
- [[2026-04-22-wf2-pixel-char|WF2 · pixel-char — PixelLab image-to-pixelart 호출의 실제]]
- [[2026-04-21-comfyui-소개-amd-zluda-설치|ComfyUI 소개 & AMD + ZLUDA 설치]]
