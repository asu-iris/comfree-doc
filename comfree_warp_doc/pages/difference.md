# How ComFree Warp Differs from MuJoCo Warp

> **See also:** [ComFree Jax difference guide](comfree-jax-difference)

`comfree_warp` is designed as a near drop-in replacement for `mujoco_warp`. The workflow and most function signatures are identical; what changes is the solver underneath and two new parameters on `put_model`.

## Migration at a Glance

```python
# MJWarp
import mujoco_warp as mjwarp
m = mjwarp.put_model(mjm)
d = mjwarp.put_data(mjm, mjd, nworld=1, nconmax=1000, njmax=5000)
mjwarp.step(m, d)
mjwarp.get_data_into(mjd, mjm, d)

# ComFree Warp — only put_model gains new parameters
import comfree_warp
m = comfree_warp.put_model(mjm, comfree_stiffness=0.1, comfree_damping=0.001)
d = comfree_warp.put_data(mjm, mjd, nworld=1, nconmax=1000, njmax=5000)
comfree_warp.step(m, d)
comfree_warp.get_data_into(mjd, mjm, d)
```

## Function Summary

| Function | vs. `mujoco_warp` | User impact |
|---|---|---|
| `put_model` | Extended | Adds `comfree_stiffness` and `comfree_damping` |
| `put_data` / `make_data` / `get_data_into` | Extended internally | Same call pattern as MJWarp |
| `reset_data` | Delegated | Identical usage |
| `step` / `forward` | Replaced | Same signature; runs ComFree solver, not MJWarp solver |
| all other helpers | Re-exported from MJWarp | Unchanged |

## What to Expect

- Simulation trajectories will differ from MJWarp — the ComFree solver uses a different contact formulation.
- Helper APIs (collision, broadphase, rendering, sensors) are re-exported from MJWarp unchanged.
- For most code, only the import line and `put_model` call need to change.
