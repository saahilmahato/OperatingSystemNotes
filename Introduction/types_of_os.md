# Types of Operating Systems

## Understanding the Classification

Operating systems aren't one-size-fits-all. Different computing environments face different challenges:

- A 1960s mainframe sharing one expensive CPU among dozens of users
- A smartphone trying to maximize battery life
- A car's anti-lock braking system that must respond in milliseconds
- A network of servers working together on a massive computation

Each of these scenarios demands a different OS design. The "type" of OS reflects the **primary design constraint** it's optimized for.

**Key insight:** These types aren't mutually exclusive. A modern smartphone OS is simultaneously:
- Time-sharing (runs multiple apps)
- Mobile (power-efficient, touch-based)
- Real-time (soft deadlines for audio/video)

Let's explore each type by understanding the **problem it solves**.

---

## 1. Batch Operating Systems

### The Problem: Expensive CPU Time in the 1960s

Imagine it's 1965. Your company has one computer—a room-sized mainframe costing millions of dollars. CPU time is **absurdly expensive**. You can't afford to waste a single second.

**The wasteful way (no OS):**
```
Operator manually loads Program A → Run → Wait → Unload
    (5 min setup) (10 min run) (2 min) (3 min cleanup)

Operator loads Program B → Run → Wait → Unload
    (5 min setup) (15 min run) (2 min) (3 min cleanup)

Total time: 45 minutes
Actual CPU running: 25 minutes
Wasted time: 20 minutes (44%!)
```

### The Solution: Batch Processing

**Batch OS approach:**
```
1. Collect a "batch" of jobs (programs)
2. Load them all at once
3. Execute sequentially, automatically
4. Produce output when all complete

Jobs: [A, B, C, D, E]
       ↓
   Queue them
       ↓
   Execute A → B → C → D → E (no human intervention)
       ↓
   Collect all outputs
```

### Key Characteristics

**No interactivity:**
- Submit your job in the morning
- Come back in the afternoon for results
- Can't change anything once submitted

**Job control:**
```
// Job Control Language (JCL) example
//JOB1    JOB  (ACCT123),'PAYROLL RUN'
//STEP1   EXEC PGM=PAYROLL
//INPUT   DD   DSN=EMPLOYEE.DATA,DISP=SHR
//OUTPUT  DD   DSN=PAYCHECKS.PRINT,DISP=(NEW,CATLG)
```

**Simple scheduler:**
```
Job Queue: [Job1, Job2, Job3, Job4]
              ↓
        Pick first job
              ↓
        Run to completion
              ↓
        Pick next job
              ↓
           Repeat
```

**Spooling for efficiency:**

Simultaneous Peripheral Operations On-Line (SPOOL):
```
While CPU runs Job 1:
  ↓
Card reader prepares Job 2 input
Printer outputs Job 0 results
  ↓
No waiting for slow I/O devices
```

### What It Does NOT Support

❌ **Interactive users** — No keyboard, no screen feedback  
❌ **Preemption** — A running job cannot be interrupted  
❌ **Multitasking** — One job at a time  
❌ **Real-time responses** — Jobs wait in queue

### Historical Context

**Peak era:** 1960s-1970s mainframes

**Why it died:** 
- CPUs became cheaper
- Users demanded interactivity
- Better OS designs emerged

**Legacy:**
- Modern batch processing still exists (e.g., nightly data processing, rendering farms)
- But as a *feature* of interactive OSs, not the entire OS

---

## 2. Multiprogramming Operating Systems

### The Problem: CPU Sitting Idle While Waiting for I/O

Even with batch processing, there's still massive waste:

```
Program execution timeline:

Program A: [CPU 2ms] [Wait for disk I/O 100ms] [CPU 3ms]
                         ↑
                 CPU does NOTHING for 100ms!

Disk is 100,000x slower than CPU.
During I/O, CPU could be doing useful work.
```

### The Solution: Keep Multiple Programs in Memory

