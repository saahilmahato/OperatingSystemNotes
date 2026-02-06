# Parts of an Operating System

## The Big Picture: How an OS is Built

An operating system isn't a single monolithic blob of code. It's a carefully structured **stack of layers**, each building on the one below, from the first instruction that runs when you press the power button all the way up to the apps you interact with.

Think of it like a skyscraper:
- **Foundation (Firmware):** First thing that activates, checks if the building is safe
- **Core structure (Kernel):** The steel framework that holds everything together
- **Utilities (System services):** Plumbing, electricity, HVAC—essential but invisible
- **Interior (User interface):** The offices and apartments people actually use
- **Tenants (Applications):** The businesses and residents who occupy the space

Let's explore each layer, starting from the moment you press the power button.

---

## Layer 0: Firmware and Boot Process

**What happens in the first milliseconds after you turn on your computer?**

The CPU doesn't know about operating systems yet. It needs something to tell it where to even start. That's the job of **firmware**—the first software that runs.

### BIOS / UEFI: The Awakening

**What it is:**
- Small program stored on a chip on your motherboard
- Executes **immediately** when power is applied
- Not part of the OS, but necessary to start it

**What it does:**

```
Power button pressed
        ↓
CPU starts executing firmware (BIOS/UEFI)
        ↓
1. Initialize CPU registers and memory controller
2. Perform POST (Power-On Self-Test)
   - Check RAM
   - Detect CPU
   - Initialize graphics card
   - Test keyboard
3. Scan for bootable devices (hard drive, USB, network)
4. Load bootloader from boot device
5. Transfer control to bootloader
```

**POST (Power-On Self-Test):**

You've probably seen this—the brief messages or logo screen before your OS loads:
```
Memory OK: 16 GB
CPU: Intel Core i7
Keyboard detected
Booting from: SSD (SATA 0)
```

**BIOS vs. UEFI:**

| Feature | BIOS (Legacy) | UEFI (Modern) |
|---------|---------------|---------------|
| **Introduced** | 1981 | ~2005 |
| **Boot time** | Slower | Faster |
| **Disk support** | Up to 2 TB (MBR) | Unlimited (GPT) |
| **Interface** | Text-based | Graphical (mouse support) |
| **Security** | None | Secure Boot |
| **Architecture** | 16-bit | 32/64-bit |

**Secure Boot (UEFI feature):**

Prevents malware from replacing your bootloader:
```
UEFI checks: "Is this bootloader signed by a trusted authority?"
    ✓ Yes → Continue booting
    ✗ No → Refuse to boot (prevents rootkits)
```

---

### Bootloader: The OS Launcher

**What it is:**
- Small program stored on your boot drive
- Lives in a special boot partition
- Knows how to load the kernel into memory

**What it does:**

```
Bootloader starts
        ↓
1. Display boot menu (if multiple OSs installed)
2. Load kernel image from disk into RAM
3. Load initial ramdisk (initramfs/initrd)
4. Pass control to kernel
        ↓
Kernel begins execution
```

**Example boot menu (GRUB):**

```
┌─────────────────────────────────────┐
│  GRUB Bootloader                    │
├─────────────────────────────────────┤
│  > Ubuntu Linux 6.5.0-27            │
│    Ubuntu Linux (recovery mode)     │
│    Windows 11                       │
│    System Setup                     │
└─────────────────────────────────────┘
```

**Common bootloaders:**

- **GRUB (Grand Unified Bootloader):** Most Linux distributions
- **systemd-boot:** Simpler, faster, used by some modern distros
- **Windows Boot Manager:** Windows systems
- **Apple bootloader:** macOS

**Why separate from the kernel?**

The bootloader is small and simple—it just needs to load the kernel. The kernel is complex and large. Separating them means:
- Bootloader can offer multi-boot options
- Kernel can be updated independently
- Recovery is easier if kernel breaks

---

## Layer 1: The Kernel (The Core)

This is the **heart of the operating system**. Everything else is either setting up the kernel (firmware/bootloader) or using the kernel (applications).

**Key characteristics:**
- Runs in **kernel mode** (privileged, unrestricted hardware access)
- Always resident in memory
- Only one kernel per running OS instance
- Manages all hardware resources
- Provides abstractions to user space

**What does "kernel mode" mean?**

