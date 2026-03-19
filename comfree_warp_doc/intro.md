# ComFree Sim

`comfree-sim` is a GPU-parallelized contact physics engine for scalable robotics simulation. It replaces iterative complementarity-based contact resolution with a complementarity-free analytical formulation that computes contact impulses in closed form, achieving near-linear scaling with contact count.

Built in Warp with a MuJoCo-compatible interface, it supports unified 6D contact modeling (tangential, torsional, and rolling friction) and targets applications such as large-scale simulation, real-time MPC, and dexterous manipulation.

## ComFree Warp

```{tableofcontents}

```