**Multiprogramming breakthrough:**

> When one program blocks waiting for I/O, switch the CPU to run another program.

**Timeline with multiprogramming:**
```
Time →   0    50   100  150  200  250

Program A: [CPU] [──I/O wait──────] [CPU]
Program B:       [CPU] [──I/O wait──] [CPU]
Program C:             [CPU] [──I/O──] [CPU]

CPU usage: ████████████████████████████
           Never idle!
```

### How It Works

**Memory layout:**
```
┌─────────────────────┐
│  Operating System   │
├─────────────────────┤
│  Program A          │ ← Running
├─────────────────────┤
│  Program B          │ ← Blocked (waiting for disk)
├─────────────────────┤
│  Program C          │ ← Ready (waiting for CPU)
├─────────────────────┤
│  Program D          │ ← Ready
└─────────────────────┘
```

**State transitions:**
```
Program A is Running
        ↓
Program A requests disk I/O
        ↓
OS marks A as Blocked
OS selects Program C (Ready → Running)
        ↓
Program C runs on CPU
        ↓
Disk interrupt: "A's I/O complete!"
        ↓
OS marks A as Ready
(Later, A will run again)
```

### Essential Requirements

**1. Dual CPU modes (User / Kernel):**

Without this, programs could:
- Interfere with each other
- Take over the CPU forever
- Corrupt OS memory

```
User Mode: Restricted (programs run here)
Kernel Mode: Privileged (OS runs here)

Mode switching protects the system.
```

**2. Memory protection:**

Program A must not access Program B's memory:
```
Hardware MMU enforces:
  Program A can only access addresses 0x10000-0x1FFFF
  Program B can only access addresses 0x20000-0x2FFFF
  Violation → Exception → OS kills offending program
```

**3. Interrupts:**

How does the OS know I/O is complete?

```
Program A requests disk read
        ↓
OS tells disk controller: "Read block 500"
        ↓
OS switches CPU to Program B
        ↓
(Time passes... disk is reading...)
        ↓
Disk completes → Sends INTERRUPT to CPU
        ↓
CPU stops Program B, jumps to OS interrupt handler
        ↓
OS: "Disk I/O for Program A is done"
        ↓
OS marks Program A as Ready
```

**4. Context switching:**

Saving/restoring program state:
```
Program A is running
        ↓
Save A's state:
  • Program Counter: 0x1234
  • Registers: [values]
  • Stack pointer: 0x8FFF
        ↓
Load B's state:
  • Program Counter: 0x5678
  • Registers: [values]
  • Stack pointer: 0x9FFF
        ↓
Program B resumes exactly where it left off
```

### What It Enables

✅ **CPU-I/O overlap** — CPU runs while disk/network work  
✅ **Better hardware utilization** — 60-90% CPU usage vs. 10-20%  
✅ **Higher throughput** — More jobs complete per hour

### What It Does NOT Guarantee

❌ **User responsiveness** — No time limit on CPU use  
❌ **Fairness** — One program could hog CPU  
❌ **Interactivity** — Still batch-oriented

**Example problem:**

```
Program A: CPU-intensive (runs for 1 hour straight)
Program B: Quick calculation (needs 2 seconds)

Multiprogramming OS: B waits 1 hour for A to finish!
User frustration: High
```

This limitation led to the next evolution...

---

## 3. Time-Sharing (Interactive) Operating Systems

### The Problem: Users Want Responsiveness

**1970s:** Computers are getting cheaper. Multiple users want to interact with the same computer **simultaneously**.

**User expectations:**
- Type a command → see result immediately
- Edit a document → see keystrokes appear instantly
- Run multiple programs and switch between them

**Multiprogramming problem:**
```
User 1 runs a computation (takes 5 minutes)
User 2 types a command
        ↓
User 2 waits 5 minutes for a response!
        ↓
Unacceptable user experience
```

### The Solution: Time-Sharing with Preemption

**Core idea:** 

