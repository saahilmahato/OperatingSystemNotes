# Types of Operating Systems

Operating systems are classified by **how they manage execution, interaction, timing, and deployment**.
Each type represents a **design response to a specific constraint** (hardware cost, user interaction, timing guarantees, scale).

---

## 1. Batch Operating Systems

### Definition

A **batch OS** executes groups of jobs sequentially with **no user interaction during execution**.

### Key Characteristics

* Jobs submitted in advance
* Sequential execution
* No preemption or interactivity
* Throughput prioritized over response time

### Core Mechanisms

* Job queues
* Simple job scheduler
* Spooling for I/O overlap

### What It Does *Not* Support

* Interactive users
* Preemptive multitasking
* Real-time guarantees

### Historical Context

Used on early mainframes where CPU time was extremely expensive.

---

## 2. Multiprogramming Operating Systems

### Definition

A **multiprogramming OS** keeps **multiple programs in memory** and switches the CPU among them to maximize utilization.

### Core Idea

> When one program blocks (e.g., for I/O), another runs.

### Essential Requirements

* Dual CPU modes (user / kernel)
* Memory protection and address translation
* Interrupts (especially I/O completion)
* Context switching

Without these → **unsafe execution**

### What It Enables

* CPU–I/O overlap
* Better hardware utilization

### What It Does *Not* Guarantee

* Fast user response
* Fairness
* Interactivity

---

## 3. Time-Sharing (Interactive) Operating Systems

### Definition

A **time-sharing OS** is a multiprogramming system optimized for **interactive use**, where users perceive responsiveness.

### Core Characteristics

* Preemptive scheduling
* Short time slices
* Rapid context switching
* Multiple concurrent users or applications

### Key Distinction from Multiprogramming

* **Response time** matters
* Not just CPU utilization

### Typical Examples

* Unix/Linux
* Windows
* macOS

---

## 4. Real-Time Operating Systems (RTOS)

### Definition

An **RTOS** guarantees that tasks meet **strict timing constraints**.

Correctness depends on **time + result**, not just result.

---

### Hard Real-Time OS

* Missing a deadline = system failure
* Used in safety-critical systems

Examples:

* Avionics
* Medical devices
* Automotive control systems

---

### Soft Real-Time OS

* Occasional deadline misses acceptable
* Performance degrades gracefully

Examples:

* Multimedia systems
* Streaming
* Online gaming

---

### Core RTOS Properties

* Deterministic scheduling
* Bounded interrupt latency
* Predictable execution
* Minimal kernel complexity

---

## 5. Distributed Operating Systems

### Definition

A **distributed OS** manages multiple computers and presents them as **one unified system**.

### Key Properties

* Single system image
* Location transparency
* Distributed resource management
* Message-based communication

### Core Challenges

* Synchronization
* Fault tolerance
* Partial failures
* Clock coordination

### Note

True distributed OSs are rare due to complexity.

---

## 6. Network Operating Systems

### Definition

A **network OS** provides **network services**, not a single unified system.

Each machine:

* Runs its own OS
* Retains autonomy

### Services Provided

* File sharing
* Authentication
* Print services

### Key Difference from Distributed OS

> **No single system image**

---

## 7. Mobile Operating Systems

### Definition

A **mobile OS** is designed for **battery-powered, sensor-rich, touch-based devices**.

### Design Priorities

* Power efficiency
* Application sandboxing
* Aggressive background process control
* Strong permission model

### Examples

* Android
* iOS

---

## 8. Embedded Operating Systems

### Definition

An **embedded OS** runs on **dedicated-function devices** with tight hardware constraints.

### Key Characteristics

* Small memory footprint
* Often real-time
* Minimal or no UI
* Long uptimes

### Typical Environments

* Microcontrollers
* Industrial systems
* IoT devices

---

## High-Value Comparison (Conceptual)

| OS Type          | Interactivity | Preemption | Timing Guarantees | System Scope      |
| ---------------- | ------------- | ---------- | ----------------- | ----------------- |
| Batch            | No            | No         | No                | Single machine    |
| Multiprogramming | No            | Optional   | No                | Single machine    |
| Time-sharing     | Yes           | Yes        | No                | Single machine    |
| Real-time        | Optional      | Yes        | **Yes**           | Single / Embedded |
| Distributed      | Yes           | Yes        | Varies            | Multiple machines |
| Network          | Yes           | Yes        | No                | Networked         |
| Mobile           | Yes           | Yes        | Soft              | Single device     |
| Embedded         | Rare          | Often      | Often             | Single device     |

---

## One-Line Mental Model (Exam + Design Gold)

* **Batch** → throughput
* **Multiprogramming** → utilization
* **Time-sharing** → responsiveness
* **Real-time** → predictability
* **Distributed** → transparency
* **Network** → services
* **Mobile** → power + isolation
* **Embedded** → constraints
