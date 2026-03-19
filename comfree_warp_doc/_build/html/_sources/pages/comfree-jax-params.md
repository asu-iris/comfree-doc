# ComFree Jax Parameters

ComFree Jax exposes cost and constraint parameters that influence the contact solver.

## Common parameters

While the exact parameter names can vary across versions, the typical ComFree control parameters include:

- `comfree_stiffness`: Contact stiffness term (higher = stiffer contacts)
- `comfree_damping`: Contact damping term (higher = more dissipation)

The parameters are usually passed when constructing or configuring the solver. For example:

```python
m = mjx.put_model(mjm, comfree_stiffness=0.1, comfree_damping=0.001)
```

## Per-environment parameterization

For batch simulations or multi-world setups, these parameters can often be arrays to support per-instance tuning.

Refer to the `mjx` source code in `comfree_jax/mjx/_src/` for the most up-to-date parameter names and defaults.