> Give each program a small "time slice" (e.g., 10-100 milliseconds). After the slice expires, forcibly switch to the next program.

**Time-sharing timeline:**
```
Time →   0ms   10ms  20ms  30ms  40ms  50ms

User 1:  [run] [───] [run] [───] [run] [───]
User 2:  [───] [run] [───] [run] [───] [run]

Each user gets CPU every 20ms
Feels simultaneous!
```

### Key Mechanism: Preemptive Scheduling

**Without preemption (multiprogramming):**
```
Program runs until it:
  • Blocks for I/O, OR
  • Terminates
```

**With preemption (time-sharing):**
```
Timer interrupt every 10ms
        ↓
OS: "Your time is up!"
        ↓
Save current program state
Load next program state
        ↓
Next program runs (even if previous wasn't blocked)
```

**Hardware timer:**
```
OS sets timer: "Interrupt me in 10ms"
        ↓
Program A runs
        ↓
(10ms passes)
        ↓
TIMER INTERRUPT!
        ↓
CPU jumps to OS
        ↓
OS performs context switch
```

### Time-Sharing Characteristics

**1. Multiple interactive users:**
```
Terminal 1 (Alice): $ ls
Terminal 2 (Bob):   $ vim file.txt
Terminal 3 (Carol): $ gcc program.c

All feel responsive simultaneously
```

**2. Rapid context switching:**
```
Scheduler queue:
[Alice's bash] → [Bob's vim] → [Carol's gcc] → [Alice's bash] → ...

Cycles through every 10-30ms
```

**3. Virtual memory:**

Each user's programs isolated:
```
Alice sees:  [her programs at 0x0000-0xFFFF]
Bob sees:    [his programs at 0x0000-0xFFFF]
Carol sees:  [her programs at 0x0000-0xFFFF]

All mapped to different physical memory
```

**4. File permissions:**
```bash
$ ls -l
-rw------- alice  thesis.txt      # Only Alice can access
-rw-r--r-- bob    shared.txt      # Bob wrote it, others can read
drwx------ carol  private_dir/    # Only Carol can enter
```

### Examples

**Unix (1970s):**
- Born as a time-sharing system
- Designed for multiple users on one minicomputer
- Terminals connected via serial lines

**Modern descendants:**
- **Linux** — Multi-user server OS
- **macOS** — Based on Unix (BSD)
- **Windows** — Multi-user desktop OS

**Typical use case:**
```
University computer (1980s):
  One VAX minicomputer
  20 terminals connected
  Students all using it simultaneously
  Each feels like they have their own machine
```

### What Changed from Multiprogramming

| Aspect | Multiprogramming | Time-Sharing |
|--------|------------------|--------------|
| **Goal** | CPU utilization | User responsiveness |
| **Switching** | On I/O block only | Fixed time slices |
| **Preemption** | No | Yes |
| **Users** | Batch jobs | Interactive users |
| **Response time** | Not a priority | Critical priority |

---

## 4. Real-Time Operating Systems (RTOS)

### The Problem: When "Fast" Isn't Enough—You Need "Predictable"

**Scenario 1: Car Anti-lock Braking System (ABS)**

```
Sensor detects: "Wheel is skidding!"
        ↓
ABS must respond within 5 milliseconds
        ↓
Response after 5ms? Too late—crash!
```

**Scenario 2: Medical Pacemaker**

```
Heart rate too slow
        ↓
Pacemaker must deliver electrical pulse within 50ms
        ↓
Late response = potential death
```

**Normal OS problem:**
```
Time-sharing OS:
  Average response: 10ms
  Maximum response: Could be 500ms+ (if unlucky)
                          ↑
                    UNACCEPTABLE
```

### The Solution: Guaranteed Timing

**Real-time OS definition:**

> Correctness = Right answer + On time

Getting the right answer **late** is considered **wrong**.

### Hard Real-Time vs. Soft Real-Time

