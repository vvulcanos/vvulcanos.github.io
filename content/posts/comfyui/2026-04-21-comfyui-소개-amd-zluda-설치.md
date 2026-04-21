---
title: "ComfyUI 소개와 설치 — Windows · AMD Radeon · ZLUDA 편"
date: 2026-04-21
category: comfyui
tags: [comfyui, stable-diffusion, amd, zluda, windows, 설치]
draft: false
---

# ComfyUI 소개와 설치 — Windows · AMD Radeon · ZLUDA 편

로컬에서 이미지 생성 AI 를 돌리고 싶은데 NVIDIA GPU 가 없다고 포기할 필요는 없다. AMD Radeon 환경에서도 **ZLUDA** 를 경유하면 CUDA 생태계의 거의 모든 툴이 동작한다. 그 중 제일 유연한 UI 가 **ComfyUI** 다. 이 글은 Windows + AMD GPU 환경에서 ComfyUI 를 처음부터 끝까지 세팅하는 실전 가이드다.

## ComfyUI 가 뭔가

한 줄로: **노드 기반 이미지 생성 파이프라인 에디터**.

Automatic1111 이나 SD.Next 같은 버튼·슬라이더 중심 UI 와 달리, ComfyUI 는 "체크포인트 로드 → 프롬프트 인코딩 → 샘플러 → VAE 디코드 → 저장" 전 과정을 **시각화된 그래프** 로 엮는다. Blender 의 셰이더 에디터를 써봤다면 친숙한 UX.

**왜 유리한가:**
- 같은 파이프라인을 JSON 으로 저장·공유 가능 — 프롬프트만 공유하는 것보다 재현 가능성이 압도적으로 높음
- 커스텀 노드로 파이프라인 확장 (다른 API 호출, 후처리, LoRA 체이닝, ControlNet 등)
- 메모리·VRAM 관리가 명시적 — 한 그래프에서 여러 모델을 순서대로 바꿔 끼우기 수월

**단점:** 러닝 커브. 노드 수십 개를 직접 잇는 게 처음엔 버겁다. 익숙해지면 버튼 UI 로 돌아갈 수 없다.

## 이 글의 환경

