# Known Issues

> **See also:** [ComFree Jax known issues](comfree-jax-known-issues)

## Solver differences

`comfree_warp.step` and `forward` use the ComFree solver, not the original MJWarp pipeline. Trajectory differences between the two backends are expected and not a bug.

Some MuJoCo XML constraint settings (e.g. `frictionloss`) may behave differently because ComFree uses a different constraint formulation. Torsional and rolling friction coefficients may also require re-tuning.

## Large initial penetration

Avoid starting simulations with significant geometry interpenetration. If you see abrupt contact responses at the start of a rollout, check the initial model configuration first.

## CPU-GPU synchronization cost

`get_data_into(...)` copies GPU state to CPU and is relatively expensive. Keep simulation on the GPU and call `get_data_into` only when you need CPU-side access.

## Multi-world parameter vectors

When `comfree_stiffness` or `comfree_damping` are lists, worlds are assigned by `world_id % k`. See [Contact Parameter Settings](cfwarp-params.md) for details.

## Version compatibility

The codebase includes compatibility logic for:
- `warp-lang >= 1.12`
- `mujoco == 3.5.0`

If something fails at import or setup, check your MuJoCo and Warp versions first.

---

**Quick checklist** when something seems wrong:
1. Are `comfree_stiffness` / `comfree_damping` set to sensible values?
2. Does the model start with large penetration?
3. Are you calling `get_data_into` more often than necessary?
4. Are MuJoCo and Warp versions compatible?
