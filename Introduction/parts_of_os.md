# Parts of an Operating System

An operating system (OS) is built in layers. Each layer has a specific job, from talking directly to hardware up to showing you a nice desktop. Below is a clear, organized explanation of every major part, with simple definitions for terms you might not know.

### 1. Firmware & Boot Process (The very first things that run)

- **BIOS / UEFI**  
  This is software stored on your motherboard's chip. When you press the power button, it wakes up first.  
  It checks if the hardware (CPU, RAM, keyboard, etc.) is working (this is called **POST** = Power-On Self-Test), turns on basic devices, and then finds and starts the **bootloader**.  
  Old computers used **BIOS**; almost all modern computers use **UEFI** (faster, supports big hard drives, has a mouse-friendly menu, and can do **secure boot** to prevent malware from starting early).

- **Bootloader**  
  A small program that knows where your operating system files are.  
  It loads the **kernel** (the heart of the OS) into RAM and hands control over to it.  
  Examples: GRUB (most Linux), systemd-boot (some Linux), Windows Boot Manager, Apple's bootloader.  
  It often shows a menu if you have multiple operating systems installed.

### 2. The Kernel (The most important and privileged part)

The **kernel** is the only software allowed to talk directly to hardware. Everything else asks the kernel for help. It runs in a special protected mode called **kernel mode**.

#### Kernel Styles (How the kernel is designed)
- **Monolithic kernel**  
  Almost everything (file system, drivers, networking) is inside one big kernel program.  
  Fast, but if one part crashes, the whole system usually crashes.  
  Example → Linux, FreeBSD.

- **Microkernel**  
  Only the smallest, most critical parts run inside the kernel. Everything else (drivers, file systems) runs as normal programs outside.  
  Safer and easier to fix bugs, but slower because parts talk via messages.  
  Examples → QNX (used in cars), Minix, seL4 (very secure research kernel).

- **Hybrid kernel**  
  Mix of both: important things run fast inside the kernel, but some parts can be loaded separately.  
  Examples → Windows (NT kernel), macOS (XNU kernel).

#### Main Jobs Inside the Kernel
- **Process & Thread Management**  
  A **process** is a running program (like your web browser). A **thread** is a small unit of work inside a process (browser can have many tabs = many threads).  
  The kernel decides which process/thread gets to use the CPU at any moment (**scheduling**), starts/stops them, and switches between them (**context switching**).

- **Memory Management**  
  Gives each program its own fake (“virtual”) memory space so they don’t overwrite each other.  
  Uses **paging** (memory divided into small pages), **page faults** (when program needs data not in RAM), **swapping** (moving unused pages to disk), and protects memory so one program can’t read another’s data.

- **File System & Storage Management**  
  Lets programs read/write files without knowing whether they are on SSD, HDD, USB, or network.  
  **Virtual File System (VFS)** → a common interface layer in Linux that hides differences between file systems (ext4, NTFS, FAT32, etc.).  
  Manages **page cache** (keeps recently used file data in RAM for speed), **I/O scheduler** (decides order of disk reads/writes), and encryption tools.

- **Device Drivers**  
  Small programs that know how to talk to specific hardware (your Wi-Fi card, graphics card, printer, keyboard).  
  In Linux these are often **kernel modules** (.ko files) that can be loaded/unloaded without rebooting.

- **Networking Stack**  
  Handles internet and local network communication.  
  Implements **TCP/IP** (rules for sending data over internet), **sockets** (way programs talk over network), firewall rules, etc.

- **Security & Protection**  
  Controls who can do what.  
  Includes **capabilities** (fine-grained permissions instead of just root/regular user), **SELinux/AppArmor** (extra strict rules), **seccomp** (limits what a program can ask the kernel to do), and **namespaces & cgroups** (used for containers):  
  → **Namespaces** → make it look like a process has its own private view of the system (own processes, network, files).  
  → **cgroups** (control groups) → limit how much CPU, memory, disk I/O a group of processes can use.

- **Inter-Process Communication (IPC)**  
  Ways programs talk to each other: pipes (| in terminal), shared memory, signals, sockets, etc.

- **Power & Thermal Management**  
  Lowers CPU speed or turns off unused parts to save battery and prevent overheating.

### 3. System Call Interface (The door between user programs and kernel)

Programs cannot directly access hardware or kernel data. They ask for help by making **system calls** (syscalls).  
Examples: open a file, read keyboard input, send data over network.  
This is the only safe bridge between normal programs and the kernel.

### 4. Core Userspace Libraries & Runtime

- **C Standard Library** (libc)  
  A collection of ready-made functions for C programs (printf, malloc, open, fork, etc.).  
  Most common: **glibc** (GNU C Library – big, feature-rich, used by most Linux distros) vs **musl** (small, simple, used in Alpine Linux, good for containers).

- **POSIX API**  
  Stands for **Portable Operating System Interface**.  
  A set of standard rules and function names (like open(), read(), fork()) that Unix-like systems (Linux, macOS, BSD) follow.  
  If a program uses only POSIX functions, it can usually run on any POSIX-compatible OS with little or no change.

### 5. System Services & Daemons (Background programs)

- **Init system**  
  The first user-space program that starts after kernel boots. It starts all other services.  
  Modern Linux → **systemd** (very powerful, handles services, logging, networking, user logins).  
  Others → OpenRC, SysV init (older), launchd (macOS).

- **Common daemons** (services that run in background)  
  - NetworkManager → handles Wi-Fi and Ethernet  
  - dbus → lets programs send messages to each other  
  - udevd → detects and sets up new devices (USB plug in)  
  - sshd → allows remote login

### 6. Graphical & User Interface Stack

- **Display Server / Protocol**  
  Manages windows, mouse, keyboard input.  
  Old → **X11 / X.Org** (not very secure).  
  Modern → **Wayland** (more secure, smoother, most new desktops use it).

- **Compositor / Window Manager**  
  Draws windows on screen, handles transparency, animations, moving/resizing windows.  
  Examples: Mutter (GNOME), KWin (KDE Plasma), Sway (tiling manager).

- **Desktop Environment (DE)**  
  Full user interface package: menus, file manager, settings app, wallpapers, themes.  
  Popular: **GNOME** (simple, modern), **KDE Plasma** (very customizable), **XFCE** (lightweight), **Cinnamon** (traditional Windows-like).

- **Widget Toolkits**  
  Libraries used to draw buttons, menus, windows: **GTK** (used by GNOME), **Qt** (used by KDE).

### 7. User Applications & Tools

- Terminal + shell (bash, zsh, fish) → command line  
- File managers (Nautilus, Dolphin)  
- Web browsers, text editors, games, etc.  
- Package managers (apt, dnf, pacman) → install software

### 8. Modern / Emerging Concepts (2025–2026)

- **eBPF** → safe way to run small programs inside the kernel without changing kernel code. Used for monitoring, security, networking.  
- **Flatpak / Snap** → new way to package apps so they work on any Linux distro.  
- **PipeWire** → modern replacement for old audio systems (better for video calls, low-latency audio).  
- **Immutable OS** → OS files are read-only; updates create a new clean version (Silverblue, NixOS).