Modern CPUs support multiple privilege levels. The kernel runs with **maximum privileges**:

```
┌─────────────────────────────────────────┐
│         User Mode (Ring 3)              │  ← Applications run here
│  • Restricted                           │     Cannot access hardware
│  • Cannot execute privileged ops        │     directly
│  • Cannot access kernel memory          │
├═════════════════════════════════════════┤
│        Kernel Mode (Ring 0)             │  ← OS kernel runs here
│  • Full hardware access                 │     Complete control
│  • Can execute ANY instruction          │
│  • Can access all memory                │
└─────────────────────────────────────────┘
```

Applications must ask the kernel for services via **system calls** (more on this later).

---

### Kernel Architecture: Design Philosophies

Not all kernels are structured the same way. There are three main architectural approaches:

#### 1. Monolithic Kernel (The All-In-One Approach)

**Philosophy:** Put everything in kernel space for maximum performance.

```
┌─────────────────────────────────────────┐
│          Kernel Space                   │
│  ┌────────────────────────────────────┐ │
│  │  Process Scheduler                 │ │
│  │  Memory Manager                    │ │
│  │  File Systems (ext4, NTFS, etc.)   │ │
│  │  Device Drivers (disk, network)    │ │
│  │  Network Stack (TCP/IP)            │ │
│  │  System Call Interface             │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
           ↑
     System Calls
           ↓
┌─────────────────────────────────────────┐
│          User Space                     │
│  (Applications)                         │
└─────────────────────────────────────────┘
```

**Advantages:**
- ✅ **Fast:** Direct function calls between components (no IPC overhead)
- ✅ **Simple communication:** Everything shares kernel memory
- ✅ **Efficient:** No context switching for kernel services

**Disadvantages:**
- ❌ **Large:** Entire kernel must be loaded
- ❌ **Less stable:** One buggy driver can crash the whole system
- ❌ **Security:** Larger attack surface

**Examples:** Linux, traditional Unix, FreeBSD

**Real-world analogy:**

Imagine a restaurant where the chef, waiter, dishwasher, and manager all work in one big open kitchen. Communication is instant, but if the dishwasher starts a fire, everyone's in danger.

---

#### 2. Microkernel (The Minimal Approach)

**Philosophy:** Keep the kernel tiny. Move everything else to user space.

```
┌─────────────────────────────────────────┐
│          User Space                     │
│  ┌────────────────────────────────────┐ │
│  │  File System Server                │ │
│  │  Device Driver Servers             │ │
│  │  Network Stack Server              │ │
│  │  Applications                      │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
           ↑
     IPC (Message Passing)
           ↓
┌─────────────────────────────────────────┐
│          Kernel Space (Minimal)         │
│  ┌────────────────────────────────────┐ │
│  │  CPU Scheduling                    │ │
│  │  IPC (Inter-Process Communication) │ │
│  │  Basic Memory Management           │ │
│  │  Low-level Hardware Access         │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

**Advantages:**
- ✅ **Stable:** Driver crash doesn't crash kernel
- ✅ **Secure:** Smaller trusted code base
- ✅ **Modular:** Easy to update components
- ✅ **Portable:** Less hardware-specific code in kernel

**Disadvantages:**
- ❌ **Slower:** IPC overhead for every operation
- ❌ **Complex:** More message passing logic
- ❌ **Debugging:** Harder to trace issues across processes

**Examples:** MINIX, QNX, seL4 (high-assurance systems)

**Real-world analogy:**

Restaurant with separate kitchen, dining room, and dishwashing area with walls between them. Fire in the kitchen? Close the door and evacuate—dining room is still safe. But now waiters need to walk through doors to communicate (slower).

---

#### 3. Hybrid Kernel (The Practical Compromise)

**Philosophy:** Mostly monolithic for performance, but with modular design for flexibility.

```
┌─────────────────────────────────────────┐
│          Kernel Space                   │
│  ┌────────────────────────────────────┐ │
│  │  Core Services (monolithic)        │ │
│  │  • Scheduler                       │ │
│  │  • Memory Manager                  │ │
│  │  • Core IPC                        │ │
│  ├────────────────────────────────────┤ │
│  │  Modular Components                │ │
│  │  • Device Drivers (can be modules) │ │
│  │  • File Systems (loadable)         │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