#### Hard Real-Time

**Definition:** Missing a deadline is a **system failure**.

**Characteristics:**
- Deterministic guarantees
- Worst-case execution time (WCET) analyzed
- No dynamic memory allocation (unpredictable timing)
- Minimal OS complexity

**Examples:**

**Aircraft flight control:**
```
Autopilot loop:
  Every 20ms:
    1. Read sensors
    2. Calculate adjustments
    3. Command actuators
  
If this takes 21ms → Unstable flight → Crash
```

**Industrial robotics:**
```
Robot arm movement:
  Position update every 1ms
  
Miss a deadline → Collision, damage, injury
```

**Automotive systems:**
```
Engine control unit:
  Fuel injection timing: ±0.1ms precision
  Airbag deployment: <10ms from impact detection
```

**Nuclear reactor control:**
```
Emergency shutdown:
  Detect critical condition → Insert control rods within 100ms
  Late response → Meltdown
```

#### Soft Real-Time

**Definition:** Occasional deadline misses are **tolerable** but degrade quality.

**Characteristics:**
- Best-effort timing
- Graceful degradation
- Higher average performance matters more than worst-case

**Examples:**

**Video streaming:**
```
Target: Decode frame every 16.67ms (60 FPS)

Miss a frame occasionally → Stutter (annoying but not fatal)
Miss too many → Poor quality
```

**Online gaming:**
```
Network update: Every 50ms
Occasional miss: Slight lag
Frequent misses: Unplayable
```

**Voice calls:**
```
Audio packet every 20ms
Occasional drop: Brief audio glitch
Consistent drops: Call quality degrades
```

**Music production software:**
```
Audio buffer: 5-10ms latency
Occasional glitch: Noticeable but survivable
Frequent glitches: Unusable
```

### RTOS Core Properties

**1. Deterministic scheduling:**

```
Priority-based preemptive scheduling:

Task priorities:
  Emergency stop: Priority 10 (highest)
  Sensor read:    Priority 5
  Logging:        Priority 1 (lowest)

High-priority task ALWAYS preempts lower priority
```

**Real-time scheduler example:**
```
Task A (Priority 10): Runs every 10ms, takes 2ms
Task B (Priority 5):  Runs every 50ms, takes 5ms
Task C (Priority 1):  Runs every 100ms, takes 20ms

Timeline:
0ms:   A starts
2ms:   A completes
2ms:   B starts
7ms:   B completes
7ms:   C starts
10ms:  A preempts C (higher priority)
12ms:  A completes, C resumes
...
```

**2. Bounded interrupt latency:**

```
Event occurs → CPU responds within X microseconds (guaranteed)

Typical RTOS: <10 microseconds
General OS:   Could be milliseconds
```

**3. Minimal kernel:**

```
RTOS kernel size: 10-50 KB
General OS kernel: 10-50 MB

Why?
  Smaller code → Less to go wrong
  Faster → More predictable
  Easier to verify → Safety certification
```

**4. No unpredictable operations:**

```
Forbidden in hard RTOS:
  ✗ Virtual memory page faults (unbounded delay)
  ✗ Dynamic memory allocation (fragmentation, unpredictable time)
  ✗ Disk I/O (highly variable latency)
  ✗ Garbage collection (pause times)

Instead:
  ✓ Static memory allocation
  ✓ Memory pools
  ✓ Bounded buffers
```

### RTOS Examples

**Hard RTOS:**
- **VxWorks** — Aerospace, defense (Mars rovers!)
- **QNX** — Automotive, medical devices
- **FreeRTOS** — Embedded systems, IoT
- **seL4** — Formally verified, highest assurance

**Soft RTOS:**
- **RT-Linux** — Linux with real-time patches
- **Windows Embedded** — Industrial automation
- **Zephyr** — IoT and embedded

### Real-Time vs. Time-Sharing Comparison

