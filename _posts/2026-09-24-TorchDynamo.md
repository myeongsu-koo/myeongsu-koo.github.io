---
title: Inside PyTorch(3) — How TorchDynamo Reuses Compiled Graphs
by: hdimmfh
date: 2026-09-24 13:00:00 +0900
categories: [GPU, PyTorch]
tags: [PyTorch, TorchDynamo, torch.compile, Guards, Graph Breaks, Dynamic Shapes]
---

🔍 **Why Does `torch.compile` Sometimes Recompile the Same Function?**

> TorchDynamo does not simply compile a Python function once and reuse it forever. It captures computation graphs under a set of runtime assumptions called **guards**. If those assumptions no longer hold, Dynamo may reuse another cached graph or compile a new one. Dynamic shapes make those assumptions less restrictive so that one compiled artifact can handle multiple tensor sizes.

---

## ① What Does TorchDynamo Actually Do?

PyTorch normally executes operations eagerly.

```python
x = a + b
y = torch.relu(x)
z = y * 2
```

In Eager Mode, each operation is dispatched as Python reaches it.

```text
Python
  ↓
a + b
  ↓
execute
  ↓
relu
  ↓
execute
  ↓
multiply
  ↓
execute
```

`torch.compile()` changes this execution model.

```text
Python Function
      ↓
TorchDynamo
      ↓
FX Graph
      ↓
Backend Compiler
      ↓
Compiled Artifact
```

TorchDynamo observes Python execution and attempts to represent compatible tensor computation as an FX graph. However, the graph is only valid under certain assumptions about the runtime environment. Those assumptions are recorded as **guards**.

---

## ② Guards Define Where a Compiled Artifact Is Valid

Consider:

```python
@torch.compile
def f(x):
    return x + 1
```

Suppose the first input is a CUDA `float32` tensor with shape `(32, 32)`. Dynamo may specialize the compiled artifact around properties such as dtype, device, rank, and shape.

```text
Compiled Artifact A
┌────────────────────┐
│ FX Graph           │
│ x → add(x, 1)      │
└────────────────────┘

Guards
├── dtype == float32
├── device == cuda
├── rank == 2
├── size[0] == 32
└── size[1] == 32
```

A guard is therefore **not the physical boundary of an FX graph**. It is better understood as a runtime validity condition.

```text
Guard PASS
    ↓
reuse compiled artifact

Guard FAIL
    ↓
try another cached artifact
or
recompile
```

In other words, guards define the runtime state in which a compiled artifact can safely be reused.

---

## ③ Graph Breaks Define Graph Capture Boundaries

A **graph break** is different from a guard failure. A graph break occurs when Dynamo cannot continue tracing a region of Python execution as part of the current graph.

```text
Python Function
      │
      ▼
┌──────────────┐
│ FX Graph #1  │
└──────────────┘
      │
      ▼
  Graph Break
      │
      ▼
Regular Python
      │
      ▼
Resume Tracing
      │
      ▼
┌──────────────┐
│ FX Graph #2  │
└──────────────┘
```

Dynamo compiles the graph captured before the break, executes the unsupported Python region, and then attempts to resume tracing afterward.

| Concept       | Meaning                                                                                    |
| ------------- | ------------------------------------------------------------------------------------------ |
| Guard         | Determines whether an existing compiled artifact is still valid                            |
| Graph Break   | Determines where one captured FX graph ends and another may begin                          |
| Resume        | Continues Dynamo tracing after a graph break                                               |
| Recompilation | Creates a new compiled artifact when existing ones cannot handle the current runtime state |

A Python function boundary itself does not necessarily create a graph break. If nested function calls can be traced, Dynamo may capture their tensor operations into the same FX graph.

---

## ④ What Happens During Recompilation?

Assume dynamic shapes are disabled.

```python
@torch.compile(dynamic=False)
def f(x):
    return x + 1
```

The first call may produce a specialization for `(32, 32)`.

```text
Artifact A
Guard: shape == (32, 32)
```

If the next input is `(64, 32)`, the existing shape guard fails.

```text
Input: (64, 32)
       │
       ▼
Artifact A
shape == (32, 32)
       │
       ▼
     FAIL
       │
       ▼
 Recompilation
```

A new specialization may then be created.

```text
Artifact A → shape == (32, 32)
Artifact B → shape == (64, 32)
```

Artifact B does not simply replace Artifact A. Multiple compiled results can exist for the same Python code. If `(32, 32)` appears again, Dynamo can reuse a cached artifact whose guards match that input.

> Recompilation creates another compiled specialization because the currently cached artifacts cannot safely execute the new runtime state.

---

## ⑤ Dynamic Shapes Reduce Shape-Based Recompilation

Static specialization works well when tensor sizes remain stable. It becomes expensive when sizes frequently change.

```text
(32, 32)
(64, 32)
(128, 32)
(256, 32)
```

