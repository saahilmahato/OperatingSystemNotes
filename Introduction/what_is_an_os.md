# Introduction to Operating Systems

## 1. What Is an Operating System?

An **Operating System (OS)** is **system software** that manages computer hardware resources and provides a controlled execution environment for programs.

It serves as an **intermediary between applications/users and the hardware**, allowing programs to run without directly managing hardware details such as CPU scheduling, memory allocation, or device control.

---

## 2. Two Fundamental Views of an Operating System

### 2.1 OS as a Resource Manager

The OS is responsible for:

* Allocating CPU time
* Managing memory space
* Controlling I/O devices
* Managing storage

Its job is to ensure:

* **Efficiency** (maximum utilization)
* **Fairness** (no starvation)
* **Isolation** (one program does not harm another)

---

### 2.2 OS as an Extended (Virtual) Machine

The OS provides:

* High-level abstractions over low-level hardware
* A simpler programming model

Instead of dealing with:

* Registers, disk sectors, interrupts

Programs deal with:

* Processes
* Virtual memory
* Files
* Sockets

This abstraction is what makes modern software development feasible.

---

## 3. Goals of an Operating System

### 3.1 Primary Goals

* Execute user programs correctly
* Make the system convenient to use

---

### 3.2 Secondary (Modern) Goals

* Efficient resource utilization
* Security and protection
* Reliability and fault tolerance
* Scalability (multiple users, cores, devices)
* Ability to evolve with new hardware

---

## 4. Core Functions of an Operating System

### 4.1 Process Management

Responsible for managing program execution.

Includes:

* Process creation and termination
* CPU scheduling
* Context switching
* Inter-process communication (IPC)
* Synchronization and deadlock handling

**Key abstraction:** process and thread

---

### 4.2 Memory Management

Controls how memory is used and shared.

Includes:

* Tracking memory usage
* Allocation and deallocation
* Virtual memory
* Address translation
* Memory protection and isolation

**Key abstraction:** virtual address space

---

### 4.3 File System Management

Provides persistent storage abstraction.

Includes:

* File and directory creation/deletion
* Read/write operations
* Metadata management
* Access control and permissions
* File system organization

**Key abstraction:** file

---

### 4.4 Device and I/O Management

Manages communication with hardware devices.

Includes:

* Device drivers
* I/O scheduling
* Buffering and caching
* Interrupt handling
* Device abstraction

**Key abstraction:** device-independent I/O

---

### 4.5 Security and Protection

Ensures system safety and isolation.

Includes:

* Authentication (users, credentials)
* Authorization (permissions)
* Process isolation
* Protection from faulty or malicious programs

---

### 4.6 User Interface and APIs

Provides interaction mechanisms.

Includes:

* Command-line interfaces (shells)
* Graphical user interfaces (GUI)
* System APIs and libraries

---

## 5. Operating System Structure (Layered View)

Typical abstraction stack (bottom → top):

Hardware
↓
Kernel (privileged mode)
↓
System call interface
↓
Core OS services (process, memory, file system, drivers)
↓
Libraries and runtime environments
↓
User applications, shell, GUI

---

### 5.1 Execution Modes

Most modern OSs operate in **two CPU modes**:

* **Kernel mode**

  * Full hardware access
  * Executes OS core code

* **User mode**

  * Restricted access
  * Executes application code

Transitions occur via:

* System calls
* Interrupts
* Traps and exceptions

This separation is critical for security and stability.

---

## 6. Key Abstractions Provided by the OS

| Hardware Resource | OS Abstraction         | Purpose                             |
| ----------------- | ---------------------- | ----------------------------------- |
| CPU               | Process / Thread       | Illusion of concurrent execution    |
| Physical memory   | Virtual memory         | Isolation and simplified addressing |
| Disk blocks       | Files & directories    | Persistent named storage            |
| Devices           | Device-independent I/O | Portability                         |
| Interrupts        | Signals / events       | Structured async handling           |

---

## 7. Types of Operating Systems

Common classifications:

* Batch operating systems (historical)
* Time-sharing / multi-user systems
* Distributed operating systems
* Network operating systems
* Real-time operating systems

  * Hard real-time
  * Soft real-time
* Embedded operating systems
* Mobile operating systems

---

## 8. Modern General-Purpose Operating Systems

Examples:

* **Linux** – monolithic kernel
* **Windows (NT family)** – hybrid kernel
* **macOS** – XNU hybrid kernel (Mach + BSD)
* **Android** – Linux kernel with custom user space
* **iOS** – Darwin/XNU kernel
* **BSD family** – FreeBSD, OpenBSD, NetBSD

---

## 9. Core Mental Model for Systems Engineers

An operating system should be understood from **four perspectives**:

1. **User perspective**
   Convenience and usability

2. **Application perspective**
   Powerful abstractions and services

3. **Hardware perspective**
   Resource allocation and control

4. **Designer perspective**
   Trade-offs between performance, security, reliability, and extensibility

---

## 10. One-Line Definition (Exam Gold)

> **An operating system is a resource manager and abstraction layer that controls hardware and provides a secure, efficient execution environment for programs.**

---