| Aspect | Time-Sharing | Real-Time |
|--------|--------------|-----------|
| **Goal** | Average responsiveness | Guaranteed deadlines |
| **Priority** | Fair sharing | Strict priority |
| **Worst case** | Not guaranteed | Must be bounded |
| **Complexity** | High (features) | Low (predictability) |
| **Use case** | Interactive users | Safety-critical control |

**Misconception:** "Real-time = fast"

**Truth:** "Real-time = predictable"

```
Time-sharing: Average 10ms, max 1000ms
Real-time:    Average 50ms, max 50ms

RTOS is "slower" on average but PREDICTABLE
```

---

## 5. Distributed Operating Systems

### The Problem: One Computer Isn't Enough

**Scenario:** Scientific computation needs 1000 CPUs working together.

**Challenge:** How do you make 1000 separate computers behave as **one unified system**?

### The Vision: Single System Image

**What users see:**
```
One coherent system
  • One file system (files can be anywhere)
  • One process space (processes can run anywhere)
  • Transparent resource access (don't care where it is)
```

**Example interaction:**
```
User: $ run simulation
System: [distributes across 100 nodes automatically]
User: $ read results.txt
System: [fetches from wherever it's stored]

User doesn't know or care about distribution
```

### Key Properties

**1. Location transparency:**

```
User opens file: /home/alice/data.txt

Distributed OS:
  • Figures out which machine has it
  • Fetches it transparently
  • User sees just a file, not "file on Machine 47"
```

**2. Migration transparency:**

```
Process starts on Machine A
Load balancer: "Machine A overloaded, move to Machine B"
        ↓
Process migrated, continues running
User: [doesn't notice]
```

**3. Replication transparency:**

```
File F replicated on Machines [5, 12, 28]
Read F: System picks closest copy
Write F: System updates all copies
User: Sees one file, not three
```

**4. Distributed resource management:**

```
Global scheduler:
  • Tracks load on all machines
  • Assigns tasks to least-loaded nodes
  • Balances work automatically
```

### Core Challenges

**1. Synchronization:**

```
Two machines try to update the same file:
  Machine A: Write "X"
  Machine B: Write "Y"
  
Which wins? How do we prevent corruption?
  
Solution: Distributed locks, consensus protocols
```

**2. Clock coordination:**

```
Machine A's clock: 10:00:05
Machine B's clock: 10:00:03

Which event happened first?
  
Solution: Logical clocks (Lamport timestamps), NTP
```

**3. Partial failures:**

```
Normal OS: Crash → entire system down
Distributed OS: One node crashes → rest should keep working

Challenge: Detect failures, recover, reassign work
```

**4. Network communication overhead:**

```
Local function call: ~1 nanosecond
Network RPC (Remote Procedure Call): ~1 millisecond

1,000,000x slower!

Challenge: Design to minimize network operations
```

### Why True Distributed OSs Are Rare

**Complexity:**
```
Number of failure modes:
  Single machine: ~10
  100-machine cluster: ~10,000+

Testing, debugging, reasoning → exponentially harder
```

**Performance:**
```
Coordination overhead often outweighs benefits
Better: Loosely-coupled systems with explicit distribution
```

**Alternative approach: Distributed systems, not distributed OSs**

Modern preference:
```
Each machine runs standard OS (Linux)
  +
Application-level coordination (Kubernetes, Hadoop)
```

### Historical Examples

**Plan 9 from Bell Labs:**
```
Everything is a file (including remote resources)
9P protocol for transparent remote access
Never widely adopted (too radical)
```

**Amoeba:**
```
Research OS by Andrew Tanenbaum
Object-based, capability security
Pool of processors as shared resource
```

**Sprite:**
```
Transparent process migration
Global file system
Research prototype
```

---

## 6. Network Operating Systems

### The Key Difference: No Illusion of Unity

**Distributed OS:** "You have **one** computer (that happens to be 100 machines)"  
**Network OS:** "You have **100** computers connected by a network"

### What It Provides

