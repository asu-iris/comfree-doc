# MJWarp Preliminary

This page is a preliminary reference for the original upstream `mujoco_warp` package.
`mujoco-warp` is a Warp-based simulator backend for MuJoCo models, designed for fast GPU-accelerated simulation while keeping a MuJoCo-like workflow (`MjModel`/`MjData` -> `put_model`/`put_data` -> `step`).

We include this MJWarp introduction because `comfree_warp` intentionally follows almost the same interface and workflow. Understanding the basic `mujoco_warp` usage pattern makes it much easier to understand `comfree_warp`, since the same core calls such as `put_model`, `put_data`, `step`, `forward`, `reset_data`, and `get_data_into` carry over directly.

For full upstream documentation, see the [mujoco-warp docs](https://mujoco.readthedocs.io/en/latest/mjwarp/).

## Imports

```python
import mujoco
import warp as wp
import mujoco_warp as mjwarp
```

## Minimal Example

```python
# 1) Load and compile a MuJoCo model
model_path = "benchmark/test_data/primitives.xml"
mjm = mujoco.MjSpec.from_file(model_path).compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

# 2) Move model/data into mujoco-warp
m = mjwarp.put_model(mjm)
d = mjwarp.put_data(mjm, mjd, nworld=1, nconmax=1000, njmax=5000)

# 3) Warm up and capture a graph for fast repeated stepping
mjwarp.step(m, d)
mjwarp.step(m, d)
with wp.ScopedCapture() as capture:
    mjwarp.step(m, d)
graph = capture.graph

# 4) Run steps and sync back to MuJoCo data if needed
for _ in range(100):
    wp.capture_launch(graph)
    wp.synchronize()
    mjwarp.get_data_into(mjd, mjm, d)
```

## Notes

- `put_model` and `put_data` convert MuJoCo structures into Warp-compatible structures.
- `nconmax` and `njmax` control contact/constraint buffer sizes for Warp simulation.
- `mjwarp.get_data_into(...)` is useful when external tools read from `mjd`.

<!-- ## comfree-sim Extends mujoco-warp

`comfree_warp` builds directly on `mujoco_warp` and re-exports its ComFree API at the top level, so the workflow above carries over with only an import change. The table below shows how each function relates to its `mujoco_warp` counterpart:


| Function        | Relationship | Notes                                                                                            |
| --------------- | ------------ | ------------------------------------------------------------------------------------------------ |
| `put_model`     | Extended     | Calls`mjwarp.put_model`, then attaches `comfree_stiffness`/`comfree_damping` arrays to the model |
| `put_data`      | Extended     | Calls`mjwarp.put_data`, then adds comfree constraint and velocity fields to the data             |
| `make_data`     | Extended     | Same extension pattern as`put_data`                                                              |
| `get_data_into` | Extended     | Same extension pattern as`put_data`                                                              |
| `reset_data`    | Delegated    | Passes through directly to`mjwarp.reset_data`                                                    |
| `step`          | Replaced     | Uses comfree's own physics pipeline instead of mjwarp's                                          |
| `forward`       | Replaced     | Uses comfree's own forward pass instead of mjwarp's                                              |

To migrate from `mujoco_warp` to `comfree_warp`, only the import line changes:

```python
# Before
import mujoco_warp as mjwarp

# After
import comfree_warp
``` -->

All `put_model`, `put_data`, `get_data_into`, and `reset_data` calls are signature-compatible — no other changes are required for basic usage.
