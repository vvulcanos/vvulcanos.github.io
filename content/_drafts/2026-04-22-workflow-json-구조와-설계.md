---
title: "ComfyUI 워크플로 — JSON 구조, 저장·공유, 그리고 쪼개기"
date: 2026-04-22
category: comfyui
tags: [comfyui, 워크플로, json, 파이프라인, 설계]
draft: false
---

ComfyUI 를 써 본 사람이 조만간 마주치는 질문 — "이걸 어떻게 관리하지?" 화면에 펼친 노드 수십 개와 그 연결이 창을 닫는 순간 사라지면 곤란하다. 다행히 ComfyUI 의 워크플로는 그 자체가 JSON 데이터라, 저장·공유·버전 관리·자동화를 전부 일반 파일처럼 다룰 수 있다.

## 워크플로 = 그래프 = JSON

Blender 의 셰이더나 Unreal 의 블루프린트처럼, ComfyUI 워크플로는 화면에 보이는 노드와 연결선 전부를 직렬화한 구조다. UI 에서 `Save (API Format)` 또는 `Save` 를 누르면 바로 `.json` 으로 떨어진다. 화면에 그린 파이프라인이 곧 JSON 이고, JSON 을 다시 드래그&드롭하면 그대로 복원된다. JSON 이 같으면 누가 돌려도 같은 파이프라인 구조가 재현되고, 모델·입력까지 동일하면 결과도 거의 같게 나온다.

프롬프트 문자열만 주고받는 문화(예: Stable Diffusion WebUI 의 PNG Info 복사)와 비교하면 재현 가능성이 한 차원 위다.

## JSON 구조 개요

API Format (= `/prompt` 엔드포인트로 POST 되는 형식) 의 최소 형태는 이렇다.

```json
{
  "3": {
    "inputs": {
      "seed": 42,
      "steps": 20,
      "cfg": 7.0,
      "sampler_name": "euler",
      "scheduler": "normal",
      "denoise": 1.0,
      "model": ["4", 0],
      "positive": ["6", 0],
      "negative": ["7", 0],
      "latent_image": ["5", 0]
    },
    "class_type": "KSampler"
  },
  "4": {
    "inputs": { "ckpt_name": "sd_xl_base_1.0.safetensors" },
    "class_type": "CheckpointLoaderSimple"
  },
  "5": {
    "inputs": { "width": 1024, "height": 1024, "batch_size": 1 },
    "class_type": "EmptyLatentImage"
  }
}
```

키 (`"3"`, `"4"`) 가 노드 ID — UI 에 박힌 고유 번호. `class_type` 이 ComfyUI 노드 클래스 이름이고, `inputs` 의 각 값은 리터럴 (`42`, `"euler"`) 이거나 `[다른노드ID, 출력인덱스]` 꼴의 링크다. 리턴은 연결 순서대로 평가되니 DAG 가 유효해야 한다.

Editor 에서 `Save` 로 저장되는 쪽은 추가로 `nodes`, `links`, 위젯 값, 시각 배치(`pos`, `size`) 까지 포함한다. API 형식은 실행용, Save 형식은 편집용 — 둘을 구분해 두는 게 좋다.

## 저장·로드·공유

UI 오른쪽 메뉴 버튼 대응. `Save` 는 편집 포맷 JSON 다운로드(배치 포함), `Save (API Format)` 은 실행 포맷만, `Load` 는 JSON 파일을 불러와 현재 그래프를 대체. 드래그&드롭도 같은 효과.

### PNG 안에 들어가는 워크플로

ComfyUI 가 출력한 PNG 에는 그 이미지를 만든 워크플로 JSON 이 임베드돼 있다. UI 에 PNG 를 드래그하면 원래 그래프가 복원된다. JSON 을 따로 건네줄 필요 없이 이미지 파일 하나로 구조·파라미터·모델 조합까지 공유 가능하다는 뜻.

