---
title: NVIDIA AI Factory(1) - Software Components
by: hdimmfh
date: 2026-09-26 15:00:00 +0900
categories: [GPU, AI Factory]
tags: [NVIDIA, NICo, NVCM, Config Manager, NetQ, UFM, NVSentinel, Spectrum-X, InfiniBand]
---

🔍 **How Do the Main NVIDIA AI Factory Software Components Relate to Each Other?**

> NVIDIA AI Factory management software is split across multiple control and observability domains. NICo, Config Manager, NetQ, UFM, and NVSentinel are not equivalent components: they are deployed in different places, manage different infrastructure layers, and interact with different hardware or workload domains.

---

## ① High-Level Architecture

![NVIDIA AI Factory Management Software Architecture](https://github.com/hdimmfh/hdimmfh.github.io/blob/main/_data/blog-img-repo/img/aifactory/ncp_sw_components.png?raw=true)
*Figure 1. NVIDIA AI Factory Software Components.*

The main relationship can be summarized as follows:

```text
AI Factory Management Services
├── NICo
│   └── Managed Hosts / DPU / SuperNIC / provisioning / network lifecycle
├── NVIDIA Config Manager
│   └── Spectrum / Spectrum-X switch configuration lifecycle
├── NetQ
│   └── Ethernet fabric telemetry / validation / operational state
└── UFM
    └── InfiniBand fabric management / monitoring

Customer / Tenant Kubernetes Cluster
└── NVSentinel
    └── GPU / node health monitoring and Kubernetes remediation
```

The important distinction is that these components do not form one flat management stack. Each component owns a different management domain.

---

## ② NICo

**NICo (NVIDIA Infra Controller)** is the infrastructure control plane for managed-host lifecycle, DPU/SuperNIC-related networking, provisioning, IPAM, and tenant infrastructure resources.

```text
NICo
 ├── Core / API
 ├── provisioning workflows
 ├── DHCP / IPAM
 ├── host and DPU lifecycle
 └── network / partition resources
        │
        ▼
Managed Hosts / DPU / SuperNIC
```

NICo should therefore be viewed primarily as an **infrastructure orchestration and lifecycle controller**, not as a general-purpose switch configuration manager.

---

## ③ NVIDIA Config Manager

**NVIDIA Config Manager** manages the configuration lifecycle of NVIDIA network devices. Its main flow is based on source-of-truth data, rendering, configuration storage, and deployment workflows.

```text
Nautobot / DCIM
      │
      ▼
Render Service
      │
      ▼
Config Store
      │
      ▼
Deployment Workflow
      │
      ▼
Spectrum / Spectrum-X Switch
```

Its role is closest to **desired-state network configuration management**.

---

## ④ NetQ

**NetQ** provides Ethernet fabric observability, telemetry, validation, and operational-state visibility.

```text
Spectrum / Spectrum-X Fabric
          │
          │ telemetry / state
          ▼
         NetQ
```

Config Manager and NetQ therefore represent different views of the same Ethernet infrastructure:

- **Config Manager**: intended configuration
- **NetQ**: observed operational state

---

## ⑤ UFM

**UFM (Unified Fabric Manager)** manages and monitors the InfiniBand fabric.

```text
UFM
 │
 ├── topology
 ├── routing
 ├── partitioning
 ├── telemetry
 └── fabric operations
        │
        ▼
InfiniBand Switches / HCAs
```

UFM is the primary management domain for **InfiniBand scale-out networking**.

---

## ⑥ NVSentinel

**NVSentinel** is associated with the customer or tenant Kubernetes cluster rather than the central AI Factory management services.

```text
Customer / Tenant Kubernetes Cluster
│
├── NVSentinel
│   ├── GPU Health Monitor
│   ├── Platform Connector
│   ├── Fault Quarantine
│   ├── Node Drainer
│   └── Fault Remediation
│
├── Kubernetes API
└── GPU Nodes / Workloads
```

Its primary role is **GPU and node health monitoring, fault handling, quarantine, drain, and remediation through Kubernetes-native workflows**.

---

## ⑦ Responsibility Boundaries

```text
NICo
→ infrastructure lifecycle / managed hosts / DPU / SuperNIC

Config Manager
→ Spectrum / Spectrum-X switch configuration

NetQ
→ Ethernet fabric observability

UFM
→ InfiniBand fabric management

NVSentinel
→ Kubernetes GPU-node health and remediation
```

These boundaries are important because the products are related operationally but are not interchangeable.

For example, NICo and Config Manager should be treated as separate management domains, while NetQ complements Config Manager by observing the resulting network state. NVSentinel belongs closer to the workload cluster because its actions are Kubernetes- and node-health-oriented.

---

## Key Takeaway

The simplest way to understand the NVIDIA AI Factory software stack is to classify each component by **deployment location, control target, and responsibility**.

```text
Infrastructure orchestration
→ NICo

Ethernet configuration
→ NVIDIA Config Manager

Ethernet observability
→ NetQ

InfiniBand management
→ UFM

GPU / Kubernetes remediation
→ NVSentinel
```

The architecture is therefore a set of domain-specific management planes rather than a single hierarchical controller.

---

## TL;DR

Q. What does NICo manage?

> Managed-host lifecycle, DPU/SuperNIC-related infrastructure, provisioning, and tenant network resources.

Q. What does NVIDIA Config Manager manage?

> Desired configuration and deployment workflows for NVIDIA Ethernet switches.

Q. What does NetQ manage?

> Ethernet telemetry, validation, and operational-state visibility.

Q. What does UFM manage?

> InfiniBand topology, routing, partitioning, monitoring, and fabric operations.

Q. Where does NVSentinel belong?

> Inside or alongside the customer Kubernetes workload cluster, where it monitors GPU/node health and drives remediation.

---

## References

- [NVIDIA Infra Controller Architecture](https://docs.nvidia.com/infra-controller/documentation/architecture/overview-and-components)
- [NVIDIA Config Manager Architecture](https://docs.nvidia.com/switch-infrastructure/config-manager/config-manager/overview/architecture)
- [NVIDIA NetQ](https://docs.nvidia.com/networking-ethernet-software/cumulus-netq/)
- [NVIDIA UFM](https://docs.nvidia.com/networking/display/ufmenterpriseum/)
- [NVIDIA NVSentinel](https://docs.nvidia.com/nvsentinel/)


