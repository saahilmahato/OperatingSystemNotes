# Parts of an Operating System

An operating system is structured in **layers**, starting from the lowest-level code that runs at power-on, up to user-facing applications. Each layer has a clearly defined responsibility and interacts with adjacent layers through well-defined interfaces.

---

## 1. Firmware and Boot Process

*(The first software that runs after power-on)*

### BIOS / UEFI

* **Firmware** stored on a non-volatile chip on the motherboard.
* Executes immediately after the system is powered on.
* Responsibilities:

  * Initialize CPU, memory, and essential devices
  * Perform **POST** (Power-On Self-Test)
  * Locate and start a bootable device
  * Transfer control to the **bootloader**

**BIOS vs UEFI**

* **BIOS**: Legacy system, limited features
* **UEFI** (modern standard):

  * Faster startup
  * Support for large disks (GPT)
  * Graphical configuration interface
  * **Secure Boot** to prevent unauthorized boot software

---

### Bootloader

* A small program stored on disk.
* Loads the **kernel** into memory and starts it.
* May provide a menu for selecting operating systems.

**Common examples**

* GRUB, systemd-boot (Linux)
* Windows Boot Manager
* Apple bootloader

---

## 2. The Kernel

*(The core of the operating system)*

The **kernel** is the most critical component of the OS.
It runs in **kernel mode**, with unrestricted access to hardware.

All other software must interact with hardware **indirectly**, through the kernel.

---

### 2.1 Kernel Architecture Styles

#### Monolithic Kernel

* Most OS services run inside the kernel:

  * Device drivers
  * File systems
  * Networking
* High performance, lower isolation

**Examples**: Linux, FreeBSD

---

#### Microkernel

* Only minimal functionality in kernel:

  * Scheduling
  * IPC
  * Basic memory management
* Other services run in user space

**Advantages**: strong isolation, reliability
**Tradeoff**: IPC overhead

**Examples**: QNX, MINIX, seL4

---

#### Hybrid Kernel

* Combines ideas from both models
* Performance-critical services in kernel space
* Modular design

**Examples**: Windows NT, macOS (XNU)

---

### 2.2 Core Responsibilities of the Kernel

#### Process and Thread Management

* **Process**: an executing program with its own address space
* **Thread**: a lightweight execution unit within a process

Kernel responsibilities:

* Process creation and termination
* CPU scheduling
* Context switching
* Thread synchronization

---

#### Memory Management

* Provides each process with a **virtual address space**
* Prevents processes from accessing each other’s memory

Key concepts:

* Paging
* Page faults
* Swapping
* Memory protection and access control

---

#### File System and Storage Management

* Abstracts storage devices into files and directories
* Allows uniform access regardless of storage type

Key components:

* **Virtual File System (VFS)** abstraction
* Page cache
* I/O scheduling
* File system permissions and encryption

---

#### Device Drivers

* Specialized kernel components that communicate with hardware
* Translate generic OS requests into device-specific commands

In Linux:

* Often implemented as **loadable kernel modules**

---

#### Networking Stack

* Implements communication protocols
* Handles packet routing, buffering, and transmission

Includes:

* TCP/IP
* Sockets API
* Firewalls and packet filtering

---

#### Security and Protection

* Enforces system security policies

Includes:

* User and process permissions
* Capabilities
* Mandatory access control (SELinux, AppArmor)
* **seccomp** (restricts system calls)
* **Namespaces** (isolation)
* **cgroups** (resource limits)

---

#### Inter-Process Communication (IPC)

Mechanisms for communication between processes:

* Pipes
* Shared memory
* Signals
* Message queues
* Sockets

---

#### Power and Thermal Management

* CPU frequency scaling
* Power-saving states
* Thermal monitoring and throttling

---

## 3. System Call Interface

*(Boundary between user space and kernel space)*

User programs cannot access hardware directly.

They request kernel services via **system calls**, which safely transfer control from:

* **User mode → Kernel mode**

Examples:

* File I/O
* Process creation
* Network communication

This interface is critical for **security and stability**.

---

## 4. Core User-Space Libraries and Runtime

### C Standard Library (libc)

* Provides higher-level functions built on system calls
* Used by almost all applications

Common implementations:

* **glibc** – feature-rich, widely used
* **musl** – small, simple, container-friendly

---

### POSIX API

* **Portable Operating System Interface**
* Defines standard behavior for Unix-like systems

Benefits:

* Source-level portability
* Consistent system programming model

---

## 5. System Services and Daemons

*(Background system processes)*

### Init System

* First user-space process started by the kernel
* Responsible for starting and managing all other services

Examples:

* systemd (modern Linux)
* OpenRC
* SysV init (legacy)
* launchd (macOS)

---

### Common System Daemons

* Network configuration services
* Device management (udev)
* Inter-process messaging (D-Bus)
* Remote access (SSH)

---

## 6. Graphical and User Interface Stack

### Display Server / Protocol

* Handles display output and input events

Examples:

* X11 (legacy)
* Wayland (modern, more secure)

---

### Window Manager / Compositor

* Controls window layout, animations, and rendering

Examples:

* Mutter
* KWin
* Sway

---

### Desktop Environment

* Complete graphical user experience

Examples:

* GNOME
* KDE Plasma
* XFCE
* Cinnamon

---

### Widget Toolkits

* UI libraries for building applications

Examples:

* GTK
* Qt

---

## 7. User Applications and Tools

* Shells and terminals
* File managers
* Browsers, editors, IDEs
* Package managers

---

## 8. Modern and Emerging OS Concepts

* **eBPF** – programmable, safe kernel extensions
* **Flatpak / Snap** – universal application packaging
* **PipeWire** – modern audio/video infrastructure
* **Immutable operating systems** – atomic updates and rollback support

---

## Final Mental Model

> **The kernel controls hardware.
> User space provides services.
> Applications consume abstractions.**

This layered structure is what makes modern operating systems **powerful, secure, portable, and scalable**.

---
