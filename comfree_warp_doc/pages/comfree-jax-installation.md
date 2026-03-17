# ComFree Jax Installation

ComFree Jax (`mjx`) is a JAX-based implementation of the ComFree contact solver.

## Requirements

- Python 3.10+
- [MuJoCo](https://mujoco.org/) (installed and configured)
- [JAX](https://github.com/google/jax) + backend (CPU/GPU/TPU)

## Install from source

```bash
cd /path/to/comfree_jax
pip install -e .
```

## Optional dependencies

- `jaxlib` with CUDA support for GPU execution
- `jaxlib` TPU builds for Cloud TPU usage


If you are using a shared environment (e.g., a cluster), ensure that MuJoCo is installed and `MUJOCO_PY_MUJOCO_PATH` / `MUJOCO_GL` are configured as needed.