**Explicit networked services:**

```
File Server:
  \\fileserver\shared\documents

Each computer knows:
  • This is a REMOTE resource
  • Accessed over the network
  • Different from local files
```

**Authentication services:**
```
Central user database
  • Login from any machine
  • Same credentials everywhere
  • Centralized access control
```

**Distributed applications:**
```
Each machine runs its own OS
Applications explicitly use network
```

### Network OS Architecture

```
┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│  Workstation 1      │  │  Workstation 2      │  │  File Server        │
│  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │
│  │   Local OS    │  │  │  │   Local OS    │  │  │  │   Server OS   │  │
│  │   (Windows)   │  │  │  │   (Windows)   │  │  │  │   (Linux)     │  │
│  └───────┬───────┘  │  │  └───────┬───────┘  │  │  └───────┬───────┘  │
│          │          │  │          │          │  │          │          │
│  ┌───────▼───────┐  │  │  ┌───────▼───────┐  │  │  ┌───────▼───────┐  │
│  │Network Client │  │  │  │Network Client │  │  │  │Network Server │  │
│  │  (SMB/CIFS)   │  │  │  │  (SMB/CIFS)   │  │  │  │    (Samba)    │  │
│  └───────┬───────┘  │  │  └───────┬───────┘  │  │  └───────┬───────┘  │
└──────────┼──────────┘  └──────────┼──────────┘  └──────────┼──────────┘
           │                        │                        │
           └────────────────────────┴────────────────────────┘
                          Network (Ethernet/WiFi)
```

### Examples

**Traditional:**
- **Novell NetWare** (1980s-1990s file servers)
- **Windows Server with Active Directory**
- **NFS (Network File System) in Unix**

**Modern:**
- Cloud storage (Dropbox, Google Drive)
- Enterprise file sharing (SharePoint)
- Cluster computing (Beowulf)

### Key Distinction Table

| Aspect | Distributed OS | Network OS |
|--------|----------------|------------|
| **System image** | Single | Multiple separate systems |
| **Transparency** | Location transparent | Explicitly remote |
| **User view** | One computer | Multiple computers |
| **Autonomy** | Central control | Each node independent |
| **Failure** | System-wide impact | Local impact |

---

## 7. Mobile Operating Systems

### The Problem: Fundamentally Different Constraints

**Traditional desktop:**
```
Power: Unlimited (wall outlet)
Lifespan: Replace every 3-5 years
Interaction: Keyboard + mouse
Security: Trusted user
```

**Mobile device:**
```
Power: Battery (must last all day)
Lifespan: Replace every 2-3 years (constrained)
Interaction: Touch, sensors (accelerometer, GPS)
Security: Untrusted apps, valuable data
```

### Design Priorities

**1. Power efficiency:**

```
Every wasted CPU cycle = shorter battery life

Mobile OS strategies:
  • Aggressive app suspension
  • CPU throttling
  • Screen dimming
  • Network batching
```

**Example: Background app limits**
```
App moves to background:
  After 30 seconds:
    • Network cut off
    • CPU time heavily restricted
    • Only critical services allowed
  
Saves power dramatically
```

**2. Application sandboxing:**

```
Each app runs in isolated container:
  • Cannot access other apps' data
  • Cannot access system files
  • Explicit permission for sensitive resources
```

**Permission model (Android):**
```java
// App cannot access location until user grants permission
if (checkSelfPermission(ACCESS_FINE_LOCATION) != GRANTED) {
    requestPermissions([ACCESS_FINE_LOCATION]);
}
```

**3. Touch-optimized UI:**

```
Desktop: Precise mouse cursor, small targets
Mobile:  Finger touch, large targets, gestures

UI design:
  • Minimum button size: 44×44 pixels
  • Gesture recognition (swipe, pinch, rotate)
  • Virtual keyboard
```

**4. Sensor integration:**

