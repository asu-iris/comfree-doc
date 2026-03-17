# Known Issues

This page collects practical limitations and caveats that users should keep in mind when working with `comfree_warp`.

The goal here is not to list every internal TODO in the source tree, but to document the issues that are most useful from a user perspective.

## 1. `put_model(...)` is the main user-visible API extension

Compared with plain `mujoco_warp`, the main API-level extension most users will notice is that `comfree_warp.put_model(...)` adds:

- `comfree_stiffness`
- `comfree_damping`

If these are omitted, defaults are used. If results are unexpectedly soft, stiff, or bouncy, this is one of the first places to check.

## 2. `step(...)` and `forward(...)` keep the same interface but not the same implementation

`comfree_warp.step(...)` and `comfree_warp.forward(...)` use the ComFree solver path, not the original MJWarp solver path.

In practice, this means:

- the function names and overall usage stay familiar
- the simulation behavior is not expected to be numerically identical to MJWarp

If you are comparing results between the two backends, differences in trajectories or contact behavior are not necessarily a bug by themselves.

## 3. Large initial penetration can still be sensitive

The ComFree constraint implementation contains logic specifically intended to reduce significant jumps when the initial penetration is large.

This suggests an important practical caution:

- large initial interpenetration is still a case worth avoiding when possible
- if you see abrupt contact responses at the beginning of a rollout, check the initial geometry configuration first

In general, cleaner initial contact states lead to more predictable behavior.

## 4. CPU-GPU synchronization can become a bottleneck

Calls such as `get_data_into(...)` copy Warp-side state back to MuJoCo CPU arrays.

This is useful for debugging, logging, or visualization, but it can be expensive if done every step.

Recommendation:

- keep simulation on the GPU as much as possible
- call `get_data_into(...)` only when you actually need MuJoCo-side access

## 5. Multi-world parameter vectors require careful interpretation

When `comfree_stiffness` or `comfree_damping` are passed as vectors, they are assigned to worlds by bucket rather than by arbitrary lookup.

Recommendation:

- read [ComFree Contact Parameter Settings](cfwarp-params.md) carefully before using vector-valued settings
- be explicit about whether you want one shared value, periodic reuse, or one value per world

## 6. Compatibility with external dependencies depends on versions

The current source includes compatibility logic around:

- `warp-lang >= 1.12`
- `mujoco == 3.5.0`

MuJoCo internal data structure updates across versions may introduce compatibility issues in `comfree_warp`.

This means older or newer dependency versions may require extra caution.

We plan to continue adding fixes so that future MuJoCo versions are better supported.

If something behaves unexpectedly at import time or during setup, verify the MuJoCo and Warp versions first.

## 7. Not every internal MJWarp API is the focus of ComFree documentation

`comfree_warp` re-exports a large amount of API surface from MJWarp, but the ComFree docs focus mainly on the engine-facing workflow:

- `put_model`
- `put_data`
- `make_data`
- `get_data_into`
- `reset_data`
- `step`
- `forward`

If you are using deeper MJWarp helper APIs, the interface may still be available, but the ComFree docs may not cover those paths in detail.

## 8. Solver differences affect some MuJoCo XML constraint settings

Because ComFree uses a different constraint solver than MuJoCo's built-in solver, some constraint-based configuration specified directly in MuJoCo XML (for example `joint` attributes such as `frictionloss`) may not have the same effect in `comfree_warp`. We will continue improving coverage of these XML parameters; if you rely on solver-specific settings, please verify behavior and report any regressions.

## 9. Torsional and rolling friction coefficient scaling may differ from MuJoCo

The scale (magnitude) of torsional and rolling friction coefficients can differ between `comfree_warp` and MuJoCo. These coefficients may require fine-tuning before deployment to achieve behavior that matches MuJoCo.

## Practical Advice

If something seems wrong, check these first:

1. Are `comfree_stiffness` and `comfree_damping` set to sensible values?
2. Is the model starting with large penetration?
3. Are you calling `get_data_into(...)` more often than necessary?
4. Are your MuJoCo and Warp versions compatible with the current codebase?

These four checks will explain a large fraction of practical issues encountered in early usage.
