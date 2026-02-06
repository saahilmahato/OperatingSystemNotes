# What is a Process?

## Definition

A **process** is a **program in execution**.

It is the operating system’s abstraction for representing a running program, including all the information needed to execute, manage, and protect that program.

A process is not just code — it is a *live execution environment*.

---

## Core Idea

When a program runs, the operating system creates a process to track:

- What instructions are executing
- What memory is being used
- What resources are allocated
- The current CPU state

This allows multiple programs to run safely and independently on the same machine.

---

## Program vs Process

A key distinction:

### Program
- Static instructions stored on disk
- Passive entity
- Does not execute by itself

### Process
- Active execution of a program
- Exists in memory
- Has runtime state managed by the OS

Example:

A text editor installed on disk is a **program**.  
Opening it creates a **process**.

Multiple processes can originate from the same program.

---

## What a Process Contains

A process is made of several essential components:

### Execution Context
The current CPU state needed to resume execution:

- Program counter (next instruction)
- CPU registers
- Status flags

This defines *where* the program is in execution.

---

### Address Space
The memory assigned to the process:

- Executable code
- Global/static data
- Dynamically allocated memory
- Function call stack

Each process gets its own protected address space.

---

### Resources
The OS may assign resources such as:

- Open files
- I/O handles
- Network connections

These belong to the process and are tracked by the OS.

---

### OS Metadata
The operating system maintains internal data structures describing the process, including identifiers and management information.

This allows the OS to control execution and enforce isolation.

---

## Process as an Abstraction

A process gives the illusion that:

- A program runs independently
- It owns its memory
- It has dedicated CPU time

In reality, the OS shares hardware among many processes while maintaining strict boundaries.

This abstraction enables safe multitasking.

---

## Isolation and Protection

Processes are isolated from one another:

- One process cannot directly read or modify another’s memory
- Faults in one process do not crash others
- Security boundaries are enforced

This isolation is critical for system stability and safety.

---

## Why Processes Matter

Processes are the foundation of modern computing because they provide:

- Safe multitasking
- Resource ownership
- Execution control
- Fault containment
- Security boundaries

Nearly every operating system feature builds on this abstraction.

---

## Mental Model

A helpful way to think about a process:

> A process is a **self-contained running workspace** where a program executes under the supervision of the operating system.

It combines execution state, memory, and resources into a manageable unit.

---

## Summary

A process is:

> A protected, managed instance of a running program that contains execution state, memory, and system resources.

It is the operating system’s primary unit for executing and controlling software.
