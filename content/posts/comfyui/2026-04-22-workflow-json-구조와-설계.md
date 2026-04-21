---
title: "ComfyUI 워크플로 — JSON 구조, 저장·공유, 그리고 쪼개기"
date: 2026-04-22
category: comfyui
tags: [comfyui, 워크플로, json, 파이프라인, 설계]
draft: false
---

# ComfyUI 워크플로 — JSON 구조, 저장·공유, 그리고 쪼개기

ComfyUI 를 써본 사람이 가장 먼저 마주치는 "어떻게 이걸 관리하지?" 라는 질문. 화면에 펼친 노드 수십 개와 그 연결이 화면을 닫는 순간 사라지면 곤란하다. 다행히 ComfyUI 의 워크플로는 **그 자체가 JSON 데이터** 라, 저장·공유·버전 관리·자동화 전부 일반 파일처럼 다룰 수 있다.

## 워크플로 = 그래프 = JSON

Blender 의 셰이더, Unreal 의 블루프린트와 마찬가지로 ComfyUI 의 "워크플로" 는 화면에 보이는 노드와 연결선 전부를 직렬화한 구조다. UI 에서 `Save (API Format)` 또는 `Save` 를 누르면 바로 `.json` 파일로 덤프된다.

즉:

- 화면에 그린 파이프라인 = JSON
- JSON 을 드래그&드롭하면 = 그 파이프라인이 화면에 복원
- JSON 이 같으면 = 누가 돌려도 **같은 파이프라인 구조** 가 재현 (모델·입력 동일하면 결과까지 거의 동일)

프롬프트 문자열만 공유하는 문화권(예: Stable Diffusion WebUI 의 PNG Info 복사) 과 비교하면 재현 가능성이 한 차원 높다.

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

핵심 문법:

- **키** (`"3"`, `"4"`) 는 노드 ID. UI 위의 고유 번호
- **`class_type`** — 어떤 ComfyUI 노드 클래스인지 (`KSampler`, `CLIPTextEncode`, 커스텀 노드 포함)
- **`inputs`** — 각 입력의 값. 리터럴(`42`, `"euler"`) 이거나 **`[다른노드ID, 출력인덱스]`** 형태의 링크
- 리턴은 연결 순서대로 평가 — DAG 가 유효해야 함

Editor 에서 `Save` 로 저장되는 형식은 추가로 `nodes`, `links`, 위젯 값, 시각 배치(`pos`, `size`) 까지 포함한다. **API 형식은 실행용**, **Save 형식은 편집용** — 둘을 구분해둘 것.

## 저장·로드·공유

UI 오른쪽 메뉴의 버튼 대응:

- `Save` — 편집 포맷 JSON 다운로드 (배치 포함)
- `Save (API Format)` — 실행 포맷만 (자동화·프로그램 연계용)
- `Load` — JSON 파일을 불러와 현재 그래프 대체
- 드래그&드롭 — 위와 동일

**깨알 팁 1 — PNG 메타데이터 임베딩**
ComfyUI 가 출력한 PNG 에는 **그 이미지를 만든 워크플로 JSON 이 임베드** 되어 있다. UI 에 PNG 를 드래그하면 원래 그래프가 복원된다. 별도로 JSON 을 건네줄 필요 없이 이미지 파일 하나로 구조·파라미터·모델 조합까지 공유 가능.