**Advantages:**
- ✅ Performance of monolithic kernel
- ✅ Some modularity and stability benefits
- ✅ Can load/unload drivers dynamically

**Examples:** Windows NT, macOS (XNU kernel), some modern BSDs

**Windows NT architecture specifics:**

```
User Mode:
  • Win32 Subsystem
  • POSIX Subsystem
  • Applications

Kernel Mode:
  • Executive Services (memory, process, I/O)
  • Kernel (scheduling, synchronization)
  • Hardware Abstraction Layer (HAL)
  • Device Drivers
```

**macOS XNU (X is Not Unix):**

Combines:
- **Mach microkernel:** IPC, scheduling, virtual memory
- **BSD:** File systems, networking, POSIX compatibility
- **IOKit:** Object-oriented driver framework

---

### Core Kernel Responsibilities

Now let's look at what the kernel actually *does*, regardless of its architecture.

#### 1. Process and Thread Management

**What it manages:** Running programs

**Key concepts:**

- **Process:** Independent program with its own address space
- **Thread:** Lightweight execution unit within a process (shares address space)

**Kernel responsibilities:**

**Process lifecycle:**
```c
// User calls fork()
pid_t pid = fork();

// Kernel does:
1. Allocate new process ID
2. Copy parent's address space (or use copy-on-write)
3. Copy file descriptors, environment variables
4. Add to scheduler queue
5. Return to user space

if (pid == 0) {
    // Child process
    exec("/bin/ls");  // Kernel replaces process image
} else {
    // Parent process
    wait(NULL);  // Kernel blocks parent until child exits
}
```

**CPU Scheduling:**

Multiple processes want the CPU. The kernel decides who runs when.

```
Scheduler algorithms:
• First-Come-First-Served (FCFS)
• Shortest Job First (SJF)
• Round Robin (give each process a time slice)
• Priority Scheduling
• Multi-level Feedback Queue (modern systems)

Example with 3 processes:

Time:  0ms   10ms  20ms  30ms  40ms  50ms
       [P1]  [P2]  [P3]  [P1]  [P2]  [P3]
        ↑
   Each gets 10ms time slice
```

**Context Switching:**

When the kernel switches from Process A to Process B:

```
1. Save Process A's state:
   - Program Counter (PC)
   - CPU registers
   - Stack pointer
   - Status flags

2. Update scheduling data

3. Switch to Process B:
   - Load B's saved state into CPU
   - Restore B's registers
   - Jump to B's saved PC

Process B resumes exactly where it left off.
```

**Thread management:**

```c
// Create thread
pthread_create(&thread_id, NULL, function, argument);

// Kernel does:
1. Allocate thread ID
2. Create new stack (in same address space as parent)
3. Set up execution context (PC, registers)
4. Add to scheduler
5. Return

// Threads share:
• Address space (memory)
• File descriptors
• Signal handlers

// Threads have separate:
• Stack
• Registers
• Thread-local storage
```

---

#### 2. Memory Management

**What it manages:** RAM allocation and protection

**Key responsibility:** Give each process the illusion of having its own private memory.

**Virtual memory:**

```
Process A sees:                    Process B sees:
┌──────────────┐                  ┌──────────────┐
│ 0xFFFFFFFF   │                  │ 0xFFFFFFFF   │
│   Stack      │                  │   Stack      │
├──────────────┤                  ├──────────────┤
│   Heap       │                  │   Heap       │
├──────────────┤                  ├──────────────┤
│   .data      │                  │   .data      │
├──────────────┤                  ├──────────────┤
│   .text      │                  │   .text      │
│ 0x00000000   │                  │ 0x00000000   │
└──────────────┘                  └──────────────┘
       ↓                                  ↓
       └─────── Both map to ──────────────┘
                      ↓
            Physical RAM (8 GB)
       ┌──────────────────────────┐
       │  [Different locations]    │
       │  Process A: 0x20000000   │
       │  Process B: 0x40000000   │
       └──────────────────────────┘
```

**Paging:**

Memory is divided into fixed-size **pages** (typically 4 KB).

```
Virtual Page → Page Table → Physical Frame

Process requests: 0x00401000 (virtual address)
                     ↓
         Page Table Lookup:
         Virtual Page 0x401 → Physical Frame 0x89A
                     ↓
         Physical Address: 0x0089A000
```

