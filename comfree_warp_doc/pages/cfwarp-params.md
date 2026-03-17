# ComFree Contact Parameter Settings

> **See also:** [ComFree Jax parameter docs](comfree-jax-params)
>
This page explains the two main ComFree contact parameters:

- `comfree_stiffness`
- `comfree_damping`

It starts with the physical intuition behind them, then explains how to pass them into `comfree_warp.put_model(...)`, and finally describes how vector-valued settings are assigned across multiple worlds.

## Intuition First

### `comfree_stiffness`

`comfree_stiffness` controls how strongly the contact resists penetration.

A useful intuition is:

- higher `comfree_stiffness` means a stiffer contact response
- lower `comfree_stiffness` means a softer, more compliant contact response

So if contact feels too soft or too permissive, stiffness is usually the first parameter to increase.

### `comfree_damping`

`comfree_damping` controls how much dissipation is added during contact.

A useful intuition is:

- higher `comfree_damping` usually reduces oscillation and bounce
- lower `comfree_damping` usually allows a more lively or oscillatory response

So if contact feels too bouncy or unstable, damping is usually the first parameter to increase.

## Practical Interpretation

In short:

- `comfree_stiffness` mainly changes how hard the contact pushes back
- `comfree_damping` mainly changes how much motion is damped out during contact

These two parameters are typically tuned together.

## Imports

```python
import mujoco
import warp as wp
import comfree_warp
```

## Setting the Parameters

Pass the parameters into `comfree_warp.put_model(...)`:

```python
m = comfree_warp.put_model(
    mjm,
    comfree_stiffness=0.1,
    comfree_damping=0.001,
)
```

Both parameters accept either:

- a scalar, meaning the same value is used everywhere
- a vector, meaning values are assigned by bucket across worlds

For example:

- `comfree_stiffness = [0.1, 0.08, 0.12]`
- `comfree_damping = [0.001, 0.002]`

This means stiffness has 3 buckets and damping has 2 buckets.

## Defaults

If not provided:

- `comfree_stiffness` defaults to `0.1`
- `comfree_damping` defaults to `0.001`


## Per-Environment Parameter Parallelization

`comfree_warp` supports per-environment parameter bucketization in multi-world simulation.

The same modulo mapping pattern can be used for parameters that are stored as world-bucketed vectors.

## Bucket Mapping Rule

In multi-world simulation, the solver selects the parameter bucket with:

`bucket_id = world_id % k`

where:

- `world_id` is the simulated world index (`0..nworld-1`)
- `k` is the vector length of `comfree_stiffness` or `comfree_damping`

This means parameter values are reused cyclically across worlds.

## Common Cases

### Case A: Scalar or length 1

If you pass a scalar like `0.1`, it behaves like a vector of length `1`.

So every world uses the same value because:

`world_id % 1 == 0`

### Case B: Vector length equals `nworld`

If `k == nworld`, each world gets its own bucket.

Example with `nworld=4`, `k=4`:

- world 0 -> bucket 0
- world 1 -> bucket 1
- world 2 -> bucket 2
- world 3 -> bucket 3

### Case C: Vector shorter than `nworld`

If `k < nworld`, buckets repeat periodically.

Example with `nworld=8`, `k=3`:

- world 0 -> bucket 0
- world 1 -> bucket 1
- world 2 -> bucket 2
- world 3 -> bucket 0
- world 4 -> bucket 1
- world 5 -> bucket 2
- world 6 -> bucket 0
- world 7 -> bucket 1

### Case D: Vector longer than `nworld`

If `k > nworld`, only the first `nworld` buckets are used in that run.

Any extra tail buckets are valid, but unused unless `nworld` increases later.

## Practical Recommendation

- use `k=1` when you want one shared contact setting across all worlds
- use `k=nworld` when you want per-world tuning for sweeps or ablation experiments

## Example

```python
# From comfree_warp/test_headless.py
nworld = 1
njmax = 5000
nconmax = 1000

model_path = "benchmark/test_data/primitives.xml"
mjm = mujoco.MjSpec.from_file(model_path).compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

comfree_stiffness_vec = [0.1, 0.08, 0.12]
comfree_damping_vec = [0.001, 0.002]

m = comfree_warp.put_model(
    mjm,
    comfree_stiffness=comfree_stiffness_vec,
    comfree_damping=comfree_damping_vec,
)
d = comfree_warp.put_data(mjm, mjd, nworld=nworld, nconmax=nconmax, njmax=njmax)

comfree_warp.step(m, d)
comfree_warp.step(m, d)

with wp.ScopedCapture() as capture:
    comfree_warp.step(m, d)

graph = capture.graph
```
