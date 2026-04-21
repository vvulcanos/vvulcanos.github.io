---
title: "ComfyUI 커스텀 노드가 메뉴에 안 뜰 때 — `nodes/` 서브패키지 shadow 버그"
date: 2026-04-21
category: comfyui
tags: [comfyui, python, 디버깅, 커스텀노드, import]
draft: false
---

# ComfyUI 커스텀 노드가 메뉴에 안 뜰 때 — `nodes/` 서브패키지 shadow 버그

ComfyUI 커스텀 노드 팩을 만들어 `custom_nodes/` 에 두고 재시작했는데, **UI 의 노드 검색 메뉴에 내가 만든 노드 이름이 나오지 않는다.** 서버 로그에는 눈에 띄는 에러도 없다. 수시간을 로그만 노려보게 만드는 이 함정 하나를 박제한다.

## 증상

패키지 구조는 이렇다.

```
custom_nodes/
└── ComfyUI-Foo/
    ├── __init__.py
    ├── nodes/                ← 문제의 이름
    │   ├── __init__.py
    │   ├── _base.py
    │   └── foo_node.py
    └── pyproject.toml
```

그리고 `__init__.py`:

```python
from nodes.foo_node import FooNode

NODE_CLASS_MAPPINGS = {"FooNode": FooNode}
NODE_DISPLAY_NAME_MAPPINGS = {"FooNode": "Foo"}
```

관례적으로 써온 구조다. 재시작하면:

- UI 검색에 `Foo` 가 **없다**
- 서버 콘솔에는 주목할 만한 traceback 이 **없다** (또는 `module has no attribute` 같은 애매한 import 에러가 한두 줄)
- `pip install -e .` 해도 동일
- 같은 디렉터리의 `__init__.py` 에 `print("HELLO")` 를 박아도 안 찍힐 때가 있다

## 원인 — 코어가 이미 `nodes` 라는 이름을 점유하고 있다

ComfyUI 는 시작 시 레포 루트의 `nodes.py` (코어 노드 정의 파일) 를 `nodes` 라는 이름으로 `sys.modules` 에 등록한다. 그 뒤 `custom_nodes/ComfyUI-Foo/__init__.py` 가 로드되면서 다음 줄을 만난다.

```python
from nodes.foo_node import FooNode
```

Python 의 import 메커니즘은 이 시점에 이미 등록된 `sys.modules['nodes']` 를 **먼저 본다**. 그게 **코어 nodes 모듈** (파일 하나짜리 모듈, 패키지 아님) 이고 거기에는 `foo_node` 라는 서브모듈이 존재하지 않으니 `ModuleNotFoundError: No module named 'nodes.foo_node'` 가 발생한다.

결정적으로 ComfyUI 는 커스텀 노드 로딩 루프에서 이 예외를 **조용히 삼키고 다음 팩으로 넘어간다**. 로그는 깨끗하고, UI 에는 아무것도 뜨지 않는다.

## 최소 재현

한 줄로 재현된다:

```python
# custom_nodes/test-shadow/__init__.py
try:
    from nodes import X  # X 는 코어 nodes.py 에 없는 어떤 이름이든
    print("[shadow test] import succeeded (shouldn't)")
except Exception as e:
    print(f"[shadow test] {type(e).__name__}: {e}")

NODE_CLASS_MAPPINGS = {}
NODE_DISPLAY_NAME_MAPPINGS = {}
```

ComfyUI 재시작 후 콘솔:

```
[shadow test] ModuleNotFoundError: No module named 'nodes.X'
```

여기서 `nodes` 가 코어 모듈이라는 확증을 얻는다. (`dir()` 찍어봐도 코어 노드 클래스들만 들어있다.)

## 해결

세 가지 선택지가 있고, 권장 순서가 다르다.

### (a) 서브패키지 이름을 바꾼다 — 권장

```bash
git mv nodes my_pack_nodes
```

모든 import 경로 재작성:

```python
from my_pack_nodes.foo_node import FooNode
```

