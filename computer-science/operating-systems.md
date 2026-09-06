# Operating Systems: From Zero to Expert

A self-contained guide that assumes no computer science background. It starts with what a computer physically is, and ends with the internals that OS engineers and systems programmers work with daily. Read it top to bottom the first time; later, use it as a reference.

---

## Table of Contents

1. [Part 0 — The Computer Itself](#part-0--the-computer-itself)
2. [Part 1 — What an Operating System Is](#part-1--what-an-operating-system-is)
3. [Part 2 — The Kernel and the Two Worlds](#part-2--the-kernel-and-the-two-worlds)
4. [Part 3 — Processes: Programs in Motion](#part-3--processes-programs-in-motion)
5. [Part 4 — Threads and Scheduling](#part-4--threads-and-scheduling)
6. [Part 5 — Memory Management and Virtual Memory](#part-5--memory-management-and-virtual-memory)
7. [Part 6 — Concurrency and Synchronization](#part-6--concurrency-and-synchronization)
8. [Part 7 — File Systems and Storage](#part-7--file-systems-and-storage)
9. [Part 8 — Input/Output and Device Drivers](#part-8--inputoutput-and-device-drivers)
10. [Part 9 — Inter-Process Communication](#part-9--inter-process-communication)
11. [Part 10 — The Boot Process](#part-10--the-boot-process)
12. [Part 11 — Security and Protection](#part-11--security-and-protection)
13. [Part 12 — Virtualization and Containers](#part-12--virtualization-and-containers)
14. [Part 13 — Networking from the OS's Point of View](#part-13--networking-from-the-oss-point-of-view)
15. [Part 14 — Performance: How Experts Think](#part-14--performance-how-experts-think)
16. [Part 15 — The Road to Expertise](#part-15--the-road-to-expertise)

---

# Part 0 — The Computer Itself

Before understanding the operating system, you need a mental model of the machine it runs on. You don't need electronics knowledge — just four ideas.

## 0.1 A computer is four things

```
+--------------------------------------------------------------+
|                                                              |
|   +---------+      +-----------+      +------------------+   |
|   |   CPU   |<---->|   RAM     |      |  Storage (disk)  |   |
|   | (brain) |      | (desk)    |      |  (filing cabinet)|   |
|   +---------+      +-----------+      +------------------+   |
|        ^                                       ^             |
|        |                                       |             |
|        v                                       v             |
|   +----------------------------------------------------+    |
|   |  I/O devices: keyboard, screen, network card, USB  |    |
|   +----------------------------------------------------+    |
|                                                              |
+--------------------------------------------------------------+
```

1. **CPU (Central Processing Unit)** — the "brain." It does exactly one kind of thing: it executes *instructions*, tiny commands like "add these two numbers" or "copy this value to that location." It does billions of them per second. It is astonishingly fast and astonishingly dumb — it has no idea what a file, a window, or a program is. It just executes the next instruction, forever.

2. **RAM (memory)** — the "desk." A huge grid of numbered slots, each holding one byte (a number from 0–255). The CPU can read or write any slot almost instantly. RAM is *volatile*: when power goes off, everything in it vanishes. Your running programs — their code and their data — live here.

3. **Storage (disk/SSD)** — the "filing cabinet." Keeps data when the power is off, but is *thousands to millions of times slower* than RAM. Files live here.

4. **I/O devices** — everything else: keyboard, display, network card, mouse. Ways for data to enter and leave the machine.

## 0.2 The one loop that runs everything

The CPU runs a single loop, forever, called the **fetch-decode-execute cycle**:

1. **Fetch** the instruction at the memory address held in a special register called the *program counter* (PC).
2. **Decode** it (figure out what it says to do).
3. **Execute** it (do the addition, the copy, the comparison...).
4. Move the program counter to the next instruction. Repeat.

A "program" is nothing more than a long list of these instructions sitting in memory. "Running a program" means "pointing the CPU's program counter at the program's first instruction."

**Registers** are a handful of tiny, ultra-fast storage slots inside the CPU itself (things like `rax`, `rsp` on x86). All actual computation happens in registers; RAM is just where values are parked when the CPU isn't actively using them.

## 0.3 The memory hierarchy: why speed differences shape everything

Almost every design decision in an OS exists because of this table. Times are rough but the *ratios* are what matter:

| Storage | Typical access time | Human-scale analogy (1 cycle = 1 second) |
|---|---|---|
| CPU register | < 1 ns | 1 second |
| L1 cache | ~1 ns | a few seconds |
| L2/L3 cache | ~4–40 ns | ~1 minute |
| RAM | ~100 ns | ~2 minutes |
| SSD | ~100 µs | ~1 day |
| Hard disk | ~10 ms | ~4 months |
| Network round trip (same city) | ~1 ms | ~2 weeks |

The lesson: **going to disk or network is catastrophically slow compared to computing.** An enormous amount of what an OS does — caching, buffering, scheduling other work during waits — is coping with this hierarchy.

## 0.4 Interrupts: how the outside world gets the CPU's attention

The CPU only executes instructions; it doesn't "watch" the keyboard. So how does a keypress get noticed?

Hardware devices can send the CPU an **interrupt** — an electrical signal that says "stop what you're doing." When an interrupt arrives, the CPU:

1. Pauses the current instruction stream,
2. Saves where it was,
3. Jumps to a pre-registered function called an **interrupt handler** (part of the OS),
4. After the handler finishes, resumes exactly where it left off.

Interrupts are the heartbeat of an operating system. Keyboard input, disk completion, network packets arriving, and the timer that lets the OS regain control from a running program — all are interrupts. Keep this concept in your pocket; it comes back constantly.

---

# Part 1 — What an Operating System Is

## 1.1 The problem an OS solves

Imagine a computer with no OS. To write even a trivial program you would have to:

- Know the exact hardware commands for *your specific* disk model to read bytes off it.
- Manually decide which RAM addresses your program may use — and hope no other program uses the same ones.
- Have complete control of the CPU, meaning if your program hangs, the machine hangs.
- Trust every other program completely, because any program could read or overwrite anything.

Early computers actually worked this way: one program at a time, loaded by hand, with full control of the machine.

An **operating system** is the program that solves these problems for all other programs. It has two jobs, and every OS topic falls under one of them:

1. **Abstraction** — present clean, simple, uniform concepts (*files*, *processes*, *sockets*) in place of messy, varied hardware (spinning platters, CPU privilege bits, network chips).
2. **Resource management** — share the machine's limited resources (CPU time, RAM, disk, network) among many competing programs, safely and fairly.

A useful slogan: **the OS turns one physical computer into many virtual ones.** Each program gets the illusion of its own private CPU (via scheduling), its own private memory (via virtual memory), and its own tidy storage (via file systems).

## 1.2 The abstractions, previewed

| Physical reality | OS abstraction | Covered in |
|---|---|---|
| One CPU switching rapidly between tasks | Each program thinks it runs continuously (**process/thread**) | Parts 3–4 |
| One shared RAM | Each program thinks it has all memory to itself (**virtual memory**) | Part 5 |
| Raw blocks on a disk | Named **files** in folders | Part 7 |
| Network cards and packets | **Sockets** — readable/writable byte streams | Part 13 |
| Wildly different device hardware | Uniform read/write interfaces (**drivers**) | Part 8 |

## 1.3 The OS family tree (what you're actually running)

- **UNIX** (1970s, Bell Labs) — the ancestor whose design ideas dominate everything below.
- **Linux** — a free UNIX-like kernel (1991); runs most servers, all of Android, most cloud infrastructure. The examples in this guide lean on Linux because it's open source — you can read every line of it.
- **macOS / iOS** — built on a UNIX foundation (Darwin/XNU kernel).
- **Windows** — a separate lineage (NT kernel), different internals, but the *same concepts*: processes, threads, virtual memory, files, drivers. Learn the concepts once and you can map them to any OS.

---

# Part 2 — The Kernel and the Two Worlds

## 2.1 Kernel mode vs. user mode

The single most important idea in OS design: the CPU hardware itself supports (at least) two privilege levels.

- **Kernel mode** — all instructions allowed, all memory and hardware accessible. The core of the OS — the **kernel** — runs here.
- **User mode** — dangerous instructions forbidden, only your own memory accessible. *Everything else* runs here: your browser, your games, your code — collectively called **user space**.

If a user-mode program tries something privileged (touch hardware directly, access another program's memory), the CPU refuses and notifies the kernel, which typically kills the program. This is what's happening behind a "Segmentation fault" on Linux or an application crash dialog on Windows: the hardware caught an illegal access, and the kernel decided the program couldn't continue.

```
+--------------------------------------------------+
|  USER MODE (restricted)                          |
|  browser | editor | your program | shell | ...   |
+-----------------------+--------------------------+
                        | system calls (the ONLY doorway)
+-----------------------v--------------------------+
|  KERNEL MODE (all-powerful)                      |
|  scheduler | memory mgmt | file systems |        |
|  drivers | network stack | security checks       |
+-----------------------+--------------------------+
                        |
+-----------------------v--------------------------+
|  HARDWARE: CPU, RAM, disks, network, devices     |
+--------------------------------------------------+
```

This split is why one crashing app doesn't take down your machine, and why malware can't (without exploiting a bug) simply read your password manager's memory.

## 2.2 System calls: the doorway between worlds

A user program cannot open a file itself — the disk is privileged hardware. Instead it asks the kernel via a **system call** (syscall): a special CPU instruction that safely switches to kernel mode, runs a specific kernel function, and switches back with the result.

The core UNIX syscalls are famously few and composable:

| Syscall | Meaning |
|---|---|
| `open(path)` | get a handle (a **file descriptor** — just a small integer) to a file |
| `read(fd, buf, n)` / `write(fd, buf, n)` | move bytes in or out through that handle |
| `close(fd)` | release the handle |
| `fork()` | create a new process (a copy of the caller) |
| `exec(program)` | replace the current process's program with a new one |
| `wait()` | wait for a child process to finish |
| `exit(code)` | terminate |
| `mmap(...)` | map memory (files or anonymous RAM) into the process |
| `kill(pid, sig)` | send a signal to a process |

Everything your computer visibly does bottoms out in syscalls. When you save a document: the editor calls `write()`; the kernel checks permissions, updates its file structures, and queues the bytes for the disk driver. You can watch this live: `strace <command>` on Linux prints every syscall a program makes — one of the best learning tools that exists.

**Mode switches are expensive** (hundreds of nanoseconds of bookkeeping — recall the hierarchy table). This is why programs and language runtimes *buffer*: instead of one `write()` per character, they accumulate thousands of bytes and write once. When you see `printf` output appear "late," buffering is why.

## 2.3 What lives inside the kernel

The kernel's major subsystems — each gets its own part in this guide:

- **Process management & scheduler** — creates/destroys processes, decides who runs (Parts 3–4)
- **Memory manager** — virtual memory, paging (Part 5)
- **Virtual File System & file systems** — files, directories, caching (Part 7)
- **Device drivers** — per-device code, the bulk of kernel source (Part 8)
- **Network stack** — TCP/IP implementation (Part 13)
- **Security** — permissions, capabilities, isolation (Part 11)

**Monolithic vs. microkernel** (a classic design debate): Linux and Windows are *monolithic-ish* — all of the above runs in kernel mode, in one address space, for speed. A *microkernel* (e.g., seL4, MINIX, QNX) keeps the kernel tiny (scheduling + message passing) and runs drivers/file systems as user-mode processes — safer (a crashed driver doesn't kill the system) but historically slower due to extra message-passing. Real systems blend both: macOS's XNU is a hybrid; Windows moved graphics drivers in and partly back out of the kernel over the years.

---

# Part 3 — Processes: Programs in Motion

## 3.1 Program vs. process

- A **program** is a passive file on disk: instructions plus data (e.g., `chrome.exe`).
- A **process** is a *running instance* of a program: the code loaded in memory, plus everything needed to run it. Launch Chrome three times → one program, three processes.

A process is the kernel's **unit of isolation and ownership**. Each process gets:

- Its own private **virtual address space** (its memory — Part 5)
- One or more **threads** of execution (Part 4)
- A table of open **file descriptors** (files, sockets, pipes)
- An identity: **PID** (process ID), owning user, working directory, environment variables
- A parent — every process is created by another process, forming a tree

Run `ps aux` (Linux/macOS) or open Task Manager (Windows) to see the live process list. `pstree` shows the ancestry tree.

## 3.2 Anatomy of a process's memory

```
High addresses
+--------------------+
|       Stack        |  function-call frames, local variables
|         |          |  (grows downward)
|         v          |
|                    |
|   (unused gap)     |
|                    |
|         ^          |
|         |          |
|        Heap        |  dynamically allocated data: malloc/new,
+--------------------+  Python objects, JS objects (grows upward)
|   BSS + Data       |  global variables
+--------------------+
|       Text         |  the program's machine instructions (read-only)
+--------------------+
Low addresses
```

- **Stack**: every function call pushes a *frame* holding its local variables and the return address. Return pops it. Infinitely recursing → frames pile up until the stack's limit → **stack overflow**.
- **Heap**: long-lived and variable-size data. Managed by the program/runtime (`malloc`, garbage collectors) which in turn asks the kernel for large chunks via `mmap`/`brk`.
- **Text**: the instructions themselves, mapped read-only (so it can be shared between processes running the same program).

## 3.3 The process lifecycle

States a process moves through:

```
            created
               |
               v
   +-------> READY  <----------------+
   |           |                     |
   | (kernel picks it to run)        | (I/O completes /
   |           v                     |  event arrives)
   |        RUNNING                  |
   |           |                     |
   |           +----> BLOCKED -------+
   |           |      (waiting on disk, network,
   |           |       keyboard, sleep, lock...)
   +-----------+
   (time slice expired / preempted)
               |
               v
            ZOMBIE ----> gone (parent collects exit status via wait())
```

The key insight: at any moment, most processes on your machine are **blocked** — waiting for input, network, or timers. A machine with 400 processes may have only 2–3 actually runnable. This is what makes multitasking feasible at all.

A **zombie** is a process that has exited but whose parent hasn't yet called `wait()` to read its exit code; the kernel keeps a tiny record around until then. An **orphan** (parent died first) gets adopted by the init process (PID 1).

## 3.4 How processes are created: fork and exec

UNIX splits "make a new process" into two orthogonal operations:

- **`fork()`** clones the calling process. Afterward there are two nearly identical processes; the call returns the child's PID in the parent and `0` in the child (that's how each knows which one it is).
- **`exec()`** *replaces* the current process's program with a new one, keeping the same PID and (by default) open file descriptors.

When you type `ls` in a shell:

1. The shell `fork()`s itself.
2. The child `exec()`s `/bin/ls` — same process, new program.
3. The parent shell `wait()`s until the child exits, then prints the next prompt.

Why the split is brilliant: *between* fork and exec, the child (still running shell code) can rearrange its own world — redirect its output to a file, change directory, drop privileges — before the new program starts. This is exactly how shell redirection (`ls > out.txt`) and pipelines are implemented, with no cooperation needed from `ls` itself.

`fork()` copying all memory sounds expensive; it isn't, thanks to **copy-on-write**, explained in Part 5. (Windows takes the other design: a single `CreateProcess()` call with many parameters.)

## 3.5 Signals

**Signals** are minimal asynchronous messages the kernel or other processes can send to a process: "a number happened to you."

| Signal | Sent when | Default effect |
|---|---|---|
| `SIGINT` | you press Ctrl+C | terminate |
| `SIGSEGV` | invalid memory access | terminate + core dump |
| `SIGKILL` | `kill -9` | terminate — cannot be caught or ignored |
| `SIGTERM` | polite kill request | terminate (catchable — apps use it to clean up) |
| `SIGCHLD` | a child process exited | ignored by default |
| `SIGSTOP`/`SIGCONT` | Ctrl+Z / `fg` | pause / resume |

A process can register a *handler* function for most signals — that's how servers reload config on `SIGHUP` and shut down gracefully on `SIGTERM`. `SIGKILL` and `SIGSTOP` are unblockable by design so the OS always retains ultimate control.

---

# Part 4 — Threads and Scheduling

## 4.1 Threads: multiple workers, one house

A **thread** is an independent stream of execution *within* a process. Threads in a process share the address space, heap, open files — everything except:

- their own **stack** (each has its own call chain and locals)
- their own **registers / program counter** (each is at a different point in the code)

| | Processes | Threads |
|---|---|---|
| Memory | isolated | shared |
| Communication | explicit IPC (Part 9) | just read/write shared variables |
| Creation cost | heavier | lighter |
| One crashes | others unaffected | whole process dies |
| Safety | high (isolation) | low (data races — Part 6) |

Why threads exist:
1. **Parallelism** — a modern CPU has many cores; a single-threaded program can use only one. Threads let one program compute on all cores at once.
2. **Concurrency of waiting** — while one thread blocks on the network, others keep working. A word processor uses one thread for typing, another for spell-check, another for autosave.

Rule of thumb: threads share by default and must isolate carefully; processes isolate by default and must share carefully. This is why browsers moved to one *process* per tab — a compromised or crashed tab can't touch the others.

## 4.2 The scheduler: the illusion of simultaneity

You have perhaps 8 CPU cores and 500 threads. The **scheduler** is the kernel component that decides, for each core, at every moment: *which thread runs now?*

The mechanism enabling this is **preemption**: a hardware timer interrupts the CPU regularly (classically every few milliseconds). The interrupt hands control to the kernel, which may decide the current thread has had enough and switch to another. Switching fast enough creates the illusion that everything runs at once. Crucially, *programs cannot refuse* — an infinitely looping program cannot hang the machine, because the timer interrupt always wrenches control back. (Compare Part 0's interrupts: this is their most important use.)

## 4.3 Context switching

Swapping the running thread is a **context switch**:

1. Timer interrupt (or syscall, or the thread blocks) → kernel gains control.
2. Kernel saves the current thread's registers and program counter into its kernel data structure.
3. Scheduler picks the next thread.
4. Kernel restores that thread's saved registers and — if it belongs to a different process — switches the address-space mapping.
5. Return to user mode; the new thread resumes exactly mid-stride, unaware it was ever paused.

Direct cost: ~1–10 µs. The bigger, sneakier cost is *cold caches*: the new thread's data isn't in the CPU caches, so it runs slow for a while. This is why "just add more threads" often makes programs slower — a key expert intuition.

## 4.4 Scheduling policies

What does "pick the next thread" optimize? Competing goals: **throughput** (total work done), **latency/responsiveness** (how quickly an interactive task reacts), **fairness**, and meeting **deadlines** (real-time). You can't maximize all at once.

Classic algorithms (worth knowing as vocabulary and intuition):

- **FIFO / First-Come-First-Served** — run to completion in arrival order. Simple; terrible latency when a long job arrives first (the *convoy effect* — a supermarket line stuck behind one full cart).
- **Shortest Job First** — provably optimal average waiting time, but you don't know job lengths in advance; long jobs can starve.
- **Round-Robin** — everyone gets a fixed time slice, in rotation. Fair and responsive; more context-switch overhead.
- **Priority + Multi-Level Feedback Queue (MLFQ)** — multiple queues by priority; new/interactive tasks start high; tasks that burn full slices sink (they're CPU-bound batch work); tasks that block often (interactive — they wait on you) stay high. Learns behavior without being told. The conceptual basis of classic UNIX/Windows scheduling.
- **Linux CFS → EEVDF** — Linux's "Completely Fair Scheduler" tracked each thread's virtual runtime and always ran the one that had received the least, using a red-black tree to find it in O(log n); since kernel 6.6 the EEVDF scheduler refines this with latency-deadline awareness. "Priority" (the `nice` value) becomes a *weight* on how fast virtual runtime accrues.
- **Real-time policies** (`SCHED_FIFO`, deadline schedulers) — for systems where late = wrong (car brakes, audio). Strict priorities, preemption guarantees; a different world with its own theory (rate-monotonic, EDF).

Modern schedulers also handle **multi-core placement**: keeping a thread on the core whose caches still hold its data (*affinity*), balancing load across cores, and grouping siblings (hyperthreads, NUMA nodes — Part 14).

---

# Part 5 — Memory Management and Virtual Memory

Widely considered the most elegant machinery in any OS. Take this part slowly.

## 5.1 The problem

Many processes, one RAM. If programs used physical RAM addresses directly:

- Two programs both built to live at address 1000 couldn't run together.
- Any program could read/corrupt any other's memory — no security, no stability.
- Memory would fragment into unusable holes.
- You couldn't run programs needing more memory than physically installed.

## 5.2 The idea: give every process a fake address space

**Virtual memory**: every memory address a process uses is *virtual* — a fiction. The CPU's **MMU** (memory management unit) translates every single access, on the fly, to a *physical* address, using per-process translation tables that the kernel maintains.

- Every process sees the same tidy private address space starting at 0 — the layout from §3.2.
- Two processes' identical virtual addresses map to *different* physical RAM. Neither can even *name* the other's memory — isolation isn't a permission check, it's an inability to express the address. This is the mechanism behind the user-space isolation promised in Part 2.

## 5.3 Paging

Memory is divided into fixed-size chunks called **pages** — 4 KB is standard. Virtual pages map to physical page-sized slots ("frames") via a **page table** per process:

```
Process A's page table:                  Physical RAM:
  virtual page 0  -> frame 8             +----------+
  virtual page 1  -> frame 3             | frame 0  |
  virtual page 2  -> (not present)       | frame 1  |
  ...                                    | frame 2  |
                                         | frame 3  | <- A's page 1
Process B's page table:                  |   ...    |
  virtual page 0  -> frame 12            | frame 8  | <- A's page 0
  virtual page 1  -> frame 8  (shared!)  |   ...    |
```

Each page table entry also carries permission bits: readable? writable? executable? user-accessible? Violate them → the CPU **faults** to the kernel → typically `SIGSEGV`. (Now you know precisely what a segfault is: the MMU caught an access to an unmapped or forbidden page.)

Fixed-size pages solve fragmentation: any free frame satisfies any page, so RAM never develops unusable odd-shaped holes (no *external* fragmentation).

Real page tables are multi-level trees (4–5 levels on x86-64) so that the vast empty regions of the address space cost no table memory. Since a naïve design would need memory lookups *per level per access*, the CPU caches recent translations in the **TLB** (translation lookaside buffer). TLB misses are a real performance factor — which is why databases and JVMs use **huge pages** (2 MB/1 GB) to cover more memory per TLB entry.

## 5.4 Page faults: laziness as a superpower

A page table entry marked "not present" makes any access trigger a **page fault** — the MMU stops the instruction and calls the kernel. The genius part: the kernel can *fix the situation and restart the instruction*, and the program never knows. This one mechanism enables:

- **Demand paging / lazy loading** — launching a program maps its file but loads *nothing*. Pages of code load from disk on first touch, one fault at a time. Programs start fast; never-used code never loads.
- **Copy-on-write (COW)** — `fork()` copies no memory: parent and child share all frames, marked read-only. Only when either *writes* a page does the fault handler copy that one page. Fork becomes nearly free. The same trick backs efficient snapshots everywhere (Redis persistence, VM snapshots, file-system snapshots).
- **Swapping** — under RAM pressure, the kernel evicts rarely-used pages to disk ("swap space"), marking them not-present. Touch one later → fault → kernel reads it back in. You can run more than your RAM holds — but recall the hierarchy: disk is ~1000× slower, and a system doing heavy swapping "thrashes" (all time spent moving pages, no work done). Eviction policy is approximately-LRU (least recently used), tracked via page-table "accessed" bits.
- **Memory-mapped files** (`mmap`) — map a file into the address space; bytes load on fault, dirty pages write back. The file becomes "just memory." Loading shared libraries works this way, which also lets *one* physical copy of libc appear in every process.
- **Zero-fill on demand** — `malloc`ing a gigabyte maps nothing; each page materializes (zeroed) on first touch. This is why allocation is instant but first-touch can be slow, and why "memory used" has multiple answers (virtual size vs. resident set size, RSS).

If one idea in this guide deserves the word *beautiful*, it's this: a hardware error-recovery mechanism repurposed as the foundation of laziness, sharing, and overcommitment.

## 5.5 What "out of memory" really means

Because of overcommit (promising more virtual memory than exists), the reckoning comes at page-fault time, not malloc time. When RAM and swap are truly exhausted, Linux's **OOM killer** picks a process (heuristically, the biggest/least important) and kills it. If a big process on a server died mysteriously, `dmesg | grep -i oom` is the first thing an expert checks.

---

# Part 6 — Concurrency and Synchronization

Threads sharing memory (Part 4) plus preemption at arbitrary moments (also Part 4) equals the hardest class of bugs in computing. This part is essential for *anyone* who writes multithreaded code, not just OS developers.

## 6.1 The race condition

Two threads both run `counter = counter + 1` (which compiles to three instructions: load, add, store):

```
Thread A: load counter (100)
                                Thread B: load counter (100)   <- preempted here!
Thread A: add 1 -> 101
Thread A: store 101
                                Thread B: add 1 -> 101
                                Thread B: store 101            <- A's increment is LOST
```

Final value: 101, not 102. This is a **race condition**: correctness depends on the accident of interleaving. Races are hideous because they're intermittent — the code is *usually* right, fails one run in a million, and disappears when you attach a debugger (timing changes). Real-world races have caused radiation-therapy deaths (Therac-25) and major blackouts (Northeast 2003).

The code region touching shared state is a **critical section**; we need **mutual exclusion**: at most one thread inside at a time.

## 6.2 The toolbox

- **Mutex (lock)** — the workhorse. `lock(m)` blocks if another thread holds it; `unlock(m)` releases. Wrap every access to the shared data in the same lock and races vanish. Implemented with special atomic CPU instructions (compare-and-swap, test-and-set) — ordinary loads/stores can't build a correct lock on modern hardware.
- **Spinlock** — a mutex that busy-waits (loops) instead of sleeping. Wasteful in general; right when waits are tiny or sleeping is impossible (inside kernel interrupt handlers).
- **Condition variable** — for *waiting until something is true* ("queue non-empty"). `wait(cv, m)` atomically releases the mutex and sleeps; `signal(cv)` wakes a waiter. Always re-check the condition in a `while` loop after waking (spurious wakeups, stolen wakeups). Mutex + condition variable = the classic solution to **producer/consumer** (a bounded queue between threads — the fundamental pattern of pipelines, thread pools, and message queues).
- **Semaphore** — a counter: `wait` decrements (blocking at 0), `post` increments. Generalizes both locking (init 1) and resource counting (init N: "at most N concurrent downloads").
- **Reader-writer lock** — many concurrent readers *or* one writer; for read-mostly data.
- **Atomics** — single operations (increment, compare-and-swap) the hardware guarantees indivisible; the basis of *lock-free* data structures (expert territory: fast, fiendishly subtle, complicated further by *memory ordering* — CPUs and compilers reorder memory operations unless told not to).

## 6.3 Deadlock

Locks cure races but introduce their own disease:

```
Thread A: lock(L1); ...wants L2...
Thread B: lock(L2); ...wants L1...
        -> both wait forever
```

**Deadlock** requires four conditions simultaneously (Coffman): mutual exclusion, hold-and-wait, no preemption of locks, and a circular wait. Break any one and deadlock is impossible. The standard practical cure: **a global lock ordering** — if every thread that needs both L1 and L2 always takes L1 first, no cycle can form. Databases instead *detect* cycles and kill a victim transaction ("deadlock detected — transaction rolled back" — now you know what that error means).

Related pathologies: **livelock** (threads actively yielding to each other forever — two people side-stepping in a hallway), **starvation** (some thread never gets its turn), and **priority inversion** — a low-priority thread holding a lock a high-priority thread needs, while medium-priority threads keep the low one from running. This famously froze the Mars Pathfinder lander; the fix, *priority inheritance* (the lock holder temporarily borrows the waiter's priority), was patched to the spacecraft from Earth.

## 6.4 Practical wisdom

- Prefer higher-level structures (queues, channels, thread pools) to raw locks; prefer message passing ("don't communicate by sharing memory; share memory by communicating" — Go's motto).
- Keep critical sections tiny; never do I/O while holding a lock.
- Coarse locking (one big lock) is safe but serializes everything (Python's GIL; early Linux's "Big Kernel Lock" — since removed). Fine-grained locking scales but multiplies deadlock risk. This tension drives much real systems engineering.
- Immutable data needs no locks. The fewer things mutate, the fewer things race.

---

# Part 7 — File Systems and Storage

## 7.1 What a disk actually offers

Storage hardware is just a giant array of numbered fixed-size **blocks** (512 B or 4 KB): "read block #82014," "write block #82015." No names, no folders, no notion of a "file." The **file system** is the kernel subsystem that builds those from raw blocks — it's a *data structure persisted on disk*, plus code to maintain it.

## 7.2 Files, directories, inodes

The UNIX model:

- A **file** is an unstructured sequence of bytes. The OS imposes no format — meaning lives in applications.
- Each file is described by an **inode**: a fixed-size record holding the metadata (size, owner, permissions, timestamps) and — crucially — *pointers to the data blocks*. The inode is identified by number, and **does not contain the file's name**.
- A **directory** is itself just a file whose contents are a list of `(name → inode number)` entries.

Resolving `/home/ana/notes.txt`: start at the root inode → read its data (a directory listing) → find `home` → read that inode → find `ana` → find `notes.txt` → inode 8817 → its block pointers give you the data. Because names live *outside* inodes:

- **Hard links**: two directory entries can point to the same inode — one file, two equally-real names. The inode keeps a link count; data is freed only when it hits zero *and* no process holds the file open. (This is why on UNIX you can delete a file a program is still using — the name disappears but the data survives until closed. "Delete" is really `unlink`.)
- **Symbolic links**: a tiny file containing a *path* — a redirect by name, which can dangle if the target moves.

Classic inodes point to blocks via direct pointers plus (for big files) indirect blocks — pointers to blocks of pointers, a small tree. Modern file systems use **extents** (start + length runs) instead, which are far more compact for large contiguous files.

## 7.3 Crash consistency: the deep problem

Operations like "create a file" touch multiple disk blocks (inode table, directory block, free-space bitmap). Power dies between writes → the structure is *inconsistent* (e.g., a block marked used that no file references; worse, a block referenced by two files). Solutions, in historical order:

1. **fsck** — after a crash, scan the *entire disk* reconciling everything. Correct, but hours on large disks.
2. **Journaling** (ext4, NTFS, XFS) — the standard today. Before modifying structures in place, write a description of the whole change to a **journal** (an on-disk log), then apply it. On crash: replay complete journal entries, discard incomplete ones. Every multi-block change becomes all-or-nothing. Trade-off dial: journal metadata only (fast, default) or data too (safer, slower).
3. **Copy-on-write file systems** (ZFS, Btrfs, APFS) — never overwrite live data; write new versions of blocks elsewhere, then flip pointers atomically at the root. Old roots = free **snapshots**. ZFS adds checksums on everything, catching silent disk corruption.
4. **Log-structured designs** — make the *entire disk* an append-only log (great for write performance and for SSDs); this idea also underlies the LSM-trees in modern databases (RocksDB, Cassandra) and SSD internals themselves.

Related trap every developer should know: `write()` returning does **not** mean data is on disk — it's in the kernel's page cache (below). Durability requires `fsync(fd)`, which is *slow* (real disk round-trip). Databases obsess over fsync correctness; famously, PostgreSQL had to rework fsync-error handling ("fsyncgate") when Linux's error semantics surprised everyone.

## 7.4 The page cache

RAM not used by programs caches recently-read disk blocks. Reads hit RAM when possible; writes go to RAM immediately ("dirty" pages) and flush to disk seconds later. This is why:

- The second launch of an app is dramatically faster than the first (code already cached).
- "Free RAM" looks low on healthy Linux boxes — unused RAM is wasted RAM, so the kernel fills it with cache (`free -h` separates "used" from "buff/cache"; the cache is instantly reclaimable).
- You must "safely eject" USB drives: unflushed dirty pages.

## 7.5 The VFS layer

Linux supports dozens of file systems (ext4, XFS, Btrfs, FAT, NTFS, network file systems like NFS...). The **Virtual File System (VFS)** is the kernel's interface layer: every file system implements the same operations (open, read, write, lookup...), so applications and the rest of the kernel are indifferent to which one backs any given path. "Mounting" grafts one file system's tree onto a directory of another, forming the single unified tree UNIX presents. The same interface uniformity explains "everything is a file": devices (`/dev/...`), kernel state (`/proc`, `/sys`), pipes, and sockets all speak read/write through file descriptors — one small toolset (including redirection and pipes) works on all of them.

---

# Part 8 — Input/Output and Device Drivers

## 8.1 Talking to devices

The CPU communicates with devices through device registers, typically via **memory-mapped I/O**: designated physical addresses that aren't RAM but wires to the device. Write to the "command register" address → the device acts.

Three strategies for moving data, in increasing sophistication:

1. **Polling** — CPU repeatedly asks "done yet?" Simple; burns CPU. Right for ultra-fast devices where the answer is "almost immediately."
2. **Interrupts** — device raises an interrupt when done (Part 0); CPU does other work meanwhile. Right for slow/sporadic devices (keyboards). But per-event interrupt overhead swamps very fast devices — a 10 Gbps NIC could interrupt millions of times a second; so high-speed drivers use hybrid schemes (Linux NAPI: interrupt once, then poll while traffic is heavy).
3. **DMA (Direct Memory Access)** — the CPU shouldn't hand-copy every byte. It tells the device (or a DMA engine) "transfer these 4 MB to RAM at address X; interrupt me when finished," and goes off to run other threads. All serious disk/network transfer is DMA.

## 8.2 Drivers

A **driver** is kernel code that speaks one device family's specific register language and presents the kernel's standard interface for that device *class* (block device, network device, character device...). Drivers are the majority of the Linux kernel by volume, and — since they run in kernel mode — the majority of crashes: a buggy driver can corrupt anything (Windows blue screens are overwhelmingly driver bugs; the 2024 CrowdStrike global outage was kernel-mode code faulting at boot).

The block layer between file systems and disk drivers also does **I/O scheduling**: merging adjacent requests and ordering them (for spinning disks, order matters enormously — seeks cost ~10 ms; for SSDs, much less, so modern kernels use simpler schedulers like mq-deadline/none, tuned instead for the massive internal parallelism NVMe exposes via multiple queues).

For the highest performance tiers, modern systems increasingly *bypass* pieces of this stack: `io_uring` (Linux's async I/O interface — submit and complete I/O via shared-memory rings with minimal syscalls), and kernel-bypass networking (DPDK) where user space drives the NIC directly.

---

# Part 9 — Inter-Process Communication

Processes are isolated (that's the point) — so the OS must supply sanctioned channels for cooperation:

- **Pipes** — a one-way byte stream between related processes; the mechanism behind the shell's `|`. `cat log | grep error | wc -l` runs three concurrent processes with the kernel moving bytes between them, blocking each when its neighbor is full/empty — automatic flow control. The pipeline idea (small tools composed by streams) is UNIX's most influential design legacy.
- **Named pipes (FIFOs)** — pipes with a filesystem name, so unrelated processes can find them.
- **UNIX domain sockets** — bidirectional, socket API, local-only; the standard for local services (Docker daemon, X11, systemd). Can even pass file descriptors between processes.
- **Network sockets (TCP/UDP)** — same API, works across machines (Part 13).
- **Shared memory** (`mmap`/`shm`) — two processes map the same physical pages; the *fastest* IPC (zero copies, no syscalls per message) but now you've reintroduced shared-state concurrency, so you need synchronization (Part 6) across processes.
- **Message queues, signals, eventfd** — kernel-mediated messages and event notifications.

And the mechanism used by every server that handles many clients at once: **I/O multiplexing** — `select`/`poll`/**`epoll`** (Linux) / `kqueue` (BSD/macOS) let one thread say "sleep until any of these 10,000 sockets has data." This is the engine inside nginx, Redis, and Node.js: one thread, an event loop over epoll, no thread-per-client — the answer to the "C10K problem." (Windows' equivalent is I/O completion ports.)

---

# Part 10 — The Boot Process

What happens between the power button and the login screen — a chain of programs each loading the next:

1. **Firmware (UEFI; historically BIOS)** — burned into the motherboard. Initializes hardware, runs self-tests, finds a boot disk, loads the **bootloader** from a special partition. (Secure Boot: the firmware verifies the bootloader's cryptographic signature — the root of a trust chain.)
2. **Bootloader (GRUB, Windows Boot Manager)** — knows just enough to read file systems, load the **kernel** image (plus an initial mini-filesystem, initramfs, holding the drivers needed to see the real disk) into RAM, and jump into it.
3. **Kernel initialization** — detect CPUs and memory, build page tables, start the scheduler, initialize drivers, mount the root file system. Then create the *first user-space process*: **PID 1**.
4. **Init system (systemd on modern Linux; launchd on macOS)** — PID 1, ancestor of every other process. Starts services (networking, display manager, daemons) per dependency graph, supervises and restarts them, adopts orphans.
5. **Login → shell/desktop** — just more user-space processes down the tree.

`systemctl status <svc>` and `journalctl -u <svc>` (service state and logs) are the everyday tools at layer 4 on Linux servers.

---

# Part 11 — Security and Protection

The OS is the enforcement point for all security on the machine. Layers, bottom-up:

1. **Hardware foundations** — user/kernel mode and virtual memory (Parts 2, 5) provide the base isolation everything relies on.
2. **Users and permissions** — every process runs *as* a user (UID); every file has an owner, group, and permission bits (`rwx` for owner/group/others — that's what `chmod 644` sets: rw- r-- r--). Every `open()` is checked. **root** (UID 0) bypasses these checks — hence the danger and the discipline: run as unprivileged users; `sudo` grants temporary, logged elevation. Windows analog: SIDs + ACLs (richer ACLs are also available on Linux).
3. **Least privilege mechanisms** — beyond all-or-nothing root: **capabilities** split root's power into grantable slices (e.g., `CAP_NET_BIND_SERVICE` to bind port 80 without full root); **seccomp** lets a process irreversibly drop the syscalls it will never need (browsers sandbox their tab processes this way: a compromised renderer can't even *ask* the kernel to open files); **setuid** binaries run with the file owner's identity (how `passwd` edits a root-owned file — historically a rich source of vulnerabilities).
4. **Mandatory access control** — SELinux/AppArmor: system-wide policy that constrains even root ("the web server may only read `/var/www`, regardless of file permissions").
5. **Exploit mitigations** — the arms race against memory-corruption bugs: **ASLR** (randomize where stack/heap/libraries land so attackers can't aim), **NX/DEP** (pages are writable *or* executable, never both — injected data can't be run; enforced by page-table bits from Part 5), stack canaries, and control-flow integrity. None is airtight; together they turn easy exploits into research projects.
6. **Sandboxing and isolation** — containers (next part) and VMs as coarse-grained boundaries.

Even hardware betrays the OS sometimes: **Meltdown/Spectre** (2018) leaked kernel memory via CPU speculative-execution side effects, forcing kernels to adopt costly countermeasures (KPTI — fully separate kernel page tables). A humbling lesson: the isolation guarantees of Part 5 are only as good as the silicon.

---

# Part 12 — Virtualization and Containers

Two different answers to "run isolated workloads on shared hardware" — understanding the difference is mandatory in the cloud era.

## 12.1 Virtual machines

A **hypervisor** (KVM, Hyper-V, Xen, VMware) applies the OS's own trick one level down: it virtualizes the *hardware*, so entire operating systems run as guests, each believing it owns a machine. Modern CPUs assist directly (Intel VT-x/AMD-V add a mode below the kernel; nested page tables translate guest-physical → host-physical addresses in hardware). Guest kernels run their own schedulers and memory managers inside their allotment; the hypervisor schedules virtual CPUs onto real cores. Every cloud instance you rent is this.

- Strong isolation (separate kernels; the hypervisor's attack surface is small)
- Run any OS on any host
- Cost: each VM carries a whole OS — GBs of RAM, minutes to boot (mitigated by lightweight microVMs — AWS Firecracker boots in ~100 ms)

## 12.2 Containers

A **container** is *not* a smaller VM. There is **one kernel**; containers are ordinary processes wearing blinders, built from three Linux kernel features:

1. **Namespaces** — per-process-group *views* of kernel resources: PID namespace (your processes think they're PIDs 1, 2, 3...), mount namespace (own filesystem tree), network namespace (own interfaces/ports), plus UTS, IPC, user namespaces. You see only your slice.
2. **cgroups** — resource *limits and accounting* per group: at most 2 CPUs' worth, 4 GB RAM (exceed it → the OOM killer visits your cgroup), I/O bandwidth caps.
3. **Layered images** (overlay file systems) + seccomp/capability drops for the security perimeter.

Docker/Kubernetes are orchestration around these primitives. Consequences that follow directly from "shared kernel": containers start in milliseconds and cost ~zero overhead; a Linux container can't run on a Windows kernel (Docker Desktop runs a hidden Linux VM); and container escape only requires one kernel bug, whereas VM escape requires beating the hypervisor — so multi-tenant clouds put VMs (or microVMs) under untrusted containers.

*(Deep dive: see [docker-deep-dive](../infra/docker-deep-dive.md) in these notes.)*

---

# Part 13 — Networking from the OS's Point of View

Full networking is its own field; here is the OS-resident core.

## 13.1 The stack lives in the kernel

The kernel implements the network protocol layers:

```
  Application (user space): your code, HTTP libraries
  ------------------- socket API boundary -------------------
  Transport (kernel):  TCP — reliable ordered byte stream:
                        acknowledgments, retransmission, ordering,
                        flow & congestion control
                       UDP — bare packets, no promises
  Network  (kernel):   IP — addressing & routing between machines
  Link     (kernel+NIC): Ethernet/Wi-Fi — local delivery
```

Your program speaks **sockets**; the kernel turns stream bytes into packets, handles loss and reordering, and drives the NIC (via DMA and interrupts — Part 8).

## 13.2 The socket API

The universal vocabulary (server side): `socket()` → `bind()` (claim port, e.g., 443) → `listen()` → `accept()` (block until a client connects; returns a *new* fd per connection) → `read`/`write` → `close`. Client side: `socket()` → `connect(ip, port)` → same read/write. A **port** is just a 16-bit number letting one machine host many conversations; ports below 1024 need privilege (or `CAP_NET_BIND_SERVICE`).

Because connections are file descriptors, everything from Part 9 applies: epoll-based event loops multiplex tens of thousands of them per thread. The kernel maintains per-socket send/receive buffers — `write()` means "kernel accepted the bytes," not "the peer got them" (an exact echo of the `write`≠`fsync` lesson from Part 7).

Useful reflexes: `ss -tlnp` (what's listening on which port, Linux), `netstat -ano` (Windows), `tcpdump`/Wireshark to watch actual packets — seeing a TCP handshake once teaches more than a chapter of prose.

---

# Part 14 — Performance: How Experts Think

The distilled mental habits this guide has been building:

1. **Know the numbers** (§0.3). Every performance mystery starts with "which level of the hierarchy are we hitting?" A program is **CPU-bound** (fix: better algorithm, more cores) or **I/O-bound** (fix: caching, batching, concurrency of waiting, faster storage) — the treatment differs completely, so diagnose first: is the CPU pegged or idle-waiting?
2. **Batch to amortize fixed costs.** Syscalls, disk seeks, network round trips, interrupts all carry per-event overhead. One 1 MB write beats a thousand 1 KB writes; hence buffering everywhere (§2.2), NAPI (§8.1), io_uring's batched submission.
3. **Cache — and know your caches.** CPU caches, TLB, page cache, application caches. Repeated cost → store the answer closer. Corollary: cache *invalidation* and cold-start effects (context switches thrash CPU caches — §4.3).
4. **Locality is speed.** Sequential beats random at every level (prefetchers, disk seeks, cache lines). Data-structure layout (arrays vs. pointer-chasing) can matter more than algorithmic constants. On multi-socket servers, **NUMA** adds "which RAM is near which CPU" to the game.
5. **Contention serializes.** Amdahl's law: the serial fraction bounds all speedup. One hot lock, one shared counter (cache-line ping-pong, false sharing), one single-threaded stage — and 64 cores perform like 3. Shard, partition, use per-CPU data (the kernel does, extensively).
6. **Measure; don't guess.** The tools, roughly in escalation order: `top`/`htop` (what's consuming CPU/RAM), `free`, `vmstat` (swapping? — §5.4), `iostat` (disk saturated?), `strace` (which syscalls — §2.2), `perf` (where CPU time *actually* goes, to the function; flame graphs visualize it), `/proc/<pid>/` (ground truth on any process), `dmesg` (kernel complaints, OOM kills). On Windows: Task Manager → Resource Monitor → Process Explorer/ETW. Brendan Gregg's *Systems Performance* is the canon here.

---

# Part 15 — The Road to Expertise

## 15.1 The layered map (what depends on what)

```
  applications, databases, browsers
        |
  runtimes & libraries (libc, JVM, Python, Node)      <- user space
  ----------------- syscall boundary ------------------
  VFS/file systems | net stack | IPC                   <- kernel services
        |               |
  scheduler | virtual memory | drivers                 <- kernel core
  ----------------------------------------------------
  CPU modes | MMU/paging | interrupts | DMA            <- hardware mechanisms
```

Everything above rests on the four hardware mechanisms at the bottom — which is why this guide introduced them first.

## 15.2 Books, in reading order

1. **Operating Systems: Three Easy Pieces (OSTEP)** — Arpaci-Dusseau. *Free online* (ostep.org). The best-written OS book in existence; virtualization/concurrency/persistence structure similar to this guide's, at 10× depth with homework simulators. **If you read one book, read this.**
2. **The Linux Programming Interface** — Kerrisk. The syscall layer in encyclopedic, superbly clear depth. The book for going from "understands concepts" to "writes real systems code."
3. **Operating System Concepts** (Silberschatz, the "dinosaur book") or **Modern Operating Systems** (Tanenbaum) — broader academic references.
4. **Systems Performance** — Gregg. Performance methodology and tooling.
5. **Understanding the Linux Kernel** / **Linux Kernel Development** (Love) + reading actual kernel source — expert tier.
6. Adjacent: **Computer Systems: A Programmer's Perspective** (CS:APP) for the hardware/software boundary; **Designing Data-Intensive Applications** for where these ideas go next (see also the [distributed-systems notes](../distributed-systems/) here).

## 15.3 Doing, not just reading (the only way it sticks)

Roughly in order of ambition:

1. **Live on the command line.** Daily driver skills: `ps`, `top`, `kill`, pipes and redirection, `/proc` spelunking. Watch programs with `strace` and `ltrace`; explain every syscall you see.
2. **Learn C.** The OS's native language; pointers *are* virtual addresses, and after Part 5 they will make sense. K&R or CS:APP as guides.
3. **Write the classics.** A shell (fork/exec/wait/pipes — cements Part 3 forever). A multithreaded producer/consumer with mutex + condvar (Part 6). A tiny HTTP server with epoll (Parts 9, 13). Use `mmap` to parse a big file (Part 5).
4. **Do OSTEP's projects / a course.** MIT 6.1810 (xv6 — a readable teaching UNIX you modify: add a syscall, change the scheduler) is the gold standard and freely available.
5. **Explore a real kernel.** Read Linux source for one subsystem you now know conceptually (the scheduler, or a simple filesystem); follow LWN.net to watch OS development happen live.
6. **Ultimate boss fight:** write a toy kernel that boots (the OSDev wiki community exists for exactly this). Even reaching "prints text and handles the timer interrupt" will teach you more than any book.

## 15.4 The ten ideas to retain if you forget everything else

1. The OS's two jobs: **abstract** the hardware, **manage** its sharing.
2. **User/kernel mode** + syscalls = all protection and all services.
3. A **process** = isolation unit; a **thread** = execution unit.
4. **Preemption via timer interrupt** = nobody can hog the machine.
5. **Virtual memory** = per-process fake address spaces, translated per-access by the MMU.
6. **Page faults are a feature**: lazy loading, copy-on-write, swap, mmap.
7. **Races and deadlocks** come from shared mutable state + arbitrary interleaving; locks and ordering discipline are the cure.
8. **File systems are persistent data structures**; crash consistency (journaling/COW) is their hard problem; `write` ≠ durable.
9. **Everything is a file descriptor** — files, pipes, sockets, devices — so one small toolset composes across all of them.
10. **The memory hierarchy explains almost all performance** — cache, batch, stay local, measure.

---

*Written 2026-08-23. Companion notes: [docker-deep-dive](../infra/docker-deep-dive.md), [distributed-systems](../distributed-systems/), [datastructures](../datastructures/probablic-datastructures/).*