```
Available sensors:
  • Accelerometer (orientation, shake detection)
  • Gyroscope (rotation)
  • GPS (location)
  • Camera
  • Microphone
  • Proximity sensor
  • Ambient light sensor

OS provides APIs to access all of these
```

**5. Aggressive resource management:**

```
Low memory:
  • Kill background apps
  • Compress memory
  • Warn user

Low battery:
  • Reduce background activity
  • Dim screen
  • Disable non-essential features
```

### Mobile OS Architecture

**Android:**
```
┌──────────────────────────────────┐
│  Applications (Java/Kotlin)      │
├──────────────────────────────────┤
│  Android Framework (APIs)        │
├──────────────────────────────────┤
│  Android Runtime (ART)           │
│  Native Libraries (C/C++)        │
├──────────────────────────────────┤
│  Hardware Abstraction Layer      │
├──────────────────────────────────┤
│  Linux Kernel (Modified)         │
│  • Power management              │
│  • Binder IPC                    │
│  • Low Memory Killer             │
└──────────────────────────────────┘
```

**iOS:**
```
┌──────────────────────────────────┐
│  Applications (Swift/Obj-C)      │
├──────────────────────────────────┤
│  Cocoa Touch Framework           │
├──────────────────────────────────┤
│  Core Services                   │
│  Core OS                         │
├──────────────────────────────────┤
│  Darwin Kernel (XNU)             │
│  • Mach microkernel              │
│  • BSD layer                     │
└──────────────────────────────────┘
```

### Key Features

**App lifecycle management:**
```
State transitions:
  Not running
       ↓
  Inactive (launching)
       ↓
  Active (foreground)
       ↓
  Background (limited time)
       ↓
  Suspended (frozen, no CPU)
       ↓
  Terminated (killed if memory needed)
```

**Push notifications:**
```
Instead of: App polls server every minute (battery drain)
Use: Server pushes notification when needed (efficient)
```

**Fast app switching:**
```
User switches apps:
  • Save app state
  • Freeze process
  • Switch UI instantly
  • Resume when user returns
```

---

## 8. Embedded Operating Systems

### The Problem: Dedicated Purpose, Extreme Constraints

**What is embedded?**

A computer system designed for a **specific function** within a larger system.

**Examples:**
- WiFi router
- Smart thermostat
- Washing machine controller
- Car engine management
- Industrial sensor

### Key Characteristics

**1. Resource constraints:**

```
Typical embedded system:
  RAM:  32 KB - 512 KB (not GB!)
  CPU:  20 MHz - 200 MHz (not GHz!)
  Storage: 128 KB - 4 MB

Desktop/Phone:
  RAM: 8-16 GB
  CPU: 2-4 GHz
  Storage: 256 GB - 1 TB
```

**2. Real-time requirements:**

```
Many embedded systems are also real-time:
  • Car ABS: Hard real-time
  • Router packet handling: Soft real-time
  • Thermostat: Periodic tasks
```

**3. No user interface (often):**

```
Industrial sensor:
  • No screen
  • No keyboard
  • Maybe status LEDs
  • Configured once, runs forever
```

**4. Long uptimes:**

```
Expected runtime: Years without reboot

Requirements:
  • Extreme reliability
  • Watchdog timers (auto-reset if hung)
  • Error recovery
```

**5. Power constraints:**

```
Battery-powered sensor:
  • Must run for 5 years on one battery
  • Sleep most of the time
  • Wake periodically to sample and transmit
```

### Embedded OS Examples

**FreeRTOS:**
```
• Kernel: ~10 KB
• Real-time scheduler
• Minimal overhead
• Used in billions of devices
```

**Embedded Linux:**
```
• Full Linux kernel, stripped down
• BusyBox userland (minimal tools)
• ~8-16 MB total size
• Good for more powerful embedded systems
```

**Zephyr:**
```
• Modern, modular
• Strong IoT support
• Multiple architecture support
• Security-focused
```

**ThreadX:**
```
• Certified for safety-critical systems
• Avionics, medical devices
• Very small footprint
```

