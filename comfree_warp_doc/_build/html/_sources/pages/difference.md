# How ComFree Warp Differs from MJWarp

`comfree_warp` is designed to stay close to `mujoco_warp` at the user-facing API level, while changing the internal contact-resolution pipeline.

The most useful mental model is:

- the outer workflow is almost the same as MJWarp
- many public helper APIs are still re-exported unchanged
- the main user-facing extension is on `put_model(...)`
- the main behavioral change is in `forward(...)` and `step(...)`

## High-Level Summary

If you already know MJWarp, the typical workflow still looks like:

1. Load a MuJoCo model and `MjData`.
2. Move them to device with `put_model(...)` and `put_data(...)`.
3. Run `step(...)` or `forward(...)`.
4. Pull state back with `get_data_into(...)` when needed.

That is why most MJWarp code can be adapted with only small changes.

````{important}
`comfree_warp` should be thought of as a near drop-in MJWarp alternative, not as a completely separate simulator interface.

However, it is not just a rename of MJWarp. The ComFree wrapper changes model parameters, data fields, and the forward/step implementation.
````

## Public API Structure

Looking at `comfree_warp/__init__.py`, the exported names fall into two categories.

### ComFree-Provided API

These are the main APIs whose behavior is defined by the ComFree wrapper or ComFree core:

- `put_model`
- `put_data`
- `make_data`
- `get_data_into`
- `reset_data`
- `forward`
- `step`
- `Model`
- `Data`
- `make_constraint`

### Unchanged MJWarp Re-Exports

Many other public helpers are re-exported directly from vendored `mujoco_warp`, including:

- collision and broadphase helpers
- rendering and ray-query helpers
- smooth-dynamics helpers
- sensor helpers
- support utilities

This is one reason the package still feels familiar to MJWarp users.

## Comparison at a Glance

The practical comparison is:

- same overall workflow
- same broad role for the main host-to-device and device-to-host APIs
- extra ComFree parameters on `put_model(...)`
- extra ComFree buffers in device-side data
- different internal contact-resolution behavior in `forward(...)` and `step(...)`

## What Stays the Same

The following points are intentionally preserved:

- the overall host-to-device workflow
- the role of `put_model`, `put_data`, `make_data`, `step`, `forward`, and `get_data_into`
- the basic device-side `Model` / `Data` abstraction
- many non-solver helper APIs re-exported from MJWarp

In day-to-day usage, this means existing MJWarp scripts usually require only limited changes in setup code.

## What Is Extended

### `put_model(...)`

This is the clearest user-visible API extension.

In the current wrapper implementation:

- `put_model(...)` first calls vendored `mujoco_warp.put_model(...)`
- it then attaches `m.comfree_stiffness`
- it also attaches `m.comfree_damping`

Both parameters are stored as device-side Warp arrays and may be provided as scalars or vectors.

Example:

```python
import comfree_warp

m = comfree_warp.put_model(
    mjm,
    comfree_stiffness=0.2,
    comfree_damping=0.001,
)
```

### `put_data(...)` and `make_data(...)`

These functions still follow the MJWarp allocation pattern, but the wrapper ensures that additional ComFree-specific device buffers exist.

From the current `api.py`, those extra fields include:

- `d.efc.efc_dist`
- `d.efc.efc_mass`
- `d.qvel_smooth_pred`
- `d.qfrc_total`

So the usage pattern remains familiar, but the resulting device data is extended to support the ComFree solver path.

### `get_data_into(...)`

This function still plays the same role as in MJWarp: it copies one selected world from device into an existing host-side `mujoco.MjData`.

The wrapper keeps the familiar call pattern while ensuring the ComFree-extended data object remains usable.

### `Model` and `Data`

`comfree_warp` also exposes `Model` and `Data` from the ComFree type definitions rather than directly exposing the vendored MJWarp versions.

Conceptually, they play the same role as the MJWarp device-side structures, but they include the extra fields needed by the ComFree solver path.

## What Is Replaced Internally

### `forward(...)`

This is one of the main behavioral differences.

In `comfree_warp`, `forward(...)` maps to `forward_comfree(...)`, not to the original MJWarp forward path.

The ComFree forward pipeline still reuses much of the MJWarp smooth-dynamics stack, but replaces the contact-resolution stage. In the current source, the pipeline includes:

- smooth kinematics and dynamics setup
- collision detection
- ComFree constraint construction
- smooth acceleration computation
- ComFree contact-force assembly
- acceleration solve using the total generalized force

### `step(...)`

`step(...)` maps to `step_comfree(...)`, not to the original MJWarp step function.

In the current implementation:

- `step(...)` first runs `forward_comfree(...)`
- it then integrates using the selected device-side integrator path
- the ComFree step path currently supports Euler and `IMPLICITFAST`

So although the function name and calling style remain familiar, the simulation behavior should not be expected to be numerically identical to MJWarp.

## Typical Migration

```python
# MJWarp
import mujoco_warp as mjwarp

m = mjwarp.put_model(mjm)
d = mjwarp.put_data(mjm, mjd, nworld=1, nconmax=1000, njmax=5000)
mjwarp.step(m, d)
mjwarp.get_data_into(mjd, mjm, d)

# ComFree Warp
import comfree_warp

m = comfree_warp.put_model(
    mjm,
    comfree_stiffness=0.2,
    comfree_damping=0.001,
)
d = comfree_warp.put_data(mjm, mjd, nworld=1, nconmax=1000, njmax=5000)
comfree_warp.step(m, d)
comfree_warp.get_data_into(mjd, mjm, d)
```

## Function-by-Function Summary

| Function | Relationship to MJWarp | User-facing meaning |
|---|---|---|
| `put_model` | Extended | Same role as MJWarp, with additional ComFree contact parameters |
| `put_data` | Extended | Same allocation/copy role, with extra ComFree device buffers |
| `make_data` | Extended | Same allocation role, with extra ComFree device buffers |
| `get_data_into` | Extended wrapper | Same role as MJWarp for copying one world back to host |
| `reset_data` | Delegated | Same role and usage pattern as MJWarp |
| `forward` | Replaced internally | Same top-level role, different contact-resolution pipeline |
| `step` | Replaced internally | Same top-level role, different solver behavior underneath |
| `Model` / `Data` | Extended types | Same conceptual role, with extra ComFree fields |
| many helper APIs | Unchanged re-export | Preserved mainly for compatibility and familiarity |

## Practical Guidance

If you are coming from MJWarp, the key points are:

- start by assuming the workflow is the same
- expect `put_model(...)` to be the main API extension
- expect `step(...)` and `forward(...)` to be the main behavioral differences
- expect many lower-level helper APIs to still be available unchanged

That combination is what makes `comfree_warp` feel familiar at the interface level while still being meaningfully different as a simulator backend.
