# How ComFree Warp Differs from MuJoCo Warp

> **See also:** [ComFree Jax difference guide](comfree-jax-difference)
>
`comfree_warp` is designed to feel familiar to users of `mujoco_warp`. For most day-to-day usage, it should feel like the same interface, with ComFree-specific behavior added underneath.

````{important}
The quickest way to think about `comfree_warp` is:

- the overall workflow is almost the same as `mujoco_warp`
- many top-level function names are the same
- for most core APIs, the input and output usage stays the same
- the most important behavioral differences are in the ComFree solver path

So `comfree_warp` should mostly read like a drop-in alternative, not like a completely different API.
````

## Start from the Public API

The cleanest way to understand the differences is to start from `comfree_warp/__init__.py`, because that file shows what the package exposes to users.

At the top level, the exported APIs fall into two groups.

#### ComFree-provided APIs

These are the APIs that `comfree_warp` provides from its own implementation:

- `put_model`
- `put_data`
- `make_data`
- `get_data_into`
- `reset_data`
- `step`
- `forward`
- `Model`
- `Data`
- `make_constraint`

These are the most important APIs to compare against `mujoco_warp`. However, for most of them, the user-facing call pattern remains the same.

#### Unchanged MJWarp Re-exports

Many other public names are re-exported directly from `mujoco_warp`, including:

- collision and broadphase helpers
- rendering and ray-query helpers
- smooth dynamics helpers
- sensor helpers
- shared types and enums

These are mainly preserved so that the package stays familiar and compatible with MJWarp-style code.

## What Stays the Same

The main simulation workflow remains almost the same:

1. Load a MuJoCo model with `mujoco`.
2. Convert it with `put_model` and `put_data`.
3. Run simulation with `step` or `forward`.
4. Copy data back with `get_data_into` if needed.

That is why most MJWarp code can be adapted with only very small changes.

### Typical Migration

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
    comfree_stiffness=0.1,
    comfree_damping=0.001,
)
d = comfree_warp.put_data(mjm, mjd, nworld=1, nconmax=1000, njmax=5000)
comfree_warp.step(m, d)
comfree_warp.get_data_into(mjd, mjm, d)
```

## What Changes

Before looking at the detailed differences, the most important compatibility point is this:

````{note}
For `put_data`, `make_data`, `get_data_into`, `reset_data`, `step`, `forward`, `Model`, `Data`, and `make_constraint`, `comfree_warp` is intended to keep the same user-facing role as `mujoco_warp`.

In practice, the main API-level extension users usually notice is on `put_model(...)`, which adds `comfree_stiffness` and `comfree_damping`.

For the rest, the function names, expected arguments, and overall input/output usage are intended to feel the same, even though the internal implementation may be extended or replaced.
````

### `step` and `forward`

These are the most important implementation differences.

Although the function names and user-facing usage are the same, `comfree_warp.step(...)` and `comfree_warp.forward(...)` do not run the original MJWarp physics pipeline. They run the ComFree solver and ComFree forward pipeline instead.

If you want to know where the simulator behavior really changes, start here.

### `put_model`

`put_model(...)` is the clearest user-visible API extension.

`comfree_warp.put_model(...)` extends the MJWarp-style API by adding:

- `comfree_stiffness`
- `comfree_damping`

These are the main new user-facing inputs for ComFree contact behavior.

```python
import comfree_warp

m = comfree_warp.put_model(
    mjm,
    comfree_stiffness=0.1,
    comfree_damping=0.001,
)
```

### `put_data`, `make_data`, and `get_data_into`

These functions still follow the MJWarp workflow and are meant to be used in the same way, but they extend the underlying Warp-side data structures with extra ComFree-related fields needed by the solver.

In normal usage, this is handled automatically. From the user side, they should still feel like the same API calls.

### `reset_data`

`reset_data` keeps the same role as in MJWarp and is effectively used in the same way.

## Function-by-Function Summary

| Function | Relationship to `mujoco_warp` | What it means for users |
|---|---|---|
| `put_model` | Extended | Same role as MJWarp, with extra ComFree parameters |
| `put_data` | Extended | Same usage pattern as MJWarp |
| `make_data` | Extended | Same usage pattern as MJWarp |
| `get_data_into` | Extended | Same usage pattern as MJWarp |
| `reset_data` | Delegated | Same usage pattern as MJWarp |
| `step` | Replaced internally | Same user-facing call, different solver behavior underneath |
| `forward` | Replaced internally | Same user-facing call, different forward behavior underneath |
| many other helpers exported in `__init__.py` | Unchanged re-export | Mostly kept for compatibility and familiarity |

## Reading Guide

If you already know MJWarp, the most useful mental model is:

- `comfree_warp` keeps the same outer workflow
- for most core APIs, the user-facing input/output pattern stays the same
- ComFree mainly changes the engine-facing solver path
- many helper APIs are still available in familiar form

That is why `comfree_warp` feels close to MJWarp while still being a different simulator backend.
