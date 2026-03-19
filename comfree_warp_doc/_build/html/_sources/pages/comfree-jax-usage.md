# ComFree Jax Usage

This page shows how to use the `comfree_jax` (aka `mjx`) API for running Contact-Free (ComFree) simulations using JAX.

`comfree_jax` mirrors the MuJoCo workflow but uses JAX for JIT compilation, automatic differentiation, and GPU/TPU execution.

## Imports

```python
import mujoco
import jax
import numpy as np
import mjx
```

## Minimal Example

```python
import mujoco
import numpy as np
import mjx

# 1) Load and compile a MuJoCo model
model_path = "path/to/model.xml"
mjm = mujoco.MjSpec.from_file(model_path).compile()

# 2) Initialize JAX-side model + data
m = mjx.put_model(mjm)
d = mjx.put_data(mjm)

# 3) Run a single ComFree step
#    `step_comfree` returns updated data in JAX arrays.
d = mjx.step_comfree(m, d)

# 4) Extract a NumPy snapshot if needed
mjd = mujoco.MjData(mjm)
mjx.get_data_into(mjd, mjm, d)
print("qpos", mjd.qpos)
```

## JAX Features

- **JIT compilation**: `jax.jit` can be applied to custom step functions.
- **Auto-differentiation**: Use `jax.grad` / `jax.jacrev` for derivatives.
- **Device support**: Runs on CPU, GPU, TPU via JAX backends.

## Key APIs

- `mjx.put_model(mjm)` - convert MuJoCo model to JAX-friendly form
- `mjx.put_data(mjm)` - allocate JAX buffers for model state
- `mjx.step_comfree(m, d)` - execute one ComFree simulation step
- `mjx.get_data_into(mjd, mjm, d)` - copy state back into MuJoCo `MjData`

For more advanced usage, see the `mjx` Python source in `comfree_jax/mjx/_src/`.