| 항목 | 값 |
|---|---|
| OS | Windows 11 Pro |
| GPU | AMD Radeon RX 9070 XT (RDNA 4, gfx1201, 16 GB) |
| Python | 3.12.x (3.14 는 AI 휠 미지원) |
| HIP SDK | 6.4.2 (ROCm 6) |
| ZLUDA | nightly rocm6 빌드 |
| 포크 | [patientx/ComfyUI-Zluda](https://github.com/patientx/ComfyUI-Zluda) |

다른 AMD GPU 라도 RDNA 2/3/4 계열이면 동일 경로로 동작한다. 단 HIP SDK 와 ZLUDA 버전 조합은 ZLUDA 릴리스 노트를 확인해야 한다.

## 왜 DirectML 이나 WSL ROCm 이 아니고 ZLUDA 인가

AMD 에서 PyTorch 를 돌리는 경로는 크게 셋이다.

1. **DirectML** — 공식 Microsoft/AMD 루트. 호환성은 높지만 속도가 1/3 ~ 1/2 수준이고 일부 연산자가 누락돼 있다.
2. **WSL ROCm** — 성능은 네이티브. 다만 Windows ↔ WSL 경계에서 모델 파일·VRAM 공유가 번거롭고, 일부 커스텀 노드가 심볼릭 링크 문제로 동작하지 않는다.
3. **ZLUDA** — CUDA API 를 HIP 로 재해석한다. CUDA 전용으로 짜인 라이브러리를 거의 수정 없이 돌릴 수 있고, 속도도 네이티브 ROCm 에 근접한다.

ComfyUI 커스텀 노드 대부분이 CUDA 특정 코드를 포함하므로 **3번이 가장 폭 넓게 동작한다.**

## 설치

### Step 1. 사전 준비

**(a) Windows Defender 예외 등록 — 필수**

ZLUDA 의 일부 DLL 이 `Trojan:Win32/Pomal!rfn` 로 **잘못 탐지**된다. 예외를 걸지 않으면 설치 중간에 DLL 이 조용히 격리되어, 처음엔 왜 안 되는지 알기 어렵다.

관리자 PowerShell:
```powershell
Add-MpPreference -ExclusionPath "C:\path\to\ComfyUI"
```

**(b) HIP SDK 6.4.2 설치**

AMD 공식 다운로드 페이지에서 6.4.2 를 받는다. **7.x 로 올리지 말 것** — 현재 ZLUDA 는 ROCm 6 전용 (2026-04 기준). 잘못 올리면 ZLUDA 가 GPU 를 못 찾는다.

**(c) Python 3.12 설치**

시스템 기본 `py` 가 3.14 라도 상관없다. `py -3.12` 로 명시 호출 가능하면 된다.

### Step 2. 클론 + venv 사전 생성

```bash
git clone https://github.com/patientx/ComfyUI-Zluda C:\path\to\ComfyUI
cd C:\path\to\ComfyUI
py -3.12 -m venv venv
```

**venv 를 미리 만들어두는 게 중요하다.** `install-n.bat` 는 venv 가 없으면 자체적으로 만드는데, 그때 시스템 `python.exe` (=3.14) 를 쓰면 이후 torch 설치가 wheel 부재로 실패한다. 먼저 3.12 venv 를 파두면 스크립트가 이를 존중하고 넘어간다.

### Step 3. 설치 스크립트 실행

```cmd
.\install-n.bat > install.log 2>&1
```

약 3분. 마지막에 자동으로 ComfyUI 를 실행하려 한다.

**이때 `HIP_VISIBLE_DEVICES` 환경변수를 절대 세팅하지 말 것.**

과거 가이드는 `HIP_VISIBLE_DEVICES=1` 을 권장했지만, HIP 6.4.2 + 현재 ZLUDA 조합에서는 어떤 값이든 세팅하면 GPU 가 숨겨진다. **unset 이 정답.** 증상이 `No CUDA GPUs are available` 로 나타나면 이 변수부터 의심한다.

### Step 4. 외부 모델 경로 연결 (선택이지만 추천)

체크포인트·LoRA·VAE 를 다른 드라이브에 두고 여러 UI 에서 공유하려면 `extra_model_paths.yaml` 을 만든다:

```yaml
base_path: D:/AI/models
is_default: true
checkpoints: checkpoints/
loras: loras/
vae: vae/
controlnet: controlnet/
clip: clip/
```

이러면 `models/` 폴더는 비어 있어도 된다. 모델 교체·백업이 훨씬 수월해진다.

### Step 5. 실행 래퍼 작성

RDNA 4 (gfx1201) 는 Triton 에 아키텍처를 명시해야 Triton-compiled 연산이 동작한다. `run-comfyui.bat`:

```bat
@echo off
set TRITON_OVERRIDE_ARCH=gfx1201
call comfyui-user.bat
```

다른 GPU 면 각자 아키텍처로 (`gfx1100` RDNA 3, `gfx1030` RDNA 2 등).

### Step 6. 첫 실행

`run-comfyui.bat` 더블클릭 → 브라우저에서 `http://127.0.0.1:8188`.

**첫 이미지 생성은 10~20분 걸린다.** MIOpen 이 현재 GPU 에 맞는 커널을 튜닝하는 시간이라 정상이다. 완료되면 캐시되어 다음부터는 즉시 실행된다. 처음 돌릴 때는 그냥 두고 기다리자.

## 자주 겪는 고장

| 증상 | 원인 | 해결 |
|---|---|---|
| `No CUDA GPUs are available` / `Triton test failed: No CUDA GPUs are available` | `HIP_VISIBLE_DEVICES` 세팅됨 (어떤 값이든) | 변수 unset. 구 가이드의 `=1` 은 현재 조합에서 작동 안 함 |
| `Error loading cublas64_11.dll` | git pull 이 ZLUDA DLL 패치를 덮어씀 | `install-n.bat` 의 DLL 복사 블록 (lines 128~136) 만 재실행 |
| ZLUDA 다운로드가 tar 오류로 실패 | Defender 가 zip 을 격리 | 예외 재확인 후 재다운로드 |
| `CUDNNWrapper: '<X>' not found to wrap` 경고 | 설치하지 않은 커스텀 노드 팩을 래핑 시도 중 | 무시 가능 (양성) |
| `DynamicVRAM support requires 2.8+` | torch 가 2.7 에 핀 됨 | 무시 가능 — 레거시 ModelPatcher 가 동작함 |

## 업그레이드 주의사항

- **torch 2.8+** 는 ZLUDA 재빌드가 필요 — 업스트림이 먼저 대응할 때까지 대기
- **HIP SDK 7.x** 금지 — ZLUDA 가 ROCm 6 전용
- `install-n.bat` 는 업데이트 시 삭제 금지 — DLL 패치 복구에 필요
- `comfyui-n.bat` / `comfyui-user.bat` 는 시작 시 자동으로 `git fetch && git pull` 을 돈다. 가끔 ZLUDA DLL 패치를 덮어쓰는데, 이때는 `install-n.bat` 의 DLL 복사 블록만 다시 돌리면 된다 (전체 재설치 불필요)

## 체크리스트

- [ ] Defender 예외 등록
- [ ] HIP SDK 6.4.2 설치 (7.x 아님)
- [ ] Python 3.12 사용 가능
- [ ] 레포 클론 **전에** venv 미리 생성
- [ ] `HIP_VISIBLE_DEVICES` unset 확인
- [ ] `TRITON_OVERRIDE_ARCH` 를 본인 GPU 아키텍처로 세팅
- [ ] 첫 실행 시 MIOpen 튜닝 10~20분 대기

## 다음 글

이 환경에서 자체 커스텀 노드를 만들다 보면 **UI 검색 메뉴에 노드가 안 뜨는** 함정에 한 번은 빠진다. 대표적인 케이스가 `nodes/` 라는 이름의 서브패키지 — ComfyUI 코어의 `nodes.py` 와 충돌한다. 다음 글에서 재현·원인·해결을 정리한다.
