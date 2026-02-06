# Process Memory Organization

## Introduction

When you run a program, the operating system doesn't just throw your code into a random chunk of memory. Instead, it creates a **process** with a carefully structured **virtual address space**—a private memory layout that makes execution predictable, safe, and efficient.

Think of this address space as a well-organized building where different floors serve different purposes. This organization isn't arbitrary; it directly reflects how programs actually execute.

---

## The Big Picture: Memory Layout

A typical process divides its memory into five key regions:

```
High Memory Addresses
┌─────────────────────┐
│       Stack         │  ← Function calls, local variables
│   (grows ↓)         │
├─────────────────────┤
│                     │
│    (free space)     │
│                     │
├─────────────────────┤
│       Heap          │  ← Dynamic memory (malloc/new)
│   (grows ↑)         │
├─────────────────────┤
│       .bss          │  ← Uninitialized global variables
├─────────────────────┤
│       .data         │  ← Initialized global variables
├─────────────────────┤
│    Text (Code)      │  ← Your compiled program instructions
└─────────────────────┘
Low Memory Addresses
```

Each region has a specific job. Let's explore them from the ground up.

---

## Region 1: Text Segment (The Instructions)

**Purpose:** Stores your compiled program code—the actual machine instructions the CPU executes.

**Characteristics:**
- **Read-only:** Prevents accidental (or malicious) code modification
- **Shareable:** Multiple processes running the same program can share one copy in physical memory
- **Fixed size:** Determined when the program is compiled

**Why it exists:**
Programs need a stable, protected place to store instructions. Making this read-only prevents bugs where you accidentally overwrite your own code, and sharing saves memory when running multiple instances of the same application.

**Example:**
```c
void foo(int x) {
    printf("%d\n", x * 2);
}

int main() {
    foo(5);
    return 0;
}
```

The compiled machine code for `foo`, `main`, and `printf` all live in the text segment.

---

## Region 2: Data Segment (Global Variables)

The data segment stores variables that live for the **entire duration** of your program. It's split into two sections based on whether variables have initial values.

### .data Section (Initialized Globals)

**Purpose:** Stores global and static variables with explicit initial values.

**Example:**
```c
int global_counter = 100;
static int file_scope_var = 42;
```

**Characteristics:**
- Initial values are **embedded in the executable file**
- The OS copies these values into memory when loading the program
- Writable at runtime (you can change `global_counter`)

**Why separate from .bss?**
Storing initial values in the executable ensures your program starts with the exact state you specified.

### .bss Section (Uninitialized Globals)

**Purpose:** Stores global and static variables that are declared but not explicitly initialized.

**Example:**
```c
int global_array[1000];
static int uninitialized;
```

**BSS** stands for "Block Started by Symbol" (historical name from assemblers).

**Characteristics:**
- **Not stored in the executable file** (saves disk space!)
- Automatically **initialized to zero** by the OS at program startup
- Writable at runtime

**Why it exists:**
If you declare a large array without initializing it, why should the executable file contain thousands of zeros? The .bss segment is a smart optimization: the OS just allocates zeroed memory when the program starts.

**Key Insight:**
```c
int x;        // → .bss (zero-initialized)
int y = 0;    // → .data (explicitly initialized)
```

Both end up zero, but `x` doesn't take space in the executable file while `y` does.

---

## Region 3: Heap (Dynamic Memory)

**Purpose:** Provides memory that you allocate and free **during program execution**.

**Example:**
```c
int *ptr = malloc(sizeof(int) * 100);  // Request memory
*ptr = 42;                              // Use it
free(ptr);                              // Release it
```

**Characteristics:**
- Size determined **at runtime**, not compile time
- You control the lifetime (unlike stack variables)
- Grows **upward** toward the stack
- Managed by memory allocator (`malloc`/`free` in C, `new`/`delete` in C++)

**Why it exists:**
Sometimes you don't know how much memory you need until the program runs. Maybe you're reading user input, loading a file, or building a dynamic data structure. The heap gives you that flexibility.

**How it works internally:**

The heap allocator maintains **metadata** to track which blocks are free and which are allocated:

```
┌──────────────┬─────────────┬──────────────┐
│ Free (32 B)  │ Used (64 B) │ Free (128 B) │
└──────────────┴─────────────┴──────────────┘
```

Common allocation strategies:
- **First-fit:** Use the first free block that's big enough
- **Best-fit:** Use the smallest free block that fits
- **Buddy allocator:** Split memory into power-of-two blocks

**Fragmentation:**
As you allocate and free memory, the heap can become fragmented—free memory exists but isn't contiguous enough for large allocations.

**Performance consideration:**
Heap allocation is slower than stack allocation because it involves:
- Searching for free blocks
- Updating metadata
- Possible system calls to request more memory from the OS

---

## Region 4: Stack (Function Execution)