**Page faults:**

What happens when you access memory that's not in RAM?

```
1. Process accesses virtual address 0x00500000
2. MMU (Memory Management Unit) checks page table
3. Page not in RAM → Page fault exception
4. CPU switches to kernel mode
5. Kernel checks:
   - Is this valid memory for this process?
     • Yes → Load page from disk (swap)
     • No → Segmentation fault (kill process)
6. Update page table
7. Resume process (retry the memory access)
```

**Memory protection:**

```
Page Table Entry contains:
┌─────────────────────────────────┐
│ Physical Frame Address          │
│ Permissions: R W X              │
│ Present: 1/0 (in RAM or not)    │
│ User/Kernel bit                 │
└─────────────────────────────────┘

Example:
• Text segment: R-X (read, execute, no write)
• Data segment: RW- (read, write, no execute)
• Kernel pages: Accessible only in kernel mode
```

**Swapping:**

When RAM is full, kernel moves inactive pages to disk:

```
RAM full:
1. Choose victim page (LRU algorithm)
2. Write to swap partition/file
3. Mark page as "not present" in page table
4. Free the RAM for new allocation

Later access triggers page fault → load from swap
```

---

#### 3. File System and Storage Management

**What it manages:** Persistent storage

**Key abstraction:** Files and directories (instead of disk sectors)

**Virtual File System (VFS):**

Provides a unified interface for different file systems:

```
User space:
   open("/home/user/file.txt")
        ↓
   System call interface
        ↓
   VFS layer (generic)
        ↓
   Specific file system driver
   • ext4 (Linux)
   • NTFS (Windows)
   • APFS (macOS)
   • FAT32 (USB drives)
        ↓
   Block device layer
        ↓
   Disk driver (SATA, NVMe)
        ↓
   Physical storage
```

**Page cache:**

The kernel caches frequently accessed file data in RAM:

```
read("/home/user/file.txt")
        ↓
Kernel checks page cache:
  Hit? → Return from RAM (fast!)
  Miss? → Read from disk, cache it, return

Subsequent reads? Served from cache.
```

**I/O Scheduling:**

Multiple processes want disk access. Kernel optimizes:

```
Request queue:
• Read sector 100 (Process A)
• Read sector 5000 (Process B)
• Read sector 120 (Process C)

Scheduler reorders to minimize disk head movement:
1. Read 100
2. Read 120  ← Adjacent! Efficient
3. Read 5000
```

**File permissions (Unix-style):**

```bash
$ ls -l file.txt
-rw-r--r-- 1 alice staff 1234 Feb 06 10:00 file.txt
│││││││││
│││││││└└─ Others: read
│││││└└─── Group: read
│││└└───── Owner: read + write
││└─────── Special bits
│└──────── File type (- = regular file)

Kernel enforces:
- Process UID matches owner? → owner permissions
- Process GID in group? → group permissions
- Else → other permissions
```

---

#### 4. Device Drivers

**What they are:** Kernel modules that know how to talk to specific hardware

**Why needed:** Hardware diversity

```
Generic kernel code:
   "Read 512 bytes from storage device"
        ↓
   Device driver translates to:
        ↓
   NVMe SSD: Send NVMe command via PCIe
   SATA HDD: Send ATA command via SATA
   USB drive: Send SCSI command via USB
```

**Linux kernel modules:**

Drivers can be loaded/unloaded without rebooting:

```bash
# List loaded modules
$ lsmod
Module                  Size  Used by
nvidia                1048576  42
e1000e                245760  0

# Load module
$ sudo modprobe usb_storage

# Unload module
$ sudo rmmod usb_storage
```

**Device files (Linux/Unix):**

Hardware appears as files in `/dev`:

```bash
/dev/sda      # First SATA disk
/dev/nvme0n1  # First NVMe disk
/dev/tty1     # First virtual terminal
/dev/random   # Random number generator
/dev/null     # Bit bucket (discards all input)

# Same API for everything:
int fd = open("/dev/sda", O_RDONLY);
read(fd, buffer, 512);
close(fd);
```

---

#### 5. Networking Stack

**What it manages:** Network communication

**OSI/TCP-IP stack in the kernel:**

