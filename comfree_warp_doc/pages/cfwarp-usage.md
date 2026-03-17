# ComFree Warp Usage

> **See also:** [ComFree Jax documentation](comfree-jax-usage)
>
> ComFree Jax provides a JAX-based implementation of the same ComFree contact solver.

This page shows how to use the `comfree_warp` API directly.

`comfree_warp` keeps the same overall workflow as `mujoco_warp`, but replaces the core solver path with the ComFree formulation. In practice, that means you still use the familiar sequence:

1. Load a MuJoCo model.
2. Convert it with `put_model` and `put_data`.
3. Run `step` or `forward`.
4. Copy data back with `get_data_into` if needed.

````{important}
`comfree_warp` is intended to be a near drop-in alternative to `mujoco_warp`.

For most core APIs, the user-facing call pattern stays the same. The main user-visible extension is on `put_model(...)`, which adds optional ComFree parameters such as `comfree_stiffness` and `comfree_damping`.

See [ComFree Contact Params Setting](cfwarp-params.md) for how these parameters are set and interpreted.
````

## Imports

```python
import mujoco
import warp as wp
import comfree_warp as cfwarp
```

## Minimal Example

```python
import mujoco
import warp as wp
import comfree_warp as cfwarp

# 1) Load and compile a MuJoCo model
model_path = "path/to/your/model.xml"
mjm = mujoco.MjSpec.from_file(model_path).compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

# 2) Move model/data into ComFree Warp
m = cfwarp.put_model(
    mjm,
    comfree_stiffness=0.1,
    comfree_damping=0.001,
)
d = cfwarp.put_data(mjm, mjd, nworld=1, nconmax=1000, njmax=5000)

# 3) Warm up and capture a graph for fast repeated stepping
cfwarp.step(m, d)
cfwarp.step(m, d)
with wp.ScopedCapture() as capture:
    cfwarp.step(m, d)
graph = capture.graph

# 4) Run simulation steps and sync back to MuJoCo data if needed
for step_idx in range(1000):
    wp.capture_launch(graph)
    wp.synchronize()
    cfwarp.get_data_into(mjd, mjm, d)

    print(f"Step {step_idx}: qpos = {d.qpos}")
```

## Core APIs

### `put_model(mjspec, comfree_stiffness=0.1, comfree_damping=0.001, ...)`

Converts a compiled MuJoCo model into a ComFree Warp model.

Parameters:

- `mjspec`: compiled MuJoCo model specification
- `comfree_stiffness`: ComFree contact stiffness, default `0.1`
- `comfree_damping`: ComFree contact damping, default `0.001`
- any additional arguments supported by the underlying MJWarp conversion path

For the detailed meaning of `comfree_stiffness` and `comfree_damping`, including per-environment parameter usage, see [Per-Env Parameter Parallelization](cfwarp-params.md).

Returns:

- a Warp-side model with ComFree parameters attached

### `put_data(mjspec, mjdata, nworld=1, nconmax=1000, njmax=5000, ...)`

Converts MuJoCo runtime data into a ComFree Warp data object.

Parameters:

- `mjspec`: MuJoCo model specification
- `mjdata`: MuJoCo data object
- `nworld`: number of simulation worlds
- `nconmax`: maximum number of contacts
- `njmax`: maximum number of constraints

Returns:

- a Warp-side data object extended with ComFree solver fields

### `make_data(mjspec, nworld=1, nconmax=1000, njmax=5000, ...)`

Creates a new ComFree Warp data object without converting an existing `mjdata`.

### `get_data_into(mjdata, mjspec, warp_data)`

Copies Warp-side simulation state back into MuJoCo CPU arrays.

Use this when you need to inspect state through MuJoCo-side tools or code.

### `reset_data(mjspec, warp_data)`

Resets the simulation state.

### `step(model, data)`

Runs one ComFree simulation step.

This is one of the main places where `comfree_warp` differs internally from `mujoco_warp`: the call looks the same, but it executes the ComFree solver pipeline.

### `forward(model, data)`

Runs the ComFree forward pipeline without integration.

## Batch Example

```python
import mujoco
import warp as wp
import comfree_warp as cfwarp

mjm = mujoco.MjSpec.from_file("model.xml").compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

m = cfwarp.put_model(
    mjm,
    comfree_stiffness=0.2,
    comfree_damping=0.002,
)

nworlds = 4
d = cfwarp.make_data(mjm, nworld=nworlds, nconmax=2000, njmax=10000)

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
        print(
            f"Step {step_idx}: "
            f"min_qpos={d.qpos.numpy().min()}, "
            f"max_qpos={d.qpos.numpy().max()}"
        )
```

## Notes

- `comfree_stiffness` and `comfree_damping` are the main ComFree-specific additions compared with plain MJWarp.
- `step` and `forward` keep the same role as in MJWarp, but run the ComFree solver and forward pipeline internally.
- `get_data_into()` is useful, but CPU-GPU synchronization is relatively expensive, so use it only when needed.
- `wp.ScopedCapture()` is recommended when you plan to repeat the same simulation step many times.
