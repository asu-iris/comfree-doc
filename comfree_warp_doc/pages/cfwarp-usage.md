# ComFree Warp Usage

> **See also:** [ComFree Jax documentation](comfree-jax-usage)

`comfree_warp` is a near drop-in replacement for `mujoco_warp`. The workflow is identical; the only user-visible addition is `comfree_stiffness` and `comfree_damping` on `put_model`.

## Minimal Example

```python
import mujoco
import warp as wp
import comfree_warp as cfwarp

# 1) Load model
mjm = mujoco.MjSpec.from_file("model.xml").compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

# 2) Convert to ComFree Warp
m = cfwarp.put_model(mjm, comfree_stiffness=0.1, comfree_damping=0.001)
d = cfwarp.put_data(mjm, mjd, nworld=1, nconmax=1000, njmax=5000)

# 3) Capture graph for fast repeated stepping
cfwarp.step(m, d)
cfwarp.step(m, d)
with wp.ScopedCapture() as capture:
    cfwarp.step(m, d)
graph = capture.graph

# 4) Run steps; sync back to CPU when needed
for step_idx in range(1000):
    wp.capture_launch(graph)
    wp.synchronize()
    cfwarp.get_data_into(mjd, mjm, d)
```

## API Reference

### `put_model(mjspec, comfree_stiffness=0.1, comfree_damping=0.001, ...)`

Converts a compiled MuJoCo model into a ComFree Warp model. Accepts all standard MJWarp arguments plus `comfree_stiffness` and `comfree_damping` (scalars or vectors).

See [Contact Parameter Settings](cfwarp-params.md) for multi-world vector usage.

### `put_data(mjspec, mjdata, nworld=1, nconmax=1000, njmax=5000, ...)`

Converts existing `MjData` into a ComFree Warp data object.

### `make_data(mjspec, nworld=1, nconmax=1000, njmax=5000, ...)`

Creates a fresh data object without an existing `MjData`.

### `get_data_into(mjdata, mjspec, warp_data)`

Copies GPU state back to CPU `MjData`. Avoid calling every step — CPU-GPU sync is expensive.

### `reset_data(mjspec, warp_data)`

Resets simulation state to initial conditions.

### `step(model, data)` / `forward(model, data)`

Run one simulation step (with integration) or the forward pass (no integration). Same signatures as MJWarp, but execute the ComFree solver internally.

## Batch Simulation

Use `make_data` with `nworld > 1` to simulate multiple environments in parallel:

```python
m = cfwarp.put_model(mjm, comfree_stiffness=0.2, comfree_damping=0.002)
d = cfwarp.make_data(mjm, nworld=4, nconmax=2000, njmax=10000)

cfwarp.step(m, d)
cfwarp.step(m, d)
with wp.ScopedCapture() as capture:
    cfwarp.step(m, d)
graph = capture.graph

for step_idx in range(500):
    wp.capture_launch(graph)
    wp.synchronize()
    if step_idx % 100 == 0:
        cfwarp.get_data_into(mjd, mjm, d)
```