단점은 개인 모델 경로나 실험 중 프롬프트가 PNG 에 그대로 박힌다는 것. 공개 배포 전엔 [`exiftool`](https://exiftool.org/) 로 메타데이터를 정리하는 게 안전하다.

```bash
exiftool -all= -overwrite_original public_sample.png
```

### 워크플로 폴더 자체를 git 으로

ComfyUI 는 `user/default/workflows/` 아래 JSON 들을 자동으로 "My Workflows" 메뉴에 띄운다. 이 폴더를 그냥 별도 git repo 로 만들면 버전 관리·동기화가 한 번에 해결된다.

```
user/default/workflows/
├── _drafts/
├── text-to-image-basic.json
├── image-to-image-refine.json
└── pipelines/
    ├── portrait-step-1-base.json
    └── portrait-step-2-pixelart.json
```

## 워크플로를 왜 쪼개나

한 그래프에 다 몰아넣어도 돌긴 돈다. 규모가 커지면 다음 문제가 하나씩 쌓인다.

노드 50개 넘는 그래프에서 한 파라미터 튜닝이 전체 재실행을 요구하고, 외부 API 호출·중대한 연산이 뒤쪽에 있으면 앞쪽에서 실패·재시도할 때마다 크레딧·시간이 낭비된다. 같은 "기반 이미지 생성" 단계를 다른 용도로도 쓰고 싶을 때 복붙이 시작되고, 한 JSON 을 둘이 편집하면 깃 충돌이 난다. 그리고 한 그래프가 너무 많은 걸 엮고 있으면 문제가 터졌을 때 어느 노드가 범인인지 좁히는 데만 한참 걸린다.

해결은 파이프라인을 의미 단위로 쪼개는 것. 각 단계가 입력·출력이 분명한 독립 JSON 이 되고, 중간 산출물은 파일시스템(또는 DB·S3)으로 넘긴다. 예를 들어 "포트레이트 아이콘 세트" 작업이라면 다음처럼.

```
WF1  베이스 이미지 생성       input: 프롬프트       output: base.png
WF2  픽셀아트 변환            input: base.png       output: pixel_64.png
WF3  8방향 로테이션           input: pixel_64.png   output: rot_*.png × 8
WF4  애니메이션 GIF 합성       input: rot_*.png × 8  output: anim.gif
```

각 단계가 실행·평가·재실행이 독립이라, WF2 결과가 마음에 안 들면 WF1 을 다시 돌릴 필요 없이 WF2 만 파라미터 바꿔 재시도하면 된다.

중간 산출물에는 공통 ID(예: Character ID = timestamp 기반 짧은 문자열)를 부여하고, 파일명에 박아 두면 여러 실행이 뒤섞이지 않는다.

```
output/
├── cid-20260422-0001_base.png
├── cid-20260422-0001_pixel_64.png
├── cid-20260422-0001_rot_east.png
├── cid-20260422-0001_rot_north.png
└── cid-20260422-0001_anim.gif
```

## 쪼갤 때의 경계 기준

자를 지점을 고를 때 쓰는 경험칙. 파일로 넘길 수 있는 곳(이미지·벡터·텍스트처럼 디스크에 올릴 수 있는 타입)이 자연스러운 경계다. 외부 API 호출이 껴 있는 단계는 반드시 독립 — 재시도·복구를 쉽게 하려고. 자주 튜닝되는 파라미터가 몰리는 지점 앞뒤도 좋은 경계다. 그 파라미터만 건드리고 싶은데 전체가 같이 재실행되면 답답해진다.

반대로 같이 자르면 안 되는 지점도 있다. Latent space 에서만 유효한 중간 값(파일로 저장하려면 VAE 디코드 필요 — 비용 발생), 그리고 랜덤 시드에 민감한 연속 노드(예: 샘플링 → 업스케일). 후자는 분리하면 재현성이 틀어질 수 있다.

## Git 올릴 때 신경 쓸 것

쪼개기를 전제로 JSON 을 repo 에 커밋한다면 몇 가지 하이진이 필요하다.

노드 ID 가 UI 저장할 때마다 재배치되면 diff 가 의미 없어지니, 수동으로 ID 를 박아 두거나 저장 후 스크립트로 정규화. 체크포인트 이름은 상대·이식 가능한 값으로 유지(로컬 머신 경로가 박히면 공유 시 깨짐). 토큰이 필요한 커스텀 노드는 위젯 값에 키를 직접 넣지 말고 환경변수나 별도 파일로 분리. JSON 자체는 `jq --sort-keys . > fmt.json` 같은 식으로 정렬·들여쓰기를 고정해 두면 diff 가 읽힌다. 실험 중 워크플로는 `_drafts/` 에 두고, 확정된 것만 루트에.

---

이 쪼개기를 실전에 적용한 예가 지금 진행 중인 포트레이트 파이프라인이다. WF1 이 무얼 하는지는 [[2026-04-22-wf1-portrait-npc|여기]], WF2 쪽은 [[2026-04-22-wf2-pixel-char|여기]].
