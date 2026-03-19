# Installation Guide

This page describes how to install `comfree_warp` based on the current upstream source layout and package metadata.

## Relationship to `mujoco_warp`

`comfree_warp` is developed on top of `mujoco_warp`. However, to minimize external dependencies and reduce version-conflict issues, the `comfree_warp` package also vendors its own in-tree copy of `mujoco_warp`.

This means:

- when you install `comfree_warp`, the vendored `mujoco_warp` implementation is installed as part of the package
- you do not need to install `mujoco_warp` separately in order to use `comfree_warp`
- if you have already installed `mujoco_warp` separately, that is still fine
- `comfree_warp` depends on its vendored `mujoco_warp`, not on a separately installed external copy

The currently vendored `mujoco_warp` snapshot corresponds to upstream commit `89be29a262e4209c8cba1ebe66f5bfd1905a3261`.

## Requirements

According to `pyproject.toml`, `comfree_warp` currently requires:

- Python `>=3.10`
- `warp-lang>=1.12`
- `mujoco==3.6.0`

You do not need to install upstream `mujoco_warp` separately in order to use `comfree_warp`.

## Install from Source

Clone the repository and install it from the project root:

```bash
git clone https://github.com/asu-iris/comfree_warp.git
cd comfree_warp
pip install .
```

If you want an editable local development install instead, use:

```bash
git clone https://github.com/asu-iris/comfree_warp.git
cd comfree_warp
pip install -e .
```

## Install with uv

If you use `uv`, the repository README also supports installing from the project root with:

```bash
git clone https://github.com/asu-iris/comfree_warp.git
cd comfree_warp
uv sync
```

## Verify the Installation

After installation, verify that the main package imports correctly:

```python
import mujoco
import warp as wp
import comfree_warp

print("MuJoCo:", mujoco.__version__)
print("Warp:", wp.__version__)
print("ComFree package:", comfree_warp.__name__)
```

For ComFree usage, the main import is:

```python
import comfree_warp
```

## Quick Sanity Checks

The upstream repository currently documents the following example entry points:

- interactive viewer: `python tests_local_viewer/test_viewer.py`
- headless simulation: `python test_headless.py`
- throughput benchmark: `python tests_local_viewer/test_throuput_hand.py`

These are useful after installation if you want to confirm the package is running correctly in your environment.

## Version Notes

The current package metadata explicitly declares:

- Python `>=3.10`
- `warp-lang>=1.12`
- `mujoco==3.6.0`

If installation or import fails, verify these versions first before debugging higher-level runtime issues.

## Next Step

Once installation is complete, continue with the [MuJoCo Warp Usage](mjwarp-usage.md) page.
