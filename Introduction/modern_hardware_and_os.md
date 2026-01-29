# Modern Hardware Features and Their Interaction with Operating Systems (2025–2026)

These notes explain *why* modern hardware looks the way it does and *how* operating systems must cooperate with it.

---

## 1. Multi-Core Processors

### What they are

* A multi-core processor contains multiple independent CPU cores on a single chip.
* Each core has:

  * Its own execution pipeline
  * Private L1 (and usually L2) caches
* Cores typically share:

  * Last-level cache (L3)
  * Memory controllers
  * Interconnect fabric

### Why they exist

* Clock frequency scaling stopped due to power and thermal limits.
* Increasing performance now comes from **parallel execution**, not higher clock speeds.

### What this means for the OS

* **Topology discovery**

  * The OS must understand which cores share caches and memory paths.
* **Per-core scheduling**

  * Each core has its own run queue to reduce contention.
* **Cache affinity**

  * The OS prefers to run a task on the same core where it previously ran.
  * This avoids cache misses and improves performance.
* **Load balancing**

  * Tasks are migrated only when necessary, because migration destroys cache locality.
* **Power and thermal coordination**

  * The OS cooperates with hardware to adjust frequency and avoid overheating.

### Key difficulty

* As core count grows, cache coherence traffic and scheduling complexity increase significantly.

---

## 2. Simultaneous Multithreading (SMT / Hyper-Threading)

### What it is

* SMT allows one physical core to execute multiple hardware threads.
* Threads share most internal resources but maintain separate architectural state.

### Resource sharing

* **Shared**: execution units, caches, TLBs, branch predictors
* **Duplicated**: registers, program counters, control state

### Why it helps

* Improves overall throughput by using idle execution units.
* Particularly effective for workloads with stalls (e.g., memory waits).

### Trade-offs

* Individual threads may run slower due to contention.
* Shared structures create security risks (side-channel attacks).

### OS responsibilities

* Expose SMT threads as logical CPUs.
* Schedule tasks to fill physical cores before SMT siblings.
* Optionally disable SMT for security or predictable latency.

---

## 3. Heterogeneous Core Architectures (P-cores and E-cores)

### What they are

* Modern CPUs combine cores with different performance and power characteristics.
* Examples: Intel P-core/E-core, Arm big.LITTLE.

### Core types

* **Performance cores**

  * High instruction-level parallelism
  * High power consumption
* **Efficiency cores**

  * Simpler pipelines
  * Much lower energy usage

### Why this exists

* Different tasks have different performance needs.
* Running all tasks on large cores wastes energy.

### OS responsibilities

* **Energy-aware scheduling**

  * Estimate the energy cost of running a task on each core type.
* **Task classification**

  * Latency-sensitive tasks → performance cores
  * Background or long-running tasks → efficiency cores
* **Migration control**

  * Avoid frequent movement between core types due to cache and pipeline costs.

---

## 4. Non-Uniform Memory Access (NUMA)

### What NUMA means

* Memory access time depends on which CPU core accesses which memory node.
* Not all memory is equally close to all cores.

### Where NUMA appears

* Multi-socket servers
* Chiplet-based CPUs
* CXL-attached memory

### OS responsibilities

* Discover memory nodes and distances.
* Use **first-touch allocation** to place memory near the accessing core.
* Migrate memory pages closer to where they are used.
* Schedule threads near their memory.

### Why it matters

* Remote memory access can be several times slower than local access.

---

## 5. Cache and Memory Bandwidth Partitioning

### Problem being solved

* Multiple processes share the last-level cache and memory bandwidth.
* One process can evict cache lines needed by another.

### Hardware support

* Cache way partitioning
* Memory bandwidth throttling

### OS responsibilities

* Assign cache and bandwidth quotas to processes or virtual machines.
* Prevent interference in multi-tenant systems.
* Provide predictable performance for critical workloads.

---

## 6. Hardware Virtualization and Confidential Computing

### Virtualization support

* Hardware-assisted page tables
* Interrupt virtualization
* Device isolation using IOMMUs

### Confidential computing

* Guest memory is encrypted and protected from the host.
* Enables secure execution in untrusted environments.

### OS and hypervisor role

* Manage encrypted memory regions.
* Handle secure boot and attestation.
* Enforce isolation between guests and devices.

---

## 7. Compute Express Link (CXL) and Memory Tiering

### What CXL provides

* Memory expansion beyond traditional DRAM limits.
* Coherent access between CPUs and devices.

### OS responsibilities

* Treat CXL memory as a separate memory tier.
* Move data between fast (DRAM) and slow (CXL) memory.
* Expose topology information to applications.

---

## 8. Performance Monitoring Units (PMU)

### What PMUs do

* Count and sample microarchitectural events.
* Provide insight into stalls, cache misses, and branch behavior.

### OS usage

* Provide interfaces for profiling tools.
* Support performance tuning and power management.
* Enable feedback-driven scheduling and optimization.

---

## Overall Impact on Operating System Design

* Scheduling must consider topology, energy, and security.
* Memory management must handle locality, tiers, and migration.
* Hardware-assisted security is now a core OS responsibility.
* Observability and profiling are essential for performance.

---

## Key Takeaway

Modern operating systems are no longer simple resource managers. They are active participants in performance, power efficiency, and security, continuously coordinating with increasingly complex hardware.
