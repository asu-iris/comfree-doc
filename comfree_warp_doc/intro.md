# ComFree Sim

```{image} teaser.jpg
:alt: ComFree Warp teaser
:width: 100%
:align: center
```

## Overview

ComFree-Sim is a GPU-parallel analytical contact physics engine for scalable contact-rich robotics simulation and control. It replaces iterative complementarity-based contact resolution with a complementarity-free analytical formulation that computes contact impulses in closed form, enabling near-linear scaling with contact count in dense-contact scenes.

## Why ComFree Warp

Built in Warp and exposed through a MuJoCo-compatible interface, `comfree_warp` can be used as a drop-in backend alternative to MuJoCo Warp (MJWarp). It inherits most of the features provided by MJWarp, while replacing the contact dynamics resolution stage with a complementarity-free contact formulation. This flattened contact-resolution pipeline delivers more than 2x higher throughput than MJWarp while maintaining comparable and tunable physical fidelity.

## Tradeoffs and Recommendation

While `comfree_warp` excels in speed, it may also be more prone to instability and require more manual tuning than the more mature and well-maintained MuJoCo stack. Because `comfree_warp` replaces the original contact solver with the ComFree formulation, some contact-dynamics behavior differences relative to MuJoCo are expected. In general, we find ComFree tends to produce more dynamic contact behavior than MuJoCo. 

At the same time, ComFree is an evolving open project, and we warmly welcome contributions from the community to improve robustness, expand features, and broaden application support.

## Potential Applications

This makes `comfree_warp` well suited for workloads where both speed and customizable contact matter, including:

- scalable contact-dense simulation
- low-latency (simulation) predictive control for contact-rich robotics
- differnetiable contact dynamics learning (real-to-sim)
- real-time predictive control for dexterous manipulation (graident-based, coming soon)

## Simulation Pipeline

The flowchart below summarizes the main computation path and highlights how the analytical contact model fits into the overall simulation pipeline.

```{image} comfree_flowchart_5.jpg
:alt: ComFree workflow chart
:width: 80%
:align: center
```

## Acknowledgments

We acknowledge the MuJoCo team and the Newton team for open-sourcing the MuJoCo simulator and the Warp language. Their tools and engineering work made the development of `comfree_warp` possible.

## Documentation Guide

```{tableofcontents}

```