With `dynamic=False`, these inputs may require separate specializations.

```text
Artifact A → Tensor[32, 32]
Artifact B → Tensor[64, 32]
Artifact C → Tensor[128, 32]
Artifact D → Tensor[256, 32]
```

Dynamic shapes allow dimensions to be represented symbolically. Instead of specializing to `Tensor[32, 32]`, Dynamo may reason about `Tensor[s0, 32]`, where `s0` represents a runtime size.

```text
              Artifact A
            Tensor[s0, 32]
                  │
        ┌─────────┼──────────┐
        ▼         ▼          ▼
       32        64         128
```

The symbolic dimension belongs to the compiler representation. The actual runtime tensor still has a concrete shape.

```text
Compile-time: Tensor[s0, 32]
Runtime:      s0 = 64 → Tensor[64, 32]
```

A dynamic compiled function therefore still produces ordinary tensors with concrete runtime sizes.

---

## ⑥ `dynamic=None`, `True`, and `False`

`torch.compile()` uses `dynamic=None` by default.

| Setting         | Behavior                                                                       |
| --------------- | ------------------------------------------------------------------------------ |
| `dynamic=False` | Specializes tensor sizes and does not generate dynamic kernels                 |
| `dynamic=True`  | Attempts to make sizes dynamic from the first compilation                      |
| `dynamic=None`  | Starts static and becomes more dynamic when runtime shape changes are observed |

The default behavior can be simplified as follows.

```text
First call: (32, 32)
        ↓
static compilation
        ↓
Tensor[32, 32]

Second call: (64, 32)
        ↓
shape guard fails
        ↓
recompile
        ↓
dimension 0 is observed to change
        ↓
Tensor[s0, 32]

Later calls:
(128, 32), (256, 32), ...
        ↓
reuse Tensor[s0, 32] when constraints match
```

This can be viewed as **runtime adaptation**. Dynamo begins with specialization and, when actual workload behavior demonstrates that a dimension changes, attempts to generalize that dimension.

However:

```text
dynamic shape ≠ no guards
dynamic shape ≠ recompilation can never happen
```

Other runtime assumptions and control-flow decisions can still require specialization.

---

## ⑦ Dynamic Shapes and Control Flow

Consider:

```python
def f(x):
    if x.shape[0] > 10:
        return x * 2
    return x + 1
```

Even if `x.shape[0]` is symbolic, Dynamo must determine which Python branch is taken while tracing. A compilation may therefore produce something conceptually similar to:

```text
Artifact A
Graph: x * 2
Guard: s0 > 10

Artifact B
Graph: x + 1
Guard: s0 <= 10
```

Dynamic shapes allow different sizes to share a graph **when the graph remains valid for those sizes**. They do not remove every specialization decision.

```text
Dynamic Shape
      ↓
size does not need to equal one exact constant
      ↓
symbolic constraints may still be required
```

---

## ⑧ Mixing Dynamic and Static Compiled Regions

Dynamic and static compiled functions can be connected normally.

```python
@torch.compile(dynamic=True)
def producer(x):
    return x * 2

@torch.compile(dynamic=False)
def consumer(x):
    return x + 1
```

The symbolic representation inside `producer()` is not permanently attached to the returned tensor.

```text
producer
Tensor[s0, 32]
      ↓
runtime output
Tensor[64, 32]
      ↓
consumer
sees Tensor[64, 32]
```

The problem is not correctness but specialization pressure. If the producer repeatedly generates different shapes, it may reuse one dynamic artifact while the static consumer produces multiple specializations.

```text
Producer
Tensor[s0, 32]
      │
      ├── (32, 32)
      ├── (64, 32)
      ├── (128, 32)
      └── (256, 32)
              ↓
           Consumer
              ↓
Consumer A → Tensor[32, 32]
Consumer B → Tensor[64, 32]
Consumer C → Tensor[128, 32]
Consumer D → Tensor[256, 32]
```

This may increase compilation latency, the number of cached specializations, downstream compiler artifacts, and warm-up cost. The configuration is valid, but it can create an inefficient **specialization boundary**.

---

## ⑨ Choosing Static or Dynamic Shapes

A practical policy is to keep dimensions static when they are known to remain stable and mark only dimensions that are known to vary as dynamic.

For example, if a model uses `[batch, sequence, hidden]` and only the first two dimensions vary:

```text
Tensor[s_batch, s_sequence, 4096]
```

is usually a more useful target than making every dimension dynamic.

If workload behavior is not known yet, `dynamic=None` is a reasonable starting point.

```text
Unknown workload
      ↓
dynamic=None
      ↓
observe recompilations
      ↓
identify changing dimensions
      ↓
mark required dimensions dynamic
      ↓
keep stable dimensions specialized
```

`dynamic=True` can be useful for experimentation, but making every possible size dynamic is not necessarily optimal.

---

## ⑩ The Runtime Mental Model