**Purpose:** Manages function calls and their local data automatically.

**Example:**
```c
void calculate(int x) {
    int result = x * 2;      // Local variable
    printf("%d\n", result);
}

int main() {
    int value = 10;          // Local variable
    calculate(value);
    return 0;
}
```

**Characteristics:**
- Grows **downward** from high addresses
- Automatic allocation and deallocation (no `malloc`/`free` needed)
- **LIFO (Last-In-First-Out):** Functions called later return first
- Very fast (just moving a pointer)

**Why it exists:**
Function calls need temporary storage that appears when you call the function and disappears when it returns. The stack provides this with zero programmer effort.

---

### Understanding Stack Frames

Every function call creates a **stack frame** (also called an **activation record**). This is a structured block of memory containing everything the function needs.

**A stack frame includes:**
1. **Function parameters**
2. **Local variables**
3. **Return address** (where to resume after the function finishes)
4. **Saved registers** (CPU state that must be restored)
5. **Frame linkage** (pointer to the previous frame)

**Example walkthrough:**

```c
int add(int a, int b) {
    int sum = a + b;
    return sum;
}

int main() {
    int x = 5;
    int result = add(x, 10);
    return 0;
}
```

**Step 1: main() starts executing**
```
┌─────────────────────┐
│ main's locals       │
│   x = 5             │
│   result (uninit)   │
├─────────────────────┤ ← Stack Pointer (SP)
│  (free space)       │
└─────────────────────┘
```

**Step 2: Calling add(x, 10)**

The CPU:
1. Pushes parameters (5 and 10)
2. Pushes return address (where to resume in `main`)
3. Allocates space for local variables (`sum`)

```
┌─────────────────────┐
│ main's frame        │
│   x = 5             │
│   result (uninit)   │
├─────────────────────┤
│ return address      │  ← Where to resume in main
├─────────────────────┤
│ saved frame pointer │
├─────────────────────┤
│ parameter b = 10    │
├─────────────────────┤
│ parameter a = 5     │
├─────────────────────┤
│ add's locals        │
│   sum (uninit)      │
├─────────────────────┤ ← Stack Pointer (SP)
│  (free space)       │
└─────────────────────┘
```

**Step 3: add() executes**
```c
int sum = a + b;  // sum = 15
```

**Step 4: add() returns**

The CPU:
1. Moves return value to a register
2. Restores Stack Pointer to remove add's frame
3. Jumps to the return address

```
┌─────────────────────┐
│ main's frame        │
│   x = 5             │
│   result = 15       │  ← Assigned from return value
├─────────────────────┤ ← Stack Pointer (SP)
│  (free space)       │
└─────────────────────┘
```

The memory used by `add` is instantly reclaimed—no need to free anything.

---

### Stack Pointer and Frame Pointer

Two CPU registers manage the stack:

**Stack Pointer (SP):**
- Points to the **current top** of the stack
- Moves up/down as memory is allocated/freed
- Marks the boundary between used and free memory

**Frame Pointer (FP):**
- Points to a **fixed location** in the current stack frame
- Provides a stable reference for accessing parameters and locals
- Allows accessing variables at **fixed offsets** even if SP moves

**Why use both?**

During function execution, SP might move (e.g., if you call another function or allocate temporary space). Having FP stay constant lets you access variables reliably:

```
FP + 16  →  parameter 'a'
FP + 12  →  parameter 'b'
FP - 4   →  local variable 'sum'
FP - 8   →  temporary storage
```

---

### Stack Benefits and Limitations

**Benefits:**
- ✅ **Automatic management:** Variables appear and disappear automatically
- ✅ **Lightning fast:** Just pointer arithmetic
- ✅ **Naturally supports recursion:** Each call gets its own frame
- ✅ **No fragmentation:** Memory is always contiguous

**Limitations:**
- ❌ **Limited size:** Typically a few MB (stack overflow if exceeded)
- ❌ **Fixed lifetime:** Variables die when the function returns
- ❌ **Can't return addresses of local variables** (they become invalid)

**Common mistake:**
```c
int* bad_function() {
    int local = 42;
    return &local;  // ❌ DANGER: local vanishes when function returns
}
```

---

## Heap vs Stack: When to Use What?

| Feature | Stack | Heap |
|---------|-------|------|
| **Speed** | Very fast (pointer manipulation) | Slower (allocator overhead) |
| **Size** | Limited (few MB) | Large (GB available) |
| **Lifetime** | Automatic (function scope) | Manual (until you free it) |
| **Use for** | Local variables, small data | Large data, variable size, persistence |

