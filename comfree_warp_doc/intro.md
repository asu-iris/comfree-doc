# ComFree Sim

`comfree-sim` is a GPU-parallelized analytical contact physics engine designed for scalable contact-rich robotics simulation and control. It replaces iterative complementarity-based contact resolution with a complementarity-free analytical formulation that computes contact impulses in closed form, enabling near-linear scaling with contact count in dense-contact scenes.

Built in Warp and exposed through a MuJoCo-compatible interface, `comfree-sim` can be used as a drop-in backend alternative to MJWarp. It supports unified 6D contact modeling, including tangential, torsional, and rolling friction, and is aimed at applications such as large-scale simulation, real-time model predictive control, and dexterous manipulation.

## ComFree Warp

```{tableofcontents}

```
