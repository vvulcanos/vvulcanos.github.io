---
title: "ComfyUI 커스텀 노드가 메뉴에 안 뜰 때 — `nodes/` 서브패키지 shadow 버그"
date: 2026-04-21
category: comfyui
tags: [comfyui, python, 디버깅, 커스텀노드, import]
draft: false
---

커스텀 노드 팩을 만들어 `custom_nodes/` 에 두고 재시작했는데 UI 의 노드 검색 메뉴에 안 뜬다. 서버 로그엔 눈에 띄는 에러도 없다. 몇 시간을 로그만 노려보게 만드는 이 함정을 한 번 걸리고 나면 다음부터는 안 걸리게 박제해 둔다.

## 증상

패키지 구조는 이렇게 생겼다.

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

관례적으로 써 온 모양새다. 재시작하면 UI 검색에 `Foo` 가 안 뜬다. 서버 콘솔엔 주목할 만한 traceback 도 없거나, 기껏해야 `module has no attribute` 비슷한 애매한 한두 줄. `pip install -e .` 을 해도 똑같고, 같은 디렉터리의 `__init__.py` 에 `print("HELLO")` 를 박아도 안 찍히는 경우까지 있다.

## 원인 — 코어가 이미 `nodes` 라는 이름을 점유하고 있다

ComfyUI 는 시작 시 레포 루트의 `nodes.py` (코어 노드 정의 파일) 를 `nodes` 라는 이름으로 `sys.modules` 에 등록해 둔다. 그 뒤 `custom_nodes/ComfyUI-Foo/__init__.py` 가 로드되면서

```python
from nodes.foo_node import FooNode
```

를 만난다. Python 은 이 시점에 이미 등록된 `sys.modules['nodes']` 를 먼저 찾는다. 그게 코어 nodes 모듈(파일 하나짜리, 패키지가 아닌)이고, 거기엔 `foo_node` 라는 서브모듈이 없으니 `ModuleNotFoundError: No module named 'nodes.foo_node'` 가 터진다.

결정적인 건 ComfyUI 가 커스텀 노드 로딩 루프에서 이 예외를 조용히 삼키고 다음 팩으로 넘어간다는 점. 로그는 깨끗하고, UI 엔 아무것도 안 뜬다.

## 최소 재현

한 줄로 재현된다.

```python
# custom_nodes/test-shadow/__init__.py
try:
    from nodes import X  # X 는 코어 nodes.py 에 없는 아무 이름
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

여기서 `nodes` 가 코어 모듈이라는 확증이 나온다 — `dir()` 찍어 보면 코어 노드 클래스들만 들어 있다.

## 해결

세 가지 선택지가 있다.

### (a) 서브패키지 이름을 바꾼다 (권장)

```bash
git mv nodes my_pack_nodes
```

모든 import 경로 재작성:

```python
from my_pack_nodes.foo_node import FooNode
```

그리고 `my_pack_nodes/` 내부끼리 import 도 전부 같은 접두어로.

```python
# my_pack_nodes/foo_node.py
from my_pack_nodes._base import Base   # OK
```

ComfyUI 코어가 점유한 이름과 안 겹치면 뭐든 된다. 관행상 팩 식별자를 접두어로 붙이는 쪽이 안전하다 (`my_pack_nodes`, `foopack_nodes` 등).

### (b) 상대 import 로 전환

```python
# ComfyUI-Foo/__init__.py
from .nodes.foo_node import FooNode
```

`.` 이 붙으면 Python 이 `sys.modules['nodes']` 를 거치지 않고 현재 패키지 내부에서 찾는다. 이미 `nodes/` 라는 이름으로 공개된 예제 코드를 최소 수정으로 구제할 때 쓸 만하다.

주의할 점은 내부 모듈끼리의 import 도 전부 상대 import 로 통일해야 한다는 것. `nodes/foo_node.py` 안에서

```python
from nodes._base import Base     # shadow 여전히 발생
from ._base import Base          # 안전
```

### (c) `sys.path` 에 팩 루트를 끼운다 (비추)

```python
# __init__.py
import os, sys
sys.path.insert(0, os.path.dirname(__file__))
```

동작은 한다. 다만 글로벌 `sys.path` 를 건드리는 부작용이 있어서, 다른 커스텀 노드 팩이 같은 트릭을 쓰면 우연히 같은 이름의 서브모듈이 충돌할 수 있다. 디버깅 난이도도 올라간다. 재명명·상대 import 둘 다 불가능할 때의 마지막 수단.

## 예방

ComfyUI 코어가 이미 점유한 이름은 서브패키지로 쓰지 말 것. 실전에서 겹치는 후보들:

- `nodes` — 코어 `nodes.py`
- `comfy` — 코어 `comfy/` 패키지
- `execution` — 코어 실행 엔진
- `server` — 코어 웹서버
- `folder_paths` — 모델 경로 유틸
- `main` — 진입점

결과적으로 이런 구조가 깔끔하다.

```
ComfyUI-Foo/
├── __init__.py
└── foo_nodes/              ← 팩 이름 접두어
    ├── __init__.py
    ├── _base.py
    └── foo_node.py
```

그리고 `__init__.py` 최상단에 import 테스트 한 줄을 깔아두면 이런 류의 조용한 실패가 콘솔에 즉시 보인다.

```python
try:
    from my_pack_nodes.foo_node import FooNode
except Exception as e:
    print(f"[ComfyUI-Foo] import failed: {type(e).__name__}: {e}")
    raise

NODE_CLASS_MAPPINGS = {"FooNode": FooNode}
```

개발 중에는 이게 단서의 전부가 되는 순간이 온다.

## 정리

에러가 삼켜지니 로그에 원인이 없고, UI 에 "안 뜸" 상태라 어디를 봐야 할지 감이 안 잡힌다. `nodes/` 는 다른 Python 프로젝트에서 흔한 이름이라 경계심도 낮다. 한 번 체득하면 다시 걸릴 일은 없는 종류의 함정이라, 이 글이 다음 사람에게 그 시간을 아껴 주면 된 거다.
