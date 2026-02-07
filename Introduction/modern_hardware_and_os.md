# Modern Hardware Features and Operating System Interaction

## Introduction: The Hardware-Software Dance

Operating systems and hardware have evolved together in a continuous feedback loop:

```
1970s-1990s:
  Hardware: Simple, single-core CPUs
  OS: Focus on multitasking and virtual memory

2000s:
  Hardware: Multi-core appears, clock speeds plateau
  OS: Must learn parallel scheduling

2010s:
  Hardware: NUMA, heterogeneous cores, deep cache hierarchies
  OS: Topology-aware, energy-aware scheduling

2020s:
  Hardware: Confidential computing, CXL memory tiers, 100+ cores
  OS: Complex placement algorithms, security-performance trade-offs
```

**The modern challenge:**

> Hardware gives us power and complexity. The OS must harness the power while hiding the complexity.

Let's explore each major hardware feature and see exactly how the OS exploits it.

---

## 1. Multi-Core Processors

### The Hardware: Multiple Independent CPUs on One Chip

**What changed:**

```
Old (single-core):
┌────────────────────┐
│   CPU Die          │
│  ┌──────────────┐  │
│  │   Core       │  │
│  │  • Pipeline  │  │
│  │  • L1 Cache  │  │
│  │  • L2 Cache  │  │
│  └──────────────┘  │
└────────────────────┘

New (multi-core):
┌─────────────────────────────────────────┐
│   CPU Die                               │
│  ┌──────────┐  ┌──────────┐  ┌────────┐│
│  │  Core 0  │  │  Core 1  │  │ Core 2 ││
│  │ • L1 I$  │  │ • L1 I$  │  │ • L1 I$││
│  │ • L1 D$  │  │ • L1 D$  │  │ • L1 D$││
│  │ • L2     │  │ • L2     │  │ • L2   ││
│  └────┬─────┘  └────┬─────┘  └────┬───┘│
│       └─────────────┼─────────────┘    │
│                     ↓                   │
│            ┌─────────────────┐          │
│            │  Shared L3 Cache│          │
│            └────────┬────────┘          │
└─────────────────────┼───────────────────┘
                      ↓
              ┌───────────────┐
              │  Main Memory  │
              └───────────────┘
```

**Typical modern CPU:**

```
Consumer (Intel Core i7-13700K):
  • 8 Performance cores (P-cores)
  • 8 Efficiency cores (E-cores)
  • Total: 16 cores, 24 threads (with SMT)
  • L3 Cache: 30 MB shared

Server (AMD EPYC 9654):
  • 96 cores
  • 192 threads (with SMT)
  • L3 Cache: 384 MB (distributed)
```

### Why Multi-Core Exists

**The power wall (early 2000s):**

```
Clock speed vs Power consumption:

Frequency:     1 GHz    2 GHz    3 GHz    4 GHz
Power:         30W      60W      120W     240W
               ↑                            ↑
           Reasonable              Impossible to cool!

Relationship: Power ≈ Frequency³

Can't go faster → Go wider (more cores)
```

**Performance comparison:**

```
Single-core 4 GHz:
  • Task A: 10 seconds
  • Task B: 10 seconds
  • Sequential: 20 seconds total

Dual-core 2 GHz (each):
  • Task A on Core 0: 20 seconds
  • Task B on Core 1: 20 seconds
  • Parallel: 20 seconds total (if tasks are independent)
  
Same total work, much lower power consumption!
```

---

### How the OS Manages Multi-Core: Scheduling

The OS scheduler is completely reimagined for multi-core systems.

#### Discovery: Understanding the Topology

**OS boot-time discovery:**

```
1. Read CPU information (CPUID instruction, ACPI tables, device tree)
2. Build topology map:

Example topology:
  Package 0 (CPU socket):
    ├─ Core 0 (L1: 32KB, L2: 256KB)
    │   ├─ Thread 0 (logical CPU 0)
    │   └─ Thread 1 (logical CPU 1) [if SMT enabled]
    ├─ Core 1 (L1: 32KB, L2: 256KB)
    │   ├─ Thread 0 (logical CPU 2)
    │   └─ Thread 1 (logical CPU 3)
    └─ Shared L3: 16 MB

  Package 1 (CPU socket):
    └─ [Similar structure]

3. Store in kernel data structures for scheduling decisions
```

**Linux example:**

```bash
$ lscpu
Architecture:        x86_64
CPU(s):              24
Thread(s) per core:  2
Core(s) per socket:  6
Socket(s):           2
NUMA node(s):        2
L1d cache:           32K (per core)
L2 cache:            256K (per core)
L3 cache:            16384K (shared)
```

#### Per-Core Run Queues

**Old single-core scheduler:**

```
┌────────────────────────────┐
│   Global Run Queue         │
│   [P1, P2, P3, P4, P5]     │
└─────────────┬──────────────┘
              ↓
         ┌─────────┐
         │   CPU   │
         └─────────┘

Problem: Lock contention when multiple cores try to access queue
```

**Modern multi-core scheduler (per-core queues):**

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Core 0       │    │ Core 1       │    │ Core 2       │
│ Run Queue:   │    │ Run Queue:   │    │ Run Queue:   │
│ [P1, P4]     │    │ [P2, P5]     │    │ [P3, P6]     │
└──────────────┘    └──────────────┘    └──────────────┘