```
Application (User Space)
   ↓ (system call)
┌──────────────────────────────┐
│  Socket Layer                │ ← Kernel
├──────────────────────────────┤
│  Transport (TCP/UDP)         │
├──────────────────────────────┤
│  Network (IP routing)        │
├──────────────────────────────┤
│  Link (Ethernet, WiFi)       │
├──────────────────────────────┤
│  Network Driver              │
└──────────────────────────────┘
   ↓
Physical network card
```

**Example: Sending data over TCP**

```c
// User space
int sock = socket(AF_INET, SOCK_STREAM, 0);
connect(sock, &server_addr, sizeof(server_addr));
send(sock, "Hello", 5, 0);

// Kernel does:
1. Socket layer: Copy data to kernel buffer
2. TCP layer: Add TCP header (seq#, ports)
3. IP layer: Add IP header (src/dst addresses)
4. Link layer: Add Ethernet header (MAC addresses)
5. Network driver: Send packet via hardware
```

**Packet filtering (firewalls):**

```
Incoming packet:
        ↓
   Netfilter hooks (kernel firewall)
        ↓
   Check rules:
   • Source IP allowed?
   • Destination port open?
   • Protocol permitted?
        ↓
   Accept or Drop
```

---

#### 6. Security and Protection

**What it manages:** System safety and access control

**User authentication:**

```
Login process:
1. User enters username/password
2. Kernel checks /etc/shadow (hashed passwords)
3. Match? → Create session, set UID
4. No match? → Deny access
```

**Process isolation:**

Modern kernels provide multiple isolation mechanisms:

**Namespaces (Linux):**

Isolate what a process can see:

```
• PID namespace: Process can't see other processes
• Network namespace: Separate network stack
• Mount namespace: Different file system view
• User namespace: Separate UID space

Example: Docker containers use namespaces
```

**cgroups (Control Groups):**

Limit resources a process can use:

```bash
# Limit CPU usage to 50%
echo "50000" > /sys/fs/cgroup/cpu/myapp/cpu.cfs_quota_us

# Limit memory to 512 MB
echo "536870912" > /sys/fs/cgroup/memory/myapp/memory.limit_in_bytes
```

**Capabilities:**

Fine-grained privileges instead of all-or-nothing root:

```
Instead of: root can do EVERYTHING
Use: Process has CAP_NET_BIND_SERVICE
     (can bind to port 80 without being root)
```

**Mandatory Access Control (MAC):**

- **SELinux (Security-Enhanced Linux):** Every object labeled, enforced policies
- **AppArmor:** Profile-based confinement

```
SELinux example:
Process: httpd (labeled: httpd_t)
File: /var/www/html/index.html (labeled: httpd_sys_content_t)
Policy: httpd_t can read httpd_sys_content_t ✓
        httpd_t cannot read user_home_t ✗
```

**seccomp (Secure Computing Mode):**

Restricts which system calls a process can make:

```c
// Allow only read, write, exit
scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
seccomp_load(ctx);

// Now trying to call open() → Process killed
```

---

#### 7. Inter-Process Communication (IPC)

**What it provides:** Ways for processes to communicate

**Why needed:** Processes are isolated by default, but sometimes they need to coordinate.

**IPC mechanisms:**

**1. Pipes:**
```c
int pipe_fd[2];
pipe(pipe_fd);  // Create pipe

if (fork() == 0) {
    // Child: Write
    write(pipe_fd[1], "Hello", 5);
} else {
    // Parent: Read
    char buf[10];
    read(pipe_fd[0], buf, 5);
}
```

**2. Shared Memory:**
```c
int shm_id = shmget(KEY, SIZE, IPC_CREAT);
char *shared = shmat(shm_id, NULL, 0);
strcpy(shared, "Shared data");
// Other process can attach and read
```

**3. Signals:**
```c
kill(pid, SIGTERM);  // Send terminate signal
// Target process handles it
signal(SIGTERM, handler_function);
```

**4. Message Queues:**
```c
int mq = msgget(KEY, IPC_CREAT);
msgsnd(mq, &message, sizeof(message), 0);
msgrcv(mq, &message, sizeof(message), 0, 0);
```

**5. Sockets:**
```c
// Most flexible—works across network too
int sock = socket(AF_UNIX, SOCK_STREAM, 0);
bind(sock, &addr, sizeof(addr));
// Processes communicate via socket
```

