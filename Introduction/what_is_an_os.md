# Introduction to Operating Systems

## What is an Operating System?

An operating system (OS) is system software that manages computer hardware, software resources, and provides common services for computer programs.

It acts as an intermediary between users/applications and the underlying hardware, enabling applications to run without needing to know the hardware details.

Two fundamental perspectives:

1. **Resource manager**  
   The OS allocates and controls resources (CPU time, memory space, I/O devices, storage) among competing programs in an efficient and fair manner.

2. **Extended machine**  
   The OS provides a simpler, more convenient abstract machine to programmers and users, hiding the complexity of the raw hardware.

## Goals of an Operating System

Primary goals:

- Execute user programs and make solving user problems easier
- Make the computer system convenient to use

Secondary (but critical modern) goals:

- Efficient operation of the computer system
- Ability to evolve (adapt to new hardware/software over time)
- Security and protection
- Reliability and fault tolerance

## Main Functions of an Operating System

1. **Process Management**  
   - Process creation and termination  
   - Process scheduling  
   - CPU allocation  
   - Inter-process communication (IPC)  
   - Synchronization

2. **Memory Management**  
   - Tracking memory usage  
   - Memory allocation and deallocation  
   - Virtual memory implementation  
   - Memory protection between processes

3. **File System Management**  
   - File creation, deletion, reading, writing  
   - Directory management  
   - Access control and permissions  
   - File system structure and organization

4. **Device / I/O Management**  
   - Device driver management  
   - I/O scheduling and buffering  
   - Interrupt handling  
   - Device abstraction

5. **Security and Protection**  
   - User authentication  
   - Access control  
   - Process isolation  
   - Protection from unauthorized access

6. **User Interface**  
   - Command-line interface (shell)  
   - Graphical user interface (GUI)  
   - Application programming interfaces (APIs)

## Operating System Structure – Layered View

Typical layered abstraction (bottom to top):
Hardware
↓
Kernel / privileged mode (full hardware access)
↓
System calls / kernel interface
↓
Operating system services (file system, memory manager, process manager, device drivers, etc.)
↓
Libraries and runtime support
↓
User applications / shell / GUI


Most modern OSs separate execution into two modes:

- **Kernel mode** (privileged) – full access to hardware and system resources  
- **User mode** – restricted access; most application code runs here

Transition between modes occurs via system calls, interrupts, and traps.

## Key Abstractions Provided by the OS

| Hardware Concept          | OS Abstraction Provided              | Benefit to Programmer/User                     |
|---------------------------|--------------------------------------|------------------------------------------------|
| CPU                       | Process / Thread                     | Illusion of multiple programs running simultaneously |
| Physical memory           | Virtual address space                | Each process gets its own large, private memory |
| Physical disk sectors     | Files and directories                | Named, hierarchical, persistent storage        |
| Hardware devices          | Device-independent I/O interface     | Same code works with different devices         |
| Interrupts & exceptions   | Signals / events                     | Structured way to handle asynchronous events   |

## Types of Operating Systems (Common Categories)

- **Batch operating systems** (historical)  
- **Time-sharing / multi-user systems**  
- **Distributed operating systems**  
- **Network operating systems**  
- **Real-time operating systems** (hard vs soft real-time)  
- **Mobile operating systems**  
- **Embedded operating systems**

## Modern General-Purpose Operating Systems (Examples)

- **Linux** (and distributions: Ubuntu, Fedora, Debian, Arch, etc.) – monolithic kernel  
- **Windows** (NT family: Windows 10, 11, Server) – hybrid kernel  
- **macOS** – XNU hybrid kernel (mach + BSD)  
- **Android** – Linux kernel + custom user-space  
- **iOS** – Darwin/XNU kernel  
- **FreeBSD**, **OpenBSD**, **NetBSD** – BSD family

## Summary – Core Mental Model

A good systems engineer thinks of an operating system in these four complementary ways:

1. **From the user perspective** → convenience and ease of use  
2. **From the application perspective** → powerful abstractions and services  
3. **From the hardware perspective** → resource allocator and arbitrator  
4. **From the designer perspective** → complex balance of performance, security, reliability, and extensibility

Understanding these viewpoints deeply is essential for mastering operating systems concepts.