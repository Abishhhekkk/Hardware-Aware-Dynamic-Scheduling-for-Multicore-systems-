# Hardware-Aware Dynamic Scheduling for Multicore Systems

### **Hardware-Aware Scheduling • Embedded Linux • Multicore Systems • Performance Counters • Adaptive CPU Placement**

> **A lightweight userspace Linux scheduler that dynamically profiles workload behavior using hardware performance counters and performs workload-aware CPU placement, load balancing, and adaptive task migration.**

![C++](https://img.shields.io/badge/C++-Systems%20Programming-00599C?style=for-the-badge\&logo=cplusplus\&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Embedded%20Systems-FCC624?style=for-the-badge\&logo=linux\&logoColor=black)
![ARM](https://img.shields.io/badge/ARM-Cortex--A53-0091BD?style=for-the-badge\&logo=arm\&logoColor=white)
![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi-3-C51A4A?style=for-the-badge\&logo=raspberrypi\&logoColor=white)

---

## 📌 Project Overview

Modern multicore embedded systems execute diverse workloads that compete for shared hardware resources such as **cache, memory bandwidth, and processor execution resources**.

Traditional Linux scheduling primarily focuses on **fairness and CPU utilization**, without directly considering the hardware behavior of individual workloads.

This project implements a **hardware-aware adaptive scheduling framework** that continuously profiles active processes using Linux hardware performance counters and dynamically adapts their CPU placement based on runtime workload behavior.

The framework operates entirely in **userspace**, without modifying the Linux kernel scheduler.

---

## 🎯 Objectives

The scheduler was designed to:

* **Identify workload behavior at runtime**
* Distinguish between **compute-intensive, memory-intensive, and mixed workloads**
* Monitor **Instructions Per Cycle (IPC)**
* Monitor **cache miss rate**
* Perform **hardware-aware CPU placement**
* Reduce **shared-resource contention**
* Perform **adaptive task migration**
* Balance workloads across CPU cores
* Support **priority-aware execution**
* Maintain scheduling stability through **migration hysteresis**
* Improve execution predictability in multicore embedded systems

---

# 🏗️ Architecture

The scheduler operates as a continuous runtime feedback loop.

```text id="8r1c5u"
                 Running Processes
                        │
                        ▼
              ┌───────────────────┐
              │ Process Discovery │
              │      /proc        │
              └─────────┬─────────┘
                        │
                        ▼
              ┌───────────────────┐
              │ Hardware Profiling│
              │  perf_event_open  │
              └─────────┬─────────┘
                        │
                ┌───────┴────────┐
                ▼                ▼
             IPC Rate       Cache Miss Rate
                │                │
                └───────┬────────┘
                        ▼
              ┌───────────────────┐
              │ Workload          │
              │ Classification    │
              └─────────┬─────────┘
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
      COMPUTE        MEMORY         MIXED
          │             │             │
          └─────────────┼─────────────┘
                        ▼
              ┌───────────────────┐
              │ Priority + Load   │
              │ Analysis          │
              └─────────┬─────────┘
                        │
                        ▼
              ┌───────────────────┐
              │ CPU Affinity      │
              │ Placement         │
              └─────────┬─────────┘
                        │
                        ▼
              ┌───────────────────┐
              │ Adaptive Migration│
              │ + Load Balancing  │
              └─────────┬─────────┘
                        │
                        ▼
                 Runtime Reprofiling
                        │
                        └───────► Loop
```

The runtime scheduling pipeline follows:

```text id="xk0z5g"
Profiling → Classification → Placement → Migration
```

and continuously repeats as workload behavior changes.

---

# 🔬 Hardware Performance Profiling

The scheduler uses Linux's `perf_event_open()` interface to collect runtime hardware behavior.

For each process, the framework monitors:

* **Instructions executed**
* **CPU cycles**
* **Cache misses**

These values are used to calculate runtime workload characteristics.

### Instructions Per Cycle

```text id="c7d3nq"
IPC = Instructions Executed / CPU Cycles
```

Higher IPC generally indicates more efficient processor execution and compute-intensive behavior.

### Cache Miss Rate

```text id="d0g5va"
Miss Rate = Cache Misses / Instructions
```

Higher cache miss rates indicate increased memory-access pressure.

---

# 🧠 Workload Classification

The scheduler dynamically classifies workloads into three categories:

```text id="f7w9f2"
┌─────────────────────────┐
│ Runtime Hardware Metrics│
│                         │
│   IPC + Cache Miss Rate │
└────────────┬────────────┘
             │
      ┌──────┴──────┐
      ▼             ▼
   COMPUTE       MEMORY
      │             │
      └──────┬──────┘
             ▼
           MIXED
```

The classification uses experimentally derived thresholds for IPC and cache miss behavior on the Raspberry Pi 3 platform.

The implemented thresholds were:

```text id="g7p1j4"
α = 0.20
β = 0.005
γ = 0.10
δ = 0.10
```

Workloads with high IPC and low cache miss rates are classified as **compute-intensive**, while low IPC or elevated cache miss behavior indicates **memory-intensive** execution.

---

# ⚙️ Adaptive CPU Placement

After workload classification, the scheduler assigns processes to appropriate CPU cores using Linux CPU affinity mechanisms.

The framework uses:

```text id="f5j3sd"
sched_setaffinity()
```

to control which cores a process can execute on.

The scheduler also monitors **per-core CPU utilization** and can override the preferred placement when another core provides better load conditions.

---

# 🔄 Adaptive Runtime Migration

Workload behavior can change during execution.

For example:

```text id="u7r9p3"
COMPUTE-INTENSIVE
       │
       │ Runtime phase change
       ▼
MEMORY-INTENSIVE
       │
       ▼
Reclassification
       │
       ▼
New CPU Placement
       │
       ▼
Task Migration
```

The scheduler continuously reprofiles workloads and detects changes in workload classification.

Migration is triggered when:

```text id="k5s8q1"
Typeₜ(Pᵢ) ≠ Typeₜ₋₁(Pᵢ)
```

Migration hysteresis is then used to prevent unnecessary task movement caused by short-lived workload fluctuations.

---

# ⚖️ Priority-Aware Scheduling

The framework also incorporates Linux task priority into CPU placement.

High-priority workloads are isolated from lower-priority background workloads to improve execution consistency and reduce interference.

Example placement policy:

| **Workload Type** | **Priority** | **Assigned Core**     |
| ----------------- | ------------ | --------------------- |
| Compute-Intensive | High         | **Core 0**            |
| Memory-Intensive  | High         | **Core 1**            |
| Compute-Intensive | Normal       | **Core 2**            |
| Memory-Intensive  | Normal       | **Core 3**            |
| Mixed / Dynamic   | Adaptive     | **Runtime Migration** |

This allows high-priority workloads to maintain dedicated execution resources while normal workloads can be dynamically balanced.

---

# 🖥️ Target Platform

| **Component**        | **Configuration**                  |
| -------------------- | ---------------------------------- |
| **Platform**         | Raspberry Pi 3 Model B             |
| **CPU**              | Quad-Core ARM Cortex-A53           |
| **Frequency**        | 1.2 GHz                            |
| **Operating System** | Ubuntu Linux                       |
| **Scheduler**        | Userspace daemon                   |
| **Profiling API**    | `perf_event_open()`                |
| **CPU Control**      | `sched_setaffinity()`              |
| **Workloads**        | Compute / Memory / Mixed / Dynamic |

## The scheduler was implemented as a persistent userspace daemon running on Ubuntu Linux and evaluated using synthetic workloads.

# 📊 Results

## System Performance

The proposed scheduler was evaluated against the default Linux **Completely Fair Scheduler (CFS)**.

| **Metric**            | **Linux CFS** | **Proposed Scheduler** |
| --------------------- | ------------: | ---------------------: |
| **System IPC**        |          0.59 |               **0.90** |
| **Compute IPC**       |     0.70–1.00 |        **0.896–1.045** |
| **Compute Miss Rate** |         0.01% |      **0.0000–0.0001** |
| **Memory IPC**        |          0.08 |        **0.066–0.095** |
| **Memory Miss Rate**  |        13.69% |         **7.81–7.85%** |

The system IPC increased from **0.59 to 0.90** under the evaluated mixed-workload conditions.

---

# 🚀 Key Performance Result

### **~52% Throughput Improvement**

The proposed scheduler achieved approximately:

```text id="f0w4jz"
Linux CFS
2.83 instructions/sec

        ↓

Proposed Scheduler
4.32 instructions/sec
```

This corresponds to an approximately **52% improvement in system throughput** under the evaluated mixed workload.

The improvement was attributed to **workload-aware core placement and reduced cache contention**.

---

# 📉 Cache Miss Reduction

The scheduler also reduced cache interference through workload-aware placement.

For the evaluated workloads, memory-intensive processes exhibited lower cache miss behavior under the proposed scheduler compared with Linux CFS.

This demonstrates the benefit of separating workloads with conflicting hardware resource requirements.

---

# ⚖️ Load-Aware Scheduling

When several workloads compete for CPU resources, the scheduler continuously monitors per-core utilization.

```text id="8b4q6x"
Before Balancing

Core 0 ─────────────── 95%
Core 1 ────────        20%
Core 2 ─────────────── 98%
Core 3 ───────         15%

             ↓

       Load Balancing

             ↓

After Balancing

Core 0 ───────── 54%
Core 1 ─────────────── 64%
Core 2 ─────────────── 64%
Core 3 ──────────────── 66%
```

The evaluation demonstrated that dynamic load balancing prevents workload concentration and reduces execution bottlenecks under heavy multicore workloads.

---

# 🔄 Adaptive Migration Results

A dynamically shifting workload was initially classified as **compute-intensive** and later transitioned into a **memory-intensive** phase.

The scheduler detected this change through runtime reprofiling and migrated the workload to a more appropriate execution core.

Example migration events included:

```text id="3u6p3d"
MEMORY → MIXED
Core 1 → Core 0

MEMORY → MIXED
Core 3 → Core 2
```

## This demonstrated that scheduling decisions adapt to **runtime workload behavior rather than static assumptions**.

# 🛡️ Scheduling Stability

Continuous reprofiling can potentially cause excessive migration when workload behavior fluctuates temporarily.

To prevent this, the scheduler implements **migration hysteresis using workload stability counters**.

Migration becomes eligible only when:

```text id="m1f9j3"
Sᵢ ≥ Sₘᵢₙ

Sₘᵢₙ = 2
```

This prevents unnecessary migration while maintaining responsiveness to genuine workload phase changes.

---

# 📋 Scheduler Comparison

| **Feature**                     | **Linux CFS** | **Proposed Scheduler** |
| ------------------------------- | ------------- | ---------------------- |
| Dynamic Workload Classification | ❌             | ✅                      |
| Runtime Reprofiling             | ❌             | ✅                      |
| Adaptive Migration              | ❌             | ✅                      |
| Priority-Aware Placement        | ❌             | ✅                      |
| Load-Aware Scheduling           | ❌             | ✅                      |
| Migration Hysteresis            | ❌             | ✅                      |

The proposed framework combines these capabilities while remaining entirely in userspace.

---

# 📈 Results Summary

| **Result**                          | **Outcome**          |
| ----------------------------------- | -------------------- |
| **System IPC**                      | **0.59 → 0.90**      |
| **Throughput**                      | **~52% improvement** |
| **Memory Cache Miss Rate**          | **13.69% → ~7.8%**   |
| **Dynamic Workload Classification** | ✅ Successful         |
| **Adaptive Task Migration**         | ✅ Successful         |
| **Load Balancing**                  | ✅ Successful         |
| **Priority Isolation**              | ✅ Successful         |
| **Migration Stability**             | ✅ Successful         |
| **Kernel Modification Required**    | **No**               |

The overall evaluation demonstrated successful runtime workload classification, adaptive migration, reduced resource contention, load balancing, priority-aware execution, and scheduling stability.

---

# 🧩 Core Linux Interfaces

### Hardware Profiling

```text id="m1g4nd"
perf_event_open()
```

Used to access hardware performance counters for runtime profiling.

### CPU Affinity

```text id="4e8z4j"
sched_setaffinity()
```

Used to control process execution across CPU cores.

### Process Discovery

```text id="c5r2kx"
/proc
```

Used for dynamic discovery and tracking of active processes.

---

# 🛠️ Technologies & Concepts

### **Systems Programming**

`C++` `C` `Linux` `Ubuntu` `Userspace Daemons`

### **Computer Architecture**

`Multicore Systems` `CPU Scheduling` `Cache Behavior` `IPC` `Memory Contention`

### **Linux Internals**

`perf_event_open()` `sched_setaffinity()` `/proc` `CPU Affinity`

### **Embedded Systems**

`Raspberry Pi 3` `ARM Cortex-A53` `Embedded Linux`

### **Performance Engineering**

`Hardware Performance Counters` `Workload Profiling` `Load Balancing` `Runtime Migration`

---

# 🔎 What This Project Demonstrates

This project demonstrates practical experience with:

* **Linux systems programming**
* **Multicore CPU architecture**
* **Hardware performance counters**
* **Low-level performance profiling**
* **CPU scheduling**
* **Cache and memory contention**
* **Runtime workload classification**
* **CPU affinity management**
* **Adaptive task migration**
* **Performance optimization**
* **Embedded Linux**
* **Systems-level benchmarking**

---

# 🔬 Experimental Workflow

```text id="q6l7r8"
          Synthetic Workloads
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
   Compute-Heavy       Memory-Heavy
        │                   │
        └─────────┬─────────┘
                  ▼
          Runtime Profiling
                  │
                  ▼
        IPC + Cache Miss Rate
                  │
                  ▼
        Workload Classification
                  │
                  ▼
        CPU Core Assignment
                  │
                  ▼
        Adaptive Migration
                  │
                  ▼
         Performance Analysis
                  │
                  ▼
       CFS vs Proposed Scheduler
```

The experimental setup included compute-intensive, memory-intensive, mixed, and dynamically shifting workloads to evaluate classification and adaptive scheduling behavior.

---

# 📷 Experimental Evidence

The repository can showcase:

* **System architecture**
* **Workload classification**
* **IPC comparison**
* **Cache miss-rate comparison**
* **Throughput comparison**
* **Per-core utilization before/after balancing**
* **Adaptive migration timeline**
* **Priority-aware placement**
* **Scheduling stability**
* **CFS vs proposed scheduler comparison**

### Recommended screenshots

Put these **three images near the top of the README**:

**① Architecture / scheduling pipeline**

**② Throughput comparison — CFS vs Proposed Scheduler**

**③ Adaptive task migration / workload behavior**

## Your report includes these experimental visualizations, including the throughput comparison and dynamic migration results.

# 🚀 Limitations & Future Work

The current evaluation primarily uses **synthetic workloads on a Raspberry Pi multicore platform**.

Potential extensions include:

* Real-world embedded workloads
* Heterogeneous CPU scheduling
* GPU-aware scheduling
* NPU-aware scheduling
* FPGA-aware scheduling
* Accelerator-aware workload offloading
* Machine-learning-assisted workload prediction
* Energy-aware scheduling
* Kernel-assisted scheduling

These extensions would enable the framework to target increasingly heterogeneous embedded and edge-computing platforms.

---

# 🎓 Academic Context

**University of Massachusetts Amherst**
**MS Electrical & Computer Engineering**

**Project:** Hardware-Aware Dynamic Scheduling for Multicore Systems

**Platform:** Raspberry Pi 3 Model B

---

# 👨‍💻 Author

### **Abishek Thirunavukkarasu**

**MS Electrical & Computer Engineering**
**University of Massachusetts Amherst**

[![Portfolio](https://img.shields.io/badge/Portfolio-000000?style=for-the-badge\&logo=google-chrome\&logoColor=white)](YOUR_PORTFOLIO_LINK)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge\&logo=linkedin\&logoColor=white)](YOUR_LINKEDIN_LINK)

---

### ⭐ Key Takeaway

> **This project demonstrates that runtime hardware behavior can be incorporated into Linux scheduling decisions to improve multicore execution efficiency without modifying the kernel scheduler.**

**Measured outcome: ~52% throughput improvement over Linux CFS under the evaluated mixed-workload conditions.**

---