---

#### 8. Power and Thermal Management

**What it manages:** Energy efficiency and heat

**CPU frequency scaling (CPUfreq):**

```
Governors (policies):
• Performance: Always max frequency
• Powersave: Always min frequency
• Ondemand: Scale based on load
• Conservative: Gradual scaling

Example:
Low load: CPU runs at 800 MHz
High load: CPU ramps to 3.5 GHz
```

**Sleep states (ACPI):**

```
S0: Running (full power)
S1: CPU stopped, RAM powered
S3: Suspend-to-RAM (low power)
S4: Suspend-to-disk (hibernate)
S5: Shutdown
```

**Thermal throttling:**

```
Temperature > threshold:
1. Kernel reduces CPU frequency
2. Activate additional cooling
3. If still too hot: Emergency shutdown
```

---

## Layer 2: System Call Interface

**What it is:** The **gateway** between user space and kernel space

**Why it exists:** Security and stability

Applications cannot directly call kernel functions. They must use **system calls**:

```
User Program (User Mode)
        ↓
   Call system call wrapper
   (e.g., write() in libc)
        ↓
   Trigger software interrupt
   (trap to kernel)
        ↓
   CPU switches to Kernel Mode
        ↓
   Kernel validates request
        ↓
   Kernel executes operation
        ↓
   CPU switches back to User Mode
        ↓
   Return to user program
```

**Example: Reading a file**

```c
// User space code
int fd = open("/home/user/file.txt", O_RDONLY);
char buffer[100];
ssize_t bytes = read(fd, buffer, 100);
close(fd);

// Each of these (open, read, close) is a system call
// Kernel validates:
• Does this file exist?
• Does the user have permission?
• Is the file descriptor valid?
```

**Common system calls:**

| Category | Examples |
|----------|----------|
| **Process** | fork, exec, exit, wait, kill |
| **File** | open, read, write, close, stat |
| **Memory** | mmap, munmap, brk, sbrk |
| **Network** | socket, bind, listen, accept, send, recv |
| **IPC** | pipe, shm_open, msgget, semget |
| **Signals** | signal, sigaction, kill |

**System call overhead:**

```
Function call (same mode): ~1-5 nanoseconds
System call (mode switch): ~50-100 nanoseconds

Why slower?
1. Save user context
2. Switch to kernel mode
3. Validate arguments
4. Execute
5. Restore context
6. Switch back to user mode
```

This overhead is why performance-critical code minimizes system calls.

---

## Layer 3: User-Space Libraries

**What they are:** Helper code that makes programming easier

### C Standard Library (libc)

**Purpose:** Provide convenient wrappers around system calls and common utilities

**Common implementations:**

- **glibc (GNU C Library):** Feature-rich, widely used on Linux
- **musl:** Lightweight, designed for embedded/containers
- **uclibc:** Tiny, for very constrained systems

**What it provides:**

```c
// Low-level: System call
ssize_t bytes = syscall(SYS_write, fd, buffer, count);

// High-level: libc wrapper
printf("Hello, world!\n");  // Calls write() internally

// libc also provides:
• String manipulation (strlen, strcpy)
• Memory allocation (malloc, free)
• Math functions (sin, sqrt)
• Time/date (time, strftime)
```

---

### POSIX API

**What it is:** Standard that defines how Unix-like systems should behave

**Purpose:** Write once, run anywhere (on POSIX systems)

**Example:**

```c
// This code works on Linux, macOS, BSD, etc.
#include <unistd.h>
#include <fcntl.h>

int fd = open("file.txt", O_RDONLY);
read(fd, buffer, 100);
close(fd);

// POSIX guarantees these functions exist and behave consistently
```

**POSIX defines:**
- System calls (open, read, fork, etc.)
- Shell behavior (sh, utilities)
- Threading (pthreads)
- Signals
- File permissions

---

## Layer 4: System Services and Daemons

**What they are:** Background processes that provide system-wide services

**Key characteristic:** Run without user interaction

### Init System: The First Process

**What it is:** First user-space process started by kernel (PID 1)

**Responsibilities:**
- Start all other services
- Restart crashed services
- Shut down system gracefully

**Evolution:**

