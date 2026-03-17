# Installation Guide

This page describes how to install `comfree_warp` based on the current source tree in this repository.

## What ComFree Warp Ships

`comfree_warp` ships with the code it needs for its ComFree interface, so you do not need to install upstream `mujoco_warp` separately just to use `comfree_warp`.

The relevant package files are:

- `comfree_warp/comfree_warp/api.py`
- `comfree_warp/comfree_warp/__init__.py`

If you separately install the official `mujoco_warp` package and want to use it directly, you can still do:

```python
import mujoco_warp
```

That standalone upstream package can be a good choice when you want the most up-to-date official `mujoco_warp` version.

The `comfree_warp` repository may pull upstream `mujoco_warp` updates periodically, but it should not be assumed to track upstream continuously in real time.

## What You Need To Install

To use `comfree_warp`, you do **not** need to install upstream `mujoco_warp` separately.

You only need:

- `mujoco`
- `warp` / `warp-lang`
- the `comfree_warp` repository itself

The main import for ComFree usage is:

- `import comfree_warp`

## Installation Methods

### Option 1: Using pip

```bash
pip install mujoco
pip install warp-lang

git clone https://github.com/asu-iris/comfree_warp.git
cd comfree_warp
pip install -e .
```

### Option 2: Using uv

`uv` is a fast Python package and environment manager. If your `comfree_warp` checkout defines dependencies in `pyproject.toml`, `uv` can install them together:

```bash
git clone https://github.com/asu-iris/comfree_warp.git
cd comfree_warp
uv sync
```

## Verification

After installation, verify that the main package imports correctly:

```python
import mujoco
import warp as wp

import comfree_warp

print("MuJoCo:", mujoco.__version__)
print("Warp:", wp.__version__)
print("ComFree package:", comfree_warp.__name__)
```

## Version Notes

The current source tree does not include a pinned package metadata file in this docs repository, so this page avoids claiming a stricter tested matrix than the code itself shows.

What the source does show is:

- the codebase includes compatibility logic around `warp-lang >= 1.12`
- the codebase includes compatibility logic around `mujoco > 3.4.0`
- the documentation build uses Python 3.10

In practice, a safe starting point is:

- Python 3.10
- a recent `mujoco` release in the 3.4+ range
- a recent `warp-lang` release around the 1.12+ range

If you maintain this repository, the best next improvement is to replace this section with the exact MuJoCo and Warp versions you have validated on your target platform.

## Import Note

For ComFree usage, import:

```python
import comfree_warp
```

If you separately installed the official upstream package and want to use that package directly instead, use:

```python
import mujoco_warp
```

## Next Step

Once installation is complete, continue with the [MuJoCo Warp Usage](mjwarp-usage.md) page.