### Use Case Example: Smart Thermostat

```
Hardware:
  • ARM Cortex-M microcontroller
  • 64 KB RAM
  • 256 KB Flash storage
  • Temperature sensor
  • WiFi module
  • Small LCD screen

OS: FreeRTOS

Tasks:
  1. Temperature monitoring (every 1 second)
  2. Display update (every 500ms)
  3. Network communication (as needed)
  4. Button handling (interrupt-driven)

Schedule:
  Priority 1: Temperature control (safety-critical)
  Priority 2: User interface
  Priority 3: Network updates
```

---

## Comparison Table: All OS Types

| OS Type | Primary Goal | Key Trade-off | Typical Use Case | Example |
|---------|-------------|---------------|------------------|---------|
| **Batch** | Throughput | No interactivity | Historical mainframes | IBM OS/360 |
| **Multiprogramming** | CPU utilization | No guarantees on response | Efficient resource use | Early Unix |
| **Time-sharing** | Responsiveness | Overhead of switching | Interactive users | Modern Linux/Windows |
| **Hard Real-time** | Predictability | Limited features | Safety-critical systems | VxWorks |
| **Soft Real-time** | Timely response | Best-effort timing | Multimedia, gaming | Modified Linux |
| **Distributed** | Unified system | Extreme complexity | Research, HPC | Plan 9, Amoeba |
| **Network** | Resource sharing | Explicit distribution | Enterprise networks | Windows Server |
| **Mobile** | Battery life | Limited multitasking | Smartphones, tablets | Android, iOS |
| **Embedded** | Reliability + size | Limited resources | IoT, industrial | FreeRTOS, Zephyr |

---

## Mental Models for Quick Recall

**One-word summaries:**

| OS Type | Core Concept |
|---------|--------------|
| Batch | **Throughput** |
| Multiprogramming | **Utilization** |
| Time-sharing | **Responsiveness** |
| Real-time | **Predictability** |
| Distributed | **Transparency** |
| Network | **Services** |
| Mobile | **Power + Isolation** |
| Embedded | **Constraints** |

---

## The Evolution Timeline

```
1950s: Batch
         ↓
1960s: Multiprogramming (keep CPU busy)
         ↓
1970s: Time-sharing (user interaction)
         ↓
1980s: Network OS (resource sharing)
         ↓
1990s: Distributed OS (research), RTOS (industrial)
         ↓
2000s: Mobile OS (smartphones emerge)
         ↓
2010s: Embedded + IoT (billions of devices)
         ↓
2020s: Hybrid systems (mobile = time-sharing + real-time + embedded concepts)
```

---

## Modern Reality: Hybrid Designs

**Important insight:** Modern systems combine multiple types!

**Smartphone (iOS/Android):**
```
• Time-sharing (run multiple apps)
• Mobile (power management, sandboxing)
• Soft real-time (audio/video playback)
• Elements of embedded (specialized chips)
```

**Linux server:**
```
• Time-sharing (multi-user, multitasking)
• Network OS (file sharing, authentication)
• Can be real-time (with RT patches)
• Can be distributed (cluster configurations)
```

**Modern car:**
```
• Hard RTOS (braking, steering)
• Embedded (sensors, actuators)
• Soft real-time (entertainment system)
• Network OS (car-to-car communication)
```

---

## Summary

Operating systems aren't monolithic categories—they're **design responses to specific constraints**:

- **Need throughput?** → Batch processing
- **Need CPU busy?** → Multiprogramming
- **Need user interaction?** → Time-sharing
- **Need timing guarantees?** → Real-time
- **Need unified cluster?** → Distributed
- **Need shared resources?** → Network
- **Need battery efficiency?** → Mobile
- **Need minimal footprint?** → Embedded

Modern systems often blend these approaches, choosing the right techniques for each subsystem. Understanding the types helps you recognize **why** an OS makes particular design decisions.