```
Historical: SysV init (scripts in /etc/init.d)
        ↓
Modern: systemd (parallel startup, dependencies)
        ↓
Alternative: OpenRC, runit (simpler, faster)
```

**systemd example:**

```bash
# Start a service
$ sudo systemctl start nginx

# Enable service to start at boot
$ sudo systemctl enable nginx

# Check status
$ sudo systemctl status nginx
● nginx.service - A high performance web server
   Active: active (running) since Feb 06 10:00:00

# View dependencies
$ systemctl list-dependencies nginx
```

**Service definition (nginx.service):**

```ini
[Unit]
Description=A high performance web server
After=network.target

[Service]
Type=forking
ExecStart=/usr/sbin/nginx
ExecReload=/usr/sbin/nginx -s reload
ExecStop=/usr/sbin/nginx -s stop

[Install]
WantedBy=multi-user.target
```

---

### Common System Daemons

**Network configuration:**
- **NetworkManager:** Manages WiFi, Ethernet connections
- **systemd-networkd:** Lightweight alternative
- **dhclient/dhcpcd:** DHCP clients for automatic IP configuration

**Device management:**
- **udev:** Detects hardware, loads drivers, creates device nodes
- **udisks2:** Manages removable media (USB drives)

**Inter-process messaging:**
- **D-Bus:** Message bus for process communication
- **Example:** Desktop notifications use D-Bus

**Logging:**
- **systemd-journald:** Collects logs from all services
- **rsyslog:** Traditional syslog daemon

**Time synchronization:**
- **chronyd/ntpd:** Keep system clock synchronized via NTP

**Remote access:**
- **sshd:** Secure shell server for remote login

---

## Layer 5: Graphical and User Interface Stack

For systems with a GUI (desktops, laptops), there's an entire stack dedicated to graphics:

### Display Server/Protocol

**What it does:** Manages display output and input events

**X11 (Legacy):**

```
┌─────────────────────────────────┐
│  Applications                   │
├─────────────────────────────────┤
│  X11 Protocol                   │
├─────────────────────────────────┤
│  X Server                       │
│  • Draws windows                │
│  • Handles input                │
├─────────────────────────────────┤
│  Kernel (DRM/KMS drivers)       │
└─────────────────────────────────┘
```

**Issues with X11:**
- Complex legacy code
- Security issues (apps can spy on each other)
- Performance overhead (network-transparent design)

**Wayland (Modern):**

```
┌─────────────────────────────────┐
│  Applications draw their own    │
│  windows (using EGL/OpenGL)     │
├─────────────────────────────────┤
│  Wayland Compositor             │
│  • Composites windows           │
│  • Handles input                │
├─────────────────────────────────┤
│  Kernel (DRM/KMS)               │
└─────────────────────────────────┘
```

**Advantages:**
- Simpler, more secure
- Better performance
- Per-window features (rotation, transparency)

---

### Window Manager / Compositor

**What it does:** Controls window placement, decorations, animations

**Examples:**

**X11 window managers:**
- **i3:** Tiling window manager
- **Openbox:** Lightweight, floating
- **Compiz:** Advanced effects

**Wayland compositors:**
- **Mutter:** GNOME's compositor
- **KWin:** KDE's compositor
- **Sway:** i3-compatible for Wayland

**Example features:**
- Window borders and title bars
- Alt-Tab switching
- Virtual desktops/workspaces
- Animations (minimize, maximize)
- Transparency and shadows

---

### Desktop Environment

**What it is:** Complete user experience (file manager, settings, etc.)

**Major desktop environments:**

**GNOME:**
- Modern, touch-friendly
- GTK-based
- Used by Ubuntu, Fedora

**KDE Plasma:**
- Highly customizable
- Qt-based
- Feature-rich

**XFCE:**
- Lightweight
- Traditional desktop metaphor
- Good for older hardware

**Cinnamon:**
- Fork of GNOME
- Windows-like interface
- Used by Linux Mint

---

### Widget Toolkits

**What they are:** Libraries for building GUI applications

**GTK (GIMP Toolkit):**
```c
GtkWidget *window = gtk_window_new(GTK_WINDOW_TOPLEVEL);
GtkWidget *button = gtk_button_new_with_label("Click me");
gtk_container_add(GTK_CONTAINER(window), button);
gtk_widget_show_all(window);
```

