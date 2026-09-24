---
title: Inside PyTorch(4) — How to Analyze GPU Runtime Performance
by: hdimmfh
date: 2026-09-24 18:40:00 +0900
categories: [GPU, PyTorch]
tags: [PyTorch, GPU Profiling, Nsight Systems, Nsight Compute, tlparse, TorchInductor]
---

🔍 **Once a Model Is Compiled, How Do We Know Whether the GPU Is Actually Running Efficiently?**

> After compilation, performance analysis should focus on the steady-state runtime: first identify where GPU time is lost, then locate expensive kernels, and finally trace inefficient kernels back to compiler decisions when necessary.

---

## ① Check Basic GPU Telemetry

Use `nvidia-smi dmon` or NVIDIA DCGM to check GPU/SM utilization, HBM activity, clocks, power, temperature, PCIe, and NVLink activity. This provides the first indication of whether the GPU is underutilized or throttled. 
> 🔗 [NVIDIA DCGM Profiling](https://docs.nvidia.com/datacenter/dcgm/latest/learn/modules/profiling.html)

## ② Profile PyTorch Execution

Use `torch.profiler` and NVTX to identify expensive forward, backward, optimizer, dataloader, and operator regions before moving to lower-level GPU analysis. 
> 🔗 [PyTorch Profiler](https://docs.pytorch.org/docs/stable/profiler)

## ③ Inspect the CPU-GPU Timeline

Use NVIDIA Nsight Systems (`nsys`) to find GPU idle gaps, delayed kernel launches, synchronization, memory copies, and CPU-side stalls. Large gaps between GPU kernels usually indicate that the GPU is waiting for work. 
> 🔗 [NVIDIA Nsight Systems User Guide](https://docs.nvidia.com/nsight-systems/UserGuide/)

## ④ Inspect Multi-GPU Communication

For distributed workloads, use Nsight Systems with NCCL tracing and DCGM to inspect AllReduce, AllGather, ReduceScatter, rank imbalance, and compute-communication overlap. 
> 🔗 [Nsight Systems — NCCL Analysis](https://docs.nvidia.com/nsight-systems/UserGuide/)

## ⑤ Inspect GPU Memory Behavior

Use Nsight Systems GPU Memory tracing and `torch.profiler` to inspect H2D/D2H transfers, allocation/deallocation behavior, peak memory usage, and unnecessary memory movement. 
> 🔗 [Nsight Systems — CUDA GPU Memory Analysis](https://docs.nvidia.com/nsight-systems/UserGuide/)

## ⑥ Investigate Unexplained GPU Idle Time

If GPU gaps cannot be explained by CPU, communication, or memory activity, use `tlparse` and `TORCH_LOGS` to inspect graph breaks, eager fallbacks, recompilations, fusion, and scheduling decisions. 
> 🔗 [PyTorch tlparse / TORCH_TRACE](https://docs.pytorch.org/docs/main/user_guide/torch_compiler/compile/programming_model.observability.html)

## ⑦ Identify Hotspot GPU Kernels

Use Nsight Systems or `torch.profiler` to rank kernels by total GPU execution time. Focus detailed analysis on the small number of kernels responsible for most of the runtime rather than profiling every kernel. 
> 🔗 [PyTorch — Profiling torch.compile Performance](https://docs.pytorch.org/docs/main/user_guide/torch_compiler/torch.compiler_profiling_torch_compile.html)

## ⑧ Analyze Individual Kernel Efficiency

Use NVIDIA Nsight Compute (`ncu`) on hotspot kernels. Check SpeedOfLight/Roofline, compute and memory throughput, Tensor Core utilization, occupancy, registers, shared memory, active/eligible warps, cache/HBM behavior, and warp stall reasons. 
> 🔗 [NVIDIA Nsight Compute Profiling Guide](https://docs.nvidia.com/nsight-compute/ProfilingGuide/)

## ⑨ Trace Kernel Problems Back to the Compiler

If a kernel shows poor hardware efficiency, return to `tlparse` and Inductor logs such as `fusion`, `schedule`, `ir_pre_fusion`, `ir_post_fusion`, and `kernel_code`. Check whether excessive fusion, insufficient fusion, scheduling, or generated Triton code explains the observed GPU behavior. 
> 🔗 [PyTorch Compiler Observability](https://docs.pytorch.org/docs/main/user_guide/torch_compiler/compile/programming_model.observability.html)

## ⑩ Validate Every Optimization

After changing batch size, fusion behavior, Triton configuration, precision, CUDA Graphs, or compiler options, repeat `torch.profiler → Nsight Systems → Nsight Compute` and compare throughput, latency, GPU idle time, hotspot kernel time, and hardware efficiency. 
> 🔗 [PyTorch — Profiling torch.compile Performance](https://docs.pytorch.org/docs/main/user_guide/torch_compiler/torch.compiler_profiling_torch_compile.html)

---

## Key Takeaway

GPU performance analysis should proceed from the system level toward individual kernels and then back toward compiler decisions only when necessary.

```text
DCGM / nvidia-smi
        ↓
torch.profiler + NVTX
        ↓
Nsight Systems
        ↓
   ┌────┴────┐
   ▼         ▼
GPU Idle   GPU Busy but Slow
   │         │
   ▼         ▼
tlparse   Nsight Compute
   │         │
   └────┬────┘
        ▼
Inductor / Triton Analysis
        ↓
A/B Validation
```

---

## TL;DR

- `Nsight Systems`: Where is runtime being lost?
- `Nsight Compute`: Why is a specific GPU kernel inefficient?
- `tlparse` / `Inductor logs`: Why did the compiler generate that execution structure?
- `DCGM` / `torch.profiler`: Isolate the problem before going deeper.
