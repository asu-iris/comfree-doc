# Installation

> **Related:** [ComFree Jax installation](comfree-jax-installation)

## Requirements

- Python 3.10
- `mujoco == 3.5.0`
- `warp-lang >= 1.12`

`comfree_warp` bundles its own MJWarp integration — no need to install upstream `mujoco_warp` separately.

## Install

### Option 1: pip

```bash
pip install mujoco warp-lang
git clone https://github.com/asu-iris/comfree_warp.git
cd comfree_warp
pip install -e .
```

### Option 2: uv

```bash
git clone https://github.com/asu-iris/comfree_warp.git
cd comfree_warp
uv sync
```

## Verify

```python
import mujoco
import warp as wp
import comfree_warp

print("MuJoCo:", mujoco.__version__)
print("Warp:", wp.__version__)
```

## Next Step

Continue with [MuJoCo Warp Usage](mjwarp-usage.md).
