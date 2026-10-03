---
title: Inside PyTorch(4) — How to Analyze GPU Runtime Performance
by: hdimmfh
date: 2026-09-24 18:40:00 +0900
categories: [GPU, PyTorch]
tags: [PyTorch, GPU Profiling , Nsight , tlparse, TorchInductor]
---

🔍 **Once a Model Is Compiled, How Do We Know Whether the GPU Is Actually Running Efficiently?**

> After compilation, performance analysis should focus on the steady-state runtime: establish a reproducible baseline, identify where GPU time is lost, locate expensive kernels, verify whether efficient implementations were selected, and finally trace inefficient kernels back to compiler decisions when necessary.

---

## ① Establish a Reproducible Performance Baseline

Before profiling, define the workload conditions that will remain fixed during optimization: representative input shapes, batch size, precision, warm-up iterations, concurrency, CUDA Graph usage, and measurement duration. Separate compilation or engine-build time from steady-state execution time, and record baseline latency and throughput before making any changes.

> 🔗 [NVIDIA TensorRT — Performance Benchmarking](https://docs.nvidia.com/deeplearning/tensorrt/latest/performance/benchmarking.html)

## ② Check Basic GPU Telemetry

Use `nvidia-smi dmon` or NVIDIA DCGM to check GPU/SM utilization, HBM activity, clocks, power, temperature, PCIe, and NVLink activity. This provides the first indication of whether the GPU is underutilized or throttled.

> 🔗 [NVIDIA DCGM Profiling](https://docs.nvidia.com/datacenter/dcgm/latest/learn/modules/profiling.html)

## ③ Profile PyTorch Execution

Use `torch.profiler` and NVTX to identify expensive forward, backward, optimizer, dataloader, and operator regions before moving to lower-level GPU analysis.

> 🔗 [PyTorch Profiler](https://docs.pytorch.org/docs/stable/profiler)

## ④ Inspect the CPU-GPU Timeline

Use NVIDIA Nsight Systems (`nsys`) to find GPU idle gaps, delayed kernel launches, synchronization, memory copies, and CPU-side stalls. Large gaps between GPU kernels usually indicate that the GPU is waiting for work.

> 🔗 [NVIDIA Nsight Systems User Guide](https://docs.nvidia.com/nsight-systems/UserGuide/)

## ⑤ Inspect Multi-GPU Communication

For distributed workloads, use Nsight Systems with NCCL tracing and DCGM to inspect AllReduce, AllGather, ReduceScatter, rank imbalance, and compute-communication overlap.

> 🔗 [Nsight Systems — NCCL Analysis](https://docs.nvidia.com/nsight-systems/UserGuide/)

## ⑥ Inspect GPU Memory Behavior

Use Nsight Systems GPU Memory tracing and `torch.profiler` to inspect H2D/D2H transfers, allocation/deallocation behavior, peak memory usage, and unnecessary memory movement.

> 🔗 [Nsight Systems — CUDA GPU Memory Analysis](https://docs.nvidia.com/nsight-systems/UserGuide/)

## ⑦ Investigate Unexplained GPU Idle Time

If GPU gaps cannot be explained by CPU, communication, or memory activity, use `tlparse` and `TORCH_LOGS` to inspect graph breaks, eager fallbacks, recompilations, fusion, and scheduling decisions.

> 🔗 [PyTorch tlparse / TORCH_TRACE](https://docs.pytorch.org/docs/main/user_guide/torch_compiler/compile/programming_model.observability.html)

## ⑧ Identify Hotspot GPU Kernels

Use Nsight Systems or `torch.profiler` to rank kernels by total GPU execution time. Focus detailed analysis on the small number of kernels responsible for most of the runtime rather than profiling every kernel.

> 🔗 [PyTorch — Profiling torch.compile Performance](https://docs.pytorch.org/docs/main/user_guide/torch_compiler/torch.compiler_profiling_torch_compile.html)

## ⑨ Inspect Algorithm and Kernel Variant Selection

A GPU operation often has multiple valid implementations with different tiling, scheduling, memory usage, workspace requirements, and hardware efficiency. Before manually optimizing a hotspot kernel, verify whether the runtime, compiler, or library selected an efficient implementation for the actual workload.

Some systems use heuristics, while others benchmark multiple candidates and select the fastest one. TensorRT profiles alternative tactics during engine building, TorchInductor `max-autotune` benchmarks Triton or template implementations, and GPU libraries such as cuDNN may select among multiple algorithms. Treat these mechanisms as a general implementation-search layer between graph optimization and individual kernel analysis.

For dynamic workloads, perform tuning with representative shapes rather than assuming that a configuration selected for one shape remains optimal for every shape. Reuse timing or tuning caches only when their hardware and software assumptions remain valid.

> 🔗 [NVIDIA TensorRT — Optimizing Performance](https://docs.nvidia.com/deeplearning/tensorrt/latest/performance/optimization.html)
> 🔗 [PyTorch — torch.compile max-autotune](https://docs.pytorch.org/docs/stable/generated/torch.compile)
> 🔗 [TorchInductor GPU Profiling](https://docs.pytorch.org/docs/main/user_guide/torch_compiler/torch.compiler_inductor_profiling.html)

## ⑩ Analyze Individual Kernel Efficiency

Use NVIDIA Nsight Compute (`ncu`) on hotspot kernels. Check SpeedOfLight/Roofline, compute and memory throughput, Tensor Core utilization, occupancy, registers, shared memory, active/eligible warps, cache/HBM behavior, and warp stall reasons.

> 🔗 [NVIDIA Nsight Compute Profiling Guide](https://docs.nvidia.com/nsight-compute/ProfilingGuide/)

## ⑪ Trace Kernel Problems Back to the Compiler

If a kernel shows poor hardware efficiency or no efficient implementation is selected, return to `tlparse` and Inductor logs such as `fusion`, `schedule`, `ir_pre_fusion`, `ir_post_fusion`, and `kernel_code`. Check whether excessive fusion, insufficient fusion, scheduling, generated Triton code, or the available implementation search space explains the observed GPU behavior.

> 🔗 [PyTorch Compiler Observability](https://docs.pytorch.org/docs/main/user_guide/torch_compiler/compile/programming_model.observability.html)

## ⑫ Validate Every Optimization

After changing batch size, fusion behavior, precision, CUDA Graphs, compiler options, algorithm selection, or autotuning configuration, repeat the same benchmark and profiling workflow. Compare throughput, latency, GPU idle time, hotspot kernel time, and hardware efficiency while also verifying numerical correctness. Measure build or compilation overhead separately from steady-state runtime improvements.

> 🔗 [PyTorch — Profiling torch.compile Performance](https://docs.pytorch.org/docs/main/user_guide/torch_compiler/torch.compiler_profiling_torch_compile.html)

---

## Key Takeaway

GPU performance optimization is not only about finding slow kernels. It should proceed from reproducible system-level measurement toward hotspot identification, implementation selection, individual kernel analysis, and finally compiler decisions when necessary.

```text
Reproducible Baseline
        ↓
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
tlparse   Hotspot Kernels
             ↓
      Algorithm / Tactic
       / Autotune Search
             ↓
       Nsight Compute
             ↓
     Inductor / Triton
          Analysis
             ↓
        A/B Validation
```

---

## TL;DR

- `Nsight Systems`: Where is runtime being lost?
- `Nsight Compute`: Why is a specific GPU kernel inefficient?
- `tlparse` / `Inductor logs`: Why did the compiler generate that execution structure?
- `DCGM` / `torch.profiler`: Isolate the problem before going deeper.
