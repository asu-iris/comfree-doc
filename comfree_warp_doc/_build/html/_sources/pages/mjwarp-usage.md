# MJWarp Preliminary

This page introduces the upstream `mujoco_warp` package only as background. It is included to explain the workflow and API pattern that `comfree_warp` largely preserves, not because `mujoco_warp` is the primary target of this documentation.

If you are here to use ComFree directly, you can treat this page as a short compatibility primer and then move on to [ComFree Warp Usage](cfwarp-usage.md).

`mujoco_warp` is a Warp-based simulation backend for MuJoCo models. In broad terms, the workflow is:

`MjModel` / `MjData` -> `put_model` / `put_data` -> `step` / `forward`

We include this MJWarp introduction because `comfree_warp` intentionally stays close to the same interface and execution pattern. Understanding the basic `mujoco_warp` usage pattern makes it much easier to understand `comfree_warp`, since the same core entry points such as `put_model`, `put_data`, `step`, `forward`, `reset_data`, and `get_data_into` carry over directly.

For full upstream documentation, see the [mujoco_warp docs](https://mujoco.readthedocs.io/en/latest/mjwarp/).

## Imports

```python
import mujoco
import warp as wp
import mujoco_warp as mjwarp
```

## Minimal Example

```python
# 1) Load and compile a MuJoCo model, then allocate runtime state
model_path = "benchmark/test_data/primitives.xml"
mjm = mujoco.MjSpec.from_file(model_path).compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

# 2) Convert the model and data into Warp-side structures
m = mjwarp.put_model(mjm)
d = mjwarp.put_data(mjm, mjd, nworld=1, nconmax=1000, njmax=5000)

# 3) Warm up once, then capture a Warp graph for repeated execution
mjwarp.step(m, d)
mjwarp.step(m, d)
with wp.ScopedCapture() as capture:
    mjwarp.step(m, d)
graph = capture.graph

# 4) Launch the captured graph and copy results back when needed
for _ in range(100):
    wp.capture_launch(graph)
    wp.synchronize()
    mjwarp.get_data_into(mjd, mjm, d)
```

## What This Example Shows

- `mujoco.MjSpec.from_file(...).compile()` produces a compiled `MjModel`, stored here in `mjm`.
- `mujoco.MjData(mjm)` allocates the corresponding runtime `MjData` structure on the host.
- `mjwarp.put_model(mjm)` places the host-side `MjModel` onto device and returns an `mjwarp.Model`.
- `mjwarp.put_data(mjm, mjd, ...)` places the host-side `MjData` onto device and returns an `mjwarp.Data`.
- `mjwarp.Model` and `mjwarp.Data` mirror their MuJoCo counterparts conceptually, but store Warp arrays on device and may omit fields associated with unsupported features.
- `mjwarp.step(m, d)` advances the device-side simulation state by executing the MJWarp simulation pipeline.
- `mjwarp.get_data_into(mjd, mjm, d)` copies one world from device back into an existing host-side `MjData` when MuJoCo-side access is needed.

## Notes

- `nworld` specifies how many parallel simulation worlds are allocated in the Warp-side data object.
- `nconmax` is the expected number of contacts per world. The total number of contacts across all worlds can exceed `nconmax` for an individual world as long as the overall batch allocation is not exceeded.
- `njmax` is the maximum number of constraints per world, and this limit is enforced per world.
- `mujoco.mj_forward(mjm, mjd)` is used here to start from a consistent host-side state before transferring data to device.
- `get_data_into(...)` is only needed when downstream code, logging, or visualization must read host-side MuJoCo state.



For basic usage, `comfree_warp` keeps the same overall call structure as `mujoco_warp`, which is why understanding this page is useful before moving on to the ComFree-specific API extensions.