**Qt:**
```cpp
QApplication app(argc, argv);
QPushButton button("Click me");
button.show();
return app.exec();
```

---

## Layer 6: User Applications

**Finally, the software you actually use:**

- **Shells:** bash, zsh, fish (command-line interfaces)
- **Terminals:** gnome-terminal, konsole, alacritty
- **File managers:** Nautilus, Dolphin, Thunar
- **Browsers:** Firefox, Chrome, Edge
- **Editors:** vim, emacs, VS Code, gedit
- **Office suites:** LibreOffice, OnlyOffice
- **Media players:** VLC, mpv
- **Package managers:** apt, dnf, pacman (manage software installation)

---

## Modern and Emerging OS Technologies

### eBPF (Extended Berkeley Packet Filter)

**What it is:** Safe, programmable kernel extensions

**Traditional problem:**
Want custom kernel behavior? Write a kernel module (risky, can crash system)

**eBPF solution:**
Write programs that run **inside** the kernel but are **verified safe**

```
User space eBPF program
        ↓
   Kernel verifier checks:
   • No infinite loops
   • No unauthorized memory access
   • Will terminate
        ↓
   JIT compile and load into kernel
        ↓
   Runs at kernel speed with safety
```

**Use cases:**
- Network filtering and monitoring
- Performance profiling
- Security enforcement
- Custom schedulers

---

### Container Technologies

**Flatpak / Snap:**
- Sandboxed applications
- Bundled dependencies
- Cross-distribution packages

**Docker / Podman:**
- Application containers
- Isolated environments
- Uses namespaces + cgroups

---

### PipeWire

**What it is:** Modern audio/video infrastructure

**Replaces:**
- PulseAudio (audio)
- JACK (pro audio)
- Video capture frameworks

**Benefits:**
- Low latency
- Unified audio/video handling
- Better security (Wayland-compatible)

---

### Immutable Operating Systems

**Concept:** OS files are read-only, updates are atomic

**Examples:**
- Fedora Silverblue
- openSUSE MicroOS
- NixOS

**Benefits:**
- Reliable updates (rollback if something breaks)
- Reproducible systems
- Better security (harder to modify)

---

## Summary: The Complete Stack

Let's put it all together from bottom to top:

```
┌─────────────────────────────────────────────────────┐
│  Layer 6: User Applications                         │
│  • Browsers, editors, games, productivity tools     │
└─────────────────────────────────────────────────────┘
                        ↑
┌─────────────────────────────────────────────────────┐
│  Layer 5: Graphical Stack (if GUI system)           │
│  • Desktop environment, window manager, display     │
└─────────────────────────────────────────────────────┘
                        ↑
┌─────────────────────────────────────────────────────┐
│  Layer 4: System Services & Daemons                 │
│  • Init system, network, logging, device management │
└─────────────────────────────────────────────────────┘
                        ↑
┌─────────────────────────────────────────────────────┐
│  Layer 3: User-Space Libraries                      │
│  • libc, POSIX API, runtime libraries               │
└─────────────────────────────────────────────────────┘
                        ↑
┌─────────────────────────────────────────────────────┐
│  Layer 2: System Call Interface                     │
│  • Gateway between user and kernel space            │
└═════════════════════════════════════════════════════┘
                        ↑
┌─────────────────────────────────────────────────────┐
│  Layer 1: Kernel (Privileged)                       │
│  • Process, memory, file system, drivers, network   │
│  • Security, IPC, power management                  │
└─────────────────────────────────────────────────────┘
                        ↑
┌─────────────────────────────────────────────────────┐
│  Layer 0: Firmware & Boot                           │
│  • BIOS/UEFI, bootloader                            │
└─────────────────────────────────────────────────────┘
                        ↑
                  ┌─────────────┐
                  │  Hardware   │
                  └─────────────┘
```

**Key principles:**

1. **Layering:** Each layer uses services from the layer below
2. **Abstraction:** Higher layers see simpler interfaces
3. **Protection:** Kernel enforces boundaries between layers
4. **Modularity:** Components can be replaced (e.g., swap desktop environments)

**Mental model:**

> The kernel **controls** hardware.
> System services **provide** functionality.
> User space **consumes** abstractions.
> Applications **deliver** value to users.

This layered architecture is what makes modern operating systems powerful, secure, portable, and maintainable.