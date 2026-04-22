---
title: "ComfyUI 소개와 설치 — Windows · AMD Radeon · ZLUDA 편"
date: 2026-04-21
category: comfyui
tags: [comfyui, stable-diffusion, amd, zluda, windows, 설치]
draft: false
---

로컬에서 이미지 생성 AI를 돌리고 싶은데 NVIDIA GPU가 없다 — 그 이유로 포기할 필요는 없다. AMD Radeon 환경에서도 ZLUDA를 경유하면 CUDA 생태계의 거의 모든 툴이 돈다. 그 중 UI 가장 유연한 쪽이 ComfyUI라서, Windows + AMD GPU 로 처음 세팅하는 길을 정리해 둔다.

## ComfyUI 가 뭔가

노드 기반 이미지 생성 파이프라인 에디터. Automatic1111 이나 SD.Next 같은 버튼·슬라이더 UI 와 달리, "체크포인트 로드 → 프롬프트 인코딩 → 샘플러 → VAE 디코드 → 저장" 전 과정을 그래프로 엮는다. Blender의 셰이더 에디터 써봤으면 첫 느낌이 비슷할 거다.

장점 몇 가지.

- 같은 파이프라인을 JSON 으로 저장·공유할 수 있어서, 프롬프트만 공유하는 문화보다 재현 가능성이 한참 높다.
- 커스텀 노드로 파이프라인을 확장하기 수월하다 — 외부 API 호출, 후처리, LoRA 체이닝, ControlNet 같은 걸 필요한 만큼 붙일 수 있다.
- 메모리·VRAM 관리가 명시적이라, 한 그래프 안에서 여러 모델을 순서대로 바꿔 끼우기 쉽다.

단점은 러닝 커브. 노드 수십 개를 직접 잇는 게 처음엔 버겁다. 한 번 익숙해지면 버튼 UI 로 돌아가기 어려워진다.

## 이 글의 환경