Benefit: Each core manages its own queue (less lock contention)
Challenge: Must balance load across cores
```

**Linux Completely Fair Scheduler (CFS) structure:**

```c
// Simplified kernel structures
struct rq {  // Per-CPU run queue
    int cpu;
    struct cfs_rq cfs;  // CFS-specific queue
    struct task_struct *curr;  // Currently running task
    u64 clock;  // Per-CPU time tracking
};

// One rq per logical CPU
struct rq runqueues[NR_CPUS];
```

#### Cache Affinity: Sticky Scheduling

**The cache problem:**

```
Process P runs on Core 0:
  ┌──────────────┐
  │   Core 0     │
  │   L1: [P's data in cache]
  │   L2: [P's data in cache]
  └──────────────┘

OS migrates P to Core 1:
  ┌──────────────┐       ┌──────────────┐
  │   Core 0     │       │   Core 1     │
  │   L1: [P's data]     │   L1: [empty]│ ← Cache miss!
  │   L2: [P's data]     │   L2: [empty]│ ← Cache miss!
  └──────────────┘       └──────────────┘
                              ↓
                    Must reload from L3/RAM
                    Penalty: 10-100x slower
```

**OS solution: CPU affinity**

```c
// Linux task_struct (process control block)
struct task_struct {
    int cpu;  // Last CPU this task ran on
    cpumask_t cpus_allowed;  // Which CPUs can run this task
    // ...
};

// Scheduler prefers to keep task on same CPU
int select_task_rq(struct task_struct *p) {
    int prev_cpu = p->cpu;
    
    // Prefer previous CPU if it's idle
    if (available(prev_cpu))
        return prev_cpu;
    
    // Otherwise, choose nearby CPU (same cache domain)
    return find_nearby_cpu(prev_cpu);
}
```

**Cache warming effect:**

```
Process P execution timeline on Core 0:

Time 0ms:   Start running (cold cache)
            L1 miss rate: 50% (slow)
Time 10ms:  Cache warming up
            L1 miss rate: 20%
Time 50ms:  Cache fully warm
            L1 miss rate: 5% (fast!)
            
If migrated to Core 1: Back to 50% miss rate!
```

#### Load Balancing: When to Migrate

**The trade-off:**

```
Scenario: Core 0 has 5 processes, Core 1 has 0 processes

Should OS migrate a process from Core 0 to Core 1?

Pros:
  ✓ Better CPU utilization
  ✓ Reduced wait time for processes on Core 0

Cons:
  ✗ Cache miss penalty during migration
  ✗ Lost cache locality

Decision: Migrate only if imbalance is significant
```

**Linux load balancing algorithm:**

```
Periodic balance (every ~4ms):
  for each CPU:
    1. Calculate load (number of runnable tasks × priority)
    2. Find most imbalanced CPU pair
    3. If imbalance > threshold:
       • Select tasks to migrate
       • Move from heavily loaded to lightly loaded CPU
       • Update affinity tracking

Idle balance (immediate):
  When a CPU becomes idle:
    1. Scan for overloaded CPUs
    2. Pull tasks immediately
    3. Avoid leaving CPU idle
```

**Example balancing decision:**

```
Before:
  Core 0: [P1 (cache-hot), P2 (cache-hot), P3 (cache-cold)]
  Core 1: []
  
Decision:
  • Don't migrate P1 or P2 (cache-hot, expensive to move)
  • Migrate P3 (cache-cold, less penalty)
  
After:
  Core 0: [P1, P2]
  Core 1: [P3]
```

#### Power Management: Dynamic Frequency Scaling

**Hardware capability:**

```
Modern CPU can adjust per-core frequency:

Core 0: 800 MHz  (idle, save power)
Core 1: 3.5 GHz  (busy, max performance)
Core 2: 2.0 GHz  (moderate load)
Core 3: 800 MHz  (idle)

OS cooperates with hardware to make these decisions
```

**Linux CPU frequency governors:**

```c
// Performance governor
void performance_governor(int cpu) {
    set_frequency(cpu, max_frequency);
    // Always run at maximum speed
}

// Powersave governor
void powersave_governor(int cpu) {
    set_frequency(cpu, min_frequency);
    // Always run at minimum speed
}

// Ondemand governor (dynamic)
void ondemand_governor(int cpu) {
    int load = get_cpu_load(cpu);
    
    if (load > 80%) {
        set_frequency(cpu, max_frequency);  // Ramp up
    } else if (load < 30%) {
        set_frequency(cpu, min_frequency);  // Ramp down
    }
    // Gradual scaling based on utilization
}
```

**Thermal throttling:**

```
Temperature monitoring:

CPU temp: 60°C  → Normal operation
CPU temp: 80°C  → OS warned, reduce frequency
CPU temp: 90°C  → Hardware throttles automatically
CPU temp: 100°C → Emergency shutdown

OS role:
  • Monitor temperature sensors
  • Reduce workload before hardware emergency
  • Migrate tasks away from hot cores
```

---

### Scheduler Impact Summary

**Multi-core forces these OS design changes:**

| Challenge | OS Solution |
|-----------|-------------|
| **Topology awareness** | Parse ACPI/device trees, build cache hierarchy map |
| **Scalability** | Per-CPU run queues to avoid global lock |
| **Cache locality** | Track last CPU, prefer same CPU for re-scheduling |
| **Load distribution** | Periodic balancing, immediate idle-pulling |
| **Power efficiency** | Dynamic frequency scaling, idle core shutdown |
| **Thermal limits** | Temperature monitoring, workload migration |

---

## 2. Simultaneous Multithreading (SMT / Hyper-Threading)

### The Hardware: Sharing One Core Between Multiple Threads

**How SMT works:**

```
Physical Core Resources:
┌─────────────────────────────────────────┐
│  Execution Units (shared):              │
│    • ALU (arithmetic)                   │
│    • FPU (floating point)               │
│    • Load/Store units                   │
│                                         │
│  Caches (shared):                       │
│    • L1 Instruction Cache               │
│    • L1 Data Cache                      │
│    • L2 Cache                           │
│                                         │
│  Branch Predictor (shared)              │
│  TLB (shared)                           │
└─────────────────────────────────────────┘
         ↑                    ↑
    Thread 0              Thread 1
    ┌─────────┐          ┌─────────┐
    │ PC: 0x1000        │ PC: 0x5000
    │ Registers:│       │ Registers:│
    │ R1=10    │        │ R1=50    │
    │ R2=20    │        │ R2=60    │
    └─────────┘         └─────────┘
    
Each thread has:
  • Own Program Counter
  • Own register set
  • Own stack pointer
  
Threads share everything else!
```

**Why it helps:**

```
Single-threaded execution:

Time →  [Compute][Wait RAM]────────[Compute][Wait RAM]────────
         ↑       ↑                  ↑       ↑
      Using ALU  Idle              Using ALU Idle
      
Execution units idle 70% of the time waiting for memory!


SMT execution (2 threads):

Thread 0: [Compute][Wait RAM]────────[Compute][Wait RAM]────────
Thread 1: ────────[Compute][Wait RAM]────────[Compute][Wait RAM]
          ↑       ↑        ↑        ↑        ↑        ↑
     Both threads share execution units, fill idle gaps

Execution units idle only 30% of the time!
```

**Performance characteristics:**

```
Workload type              Speedup from SMT
────────────────────────  ─────────────────
Memory-bound (DB, cache)   1.3-1.5x
Mixed workload             1.2-1.3x
Compute-bound (crypto)     1.0-1.1x
Adversarial (cache thrash) 0.8x (slower!)
```

---

### How the OS Manages SMT

#### OS Perspective: Logical CPUs

```
Hardware view:
  Physical Core 0
    └─ Thread 0 (T0)
    └─ Thread 1 (T1)

OS view:
  CPU 0 (maps to Core 0, Thread 0)
  CPU 1 (maps to Core 0, Thread 1)
  
OS treats each thread as a separate CPU
```

**Topology discovery:**

```bash
$ cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list
0,8  # CPUs 0 and 8 are SMT siblings (same physical core)

$ cat /sys/devices/system/cpu/cpu0/topology/core_id
0    # Both map to physical core 0
```

#### Scheduling Strategy: Fill Physical Cores First

**Naive approach (bad):**

```
Spread tasks across SMT siblings immediately:

Core 0: [CPU 0: Task A] [CPU 1: Task B]
Core 1: [CPU 2: idle  ] [CPU 3: idle  ]
Core 2: [CPU 4: idle  ] [CPU 5: idle  ]

Problem: Tasks A and B compete for Core 0's resources
         Cores 1 and 2 completely idle
```

**Smart approach (good):**

```
Fill physical cores before using SMT:

Core 0: [CPU 0: Task A] [CPU 1: idle  ]
Core 1: [CPU 2: Task B] [CPU 3: idle  ]
Core 2: [CPU 4: idle  ] [CPU 5: idle  ]

Later, when more tasks arrive:

Core 0: [CPU 0: Task A] [CPU 1: Task D]
Core 1: [CPU 2: Task B] [CPU 3: Task E]
Core 2: [CPU 4: Task C] [CPU 5: idle  ]

Each task gets a full core's resources until cores are saturated
```

**Linux implementation:**

```c
// Scheduling domain structure
struct sched_domain {
    int level;  // SMT, MC (multi-core), NUMA, etc.
    int prefer_sibling;  // Prefer filling siblings?
};

// SMT domain: prefer_sibling = 1 (fill siblings last)
// MC domain: prefer_sibling = 0 (spread across cores)
```

#### Security Considerations: Side-Channel Attacks

**The problem:**

```
Malicious Thread 0 on CPU 0:
  1. Fill L1 cache with probe data
  2. Wait for victim to execute
  3. Measure cache timing
  4. Infer what victim accessed

Victim Thread 1 on CPU 1 (same core):
  1. Accesses secret key
  2. Unknowingly evicts attacker's cache lines
  
Attacker learns secret through timing!

Shared cache = side channel
```

**Examples of attacks:**

- **Spectre:** Exploit speculative execution + shared cache
- **Meltdown:** Exploit out-of-order execution + shared TLB
- **L1TF/Foreshadow:** Exploit shared L1 cache in VMs

**OS mitigations:**

```c
// Option 1: Disable SMT entirely
echo 0 > /sys/devices/system/cpu/smt/control

// Option 2: Schedule only trusted tasks as siblings
struct task_struct {
    int security_domain;
    // ...
};

// Scheduler ensures:
// CPU 0: Task from domain A
// CPU 1: Task from domain A (same security level)
// Never mix domains on same core

// Option 3: Flush caches on context switch (expensive!)
void context_switch(struct task_struct *prev, 
                   struct task_struct *next) {
    if (prev->security_domain != next->security_domain) {
        flush_l1_cache();  // Performance penalty
    }
    // ...
}
```

**Trade-off in practice:**

```
SMT enabled:
  ✓ 20-30% better throughput
  ✗ Security vulnerabilities

SMT disabled:
  ✓ No side-channel risk
  ✗ 20-30% lower throughput

Many cloud providers: Disable SMT on multi-tenant systems
Many desktops: Keep SMT enabled (single user)
```

---

## 3. Heterogeneous Core Architectures (big.LITTLE / P-cores + E-cores)

### The Hardware: Different Cores for Different Jobs

**Intel's hybrid architecture (12th gen+):**

```
┌───────────────────────────────────────────────────┐
│  CPU Die                                          │
│                                                   │
│  Performance Cores (P-cores):                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  P-Core 0│  │  P-Core 1│  │  P-Core 2│  ...   │
│  │ • OOO    │  │ • OOO    │  │ • OOO    │        │
│  │ • SMT    │  │ • SMT    │  │ • SMT    │        │
│  │ • 5.2GHz │  │ • 5.2GHz │  │ • 5.2GHz │        │
│  │ • High IPC│ │ • High IPC│ │ • High IPC│       │
│  └──────────┘  └──────────┘  └──────────┘        │
│                                                   │
│  Efficiency Cores (E-cores):                      │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐      │
│  │E-C0│ │E-C1│ │E-C2│ │E-C3│ │E-C4│ │E-C5│ ...  │
│  │In- │ │In- │ │In- │ │In- │ │In- │ │In- │      │
│  │order│order│ order│ order│ order│ order│      │
│  │3.8 │ │3.8 │ │3.8 │ │3.8 │ │3.8 │ │3.8 │      │
│  │GHz │ │GHz │ │GHz │ │GHz │ │GHz │ │GHz │      │
│  └────┘ └────┘ └────┘ └────┘ └────┘ └────┘      │
└───────────────────────────────────────────────────┘
```

**Characteristics:**

| Feature | P-cores | E-cores |
|---------|---------|---------|
| **Pipeline** | Out-of-order, speculative | In-order, simple |
| **IPC** | ~5-6 instructions/cycle | ~3-4 instructions/cycle |
| **Frequency** | Up to 5.5 GHz | Up to 4.0 GHz |
| **Power** | 10-20W per core | 1-3W per core |
| **Die area** | Large (complex logic) | Small (4 E-cores ≈ 1 P-core) |
| **Best for** | Latency-sensitive, bursty | Throughput, background |

**Energy efficiency comparison:**

```
Task: Transcode 1 hour video

P-core:
  Time: 10 minutes
  Power: 15W
  Energy: 15W × 10min = 150 Watt-minutes

E-core:
  Time: 15 minutes
  Power: 2W
  Energy: 2W × 15min = 30 Watt-minutes
  
E-core uses 5x less energy! (for throughput workloads)
```

---

### How the OS Manages Heterogeneous Cores

#### Intel Thread Director (Hardware Hints)

**The challenge:**

```
OS receives task to schedule:
  • Is this latency-sensitive? (needs P-core)
  • Or background work? (use E-core)
  
How does OS know?
```

**Intel's solution: Hardware telemetry**

```
Thread Director (hardware component):
  1. Monitors each thread:
     • Instruction mix (compute vs memory)
     • Branch behavior
     • Pipeline utilization
  2. Assigns "class hint":
     • Class 0: Needs P-core (latency-sensitive)
     • Class 1: Okay on E-core (throughput)
     • Class 2: Background (E-core preferred)
  3. Exports hints to OS via MSR registers

OS reads hints:
  int hint = read_thread_class_hint(thread_id);
  if (hint == 0)
      schedule_on_p_core(thread);
  else
      schedule_on_e_core(thread);
```

**Example classification:**

```
Foreground tasks:
  • Web browser UI thread → Class 0 (P-core)
  • Video game render loop → Class 0 (P-core)
  • IDE editor thread → Class 0 (P-core)

Background tasks:
  • Video encoding → Class 1 (E-core)
  • File indexing → Class 2 (E-core)
  • Antivirus scan → Class 2 (E-core)
```

#### OS Scheduling Policy

**Windows 11 hybrid scheduler:**

```
Task priority-based placement:

Real-time priority (highest):
  → Always on P-cores
  
High priority (foreground):
  → Prefer P-cores
  → Move to E-cores only if all P-cores busy
  
Normal priority:
  → Start on E-cores
  → Promote to P-cores if performance needed
  
Low priority (background):
  → Always on E-cores
  → Never migrate to P-cores
```

**Linux heterogeneous scheduling (WIP as of 2024):**

```c
// Energy-Aware Scheduling (EAS)
struct energy_env {
    int src_cpu;  // Current CPU
    int dst_cpu;  // Potential target CPU
    struct task_struct *task;
};

// Calculate energy cost of placement
unsigned long compute_energy(struct energy_env *eenv) {
    // Model: Energy = Power × Time
    
    int task_compute = estimate_compute_time(eenv->task);
    int cpu_power = get_cpu_power(eenv->dst_cpu);
    
    return cpu_power * task_compute;
}

// Choose CPU with minimum energy cost
int energy_aware_select_cpu(struct task_struct *p) {
    int best_cpu = -1;
    unsigned long min_energy = ULONG_MAX;
    
    for_each_possible_cpu(cpu) {
        struct energy_env eenv = {
            .task = p,
            .dst_cpu = cpu,
        };
        
        unsigned long energy = compute_energy(&eenv);
        if (energy < min_energy) {
            min_energy = energy;
            best_cpu = cpu;
        }
    }
    
    return best_cpu;
}
```

#### Migration Penalties

**Cache architecture difference:**

```
P-core cache:
  L1: 32 KB + 32 KB (I+D)
  L2: 1.25 MB per core
  
E-core cache:
  L1: 32 KB + 64 KB (I+D)  
  L2: 2 MB shared per 4 E-cores

Migrating P-core → E-core:
  • Different L2 (cold start)
  • Possible L1 size difference
  • Pipeline characteristics change
  
Penalty: 100-500 microseconds of slowdown
```

**OS migration policy:**

```
Don't migrate unless:
  1. CPU is idle for > 1ms (worth the cost)
  2. Load imbalance is significant
  3. Energy savings justify penalty
  
Example decision tree:
  Task on P-core, P-core 80% utilized
    → Stay (migration overhead not worth it)
  
  Task on P-core, P-core 100% utilized, E-core idle
    → If task is low priority: Migrate to E-core
    → If task is high priority: Keep on P-core, migrate other task
```

---

## 4. Non-Uniform Memory Access (NUMA)

### The Hardware: Memory Proximity Matters

**Single-socket (UMA - Uniform Memory Access):**

```
┌──────────┐
│   CPU    │
└────┬─────┘
     ↓
┌────────────┐
│   Memory   │  ← All memory equally close
└────────────┘

Access time: Always 100ns
```

**Multi-socket (NUMA - Non-Uniform Memory Access):**

```
┌─────────────┐              ┌─────────────┐
│  CPU 0      │─────────────│  CPU 1      │
└──────┬──────┘              └──────┬──────┘
       │                            │
┌──────▼──────┐              ┌──────▼──────┐
│  Memory 0   │              │  Memory 1   │
│  (Node 0)   │              │  (Node 1)   │
└─────────────┘              └─────────────┘

CPU 0 accessing Memory 0: 100ns (local)
CPU 0 accessing Memory 1: 200ns (remote, through interconnect)
CPU 1 accessing Memory 0: 200ns (remote)
CPU 1 accessing Memory 1: 100ns (local)

2x latency penalty for remote access!
```

**Modern systems with chiplets:**

```
AMD EPYC (chiplet architecture):

┌─────────────────────────────────────┐
│  CPU Package                        │
│  ┌────────┐ ┌────────┐ ┌────────┐  │
│  │Chiplet0│ │Chiplet1│ │Chiplet2│  │
│  │8 cores │ │8 cores │ │8 cores │  │
│  │NUMA 0  │ │NUMA 1  │ │NUMA 2  │  │
│  └───┬────┘ └───┬────┘ └───┬────┘  │
│      └──────────┼──────────┘        │
│            Infinity Fabric           │
└─────────────────┼───────────────────┘
                  ↓
          ┌───────────────┐
          │  Memory (DDR5)│
          │  Distributed  │
          └───────────────┘

Even single-socket systems can have NUMA!
```

**Access latency matrix:**

```
         To Node 0   To Node 1   To Node 2
From 0:     100ns       150ns       200ns
From 1:     150ns       100ns       150ns
From 2:     200ns       150ns       100ns

Diagonal (local): Fast
Off-diagonal (remote): Slower
```

---

### How the OS Manages NUMA

#### Discovery and Topology Representation

**System boot:**

```bash
# OS discovers NUMA topology
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 64 GB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 64 GB
node distances:
node   0   1
  0:  10  20  # Distance to self: 10, to node 1: 20 (2x)
  1:  20  10
```

**Kernel data structure:**

```c
// Linux NUMA node representation
struct pglist_data {  // "pg_data_t"
    int node_id;
    struct zone *node_zones;  // Memory zones
    unsigned long node_start_pfn;  // Start page frame number
    unsigned long node_present_pages;  // Total pages
    struct task_struct *kswapd;  // Per-node memory reclaim thread
};

// Array of all nodes
pg_data_t *node_data[MAX_NUMNODES];
```

#### First-Touch Memory Allocation

**The policy:**

> Memory is allocated on the NUMA node of the CPU that first touches (accesses) it.

**Why:**

```
Alternative: Allocate memory on a fixed node
  Problem: All remote access if process runs elsewhere

First-touch: Memory follows the accessing CPU
  Benefit: Likely local access in common case
```

**Example:**

```c
// Process running on CPU 0 (Node 0)
int *array = malloc(1000 * sizeof(int));  // Virtual memory allocated

// First touch: Write to array
for (int i = 0; i < 1000; i++) {
    array[i] = i;  // Physical pages allocated on Node 0
}

// Later, if process migrates to CPU 8 (Node 1):
// Accessing array[i] is now remote (slower)
```

**Kernel implementation:**

```c
// Page fault handler (simplified)
void do_page_fault(unsigned long address) {
    struct page *page;
    int node = numa_node_id();  // Current CPU's NUMA node
    
    // Allocate physical page from local node
    page = alloc_page_node(node, GFP_KERNEL);
    
    // Map virtual address to physical page
    map_page(address, page);
}
```

#### NUMA-Aware Scheduling

**Goal:** Keep tasks on the same NUMA node as their memory.

**Scheduler considerations:**

```c
// When choosing CPU for task
int select_task_rq_numa(struct task_struct *p) {
    int task_node = task_numa_node(p);  // Where is task's memory?
    
    // Prefer CPUs on the same node
    for_each_cpu_on_node(cpu, task_node) {
        if (cpu_idle(cpu))
            return cpu;
    }
    
    // If all local CPUs busy, use remote CPU
    return find_idlest_cpu();
}
```

**Example:**

```
Process P's memory mostly on Node 0

Scheduling options:
  CPU 0 (Node 0): Load 50%, local memory access
  CPU 8 (Node 1): Load 10%, remote memory access
  
Naive: Choose CPU 8 (less loaded)
NUMA-aware: Choose CPU 0 (local memory more important)
```

#### Automatic NUMA Balancing

**The problem:**

```
Initial state:
  Process starts on Node 0
  First-touch allocates all memory on Node 0
  
Later:
  Process migrates to Node 1
  All memory accesses now remote (slow!)
```

**Linux solution: Automatic NUMA Balancing**

```
1. Periodically mark pages as "not present" (but remember they exist)
2. When accessed, page fault occurs
3. Track which node is accessing which pages
4. If many remote accesses detected:
   → Migrate pages to the accessing node
   
Result: Memory follows the workload
```

**Implementation:**

```c
// Periodic NUMA page marking
void task_numa_work(struct task_struct *p) {
    // Every ~1 second, mark some pages as NUMA-fault
    for (each page in process) {
        if (should_scan(page)) {
            // Remove present bit, set NUMA special bit
            pte_mknuma(page);  
        }
    }
}

// On page fault
void do_numa_page_fault(unsigned long address) {
    struct page *page = get_page(address);
    int page_node = page_to_nid(page);
    int cpu_node = numa_node_id();
    
    if (page_node != cpu_node) {
        // Remote access detected
        numa_migrate_page(page, cpu_node);  // Move page here
    }
    
    // Restore present bit
    pte_mkpresent(page);
}
```

**Trade-off:**

```
Page migration cost: ~10 microseconds
Benefit: Faster local access (2x)

Migrate if:
  • Page accessed frequently
  • Remote access cost > migration cost
  
Don't migrate if:
  • Page accessed rarely
  • Task migrating frequently (thrashing)
```

---

## 5. Cache and Memory Bandwidth Partitioning

### The Problem: Noisy Neighbors

**Shared cache scenario:**

```
┌──────────────────────────────────────┐
│         Shared L3 Cache (20 MB)      │
└──────────────────────────────────────┘
         ↑                    ↑
    Process A            Process B
  (Database)         (Batch job)
  
Without partitioning:
  Process B fills entire cache
  Process A's data evicted
  Database performance tanks
```

**Memory bandwidth problem:**

```
Memory bandwidth: 100 GB/s total

Process A (critical): Needs 20 GB/s
Process B (background): Using 90 GB/s

Result: A starved, performance degraded
```

---

### Hardware Support: Intel RDT (Resource Director Technology)

**Cache Allocation Technology (CAT):**

```
Divide L3 cache into partitions:

┌────────────────────────────────────────┐
│  L3 Cache (20 MB, 20 ways)             │
├────────────────┬───────────────────────┤
│ Ways 0-9       │ Ways 10-19            │
│ (10 MB)        │ (10 MB)               │
│ Process A      │ Process B             │
│ (guaranteed)   │ (guaranteed)          │
└────────────────┴───────────────────────┘

Hardware enforces: A cannot use B's cache ways
```

**Memory Bandwidth Allocation (MBA):**

```
Throttle memory bandwidth per process:

Process A: 100% bandwidth (critical)
Process B: 30% bandwidth (background, throttled)

Hardware ensures: B cannot steal all bandwidth
```

---

### How the OS Uses Cache Partitioning

**Linux resctrl filesystem:**

```bash
# Create resource control group
$ mkdir /sys/fs/resctrl/critical_app

# Assign cache ways (bitmask: bits 0-9 = ways 0-9)
$ echo "L3:0=0x3ff" > /sys/fs/resctrl/critical_app/schemata
#                      ↑
#                  0x3ff = 0b1111111111 (10 ways)

# Assign process to group
$ echo 1234 > /sys/fs/resctrl/critical_app/tasks

# Process 1234 now has dedicated 10 cache ways
```

**Use cases:**

```
Cloud VM isolation:
  VM 1: 50% of L3 cache
  VM 2: 30% of L3 cache
  VM 3: 20% of L3 cache
  → Predictable performance for tenants

Real-time workload:
  RT task: Dedicated cache partition
  → No interference, bounded latency

Database co-location:
  DB 1: 70% cache + 80% bandwidth
  DB 2: 30% cache + 20% bandwidth
  → Fair resource sharing
```

---

## 6. Hardware Virtualization and Confidential Computing

### Hardware-Assisted Virtualization

**The problem without hardware support:**

```
Guest OS tries to execute privileged instruction:
  → Would fail (running in user mode)
  → Hypervisor must trap and emulate
  → 100-1000x slower!
```

**Intel VT-x / AMD-V solution:**

```
CPU modes:
  VMX root mode (Host/Hypervisor runs here)
  VMX non-root mode (Guest OS runs here)
  
Guest OS can execute privileged instructions directly!
  → Hardware traps to hypervisor only when needed
  → Near-native performance
```

**Extended Page Tables (EPT) / Nested Page Tables (NPT):**

```
Traditional: Guest Virtual → Guest Physical → Host Physical
  Problem: Every memory access needs 2 translations (slow)

EPT/NPT: Hardware does both translations in one step
  Result: Near-native memory performance
```

---

### Confidential Computing: Encrypted VMs

**The threat:**

```
Cloud scenario:
  Tenant: Runs VM with sensitive data
  Cloud provider: Has physical access to servers
  
Problem: Provider can inspect VM memory (passwords, keys, data)
```

**Hardware solution: Intel TDX / AMD SEV-SNP**

```
┌─────────────────────────────────────┐
│  Hypervisor (Untrusted)             │
│  ┌─────────────────────────────┐    │
│  │  Guest VM                   │    │
│  │  ┌───────────────────────┐  │    │
│  │  │  Memory (encrypted)   │  │    │
│  │  │  Key in CPU only      │  │    │
│  │  └───────────────────────┘  │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘

Hypervisor sees:
  Memory contents: Encrypted garbage
  CPU registers: Hidden
  
Only the VM and CPU know the key
```

**OS/Hypervisor responsibilities:**

```
1. Attestation:
   • VM proves it's running on genuine secure hardware
   • Remote party verifies before sending secrets

2. Memory encryption management:
   • OS marks pages as encrypted/unencrypted
   • Hardware encrypts on memory write, decrypts on read

3. I/O handling:
   • DMA must use bounce buffers (encrypted → plaintext)
   • Shared memory with hypervisor must be explicit
```

---

## 7. Compute Express Link (CXL) and Memory Tiering

### CXL: Expanding Memory

**Traditional memory hierarchy:**

```
CPU ↔ DDR Memory (64 GB)
  ↓
SSD (1 TB)
  ↓
Hard Disk (10 TB)

Gap: No option between 64 GB RAM and slow SSD
```

**CXL adds memory tier:**

```
CPU ↔ DDR Memory (64 GB) ← Tier 0 (fast)
  ↓
CPU ↔ CXL Memory (512 GB) ← Tier 1 (medium speed)
  ↓
SSD (1 TB) ← Tier 2 (slow)

CXL: PCIe-based, coherent, 100+ GB/s bandwidth
     Slower than DDR but much faster than SSD
```

**Latency comparison:**

```
DDR5:        ~100ns
CXL memory:  ~200-300ns (2-3x slower)
NVMe SSD:    ~50,000ns (500x slower)

CXL fills the gap!
```

---

### How the OS Manages Memory Tiers

**Tier classification:**

```bash
# Linux exposes memory tiers
$ cat /sys/devices/system/node/node0/memory_tier
Tier 0  # DDR (fast)

$ cat /sys/devices/system/node/node2/memory_tier  
Tier 1  # CXL (medium)
```

**Allocation policy:**

```
New allocation:
  1. Try Tier 0 (DDR) first
  2. If full, use Tier 1 (CXL)
  3. If both full, swap to disk
```

**Automatic tiering (promotion/demotion):**

```
Kernel monitors page access patterns:

Hot page (frequently accessed):
  • If on Tier 1 (CXL) → Promote to Tier 0 (DDR)
  
Cold page (rarely accessed):
  • If on Tier 0 (DDR) → Demote to Tier 1 (CXL)
  
Goal: Keep hot data in fast memory, cold data in slower memory
```

**Implementation:**

```c
// Periodic page scanning
void memory_tiering_daemon(void) {
    for_each_page(page) {
        int accesses = get_access_count(page);
        int current_tier = page_tier(page);
        
        if (accesses > HOT_THRESHOLD && current_tier == 1) {
            // Promote hot page from CXL to DDR
            migrate_page(page, TIER_0);
        } else if (accesses < COLD_THRESHOLD && current_tier == 0) {
            // Demote cold page from DDR to CXL
            migrate_page(page, TIER_1);
        }
    }
}
```

**Use case:**

```
Database server:
  Active index: 32 GB → DDR (fast)
  Archive data: 256 GB → CXL (medium)
  Cold backups: SSD (slow)
  
Result: Hot data fast, total capacity large
```

---

## 8. Performance Monitoring Units (PMU)

### The Hardware: Counting What Actually Happens

**PMU capabilities:**

```
Hardware event counters:
  • CPU cycles executed
  • Instructions retired
  • L1/L2/L3 cache misses
  • Branch mispredictions
  • TLB misses
  • Memory bandwidth used
  • Power consumed
```

**How it works:**

```
CPU registers:
  PMC0: Count L3 cache misses
  PMC1: Count branch mispredictions
  PMC2: Count instructions retired
  ...
  
On each event:
  PMC[i]++;  // Hardware increments counter
  
OS reads counters to understand performance
```

---

### How the OS Uses PMU

#### Profiling: Finding Hotspots

**Sampling-based profiling:**

```
1. Configure PMU: "Interrupt every 100,000 CPU cycles"
2. CPU executes instructions
3. After 100,000 cycles: Hardware interrupt
4. OS records: Which instruction was executing?
5. Repeat
6. Statistical hotspot map:
   function_a: 5000 samples (50% of time)
   function_b: 3000 samples (30% of time)
   function_c: 2000 samples (20% of time)
```

**Linux `perf` tool:**

```bash
# Profile program and find hotspots
$ perf record ./my_program
$ perf report

# Output:
# 45.00%  my_program  [.] hot_function
# 30.00%  my_program  [.] medium_function  
# 20.00%  my_program  [.] cold_function

# Detailed event counts
$ perf stat ./my_program

Performance counter stats:
  1,234,567,890 cycles
  1,000,000,000 instructions  (0.81 IPC)
     50,000,000 cache-misses  (4% miss rate)
     10,000,000 branch-misses (1% miss rate)
```

#### Feedback-Driven Scheduling

**Using PMU for smart decisions:**

```c
// Measure task characteristics
struct task_profile {
    u64 cache_misses;
    u64 instructions;
    u64 memory_accesses;
};

// Classify task based on PMU data
enum task_type classify_task(struct task_struct *p) {
    struct task_profile *prof = &p->pmu_profile;
    
    double miss_rate = (double)prof->cache_misses / prof->memory_accesses;
    
    if (miss_rate > 0.2) {
        return MEMORY_BOUND;  // Schedule on core with large cache
    } else {
        return COMPUTE_BOUND;  // Schedule on high-frequency core
    }
}
```

**Application:**

```
Heterogeneous scheduling with PMU feedback:

Task characteristics (from PMU):
  • High cache miss rate → Memory-bound
    → Schedule on E-core (won't benefit from P-core speed)
  
  • High IPC, low miss rate → Compute-bound
    → Schedule on P-core (needs raw compute power)
```

#### Power Management

**RAPL (Running Average Power Limit):**

```bash
# Read CPU power consumption (via PMU)
$ perf stat -e power/energy-pkg/ sleep 1

 15.67 Joules power/energy-pkg/
```

**OS power budget enforcement:**

```c
// Monitor power consumption
u64 current_power = read_rapl_energy();

if (current_power > power_budget) {
    // Reduce frequency or throttle cores
    reduce_cpu_frequency();
}
```

---

## Summary: The Modern OS Challenge

### The Complexity Explosion

**From simple to complex:**

```
1990s OS:
  • Manage a single CPU
  • Allocate flat memory
  • Schedule processes fairly
  
2025 OS:
  • Manage 100+ heterogeneous cores (P-cores, E-cores, SMT)
  • Navigate NUMA topology and memory tiers
  • Balance performance, power, and security
  • Partition caches and bandwidth
  • Handle encrypted VMs
  • Optimize for specific microarchitectural behaviors
  
Complexity: 100x increase
```

### Key Principles for Modern OS Design

**1. Topology Awareness:**
```
Don't treat all CPUs/memory as equal
  → Discover hierarchy
  → Make placement decisions based on locality
```

**2. Energy Consciousness:**
```
Performance isn't the only goal
  → Measure energy cost
  → Choose efficient cores when possible
  → Scale frequency dynamically
```

**3. Security as Priority:**
```
Hardware features can leak information
  → Understand side channels
  → Mitigate (disable SMT, flush caches)
  → Balance security vs performance
```

**4. Adaptive Behavior:**
```
Workload characteristics change
  → Monitor with PMU
  → Adjust scheduling, frequency, placement
  → Learn and optimize over time
```

**5. Explicit Trade-offs:**
```
No perfect solution exists
  → Cache affinity vs load balance
  → Power savings vs latency
  → Security vs throughput
  
OS makes informed trade-offs based on priorities
```

---

## The Future: OS Evolution Continues

**Emerging challenges:**

```
2025-2030 trends:
  • 1000+ core systems
  • Optical/3D-stacked memory
  • Domain-specific accelerators (AI, crypto)
  • Quantum co-processors
  • Brain-inspired neuromorphic chips

OS must continue adapting:
  • New scheduling algorithms
  • New memory models
  • New security paradigms
  • New abstractions
```

**The constant:**

> Hardware and OS co-design
> OS must understand and exploit hardware
> Hardware must provide interfaces OS can use
> Together, they deliver the computing we depend on

**Bottom line:**

Modern operating systems are no longer simple resource managers. They are complex, adaptive systems that must understand hardware at a deep level, make continuous trade-offs between conflicting goals, and provide both high performance and strong guarantees—all while making it look simple to applications and users.