그리고 `my_pack_nodes/` 내부의 서로 간 import 도 모두 같은 접두어로.

```python
# my_pack_nodes/foo_node.py
from my_pack_nodes._base import Base   # ✅
```

ComfyUI 코어가 점유한 이름과 겹치지 않으면 무엇이든 좋다. 관행상 **팩 식별자 접두어를 붙이는 게 안전하다** (`my_pack_nodes`, `foopack_nodes` 등).

### (b) 상대 import 로 전환

```python
# ComfyUI-Foo/__init__.py
from .nodes.foo_node import FooNode
```

`.` 접두어가 붙으면 Python 은 `sys.modules['nodes']` 를 거치지 않고 **현재 패키지 내부** 에서 찾는다. 이미 `nodes/` 라는 이름으로 공개된 예시 코드를 최소 수정으로 구제할 때 적합.

주의: **내부 모듈끼리의 import 도 모두 상대 import 로 통일** 해야 한다. `nodes/foo_node.py` 안에서

```python
from nodes._base import Base     # ❌ shadow 여전히 발생
from ._base import Base          # ✅ 안전
```

### (c) `sys.path` 에 팩 루트를 끼운다 — 비추

```python
# __init__.py
import os, sys
sys.path.insert(0, os.path.dirname(__file__))
```

동작은 한다. 하지만 **글로벌 `sys.path` 를 건드리는 부작용** 이 있다. 다른 커스텀 노드 팩이 같은 트릭을 쓰면 우연히 같은 이름의 서브모듈이 충돌할 수 있고, 디버깅 난이도가 올라간다. 재명명·상대 import 둘 다 불가능할 때의 마지막 수단으로만.

## 예방 규칙

ComfyUI 코어가 이미 점유한 이름은 서브패키지로 쓰지 말 것. 실전에서 겹치는 후보들:

- `nodes` — 코어 `nodes.py`
- `comfy` — 코어 `comfy/` 패키지
- `execution` — 코어 실행 엔진
- `server` — 코어 웹서버
- `folder_paths` — 모델 경로 유틸
- `main` — 진입점

결과적으로 이런 구조가 가장 깔끔하다.

```
ComfyUI-Foo/
├── __init__.py
└── foo_nodes/              # ✅ 팩 이름 접두어
    ├── __init__.py
    ├── _base.py
    └── foo_node.py
```

## 왜 이 버그가 아픈가

- **조용히 실패** — 에러가 삼켜지므로 로그에 원인이 뜨지 않는다
- **증상이 부재** — UI 에 "안 뜸" 상태라 어디를 봐야 할지 단서가 없다
- **유사 이름 관행** — 다른 Python 프로젝트에서 `nodes/` 는 흔한 이름이라 경계심이 낮다

예방책으로 **커스텀 노드 팩을 만들 때 `__init__.py` 최상단에 import 테스트 한 줄** 을 깔아두는 걸 추천한다. 의심 import 하나가 실패하면 콘솔에 즉시 눈에 띄게 출력되도록.

```python
# __init__.py 시작부
try:
    from my_pack_nodes.foo_node import FooNode
except Exception as e:
    print(f"[ComfyUI-Foo] import failed: {type(e).__name__}: {e}")
    raise

NODE_CLASS_MAPPINGS = {"FooNode": FooNode}
```

개발 중에는 이게 단서의 전부가 되는 순간이 온다.

## 정리

- 증상: 커스텀 노드가 UI 검색 메뉴에 뜨지 않고 로그도 깨끗하다
- 원인: 서브패키지 이름이 ComfyUI 코어 모듈명과 겹치면서 `sys.modules` 상에서 shadow 됨
- 해결: 서브패키지 재명명 (권장) 또는 상대 import 전환
- 예방: 코어 점유 이름 피하기 + `__init__.py` 최상단 import 체크

재현이 한 줄이라 한 번 체득하면 다시 걸리진 않는다. 다만 처음 걸렸을 때 원인에 도달하는 시간이 너무 길어서, 이 글이 그 시간을 0 분으로 줄여주길.
