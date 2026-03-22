# Known Issues

This page focuses on the main practical caveats users should keep in mind when working with `comfree_warp`.

## 1. Large Initial Penetration Can Still Be Sensitive

Large initial interpenetration can still lead to sensitive or abrupt contact behavior, especially at the beginning of a rollout. The main reason is that deep initial penetration can generate very large contact forces in the first few simulation steps, which can in turn produce visible jumps or abrupt corrections.

In practice:

- avoid deeply intersecting initial configurations when possible
- if a body appears to "pop out" or jump strongly at startup, large initial penetration is one of the first things to inspect

Cleaner initial contact states usually lead to more predictable behavior.

## 2. Most Non-Solver Issues Are Inherited from MJWarp

`comfree_warp` changes the solver path, but much of the surrounding simulation stack is still inherited from vendored `mujoco_warp`. This means that many non-solver issues are likely to come from `mujoco_warp`, not from the ComFree solver itself. A typical example is collision detection.

More generally, behaviors related to geometry processing, contact generation, broadphase, narrowphase, rendering, and other non-solver infrastructure may still reflect MJWarp behavior.

For MJWarp-related issues, limitations, or upstream fixes, refer to the official repository:

- [google-deepmind/mujoco_warp](https://github.com/google-deepmind/mujoco_warp)

As a rule of thumb:

- if the issue is not obviously solver-related, it is often inherited from `mujoco_warp`
- `comfree_warp` periodically pulls upstream MJWarp changes to update the vendored `mujoco_warp` version

## 3. Very Small Dynamics Drift Can Occur

Because ComFree uses an impedance-style contact handling mechanism rather than a complementarity or optimization-complete solver,  very small dynamics drift can occur in some cases. This should be treated as an expected tradeoff of the complementarity-free formulation rather than as an automatic sign of a bug.

More broadly, because `comfree_warp` replaces the original MJWarp contact solver with ComFree, some differences in contact-dynamics behavior are expected relative to MJWarp. Differences in contact force profiles, penetration recovery, sticking/sliding transitions, or settling behavior do not automatically indicate an implementation problem.

This can potentially be fixed or further reduced in future development.



## 4. Step Size Requires More Care Than in MJWarp

In our experience, `comfree_warp` does not support time steps as large as MJWarp in all cases, although it can still handle reasonably large step sizes. This is because  impedance-style contact response is using the timestep to do penetration prediction. As the time step changes, the effective contact behavior changes as well, so the same stiffness and damping values may no longer produce the same simulation quality.

In practice:

- if you increase the simulation time step significantly, re-tune `comfree_stiffness` and `comfree_damping`
- if behavior becomes unstable or overly soft after changing the time step, check contact tuning before assuming a deeper issue

## Practical Guidance

If results look off, check these first:

1. Is the model starting with large initial penetration?
2. Is the issue solver-related, or is it more likely inherited from `mujoco_warp`?
4. Have you re-tuned `comfree_stiffness` and `comfree_damping`?