단점은 **개인 모델 경로나 실험 중 프롬프트가 PNG 에 박제된다는 것**. 공개 배포 전에는 [`exiftool`](https://exiftool.org/) 같은 도구로 메타데이터를 정리하는 게 안전하다.

```bash
exiftool -all= -overwrite_original public_sample.png
```

**깨알 팁 2 — `workflow` 디렉터리 관리**
ComfyUI 는 `user/default/workflows/` 아래에 저장된 JSON 들을 자동으로 "My Workflows" 메뉴에 띄운다. 이 폴더를 **별도 git repo 로 만들어** 볼트처럼 관리하면 버전 관리·동기화가 즉시 해결된다.

```
user/default/workflows/
├── _drafts/
├── text-to-image-basic.json
├── image-to-image-refine.json
└── pipelines/
    ├── portrait-step-1-base.json
    └── portrait-step-2-pixelart.json
```

## 워크플로를 쪼개는 이유

한 그래프에 전부 몰아넣을 수도 있지만, 규모가 커지면 아래 문제가 쌓인다.

1. **디버깅 지옥** — 50개 이상 노드가 뒤엉키면 한 파라미터 튜닝이 전체 재실행을 요구
2. **실패 비용** — API 호출·중대한 연산이 뒤쪽에 있으면 앞쪽 실패·재실행으로 크레딧·시간 낭비
3. **재사용 불가** — 같은 "기반 이미지 생성" 단계를 다른 용도로도 쓰고 싶을 때 복붙
4. **협업 난이도** — 한 JSON 을 둘이 편집하면 충돌
5. **재현성 약화** — 한 번에 너무 많은 것이 엮여 있어, 문제가 터졌을 때 어느 노드가 범인인지 좁히기 어려움

**해법은 파이프라인을 의미 단위로 쪼개는 것.** 각 단계는 입력과 출력이 명확한 독립 JSON 이고, 중간 산출물은 파일시스템 (또는 DB·S3) 으로 넘긴다.

예를 들어 "포트레이트 아이콘 세트 생성" 같은 작업을 이렇게 나눌 수 있다.

```
WF1  베이스 이미지 생성         input: 프롬프트         output: base.png
WF2  픽셀아트 변환              input: base.png         output: pixel_64.png
WF3  8방향 로테이션              input: pixel_64.png    output: rot_*.png × 8
WF4  애니메이션 GIF 합성          input: rot_*.png × 8   output: anim.gif
```

각 단계가 **실행·평가·재실행** 이 독립이 된다. WF2 결과가 마음에 안 들면 WF1 을 다시 돌릴 필요 없이 WF2 만 파라미터 바꿔 재시도하면 된다.

중간 산출물에는 **공통 ID** (예: Character ID = timestamp 기반 짧은 문자열) 를 부여하고, 파일명에 박아두면 여러 실행이 뒤섞이지 않는다.

```
output/
├── cid-20260422-0001_base.png
├── cid-20260422-0001_pixel_64.png
├── cid-20260422-0001_rot_east.png
├── cid-20260422-0001_rot_north.png
└── cid-20260422-0001_anim.gif
```

## 쪼갤 때의 경계 선택 기준

노드를 자를 지점을 고를 때 쓰는 경험칙:

- **파일로 넘길 수 있는 곳** — 이미지·벡터·텍스트처럼 디스크에 올릴 수 있는 타입이면 자연스러운 경계
- **크레딧이나 시간을 잡아먹는 곳의 앞뒤** — 외부 API 호출이 포함된 단계는 반드시 독립
- **자주 튜닝되는 파라미터가 모이는 곳의 앞뒤** — 그 파라미터만 건드리고 싶은데 전체가 같이 재실행되면 비효율

반대로 **같이 잘라선 안 되는** 것:

- Latent space 에서만 유효한 중간 값 (파일로 저장하려면 VAE 디코드 필요 — 비용 발생)
- 랜덤 시드에 민감한 연속 노드 (예: 샘플링 → 업스케일) — 분리하면 재현이 틀어질 수 있음

## Git 친화적으로 만들기

쪼개기를 전제로 JSON 을 repo 에 커밋한다면 몇 가지 하이진이 필요하다.

1. **안정된 노드 ID** — UI 에서 저장할 때마다 ID 순서가 바뀌면 diff 가 의미 없어진다. 수동으로 ID 를 박거나, 저장 후 스크립트로 정규화
2. **절대 경로 제거** — 체크포인트 이름은 상대·이식 가능한 값으로. 로컬 머신 경로가 박히면 공유 시 깨짐
3. **시크릿 분리** — 토큰이 필요한 커스텀 노드는 위젯 값에 키를 직접 넣지 말고 환경변수·별도 파일 참조
4. **예쁘게 포매팅** — JSON 을 정렬·들여쓰기 고정 (`jq --sort-keys . > fmt.json`) 하면 diff 가 의미 있게 나옴
5. **드래프트 디렉터리** — 실험 중인 워크플로는 `_drafts/` 에, 확정된 것만 루트 바로 아래에

## 정리

- 워크플로 = JSON = 재현 단위. 다루기가 "파일 하나" 수준으로 단순
- API 형식(실행용)과 Save 형식(편집용) 둘이 있다
- PNG 에 임베드되는 메타데이터는 **공유에 편리, 공개에 주의**
- 그래프가 커지기 시작하면 **의미 단위로 쪼개기** — 파일 경계 + CID 규약
- Git 에 올릴 땐 ID 안정화 · 시크릿 분리 · 포맷 정규화

다음 글은 이 쪼개기 원칙을 실전에 적용한 "포트레이트 파이프라인" 사례를 다룰 예정. 왜 4개로 나눴고 각 단계의 입출력이 무엇인지.
