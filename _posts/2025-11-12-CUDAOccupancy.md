---
title: CUDA(2) — Occupancy
by: hdimmfh
date: 2025-11-12 10:55:00 +0900
categories: [GPU, CUDA]
tags: [GPU, CUDA, Occupancy]
---

🔍**Understanding Occupancy in CUDA**

## ① What Is Occupancy?

**Occupancy** refers to the ratio between the number of *active warps*
running on a Streaming Multiprocessor (SM) and the *maximum number of
warps* that the SM can theoretically support.

![Calculating Theoretical Occupacy](https://github.com/hdimmfh/hdimmfh.github.io/blob/main/_data/blog-img-repo/img/gpu/architecture/sm_block_thread_occupancy.png?raw=true)
*Figure 1. Theoretical Occupacy.*

> **Formula:**\
> Occupancy = `Active Warps per SM` / `Maximum Warps per SM`

It measures how effectively the GPU's parallel resources are being
utilized.

---

## ② Example

For instance, the NVIDIA A100 GPU can host up to `64 warps per SM`.\
If your kernel launches only `32 warps on each SM`, then:

> Occupancy = 32 / 64 = 50%

That means only half of the SM's execution slots are actively used.

---

## ③ Why It Matters

Higher occupancy generally improves **latency hiding**.\
While one warp waits for memory access, another can execute --- keeping
the SM busy.

However, occupancy is **not the sole performance indicator**.

-   ✓ Higher occupancy → better latency hiding and concurrency\
-   Too high occupancy → less register/shared memory per thread,
    which can reduce efficiency

> In practice, `60~80% occupancy` often yields the best performance --- not 100%.

---

## ④ What Limits Occupancy

| Factor | Description |
|--------|--------------|
| **Threads per block** | More threads per block → fewer blocks can fit on the same SM |
| **Registers per thread** | More register usage → fewer warps can be active simultaneously |
| **Shared memory per block** | Large shared memory per block → fewer concurrent blocks per SM |

---

## ⑤ Theoretical vs Achieved Occupancy

-   **Theoretical Occupancy**\
    Calculated from kernel launch parameters and SM hardware limits
    (e.g., threads, warps, blocks).\
    It assumes ideal conditions with all available resources used
    efficiently.

-   **Achieved Occupancy**\
    Measured after execution, showing how many warps were actually
    active during runtime.\
    Tools like `nvprof` or **Nsight Compute** can report this metric.

> **Important Note:**\
> Theoretical occupancy always assumes the **maximum blocks per SM**.\
> Therefore, it is **usually higher than the achieved occupancy**
> measured in real runs.

---

## Summary

| Type | Description |
|------|--------------|
| **Occupancy** | Ratio of active to maximum warps per SM |
| **Theoretical Occupancy** | Estimated value based on hardware limits |
| **Achieved Occupancy** | Real value measured at runtime |
| **Purpose** | Evaluates GPU parallel efficiency and latency hiding capability |
| **Caution** | 100% occupancy ≠ best performance |

---

## References

-   NVIDIA CUDA Programming Guide, v12.4
-   Nsight Compute Metric: `sm__warps_active.avg.pct_of_peak_sustained_active`
-   NVIDIA CUDA Occupancy Calculator
