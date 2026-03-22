# ComFree Warp Usage

This page describes the public `comfree_warp` API as implemented in the source tree.

At the API level, `comfree_warp` is intentionally close to `mujoco_warp`. The package reuses the MJWarp-style model/data conversion path, but replaces the forward and stepping pipeline with the ComFree contact formulation.

## Workflow Overview

In broad terms, the workflow is:

1. Load a MuJoCo model and allocate `MjData`.
2. Move the model and data onto device with `put_model(...)` and `put_data(...)`.
3. Run `step(...)` or `forward(...)`.
4. Copy one world back into MuJoCo with `get_data_into(...)` when host-side access is needed.

````{important}
`comfree_warp` is designed to be a near drop-in alternative to `mujoco_warp`, but it is not just a rename.

In the current source tree:

- `put_model(...)` extends the MJWarp model with `comfree_stiffness` and `comfree_damping`
- `put_data(...)` and `make_data(...)` reuse the MJWarp allocation path and then add ComFree-specific device buffers
- `forward(...)` maps to `forward_comfree(...)`
- `step(...)` maps to `step_comfree(...)`

Since the original MJWarp contact solver path is replaced by ComFree, some differences in contact-dynamics behavior are expected even when the API usage looks almost identical.

See [ComFree Contact Parameter Settings](cfwarp-params.md) for the parameter semantics of `comfree_stiffness` and `comfree_damping`.
````

## Quick Start

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

# 1) Load and compile a MuJoCo model, then allocate host-side runtime state
model_path = "path/to/your/model.xml"
mjm = mujoco.MjSpec.from_file(model_path).compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

# 2) Move the model and data onto device
m = cfwarp.put_model(
    mjm,
    comfree_stiffness=0.2,
    comfree_damping=0.001,
)
d = cfwarp.put_data(mjm, mjd, nworld=1, nconmax=1000, njmax=5000)

# 3) Warm up once, then capture a Warp graph for repeated stepping
cfwarp.step(m, d)
cfwarp.step(m, d)
with wp.ScopedCapture() as capture:
    cfwarp.step(m, d)
graph = capture.graph

# 4) Launch the graph and pull one world back into MuJoCo when needed
for step_idx in range(1000):
    wp.capture_launch(graph)
    wp.synchronize()
    cfwarp.get_data_into(mjd, mjm, d, world_id=0)

    print(f"Step {step_idx}: qpos = {mjd.qpos}")
```

## What Changes Relative to MJWarp

From the current `api.py` implementation, the main ComFree-specific additions are:

- `put_model(...)` attaches `m.comfree_stiffness` and `m.comfree_damping` as device arrays
- scalar or vector inputs are both accepted; internally they are converted with `np.atleast_1d(...)`
- `put_data(...)` and `make_data(...)` ensure extra ComFree device fields exist
- the additional ComFree buffers include `d.efc.efc_dist`, `d.efc.efc_mass`, `d.qvel_smooth_pred`, and `d.qfrc_total`
- `forward(...)` runs the ComFree forward pipeline, including ComFree constraint construction and ComFree force assembly
- `step(...)` runs the ComFree step pipeline and then integrates with the selected MJWarp integrator path

## API Reference

### `put_model(mjm, comfree_stiffness=0.2, comfree_damping=0.001, ...)`

Creates a device-side model from a host-side `mujoco.MjModel`, then attaches the ComFree contact parameters.

In the current source:

- `comfree_stiffness` defaults to `0.2`
- `comfree_damping` defaults to `0.001`
- both are stored on device as Warp arrays
- the underlying model conversion is still performed by vendored `mujoco_warp.put_model(...)`

### `put_data(mjm, mjd, nworld=1, nconmax=None, nccdmax=None, njmax=None, naconmax=None, naccdmax=None)`

Creates a device-side `Data` object from host-side `mujoco.MjData`, using the vendored MJWarp allocation path and then adding ComFree-specific buffers.

This is the right entry point when you already have a host-side `MjData` whose state you want to copy onto device.

### `make_data(mjm, nworld=1, nconmax=None, nccdmax=None, njmax=None, naconmax=None, naccdmax=None)`

Allocates a new device-side `Data` object without copying an existing host-side `MjData`.

Use this when you want batched or freshly initialized device-side state.

### `get_data_into(result, mjm, d, world_id=0)`

Copies one selected world from device back into an existing host-side `mujoco.MjData`.

This function updates the provided `result` object in place. It is primarily useful for:

- debugging
- logging
- visualization
- interoperability with MuJoCo-side tools

### `reset_data(m, d, reset=None)`

Resets device-side data to defaults. The optional `reset` argument is a per-world boolean mask.

### `forward(m, d, factorize=True)`

Runs the ComFree forward pipeline without integrating one time step.

From the current source, this path performs:

- kinematics and smooth dynamics setup
- collision detection
- ComFree constraint construction
- smooth acceleration computation
- ComFree constraint-force assembly
- acceleration solve

### `step(m, d)`

Runs one ComFree simulation step.

In the current implementation, `step(...)` calls `forward_comfree(...)` and then integrates using the device-side integrator selected in the model options.

At present, the ComFree step implementation supports:

- Euler integration
- `IMPLICITFAST`

Other integrator modes are not documented here as supported by the current ComFree step path.

## Multi-World Example

```python
import mujoco
import warp as wp
import comfree_warp as cfwarp

mjm = mujoco.MjSpec.from_file("model.xml").compile()

m = cfwarp.put_model(
    mjm,
    comfree_stiffness=[0.2, 0.1],
    comfree_damping=[0.002, 0.001],
)

nworld = 4
d = cfwarp.make_data(mjm, nworld=nworld, nconmax=2000, njmax=10000)

cfwarp.step(m, d)
cfwarp.step(m, d)
with wp.ScopedCapture() as capture:
    cfwarp.step(m, d)
graph = capture.graph

mjd = mujoco.MjData(mjm)
for step_idx in range(500):
    wp.capture_launch(graph)
    wp.synchronize()

    if step_idx % 100 == 0:
        cfwarp.get_data_into(mjd, mjm, d, world_id=0)
        print(f"Step {step_idx}: qpos0 = {mjd.qpos[0]}")
```

## Practical Notes

- `comfree_warp` uses vendored `mujoco_warp` for most model/data conversion and many unchanged helper APIs.
- The main user-facing extensions are `comfree_stiffness` and `comfree_damping` on `put_model(...)`.
- `nworld`, `nconmax`, `njmax`, `naconmax`, and related allocation parameters follow the same basic semantics as MJWarp.
- `get_data_into(...)` introduces host-device synchronization, so avoid calling it every step unless you actually need host-side access.
- `wp.ScopedCapture()` is useful when you plan to repeat the same step graph many times.
