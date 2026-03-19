# ComFree Contact Parameters

This page explains the two main ComFree contact parameters exposed by `comfree_warp.put_model(...)`:

- `comfree_stiffness`
- `comfree_damping`

These parameters control the strength and dissipation of the ComFree contact response. They can be provided either as scalars or as vectors for multi-world simulation.

## Parameter Summary

### `comfree_stiffness`

`comfree_stiffness` controls how strongly the contact response pushes against predicted penetration.

In practice:

- higher `comfree_stiffness` produces a stiffer contact response
- lower `comfree_stiffness` produces a softer, more compliant contact response

If contacts feel too soft or objects penetrate too much, this is usually the first parameter to increase.

### `comfree_damping`

`comfree_damping` controls how much damping is applied in the contact response.

In practice:

- higher `comfree_damping` usually reduces oscillation and bounce
- lower `comfree_damping` usually produces a more lively but potentially less stable response

If contacts feel too bouncy or oscillatory, this is usually the first parameter to increase.

## How They Are Used

In the current implementation, both parameters are stored as device arrays on the model and are consumed in the ComFree contact-force computation.

At a high level:

- `comfree_stiffness` scales the penetration-restoring part of the contact response
- `comfree_damping` scales the velocity-damping part of the contact response

These two parameters are usually tuned together.

## Defaults

If not provided to `put_model(...)`:

- `comfree_stiffness` defaults to `0.2`
- `comfree_damping` defaults to `0.001`

## Basic Usage

Pass the parameters directly into `comfree_warp.put_model(...)`:

```python
import mujoco
import comfree_warp

mjm = mujoco.MjSpec.from_file("model.xml").compile()

m = comfree_warp.put_model(
    mjm,
    comfree_stiffness=0.2,
    comfree_damping=0.001,
)
```

## Scalar and Vector Inputs

Both parameters accept:

- a scalar, meaning the same value is used for every world
- a vector, meaning values are assigned across worlds by bucket

For example:

```python
m = comfree_warp.put_model(
    mjm,
    comfree_stiffness=[0.2, 0.15, 0.25],
    comfree_damping=[0.001, 0.002],
)
```

In this case:

- stiffness has 3 buckets
- damping has 2 buckets

## Multi-World Bucket Mapping

In multi-world simulation, the current implementation selects parameter values by:

`bucket_id = world_id % k`

where:

- `world_id` is the simulated world index
- `k` is the vector length of `comfree_stiffness` or `comfree_damping`

This means parameter buckets are reused cyclically across worlds.

## Common Cases

### Shared Setting Across All Worlds

If you pass a scalar such as `0.2`, it behaves like a length-1 vector, so every world receives the same value.

### One Setting Per World

If the vector length equals `nworld`, each world receives its own parameter bucket.

Example with `nworld = 4`:

- world 0 -> bucket 0
- world 1 -> bucket 1
- world 2 -> bucket 2
- world 3 -> bucket 3

### Periodic Reuse Across Worlds

If the vector length is smaller than `nworld`, the buckets repeat periodically.

Example with `nworld = 8` and `k = 3`:

- world 0 -> bucket 0
- world 1 -> bucket 1
- world 2 -> bucket 2
- world 3 -> bucket 0
- world 4 -> bucket 1
- world 5 -> bucket 2
- world 6 -> bucket 0
- world 7 -> bucket 1

### Extra Buckets

If the vector length is larger than `nworld`, only the first `nworld` buckets are used in that run.

## Practical Recommendations

- start with the defaults: `comfree_stiffness = 0.2`, `comfree_damping = 0.001`
- increase `comfree_stiffness` first if contacts are too soft
- increase `comfree_damping` first if contacts are too oscillatory or too bouncy
- use a scalar when you want identical contact behavior across all worlds
- use a length-`nworld` vector when running per-world sweeps or ablations
- for tasks that need task-specific contact tuning, these parameters can also be tuned with learning or automated search

## Example

```python
import mujoco
import warp as wp
import comfree_warp

nworld = 4
njmax = 5000
nconmax = 1000

mjm = mujoco.MjSpec.from_file("benchmark/test_data/primitives.xml").compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

m = comfree_warp.put_model(
    mjm,
    comfree_stiffness=[0.2, 0.15, 0.25, 0.3],
    comfree_damping=[0.001, 0.002, 0.0015, 0.0008],
)
d = comfree_warp.put_data(
    mjm,
    mjd,
    nworld=nworld,
    nconmax=nconmax,
    njmax=njmax,
)

comfree_warp.step(m, d)
comfree_warp.step(m, d)

with wp.ScopedCapture() as capture:
    comfree_warp.step(m, d)

graph = capture.graph
```
