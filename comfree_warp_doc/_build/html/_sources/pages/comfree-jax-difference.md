# ComFree Jax vs ComFree Warp

Both ComFree Jax and ComFree Warp implement the ComFree contact solver, but they target different execution frameworks:

- **ComFree Warp** (`comfree_warp`) is built on top of Warp and integrates with MuJoCo/Warp for fast simulation on CPU/GPU.
- **ComFree Jax** (`mjx`) is designed for JAX users, enabling JIT compilation, automatic differentiation, and access to JAX devices (CPU/GPU/TPU).

### Key differences

- **Execution model**
  - `comfree_warp`: uses Warp kernels and relies on `wp.capture_launch` for fast repeated execution.
  - `comfree_jax`: uses JAX `jit` and `jit`-compiled functions like `step_comfree`.

- **Use cases**
  - `comfree_warp`: best when you want a drop-in MuJoCo/Warp replacement.
  - `comfree_jax`: best when you need gradients, JIT-compiled pipelines, or integration with JAX ML stacks.

### Shared concepts

Both packages follow a similar workflow:
1. Load a MuJoCo model
2. Convert/compile the model for the respective backend
3. Step the simulation
4. Retrieve state for analysis or visualization