The overall Dynamo execution model can be simplified as follows.

```text
                    Python Function
                           │
                           ▼
                      TorchDynamo
                           │
                           ▼
                     Trace Program
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
             FX Graph               Guards
                │                     │
                └──────────┬──────────┘
                           ▼
                  Compiled Artifact
                           │
                     next invocation
                           ▼
                     Check Guards
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
               PASS                 FAIL
                 │                   │
                 ▼                   ▼
               Reuse          Search Cached
                                Artifacts
                                   │
                         ┌─────────┴─────────┐
                         ▼                   ▼
                       Match              No Match
                         │                   │
                         ▼                   ▼
                       Reuse             Recompile
```

Graph breaks describe a separate process.

```text
Dynamo tracing
     ↓
FX Graph #1
     ↓
Graph Break
     ↓
Python Execution
     ↓
Resume
     ↓
FX Graph #2
```

| Concept            | Responsibility                                                                               |
| ------------------ | -------------------------------------------------------------------------------------------- |
| Graph              | Represents captured computation                                                              |
| Guard              | Defines when a compiled artifact remains valid                                               |
| Graph Break        | Ends the current graph capture region                                                        |
| Recompilation      | Creates another specialization when cached artifacts cannot handle the current runtime state |
| Dynamic Shape      | Generalizes changing dimensions so one artifact can cover multiple sizes                     |
| Runtime Adaptation | Uses observed runtime variation to reduce unnecessary future specialization                  |

---

## Key Takeaway

TorchDynamo is not simply a system that converts Python functions into permanent static graphs. It is a **speculative JIT compiler**. Dynamo observes Python execution, captures tensor computation, and specializes the resulting graph under a set of assumptions.

```text
Python Execution
      ↓
Dynamo Tracing
      ↓
Graph + Guards
      ↓
Compiled Artifact
```

Guards determine whether that specialization remains valid.

```text
Runtime Input
      ↓
Guard Check
      │
      ├── valid → reuse
      └── invalid → another cached artifact or recompilation
```

Graph breaks determine where graph capture stops and later resumes. Dynamic shapes reduce unnecessary specialization by representing changing dimensions symbolically when appropriate.

The central optimization question is therefore not simply:

> Should the model be static or dynamic?

A more useful question is:

> Which properties of the workload are truly stable, and which properties should remain dynamic so that compiled artifacts can be reused efficiently?

---

## TL;DR

Q. What does TorchDynamo do?

> TorchDynamo observes Python execution and captures compatible computation into FX graphs that can be passed to backend compilers.

Q. What is a guard?

> A guard is a runtime condition that determines whether a previously compiled artifact is still valid for the current execution.

Q. Is a guard the boundary of an FX graph?

> No. A guard defines the validity domain of a compiled artifact. A graph break defines where graph capture ends.

Q. What is a graph break?

> A graph break occurs when Dynamo stops the current graph capture, executes part of the program outside that graph, and later resumes tracing.

Q. What causes recompilation?

> Recompilation occurs when existing cached compiled artifacts cannot safely handle the current runtime state because their guards do not match.

Q. Does recompilation replace the previous artifact?

> Not necessarily. Multiple compiled specializations can exist for the same Python code and can be reused when their guards match future inputs.

Q. What does `dynamic=False` do?

> It specializes tensor sizes, which can cause recompilation when those sizes change.

Q. What does `dynamic=True` do?

> It attempts to make sizes dynamic from the first compilation so that one compiled artifact can handle multiple sizes when possible.

Q. What does `dynamic=None` do?

> It initially specializes statically and attempts to generalize changing dimensions after observing shape-driven recompilation.

Q. Is a symbolic shape attached to the runtime tensor?

> No. Symbolic shapes are compiler representations. Runtime tensors still have concrete sizes.

Q. Can a dynamic compiled function feed a static compiled function?

> Yes. However, varying upstream output shapes may cause the static downstream function to accumulate multiple specializations.

Q. What is a reasonable shape policy?

> Keep known stable dimensions static, explicitly mark known changing dimensions dynamic, and use the default automatic behavior when the workload has not yet been characterized.

---

## References

* [PyTorch Compiler](https://docs.pytorch.org/docs/stable/user_guide/torch_compiler/torch.compiler.html)
* [torch.compile API](https://docs.pytorch.org/docs/stable/generated/torch.compile.html)
* [TorchDynamo Overview](https://docs.pytorch.org/docs/stable/torch.compiler_dynamo_overview.html)
* [Dynamo Core Concepts](https://docs.pytorch.org/docs/stable/compile/programming_model.dynamo_core_concepts.html)
* [Dynamic Shapes](https://docs.pytorch.org/docs/stable/torch.compiler_dynamic_shapes.html)
* [Dynamic Shapes Core Concepts](https://docs.pytorch.org/docs/stable/compile/dynamic_shapes_core_concepts.html)