| 항목 | 값 |
|---|---|
| OS | Windows 11 Pro |
| GPU | AMD Radeon RX 9070 XT (RDNA 4, gfx1201, 16 GB) |
| Python | 3.12.x (3.14는 AI 휠 미지원) |
| HIP SDK | 6.4.2 (ROCm 6) |
| ZLUDA | nightly rocm6 빌드 |
| 포크 | [patientx/ComfyUI-Zluda](https://github.com/patientx/ComfyUI-Zluda) |

다른 AMD GPU 라도 RDNA 2/3/4 계열이면 같은 경로로 동작한다. HIP SDK 와 ZLUDA 버전 조합은 ZLUDA 릴리스 노트에서 확인.

## 왜 DirectML 이나 WSL ROCm 이 아니고 ZLUDA 인가

AMD 에서 PyTorch 돌리는 경로는 크게 셋이다.

DirectML 은 공식 Microsoft/AMD 루트인데, 호환성은 높지만 속도가 네이티브의 1/3~1/2 수준이고 누락된 연산자가 있다. WSL ROCm 은 속도는 네이티브에 근접하지만 Windows↔WSL 경계에서 모델 파일·VRAM 공유가 번거롭고, 일부 커스텀 노드가 심볼릭 링크 문제로 안 돈다. ZLUDA 는 CUDA API 를 HIP 로 재해석해서 돌리는 방식이라, CUDA 전용으로 짜인 라이브러리를 거의 수정 없이 굴릴 수 있고 속도도 네이티브 ROCm 수준이 나온다.

ComfyUI 커스텀 노드 대부분이 CUDA 특정 코드를 포함하다 보니 ZLUDA 쪽이 가장 폭넓게 돈다.

## 설치

### Step 1. 사전 준비

Windows Defender 예외 등록부터. ZLUDA 의 일부 DLL 이 `Trojan:Win32/Pomal!rfn` 로 오탐된다. 예외를 안 걸면 설치 중간에 DLL 이 조용히 격리돼서, 처음엔 왜 실패하는지 단서를 찾기 어렵다. 관리자 PowerShell 에서:

```powershell
Add-MpPreference -ExclusionPath "C:\path\to\ComfyUI"
```

다음 HIP SDK 6.4.2 설치. AMD 공식 다운로드 페이지에서 6.4.2 를 받는다. 7.x 로 올리지 말 것 — 현재 ZLUDA 는 ROCm 6 전용이다 (2026-04 기준). 잘못 올리면 GPU 를 못 찾는 상태가 된다.

그리고 Python 3.12. 시스템 기본 `py` 가 3.14 여도 상관없고, `py -3.12` 로 명시 호출만 되면 된다.

### Step 2. 클론 + venv 사전 생성

```bash
git clone https://github.com/patientx/ComfyUI-Zluda C:\path\to\ComfyUI
cd C:\path\to\ComfyUI
py -3.12 -m venv venv
```

venv 를 먼저 만들어 두는 게 포인트다. `install-n.bat` 는 venv 가 없으면 자체적으로 하나 만드는데, 그때 시스템 `python.exe`(=3.14)를 쓴다. 이후 torch 설치가 wheel 부재로 실패한다. 3.12 venv 를 미리 파 두면 스크립트가 존중하고 넘어간다.

### Step 3. 설치 스크립트 실행

```cmd
.\install-n.bat > install.log 2>&1
```

3분 정도. 끝에서 ComfyUI 를 자동으로 실행하려 한다. 여기서 `HIP_VISIBLE_DEVICES` 환경변수를 절대 세팅하지 말 것. 과거 가이드는 `HIP_VISIBLE_DEVICES=1` 을 권장했지만, HIP 6.4.2 + 현재 ZLUDA 조합에서는 어떤 값이든 세팅하는 순간 GPU 가 숨겨진다. unset 이 정답이고, 증상이 `No CUDA GPUs are available` 로 나타나면 제일 먼저 이 변수부터 확인하면 된다.

### Step 4. 외부 모델 경로 연결 (선택이지만 추천)

체크포인트·LoRA·VAE 를 다른 드라이브에 두고 여러 UI 에서 공유하려면 `extra_model_paths.yaml`:

```yaml
base_path: D:/AI/models
is_default: true
checkpoints: checkpoints/
loras: loras/
vae: vae/
controlnet: controlnet/
clip: clip/
```

`models/` 폴더는 비어 있어도 된다. 모델 교체·백업이 훨씬 편해진다.

### Step 5. 실행 래퍼

RDNA 4 (gfx1201) 는 Triton 에 아키텍처를 명시해야 Triton 컴파일 연산이 돈다. `run-comfyui.bat`:

```bat
@echo off
set TRITON_OVERRIDE_ARCH=gfx1201
call comfyui-user.bat
```

다른 GPU 면 각자 아키텍처로 (`gfx1100` = RDNA 3, `gfx1030` = RDNA 2 등).

### Step 6. 첫 실행

`run-comfyui.bat` 더블클릭, 브라우저에서 `http://127.0.0.1:8188`. 첫 이미지 생성이 10~20분 걸리는데, MIOpen 이 현재 GPU 에 맞는 커널을 튜닝하는 시간이다. 끝나면 캐시되고 이후엔 즉시 뜬다. 그냥 두고 기다리자.

## 자주 겪는 고장

| 증상 | 원인 | 해결 |
|---|---|---|
| `No CUDA GPUs are available` / `Triton test failed: No CUDA GPUs are available` | `HIP_VISIBLE_DEVICES` 가 어떤 값이든 세팅됨 | 변수 unset. 구 가이드의 `=1` 은 현재 조합에서 동작 안 함 |
| `Error loading cublas64_11.dll` | git pull 이 ZLUDA DLL 패치를 덮어씀 | `install-n.bat` 의 DLL 복사 블록 (lines 128~136) 만 재실행 |
| ZLUDA 다운로드가 tar 오류로 실패 | Defender 가 zip 을 격리 | 예외 재확인 후 재다운로드 |
| `CUDNNWrapper: '<X>' not found to wrap` 경고 | 설치하지 않은 커스텀 노드 팩을 래핑 시도 | 무시 가능 |
| `DynamicVRAM support requires 2.8+` | torch 가 2.7 에 핀 됨 | 무시 가능 — 레거시 ModelPatcher 로도 돈다 |

## 업그레이드 주의

torch 2.8+ 는 ZLUDA 재빌드가 필요하니 업스트림이 먼저 대응할 때까지 기다리는 게 안전하다. HIP SDK 도 7.x 로 올리지 말 것. ZLUDA 가 ROCm 6 전용이다. `install-n.bat` 은 업데이트 시 삭제 금지 — DLL 패치 복구에 계속 쓰인다. `comfyui-n.bat` / `comfyui-user.bat` 가 시작 시 `git fetch && git pull` 을 자동으로 돌리는데, 가끔 그 과정에서 ZLUDA DLL 패치를 덮어쓴다. 그러면 `install-n.bat` 의 DLL 복사 블록만 다시 돌리면 된다 (전체 재설치까지 필요 없다).

## 출발 체크리스트

- [ ] Defender 예외 등록
- [ ] HIP SDK 6.4.2 (7.x 아님)
- [ ] Python 3.12 사용 가능
- [ ] 레포 클론 뒤 venv 미리 생성
- [ ] `HIP_VISIBLE_DEVICES` unset
- [ ] `TRITON_OVERRIDE_ARCH` 를 본인 GPU 아키텍처로
- [ ] 첫 실행 시 MIOpen 튜닝 10~20분 대기

자체 커스텀 노드를 만들다 보면 UI 검색 메뉴에 노드가 안 뜨는 함정에 한 번은 빠진다. 그 얘기는 [[2026-04-21-custom-node-shadow-collision|여기]].
