# MJWarp Preliminary

> **See also:** [ComFree Jax documentation](comfree-jax-usage)

`mujoco_warp` is a Warp-based GPU simulator for MuJoCo models. `comfree_warp` follows the same interface, so understanding the basic MJWarp workflow transfers directly.

For full upstream docs, see [mujoco-warp](https://mujoco.readthedocs.io/en/latest/mjwarp/).

## Minimal Example

```python
import mujoco
import warp as wp
import mujoco_warp as mjwarp

# 1) Load model
mjm = mujoco.MjSpec.from_file("model.xml").compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

# 2) Convert to Warp
m = mjwarp.put_model(mjm)
d = mjwarp.put_data(mjm, mjd, nworld=1, nconmax=1000, njmax=5000)

# 3) Capture graph for fast repeated stepping
mjwarp.step(m, d)
mjwarp.step(m, d)
with wp.ScopedCapture() as capture:
    mjwarp.step(m, d)
graph = capture.graph

# 4) Run steps; sync back to CPU when needed
for _ in range(100):
    wp.capture_launch(graph)
    wp.synchronize()
    mjwarp.get_data_into(mjd, mjm, d)
```

## Notes

- `nconmax` / `njmax` set the contact and constraint buffer sizes.
- `get_data_into` copies GPU state back to CPU `MjData` — use sparingly.
- All core calls (`put_model`, `put_data`, `step`, `forward`, `reset_data`, `get_data_into`) carry over to `comfree_warp` with the same signatures.