**Rule of thumb:**
- Use **stack** by default (it's automatic and fast)
- Use **heap** when you need:
  - Data larger than stack limits
  - Data that outlives the function
  - Size determined at runtime

---

## Advanced Concepts

### Virtual Memory and Isolation

Each process gets its own **virtual address space**. What does this mean?

**Without virtual memory:**
```
Process A sees:  [0x1000: its code] [0x2000: its data]
Process B sees:  [0x1000: its code] [0x2000: its data]  ❌ Conflict!
```

**With virtual memory:**
```
Process A virtual:  [0x1000: A's code] → Physical: [0x5000]
Process B virtual:  [0x1000: B's code] → Physical: [0x9000]  ✅ Isolated!
```

The **Memory Management Unit (MMU)** translates virtual addresses to physical addresses using **page tables**:

```
Virtual Address Space          Physical RAM
┌────────────────────┐        ┌─────────────────┐
│ Stack (0x7FFF...)  │───────→│ Frame 0xA000... │
├────────────────────┤        ├─────────────────┤
│ Heap (0x6000...)   │───────→│ Frame 0xB000... │
├────────────────────┤        ├─────────────────┤
│ .data (0x2000...)  │───────→│ Frame 0xC000... │
├────────────────────┤        ├─────────────────┤
│ Text (0x1000...)   │───────→│ Frame 0xD000... │
└────────────────────┘        └─────────────────┘
```

**Benefits:**
- Each process thinks it has its own complete memory
- One process can't accidentally (or maliciously) access another's memory
- OS can swap unused pages to disk, using RAM efficiently

---

### Memory Protection

The OS enforces **permissions** on each memory region:

| Region | Read | Write | Execute |
|--------|------|-------|---------|
| Text   | ✅   | ❌    | ✅      |
| .data  | ✅   | ✅    | ❌      |
| .bss   | ✅   | ✅    | ❌      |
| Heap   | ✅   | ✅    | ❌      |
| Stack  | ✅   | ✅    | ❌      |

**Why this matters:**

Making the text segment read-only prevents:
- Bugs from corrupting your code
- Buffer overflow attacks from injecting malicious code

Making data segments non-executable prevents:
- Code injection attacks
- Accidentally executing data as code

---

### Compiler Optimization: Escape Analysis

Smart compilers perform **escape analysis** to determine if a variable's lifetime is strictly within a function.

**If a variable doesn't "escape":** Allocate it on the stack (faster!)

**If it escapes:** Must allocate on the heap

**Example (Go):**
```go
func noEscape() int {
    x := 42        // ✅ Doesn't escape → stack allocated
    return x
}

func escapes() *int {
    x := 42        // ❌ Escapes (pointer returned) → heap allocated
    return &x
}
```

This optimization reduces garbage collection pressure and improves performance in managed languages.

---

## Putting It All Together: A Complete Example

```c
#include <stdio.h>
#include <stdlib.h>

int global_init = 100;      // → .data
int global_uninit;          // → .bss

void process(int x) {       // → text
    int local = x * 2;      // → stack (local variable)
    printf("%d\n", local);  // → text (printf code)
}

int main() {                // → text
    int a = 5;              // → stack (main's local)
    int *ptr = malloc(10);  // ptr on stack, allocated memory on heap
    *ptr = 20;              // Write to heap
    
    process(a);             // Creates stack frame for process()
    
    free(ptr);              // Return heap memory
    return 0;
}
```

**Memory snapshot while process() is executing:**

```
High Addresses
┌─────────────────────────┐
│ Stack:                  │
│   process's frame       │
│     local = 10          │
│     return address      │
│   main's frame          │
│     a = 5               │
│     ptr = 0x6000        │
├─────────────────────────┤
│ Heap:                   │
│   [0x6000] = 20         │
├─────────────────────────┤
│ .bss:                   │
│   global_uninit = 0     │
├─────────────────────────┤
│ .data:                  │
│   global_init = 100     │
├─────────────────────────┤
│ Text:                   │
│   main() {...}          │
│   process() {...}       │
│   printf() {...}        │
└─────────────────────────┘
Low Addresses
```

---

## Summary

A process's memory is organized into specialized regions, each serving a distinct purpose:

| Region | Purpose | Lifetime | Size |
|--------|---------|----------|------|
| **Text** | Executable code | Entire process | Fixed at compile time |
| **.data** | Initialized globals | Entire process | Fixed at compile time |
| **.bss** | Uninitialized globals (zeroed) | Entire process | Fixed at compile time |
| **Heap** | Dynamic allocation | Until freed | Grows as needed |
| **Stack** | Function calls and locals | Function scope | Fixed limit, grows/shrinks |

**Key Takeaways:**

1. **Separation creates safety** — each region has specific permissions and purposes
2. **Stack is automatic and fast** — use it for most local variables
3. **Heap is flexible but manual** — use it for dynamic or large data
4. **Virtual memory provides isolation** — each process has its own protected space
5. **Understanding memory layout helps you write better code** — knowing where data lives explains performance, bugs, and security issues

This organization isn't accidental—it's the result of decades of operating system design evolution, balancing performance, safety, and programmer convenience.