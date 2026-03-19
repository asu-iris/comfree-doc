# Contact Parameter Settings

> **See also:** [ComFree Jax parameter docs](comfree-jax-params)

Two parameters control ComFree contact behavior, both passed to `put_model`:

```python
m = comfree_warp.put_model(
    mjm,
    comfree_stiffness=0.1,   # default
    comfree_damping=0.001,   # default
)
```

| Parameter | Effect | Tune when... |
|---|---|---|
| `comfree_stiffness` | Resistance to penetration | Contact feels too soft/permissive → increase |
| `comfree_damping` | Energy dissipation during contact | Contact is too bouncy/oscillatory → increase |

## Multi-World (Per-Environment) Parameters

Both parameters accept a scalar or a list. In multi-world simulation, world `i` uses bucket `i % k`, where `k` is the list length:

```python
# 3 stiffness buckets, 2 damping buckets
m = comfree_warp.put_model(
    mjm,
    comfree_stiffness=[0.1, 0.08, 0.12],
    comfree_damping=[0.001, 0.002],
)
```

With `nworld=8` and `k=3`: world 0→bucket 0, world 1→bucket 1, world 2→bucket 2, world 3→bucket 0, ...

**Common patterns:**
- `k=1` (scalar): all worlds share one value
- `k=nworld`: each world gets its own value — useful for parameter sweeps
