# What Happens When You Run a Program

A complete, layer-by-layer trace of a single command — from the moment you press Enter in a shell to the moment the last transistor stops toggling on your behalf. The running example is a Linux/x86-64 machine executing a small C program, but every stage has a direct equivalent on macOS, Windows, and ARM; the differences are named where they matter.

The document is organised as a **descent and return**: the request moves *down* through the shell, the kernel, the loader, the CPU, the caches, and the silicon, and the result comes back *up* through the same layers. Each section says what the layer receives, what it does, and what it hands to the layer below. Read it once end-to-end, then use the table of contents as an index.

Companion notes: [Computer Architecture](../computer-architecture/computer-architecture.md) (the hardware map this walkthrough moves through), [How Computers Work](../computer-science/01-how-computers-work.md) (gates → CPU), [Operating Systems](../computer-science/operating-systems.md), [Compilers and Languages](../computer-science/08-compilers-and-languages.md), [ext4 Internals](../operating-system/filesystem/ext4-filesystem-internals.md).

---

## Table of Contents

0. [The Whole Journey on One Page](#0--the-whole-journey-on-one-page)
1. [Before You Run It: What a Program File Is](#1--before-you-run-it-what-a-program-file-is)
2. [Stage 1 — The Keystroke Reaches the Shell](#2--stage-1--the-keystroke-reaches-the-shell)
3. [Stage 2 — The Shell Finds and Forks](#3--stage-2--the-shell-finds-and-forks)
4. [Stage 3 — `execve`: The Kernel Replaces the Process Image](#4--stage-3--execve-the-kernel-replaces-the-process-image)
5. [Stage 4 — The Dynamic Linker Finishes the Job in User Space](#5--stage-4--the-dynamic-linker-finishes-the-job-in-user-space)
6. [Stage 5 — The Scheduler Puts It on a CPU](#6--stage-5--the-scheduler-puts-it-on-a-cpu)
7. [Stage 6 — The First Instruction: Virtual Memory Wakes Up](#7--stage-6--the-first-instruction-virtual-memory-wakes-up)
8. [Stage 7 — Inside the Core: What Executing an Instruction Means](#8--stage-7--inside-the-core-what-executing-an-instruction-means)
9. [Stage 8 — Below the Core: Caches, DRAM, and Electrons](#9--stage-8--below-the-core-caches-dram-and-electrons)
10. [Stage 9 — The Program Asks for Something: System Calls](#10--stage-9--the-program-asks-for-something-system-calls)
11. [Stage 10 — Output Reaches Your Eyes](#11--stage-10--output-reaches-your-eyes)
12. [Stage 11 — Interruptions: Timers, Faults, and Sharing the CPU](#12--stage-11--interruptions-timers-faults-and-sharing-the-cpu)
13. [Stage 12 — Exit: Unwinding Everything](#13--stage-12--exit-unwinding-everything)
14. [Variations: Interpreted, JIT-Compiled, and Managed Programs](#14--variations-interpreted-jit-compiled-and-managed-programs)
15. [Variations: Windows, macOS, Containers, and VMs](#15--variations-windows-macos-containers-and-vms)
16. [The Numbers: Where the Time Actually Goes](#16--the-numbers-where-the-time-actually-goes)
17. [Observing Every Stage Yourself](#17--observing-every-stage-yourself)
18. [Mental Models to Keep](#18--mental-models-to-keep)
19. [Further Reading](#19--further-reading)

---

# 0 — The Whole Journey on One Page

The running example throughout:

```c
// hello.c
#include <stdio.h>
int main(void) {
    printf("Hello, world\n");
    return 0;
}
```

```sh
$ gcc -O2 -o hello hello.c
$ ./hello
Hello, world
```

Between the `./hello` keystroke and the text appearing, roughly this happens, top to bottom:

```text
 YOU        press Enter
  │
  ▼
 TERMINAL   the terminal emulator sends "./hello\n" through a pseudo-terminal (pty) to the shell
  │
  ▼
 SHELL      bash parses the line, decides it is an external command, resolves the path,
            fork()s a child, and the child calls execve("./hello", argv, envp)
  │
  ▼
 KERNEL     execve: opens the file, reads the ELF header, tears down the old address space,
            builds a new one (mmap of text/data segments, stack, auxv), maps the dynamic
            linker ld-linux.so, sets the instruction pointer to ld.so's entry, returns to user mode
  │
  ▼
 LOADER     ld.so (in user space) finds libc.so.6, maps it, relocates symbols, runs
 (ld.so)    initialisers, then jumps to hello's _start → __libc_start_main → main
  │
  ▼
 SCHEDULER  somewhere in here the kernel's CFS/EEVDF scheduler chose a core for the new
            task and context-switched onto it
  │
  ▼
 CPU        fetches the first instruction at the virtual address in RIP → TLB miss → page walk
            → page not present → PAGE FAULT → kernel reads the page from the page cache (or disk)
            → maps it → retries. Repeat for each newly touched page.
  │
  ▼
 CORE       fetch → decode → rename → dispatch → execute → retire, hundreds of instructions
            in flight, branch prediction guessing the path, out-of-order engine hiding latency
  │
  ▼
 CACHES     each load/store checks L1 (~1 ns) → L2 (~4 ns) → L3 (~12 ns) → DRAM (~80 ns),
 & DRAM     with the coherence protocol keeping other cores' copies consistent
  │
  ▼
 SILICON    every cycle, billions of transistors switch; a clock edge latches register values;
            a DRAM read senses a few femtocoulombs of charge on a capacitor
  │
  ▼  (the program calls printf → write(1, "Hello, world\n", 13))
 KERNEL     SYSCALL trap → sys_write → tty layer → pty master → terminal emulator wakes up
  │
  ▼
 TERMINAL   draws glyphs into a framebuffer → GPU → display controller → pixels
  │
  ▼  (main returns → exit_group(0))
 KERNEL     tears down the address space, closes files, notifies the parent (bash) via SIGCHLD,
            bash's wait4() returns, bash prints the next prompt
  │
  ▼
 YOU        see "Hello, world" and a new prompt, ~1 ms after pressing Enter
```

Two facts organise everything that follows:

1. **The CPU never "runs a program".** It executes one instruction at a time from wherever its instruction pointer points, in whichever privilege mode it is in. A "program" is a *convention* maintained by the kernel: a set of page tables, a register save area, some file descriptors, and a scheduling entity. Run means: the kernel has arranged for the instruction pointer to end up inside your bytes, in user mode, with your page tables loaded.
2. **Every boundary crossing is expensive and deliberate.** User → kernel costs ~100 ns and a trap instruction; kernel → user costs a privileged return; CPU → memory costs ~80 ns; memory → disk costs ~10–100 µs. The design of every layer is about *avoiding* the crossing below it: caches avoid DRAM, the page cache avoids disk, the vDSO avoids syscalls, the shell's hash table avoids `stat` calls on `$PATH`.

---

# 1 — Before You Run It: What a Program File Is

You cannot understand execution without knowing what the kernel is handed. `./hello` is an **ELF** (Executable and Linkable Format) file. The equivalents are **PE/COFF** on Windows and **Mach-O** on macOS; the structure is the same in spirit.

## 1.1 From source to executable

```text
hello.c ──preprocess──▶ hello.i ──compile──▶ hello.s ──assemble──▶ hello.o ──link──▶ hello
        (cpp: expand           (cc1: parse,          (as: text →           (ld: merge sections,
         #include, macros)      optimise, emit asm)   machine code,         resolve symbols,
                                                       relocations)          emit segments & PLT/GOT)
```

- The **compiler** turns `printf("Hello, world\n")` into a call to an *undefined symbol* `printf` (or, with `-O2`, into `puts` — the compiler notices the format string has no conversions). `hello.o` records "there is a `call` here whose target I don't know; patch it at link time" — a **relocation**.
- The **linker** combines `hello.o` with `crt1.o`, `crti.o`, `crtbegin.o` (C runtime startup: the `_start` symbol), and `crtend.o`/`crtn.o`. It does *not* copy `puts` into the binary; by default it records a **dynamic dependency** on `libc.so.6` and emits a **PLT** (Procedure Linkage Table) stub plus a **GOT** (Global Offset Table) slot so the address can be filled in at load time.
- Modern toolchains emit **PIE** (Position-Independent Executable) by default: the binary can be loaded at any base address, so the kernel randomises it (ASLR).

## 1.2 Anatomy of the ELF file

```text
┌────────────────────────────┐ offset 0
│ ELF header                 │  magic 0x7F 'E' 'L' 'F', class (64-bit), machine (x86-64),
│                            │  e_type = ET_DYN (PIE), e_entry = virtual address of _start,
│                            │  offsets of the program-header and section-header tables
├────────────────────────────┤
│ Program headers (Phdrs)    │  the LOADER'S view — "segments": what to mmap, where, with what perms
│  PT_PHDR                   │
│  PT_INTERP                 │  → "/lib64/ld-linux-x86-64.so.2"   (who finishes loading me)
│  PT_LOAD  R    (headers)   │
│  PT_LOAD  R-X  (.text)     │  → mmap PROT_READ|PROT_EXEC
│  PT_LOAD  R--  (.rodata)   │  → mmap PROT_READ            ("Hello, world\n" lives here)
│  PT_LOAD  RW-  (.data/.bss)│  → mmap PROT_READ|PROT_WRITE, .bss zero-filled
│  PT_DYNAMIC                │  → the dynamic section: DT_NEEDED libc.so.6, relocation tables, symbol tables
│  PT_GNU_STACK  RW-         │  → stack is non-executable
│  PT_GNU_RELRO              │  → make GOT read-only after relocation
├────────────────────────────┤
│ .text     machine code     │  _start, main, PLT stubs
│ .rodata   constants        │
│ .data     initialised vars │
│ .bss      (no file bytes)  │  size only; zeroed at load
│ .dynsym / .dynstr          │  symbols the loader needs to resolve (puts, __libc_start_main …)
│ .rela.dyn / .rela.plt      │  relocations: "write the address of X into GOT slot Y"
│ .got / .got.plt            │
│ .init_array/.fini_array    │  constructors/destructors run by ld.so / at exit
├────────────────────────────┤
│ Section headers            │  the LINKER'S/DEBUGGER'S view — not needed at run time; `strip` removes them
└────────────────────────────┘
```

Inspect it yourself: `readelf -h hello` (header), `readelf -l hello` (program headers), `readelf -d hello` (dynamic section), `objdump -d hello` (disassembly), `ldd hello` (resolved libraries).

## 1.3 What is *not* in the file

The file contains no stack, no heap, no environment, no address of `puts`, and no thread. All of those are **created at run time** by the kernel and the loader. The executable is a *recipe*; the process is the *dish*.

---

# 2 — Stage 1 — The Keystroke Reaches the Shell

Even before the shell sees `./hello`, hardware and kernel have been at work.

```text
keyboard ──USB──▶ xHCI controller ──IRQ──▶ kernel USB HID driver ──▶ input subsystem (evdev)
   ──▶ display server (Wayland/X11) or console driver ──▶ terminal emulator process (gnome-terminal, iTerm2, …)
   ──write()──▶ /dev/ptmx (pty master) ──▶ kernel line discipline (N_TTY) ──▶ pty slave /dev/pts/N
   ──read() returns──▶ bash
```

- The keyboard's microcontroller detects the key matrix change and sends a USB HID report. The host controller raises an **interrupt**; the kernel's HID driver decodes it into an `EV_KEY` event.
- The terminal emulator (a user-space program) receives the event, decides it is Enter, and **writes the byte `\n`** into the master side of a **pseudo-terminal**. Everything typed before it was likewise written byte by byte.
- The kernel's **line discipline** buffers the line, handles backspace and Ctrl-C (turning it into `SIGINT`), echoes characters back, and, on `\n`, makes the accumulated line `./hello\n` available on the slave side.
- **bash** has been blocked in `read(0, …)` on the slave. The kernel marks it runnable; the scheduler runs it; `read` returns 8 bytes.

Everything above is the same mechanism the program itself will later use to print. The pty is a bidirectional pipe with terminal semantics; the terminal emulator sits on one end and the shell (and its children) on the other.

---

# 3 — Stage 2 — The Shell Finds and Forks

bash now has the string `./hello`. What it does is ordinary user-space programming, but the sequence is worth knowing because it defines the environment your program inherits.

## 3.1 Parse and classify

1. **Tokenise and expand**: split on whitespace, apply alias expansion, brace expansion, tilde expansion, parameter expansion (`$VAR`), command substitution, word splitting, globbing (`*.c`), and quote removal. `./hello` survives untouched.
2. **Classify the command word**: is it a shell keyword (`if`, `for`)? A function? A **builtin** (`cd`, `echo`, `export` — these run inside bash's own process and never fork)? Otherwise it is an **external command**.
3. **Resolve the path**: because `./hello` contains a `/`, bash uses it directly. For a bare `hello`, bash would walk `$PATH` left to right calling `stat()` on each `dir/hello` until one is an executable regular file, caching the result in its command hash table (`hash -r` clears it).

## 3.2 Fork

bash calls `fork()` (actually `clone()` under the hood in glibc). The kernel:

- Allocates a new `task_struct` and a new **PID**.
- **Copies** the parent's page tables, marking all writable private pages **copy-on-write** (COW). No user memory is copied at this instant; the child and parent share physical pages until one writes.
- Duplicates the file-descriptor table (so the child has the same 0, 1, 2 → the pty), the signal handler table, the current directory, the umask, resource limits, and the credentials.
- Returns twice: `0` in the child, the child's PID in the parent.

`fork` followed immediately by `exec` is a historical oddity — the child duplicates an address space only to discard it — which is why `posix_spawn` and `vfork` exist, and why COW page tables make it cheap enough not to matter (~50–100 µs for a large shell).

## 3.3 Between fork and exec: the child sets up its world

In the child, *before* `exec`, bash performs the redirections and job-control setup that the command line asked for. This is the only window in which it can, because after `exec` bash's code is gone:

- `>out.txt` → `open("out.txt", O_WRONLY|O_CREAT|O_TRUNC)` then `dup2(fd, 1)`.
- `cmd1 | cmd2` → a `pipe()` created before both forks; each child `dup2`s the right end onto 0 or 1.
- Restore default signal dispositions that bash had changed (e.g. `SIGINT` back to default so Ctrl-C kills the child, not the shell).
- Put the child in its own **process group** and, for foreground jobs, make that group the pty's foreground group (`tcsetpgrp`) — this is what makes Ctrl-C reach `hello` and not `bash`.
- Apply `ulimit`s, `nice`, environment assignments (`FOO=bar ./hello`).

Then the child calls:

```c
execve("./hello", {"./hello", NULL}, environ);
```

Meanwhile the parent bash calls `waitpid(child_pid, &status, WUNTRACED)` and blocks. It will not print a prompt until the child exits or stops.

---

# 4 — Stage 3 — `execve`: The Kernel Replaces the Process Image

`execve` is the single most consequential system call in this story. It keeps the process (same PID, same open files unless `O_CLOEXEC`, same parent) but throws away *everything* about what it was executing and builds a new image. Roughly, in `fs/exec.c`:

## 4.1 Open and identify

1. **Path lookup**: `./hello` → the dentry cache (dcache) resolves `.` and `hello` component by component, checking `x` permission on each directory and on the file for the calling UID/GID. If the filesystem is mounted `noexec`, fail with `EACCES`.
2. **Open the file** for reading; take a reference on the inode. Check `ELOOP`, `ETXTBSY` (someone has it open for writing), and that it is a regular file.
3. **Read the first 256 bytes** (`bprm->buf`) and hand them to each registered **binary format handler** in turn: `binfmt_script` (`#!` → recursively exec the interpreter with the script path as argument), `binfmt_misc` (user-registered magic → e.g. run `.jar` files or foreign-architecture binaries via QEMU), and `binfmt_elf`. The ELF magic `\x7fELF` matches.

## 4.2 Point of no return

Before this point, any failure returns an error to the still-intact bash child. After it, failure kills the process with `SIGSEGV`/`SIGKILL` because there is nothing to return to.

4. **Check the ELF header**: correct class, machine, endianness, `e_type` is `ET_EXEC` or `ET_DYN`.
5. **Read program headers**. If `PT_INTERP` exists, open the interpreter (`/lib64/ld-linux-x86-64.so.2`) and read *its* ELF header too.
6. **`de_thread`**: if the process is multithreaded, kill every other thread — exec is per-process, and the new image has exactly one thread.
7. **Create a new `mm_struct`** (address space) and drop the old one (`exec_mmap`). Every mapping bash's child had is gone. The COW pages shared with the parent simply lose a reference.
8. **Reset signal handlers** to default (ignored signals stay ignored). Close all `O_CLOEXEC` file descriptors. Clear pending alarms and timers. Drop most capabilities unless the file is set-uid or has file capabilities, in which case compute the new credentials (this is where `sudo`'s magic starts).
9. **Set the personality and `comm`** (the 16-byte task name shown by `ps`, from the basename).

## 4.3 Build the new address space

This is `load_elf_binary`. Nothing is read from disk yet beyond the headers; the kernel only creates **VMAs** (virtual memory areas) — descriptors saying "this range of virtual addresses is backed by bytes X–Y of this file, with these permissions". Physical pages arrive later, on demand, via page faults (Stage 6).

```text
Virtual address space of the new process (x86-64, 47-bit user half, ASLR applied)
0x0000_7fff_ffff_f000 ┌──────────────────────┐
                      │ [vsyscall] (legacy)  │
0x0000_7ffc_xxxx_xxxx ├──────────────────────┤  ◀── random stack base
                      │ stack                │  argv strings, envp strings, auxv, then argc/argv/envp pointers at RSP
                      │   ↓ grows down       │  guard gap below; grows via faults up to RLIMIT_STACK (8 MiB default)
                      ├──────────────────────┤
                      │ [vdso] [vvar]        │  kernel-provided page: clock_gettime, getcpu without a trap
                      ├──────────────────────┤
0x0000_7fxx_xxxx_xxxx │ ld-linux-x86-64.so.2 │  the interpreter, mapped by the KERNEL at a random base
                      ├──────────────────────┤
                      │ (later: libc.so.6)   │  mapped by ld.so, not by the kernel
                      │ (later: mmap arena)  │  large mallocs, thread stacks
                      │   ↓ grows down       │
                      ├──────────────────────┤
                      │ heap [brk]           │  starts just after .bss (randomised gap); grows up via brk()
0x0000_55xx_xxxx_xxxx ├──────────────────────┤  ◀── random PIE base
                      │ .bss  RW  anonymous  │  zero-fill on demand
                      │ .data RW  file-backed│  private COW mapping of hello
                      │ .rodata R            │
                      │ .text R X            │  "Hello, world\n" is in .rodata here
                      │ ELF headers R        │
                      └──────────────────────┘
0x0000_0000_0000_0000   (first page never mapped: NULL dereference → SIGSEGV)
```

Steps:

10. For each `PT_LOAD`, `mmap` the file range at `base + p_vaddr` with `p_flags` → `PROT_*`, `MAP_PRIVATE|MAP_FIXED`. For PIE, `base` is chosen randomly (`arch_mmap_rnd`). If `p_memsz > p_filesz` (`.bss`), the trailing part is an anonymous zero mapping and the partial last page is zeroed.
11. Set the **brk** (heap start) just past the last segment, plus a random offset.
12. Map the **interpreter** ld.so's `PT_LOAD` segments at a random base, the same way.
13. Create the **stack VMA** at a random top-of-stack address. Copy `argv[]` strings, `envp[]` strings, and the executable's filename onto it. Then push the **auxiliary vector** (`auxv`) — the kernel's message to ld.so: `AT_PHDR` (where hello's program headers are), `AT_ENTRY` (hello's `_start`), `AT_BASE` (where ld.so was loaded), `AT_PAGESZ`, `AT_RANDOM` (16 random bytes for stack canaries and pointer mangling), `AT_SYSINFO_EHDR` (vDSO address), `AT_HWCAP`/`AT_HWCAP2` (CPU features), `AT_SECURE`, `AT_UID`/`AT_EUID`. Then `envp` pointers, `argv` pointers, and finally `argc` at the very top so that `RSP` points at it.
14. Map the **vDSO** and **vvar** pages.
15. **Set the register state** that will be restored on return to user mode: `RIP = ld.so's e_entry + AT_BASE` (not hello's `main`!), `RSP = top of stack`, all other GPRs zeroed, segment registers for 64-bit user code, `RFLAGS` sane.
16. Install the new `mm_struct`: write its page-table root into **CR3**. The TLB is flushed (or the ASID/PCID changed), because every old translation is now meaningless.

## 4.4 Return to user space

`execve` never "returns" in the normal sense — the code that called it is unmapped. The syscall exit path does `SYSRET`/`IRET` with the freshly written register frame, and the CPU's next instruction fetch is at ld.so's `_start`, in ring 3, with `hello`'s page tables loaded. From the CPU's point of view, a system call has simply returned to a very different place than it was called from.

Total cost of `execve` for a small dynamic binary: roughly 200–500 µs, dominated by path lookup, VMA construction, and the page faults that immediately follow.

---

# 5 — Stage 4 — The Dynamic Linker Finishes the Job in User Space

The kernel has handed control to `/lib64/ld-linux-x86-64.so.2`, a *statically linked, position-independent* program whose sole job is to complete the loading of `hello`. Everything it does is ordinary user-mode code using `open`, `mmap`, `mprotect`, `read`, and `close`. You can watch it with `LD_DEBUG=all ./hello`.

## 5.1 Bootstrap itself

ld.so has no libc yet — it *is* the thing that loads libc. Its first job (`_dl_start`) is to relocate **itself** using only position-independent code and the base address it computes from `AT_BASE` or its own `RIP`. Then it reads `auxv` off the stack to learn where `hello`'s headers are.

## 5.2 Build the dependency graph

1. Parse `hello`'s `PT_DYNAMIC`: `DT_NEEDED` entries name `libc.so.6`. (Run `readelf -d hello | grep NEEDED`.)
2. For each needed library, **search** in order: `DT_RPATH` (deprecated), `LD_LIBRARY_PATH` (ignored for set-uid), `DT_RUNPATH`, then `/etc/ld.so.cache` (a prebuilt hash from `ldconfig`), then `/lib64`, `/usr/lib64`.
3. `open()` + `read()` the ELF header, `mmap()` each `PT_LOAD` segment of `libc.so.6` at a random base (one big `PROT_NONE` reservation then `mprotect`/`mmap` fixed within it, so the segments keep their relative offsets), `close()` the fd. The file's pages are almost certainly already in the **page cache** because every process on the system uses libc — the mapping costs nothing on disk.
4. Recurse into libc's own `DT_NEEDED` (on modern glibc, libc.so.6 depends on ld-linux only; older splits had libpthread, libdl, etc.). Build a breadth-first **search scope** — the order in which symbol lookups will try libraries.

## 5.3 Relocate

Each loaded object has relocation tables saying "the value at this GOT slot must become the address of symbol S". ld.so walks them:

- **`R_X86_64_RELATIVE`**: add the load base to a stored offset. No symbol lookup; cheap. Most relocations in a PIE are these.
- **`R_X86_64_GLOB_DAT`** / **`R_X86_64_64`**: look up symbol `S` by name (hashing via `DT_GNU_HASH`) across the search scope; write its absolute address. This is how `hello`'s references to `stdout` or `environ` get resolved.
- **`R_X86_64_JUMP_SLOT`** (PLT entries, e.g. `puts`): by default these are **lazily** bound. ld.so writes into the GOT slot the address of a *resolver stub* in the PLT; the first call to `puts` will go PLT → resolver → `_dl_runtime_resolve` → look up `puts` → patch the GOT → jump to `puts`. Every later call reads the patched GOT and goes straight through. `LD_BIND_NOW=1` or linking with `-z now` (the default under `-z relro` hardening on most distros) resolves them all up front instead.
- **Symbol versioning**: `puts@GLIBC_2.2.5` — the lookup must match the version the binary was linked against, so a new libc keeps old binaries working.
- **RELRO**: after relocation, `mprotect` the GOT and other relocated data read-only, so an attacker with a write primitive cannot redirect function pointers.

## 5.4 TLS, initialisers, and the jump

5. Set up **Thread-Local Storage**: allocate the static TLS block, point `FS` base at the thread control block (`arch_prctl(ARCH_SET_FS)`), so `errno` and `__thread` variables work.
6. Run **initialisers**: each library's `DT_INIT` and `DT_INIT_ARRAY` in dependency order (libc's sets up `stdout` buffering mode, locale, `_IO_2_1_stdout_`, the `malloc` arena state, `getauxval` cache …). C++ static constructors live here too.
7. Register the destructors (`DT_FINI_ARRAY`) to run at exit.
8. **Jump to `AT_ENTRY`** — `hello`'s `_start`, from `crt1.o`:

```asm
_start:
    xor   %ebp, %ebp            ; mark outermost frame
    mov   %rdx, %r9             ; rtld_fini (from ld.so) → 6th arg
    pop   %rsi                  ; argc
    mov   %rsp, %rdx            ; argv
    and   $-16, %rsp            ; align stack (ABI requires 16-byte alignment at call)
    push  %rax ; push %rsp
    xor   %r8d, %r8d ; xor %ecx, %ecx   ; fini, init (unused in modern glibc)
    mov   main@GOTPCREL(%rip), %rdi     ; main
    call  *__libc_start_main@GOTPCREL(%rip)
    hlt                          ; never reached
```

`__libc_start_main` then: stores `argv`/`envp`/`auxv`, sets up the stack-protector canary from `AT_RANDOM`, installs `atexit` handlers, calls `__libc_init_first`, runs the program's own `.init_array` (constructors in `hello`), and finally **calls `main(argc, argv, envp)`**. When `main` returns, it calls `exit(ret)`.

At this moment — several hundred microseconds and a few thousand instructions after you pressed Enter — your first line of code runs.

---

# 6 — Stage 5 — The Scheduler Puts It on a CPU

The narrative so far pretended the process was continuously executing. In reality it was preempted, blocked, and rescheduled several times, and the timeline only makes sense with the **scheduler** in view.

## 6.1 What the kernel schedules

The unit is a **task** (`task_struct`) — a thread. A process with one thread is one task. Each task is always in one of a handful of states:

| State | Meaning | How you get there |
|---|---|---|
| `TASK_RUNNING` | runnable or actually on a CPU | fork, wakeup, preemption |
| `TASK_INTERRUPTIBLE` | sleeping, wakes on event *or* signal | `read` on an empty pty, `waitpid`, `futex_wait`, `nanosleep` |
| `TASK_UNINTERRUPTIBLE` | sleeping, wakes only on event (the `D` state) | disk I/O completion, some locks |
| `TASK_STOPPED` / `TRACED` | frozen by `SIGSTOP` / a debugger | Ctrl-Z, `ptrace` |
| `EXIT_ZOMBIE` | finished, waiting for parent to `wait()` | `exit` |

## 6.2 Where scheduling decisions happened in our story

- `fork()` created a new runnable task and placed it on a run queue — usually the **same CPU** as the parent for cache warmth, unless load balancing moves it. The kernel often lets the *child* run first (so it can `exec` quickly and avoid COW copies when the parent writes).
- bash's `waitpid` put the *parent* to sleep (`TASK_INTERRUPTIBLE`), freeing the CPU.
- During `execve`, if the file's pages were not in the page cache, the task blocked on disk I/O (`TASK_UNINTERRUPTIBLE`) and another task ran meanwhile.
- Every **timer tick** (1–4 ms, or tickless on idle CPUs) and every wakeup of a higher-priority task can **preempt** the running task.

## 6.3 How a CPU is chosen and a switch happens

Linux's default class (CFS, since 6.6 **EEVDF**) tracks per-task **virtual runtime** weighted by nice value and picks the task with the earliest eligible virtual deadline. Each CPU has its own run queue; periodic and idle-time **load balancing** migrates tasks across cores, preferring to stay within the same L2/L3 domain and NUMA node (the scheduler knows the cache topology).

A **context switch** (`__schedule` → `context_switch` → `switch_to`):

1. Save the outgoing task's callee-saved registers and kernel stack pointer into its `thread_struct`.
2. If the incoming task has a different `mm_struct`, write its page-table root into **CR3**. With PCID (x86) / ASID (ARM) the TLB keeps entries tagged by address space so a full flush is avoided; without it, the TLB is flushed and the new task pays TLB misses for a while.
3. Switch the kernel stack pointer, `FS`/`GS` bases (TLS), and, lazily, the FPU/SIMD state (`XSAVE`/`XRSTOR`, 1–2 KiB).
4. Update the per-CPU "current task" pointer and the `TSS`'s ring-0 stack pointer for the next trap.
5. Return — but into the *incoming* task's saved kernel context, which then does its own return-to-user-mode.

Cost: ~1–3 µs direct, plus the indirect cost of cold caches and TLB for the new task, which is usually larger.

---

# 7 — Stage 6 — The First Instruction: Virtual Memory Wakes Up

The kernel `SYSRET`s to ld.so's entry. The CPU tries to fetch the instruction at, say, `0x7f3a_1c00_2b40`. Not a single byte of ld.so, libc, or hello is in physical memory yet under *this* process's page tables. What happens next is the mechanism by which the entire program gets into memory — one 4 KiB page at a time, on demand.

## 7.1 Address translation

Every memory access by user code uses a **virtual address**. The **MMU** translates it via a four- (or five-) level radix tree of page tables rooted at CR3:

```text
64-bit virtual address (4-level, 4 KiB pages)
 63        48 47      39 38      30 29      21 20      12 11         0
┌────────────┬──────────┬──────────┬──────────┬──────────┬────────────┐
│ sign ext.  │ PML4 idx │ PDPT idx │  PD idx  │  PT idx  │   offset   │
└────────────┴──────────┴──────────┴──────────┴──────────┴────────────┘
                 │           │          │           │
    CR3 ──▶ PML4 ─┴─▶ PDPT ──┴─▶ PD ────┴─▶ PT ─────┴─▶ PTE: physical frame number | flags
                                                          flags: Present, R/W, User, Accessed,
                                                          Dirty, NX, Global, PCD/PWT, …
```

- First the **TLB** (Translation Lookaside Buffer) is checked: a small fully-associative cache of recent translations (L1 iTLB/dTLB ~64–128 entries, L2 STLB ~1,500–2,000). Hit → physical address in the same cycle as the L1 cache lookup (they are looked up in parallel; L1 is virtually indexed, physically tagged).
- Miss → the **page walker** (hardware) reads four page-table entries from memory (each themselves cached in L1/L2 and in dedicated **paging-structure caches**). ~20–100 cycles.
- If the final PTE's **Present** bit is clear, or the access violates the permission bits (write to read-only, user access to supervisor page, execute of NX page), the CPU raises a **page fault** exception (`#PF`, vector 14), pushing the error code and storing the faulting address in **CR2**.

## 7.2 The page fault handler

The CPU switches to ring 0, to the kernel stack, and jumps through the IDT to `asm_exc_page_fault` → `do_user_addr_fault`. The kernel:

1. Reads CR2 and the error code (present? write? user? instruction fetch?).
2. Finds the **VMA** covering the address (`find_vma`, a maple tree lookup). No VMA → `SIGSEGV`. VMA exists but permissions forbid the access → `SIGSEGV` too (or `SIGBUS` for file-backed beyond EOF).
3. Otherwise this is a legitimate **demand fault**. `handle_mm_fault` walks the software page tables, allocating intermediate levels as needed, and dispatches on the VMA type:
   - **File-backed, read** (`.text`, `.rodata`, libc's code): call the filesystem's `->fault` (`filemap_fault`). Look up the file's **page cache** (an xarray of pages indexed by file offset). Hit — the common case for libc — map the *existing* physical page, shared with every other process using that file. Miss — allocate a page, issue a **read** to the block device, put the task to sleep, and retry when the I/O completes. Readahead pulls in neighbouring pages speculatively. **Fault-around** maps up to 16 already-cached neighbouring pages in the same fault to cut the fault count.
   - **File-backed, write** (`.data`): first read as above, then because the mapping is `MAP_PRIVATE`, allocate a fresh page, copy, and map it writable — **copy-on-write**. The file itself is never modified.
   - **Anonymous, read** (`.bss`, stack, heap before first write): map the single shared **zero page** read-only.
   - **Anonymous, write**: allocate a zeroed page from the buddy allocator (per-CPU page lists → per-zone free lists), map it writable. This is what `malloc`'d memory costs on first touch — `malloc` itself only moved a pointer.
   - **Stack growth**: the stack VMA is expanded downward if the fault is just below it and within `RLIMIT_STACK`.
4. Write the PTE (`set_pte_at`), update `rss` counters, and add the page to the **LRU lists** so it can be reclaimed later.
5. `IRET` back to user mode. The CPU **re-executes the faulting instruction**; this time the TLB misses, the walker finds a present PTE, fills the TLB, and the fetch proceeds.

A "Hello, world" process takes on the order of **50–100 page faults** before `main` runs — nearly all minor faults (page cache hits, ~0.5–1 µs each). `perf stat -e page-faults,minor-faults,major-faults ./hello` shows the count; `/usr/bin/time -v` shows it too.

## 7.3 Why it is built this way

Demand paging means: (a) a 100 MB binary of which you use 5 MB costs 5 MB; (b) `fork` is cheap; (c) the same physical libc pages serve every process on the system; (d) memory can be **overcommitted** and pages **reclaimed** (written to swap or simply dropped if clean and file-backed) under pressure, transparently to the program. The price is unpredictable latency on first touch — which is why latency-sensitive programs `mlock` or pre-fault (`MAP_POPULATE`).

---

# 8 — Stage 7 — Inside the Core: What Executing an Instruction Means

The instruction bytes are now in the L1 instruction cache. What does "execute" mean on a modern out-of-order core? Consider the body of `main` as GCC actually emits it:

```asm
main:
    sub   $8, %rsp                  ; align stack
    lea   .LC0(%rip), %rdi          ; rdi = &"Hello, world"
    call  puts@PLT                  ; → PLT stub → GOT → (first time) resolver → libc puts
    xor   %eax, %eax                ; return 0
    add   $8, %rsp
    ret
```

Six instructions. A single core will process them through a pipeline that looks like this:

```text
 FRONT END (in order)                              BACK END (out of order)                    (in order)
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────────────────┐  ┌────────┐
│  Branch  │─▶│  Fetch   │─▶│  Decode  │─▶│  Rename  │─▶│ Schedule/ │─▶│  Execute on ports │─▶│ Retire │
│ Predict  │  │ 16-32B/  │  │ x86 →    │  │ arch reg │  │ Dispatch  │  │ ALU ALU ALU ALU   │  │ commit │
│ (BTB,    │  │ cycle    │  │ µops     │  │ → phys   │  │ (RS,      │  │ LOAD LOAD STORE   │  │ in     │
│ TAGE)    │  │ from L1i │  │ (+µop $) │  │ reg; ROB │  │ ~100-200  │  │ BR   FP/SIMD …    │  │ program│
│          │  │          │  │          │  │ alloc)   │  │ entries)  │  │                   │  │ order  │
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └───────────┘  └──────────────────┘  └────────┘
     ▲                                                                   │ loads/stores
     └──────────── mispredict: flush everything younger, restart ────────┤
                                                                         ▼
                                                                 L1D ⇄ L2 ⇄ L3 ⇄ DRAM  (Stage 8)
```

## 8.1 Fetch and predict

The **branch predictor** runs *ahead* of fetch. Before the core has even decoded `call puts@PLT`, the predictor has guessed (from the Branch Target Buffer, indexed by the address of the `call`) where it will go, and fetch is already pulling those bytes. For the `ret` at the end of `puts`, a **return stack buffer** predicts the return address. A correct prediction costs nothing; a mispredict costs ~15–20 cycles of flushed work. Predictors are ~95–99 % accurate on typical code, which is the only reason deep pipelines are viable.

## 8.2 Decode into micro-operations

x86 instructions are variable-length (1–15 bytes) and semantically rich. The decoders convert each into one or more fixed-format **µops**: `add $8,%rsp` → one µop; `call` → a store of the return address plus a jump; `push`/`pop` → address generation plus memory op (often **fused**). Decoded µops are cached in the **µop cache** so hot loops skip decoding entirely. ARM's fixed 4-byte encoding makes this stage simpler and wider.

## 8.3 Rename and the illusion of sequential execution

The program names 16 general registers; the core has ~200–300 physical ones. **Register renaming** maps each *write* to a fresh physical register, eliminating false dependencies (WAR/WAW hazards). `xor %eax,%eax` is recognised as a zeroing idiom and doesn't even need an execution unit. Each µop gets a slot in the **Reorder Buffer (ROB)**, which records program order so results can be made visible in order even though they are computed out of order.

## 8.4 Out-of-order execution

µops sit in the **scheduler** (reservation station) until their inputs are ready, then issue to whichever execution **port** can handle them — several per cycle. While `call puts` is waiting for the instruction bytes of `puts` to arrive from L2, the core is already executing later independent µops it has speculated past the call. A load that misses all the way to DRAM (~80 ns ≈ 300 cycles) stalls *that* µop, but up to ~200 others can proceed around it — this **memory-level parallelism** is the whole point of the design.

Loads and stores go through the **load/store queues**. Stores don't write the cache until they **retire** (so a mispredicted store never becomes visible); loads can be **forwarded** data from an older in-flight store to the same address without waiting for it to reach the cache.

## 8.5 Retire

The oldest µops in the ROB whose execution is complete **retire** in program order, up to 4–8 per cycle: their results become architecturally visible, stores drain to L1D, and exceptions are delivered precisely at the faulting instruction. If a branch turns out mispredicted at retirement, everything younger in the ROB is discarded and fetch restarts from the correct target. **Speculation is invisible** to the program — except through timing side channels, which is what Spectre exploited.

## 8.6 What "one instruction" costs

| Event | Approximate cost |
|---|---|
| Simple ALU µop, in flight | 1 cycle latency, up to 4–6 per cycle throughput |
| L1D hit | 4–5 cycles |
| L2 hit | ~14 cycles |
| L3 hit | ~40–70 cycles |
| DRAM | ~250–400 cycles (80–100 ns) |
| Branch mispredict | ~15–20 cycles |
| TLB miss, page walk with cached PTEs | ~20–30 cycles |
| Page fault (minor) | ~1,000–3,000 cycles (in the kernel) |
| System call round trip (bare) | ~200–500 cycles; with mitigations more |

A 3 GHz core does ~3 billion cycles per second; the entire `hello` program runs in roughly a million cycles, of which `main` itself is a few hundred.

---

# 9 — Stage 8 — Below the Core: Caches, DRAM, and Electrons

Every load and store, and every instruction fetch, is a request into the **memory hierarchy**. The `lea .LC0(%rip),%rdi` computes an address; `puts` will then *load* the string bytes from it. Follow one such load.

## 9.1 The cache hierarchy

```text
core 0                core 1                …               core N
┌────────┬────────┐   ┌────────┬────────┐                   ┌────────┬────────┐
│ L1i    │ L1d    │   │ L1i    │ L1d    │                   │ L1i    │ L1d    │   32-48 KiB each, 8-12 way,
│ 32K    │ 48K    │   │ 32K    │ 48K    │                   │ 32K    │ 48K    │   64-byte lines, ~4 cy
├────────┴────────┤   ├────────┴────────┤                   ├────────┴────────┤
│ L2  1-2 MiB     │   │ L2  1-2 MiB     │                   │ L2  1-2 MiB     │   private, ~14 cy
└────────┬────────┘   └────────┬────────┘                   └────────┬────────┘
         └───────────────┬─────┴────────── ring / mesh ──────────────┘
                 ┌───────┴────────────────────────────────────────────┐
                 │ L3 (LLC)  shared, sliced, 32-96+ MiB, ~50 cy       │   + snoop filter / directory
                 └───────┬────────────────────────────────────────────┘
                 ┌───────┴──────────┐
                 │ Memory controller│  ── DDR5 channels ──▶ DIMMs   ~80-100 ns
                 └──────────────────┘
```

The load's physical address (from the TLB) is split into **tag | set index | line offset**. The L1D looks up the set, compares the tag against all ways in parallel, and on a hit returns the 8 bytes from the 64-byte line. On a miss, the request goes to L2, then L3, then to the **memory controller**. The line comes back and is installed at every level (in an *inclusive* or *non-inclusive* policy depending on the design), evicting a victim chosen by pseudo-LRU. Hardware **prefetchers** watch the stream of misses and pull in the next lines of a sequential pattern before they are asked for — which is why iterating an array forward is fast and pointer-chasing a linked list is not.

## 9.2 Coherence: many cores, one memory

The string `"Hello, world\n"` lives in a page shared by no one, but libc's `stdout` buffer struct and every kernel data structure are touched by many cores. **Cache coherence** (MESI/MOESI/MESIF) guarantees that all cores see a single value per address: a line can be **Modified** in exactly one cache, **Shared** read-only in many, or **Invalid**. A store to a Shared line must first broadcast an invalidation (or consult a **directory**) and obtain **Exclusive** ownership — a ~50–100 ns round trip. Two cores hammering the same line ping-pong it back and forth (**false sharing** when they are actually touching different variables that happen to share a line). This is the hardware reality beneath every mutex, atomic counter, and lock-free algorithm.

**Memory ordering** is a separate question from coherence: x86 (TSO) lets stores be delayed in a store buffer so a later load can appear before an earlier store; ARM is weaker still. `mfence`/`dmb`, `lock`-prefixed instructions, and acquire/release semantics in C++/Java/Rust are how software constrains this.

## 9.3 DRAM

A miss that reaches the memory controller becomes a transaction on a **DDR** channel:

1. The controller maps the physical address to **channel, rank, bank group, bank, row, column** — interleaving addresses so consecutive lines hit different banks and can proceed in parallel.
2. If the target **row** is not already open in the bank's **row buffer**, issue `PRECHARGE` (close current row) then `ACTIVATE` (open the new row: assert the word line, let ~8 KiB of cell capacitors dump their charge onto bit lines, sense amplifiers resolve each to 0/1 — this is destructive, so the row buffer now holds the only copy until write-back). Then `READ` with the column address; 64 bytes come out over the 64-bit bus in a burst of 8 transfers at the DDR clock. Row hit ≈ 15 ns; row miss ≈ 45 ns; plus queueing and controller overhead brings total observed latency to 80–100 ns.
3. Every ~64 ms each row must be **refreshed** because the capacitors leak — the controller steals cycles to do this, which is why DRAM is *dynamic*.
4. **ECC** DIMMs carry extra bits so single-bit flips (cosmic rays, Rowhammer) are corrected on the fly.

## 9.4 Down to the transistor

A **DRAM cell** is one transistor and one capacitor storing ~20–30 fF — a few tens of thousands of electrons for a "1". An **SRAM cell** in a cache is six transistors — two cross-coupled inverters and two access transistors — that hold their value as long as power is applied, no refresh, but 6× the area, which is why caches are small and DRAM is large.

A **register** is a row of flip-flops. A **flip-flop** is two latches in series clocked on opposite phases so the output changes only at the clock edge. The **clock** itself is a ~3 GHz square wave distributed by a tree of buffers across the die; on every rising edge, every flip-flop in the pipeline stage samples its input, and combinational logic (adders, muxes, comparators) built from CMOS gates has until the next edge (~330 ps at 3 GHz) to compute the next value. The critical path — the longest gate chain between two flip-flops — sets the maximum frequency.

A **CMOS inverter** is a PMOS transistor stacked on an NMOS: gate high → NMOS conducts, output pulled to ground; gate low → PMOS conducts, output pulled to Vdd. A NAND gate is two of each. Every adder, decoder, and cache comparator is built from these. A transistor switches when the voltage on its gate (a few hundred millivolts on a ~0.7–1.0 V supply) creates a conducting channel a few nanometres long between source and drain. Power is burned mostly on the switching itself (charging and discharging gate and wire capacitances, `P ∝ C·V²·f`), which is why frequency scaling stopped around 2005 and core counts went up instead.

At this level "running a program" is: the pattern of gate voltages across ~10–50 billion transistors evolving, one clock edge at a time, in a way constrained by the wiring to implement the instruction set semantics — with the instruction stream you compiled selecting which paths are exercised.

---

# 10 — Stage 9 — The Program Asks for Something: System Calls

`main` calls `puts`. libc's `puts` copies the string into `stdout`'s buffer. Because `stdout` is connected to a terminal, it is **line-buffered**: the `\n` triggers a flush, which is `write(1, "Hello, world\n", 13)`. (Redirect to a file and it becomes fully buffered — the `write` happens at exit. This is the single most common source of "my output disappeared when the program crashed" confusion.)

## 10.1 The user side

glibc's `write` wrapper loads the syscall number (`1` for `write` on x86-64) into `RAX`, the arguments into `RDI, RSI, RDX`, and executes **`SYSCALL`**. Contrast: `int 0x80` on 32-bit x86, `SVC #0` on ARM64, and for a few calls (`clock_gettime`, `getcpu`, `gettimeofday`) the **vDSO** answers entirely in user space by reading a kernel-updated shared page, no trap at all.

## 10.2 The hardware transition

`SYSCALL` in one instruction:

- Saves `RIP` → `RCX`, `RFLAGS` → `R11`.
- Loads `RIP` from `IA32_LSTAR` MSR (the kernel's entry point, `entry_SYSCALL_64`), `CS`/`SS` from `IA32_STAR`, masks `RFLAGS` per `IA32_FMASK` (clearing IF, so interrupts are briefly off).
- Switches **CPL from 3 to 0**. It does *not* switch stacks — the very first thing the kernel does is `swapgs` to reach per-CPU data and load the kernel stack pointer from there.

## 10.3 The kernel side

1. `entry_SYSCALL_64`: `swapgs`, switch to the task's **kernel stack**, push a `pt_regs` frame with all user registers. With **KPTI** (Meltdown mitigation) also switch to the kernel's page-table root (user page tables don't map most of the kernel).
2. `do_syscall_64`: bounds-check `RAX`, index the **syscall table**, call `sys_write(fd=1, buf, count=13)`. Optionally invoke seccomp filters, ptrace, audit.
3. `ksys_write` → `fdget_pos(1)` — look up fd 1 in the process's file table → `struct file` → `vfs_write` → the file's `f_op->write_iter`. For our pty slave this is `tty_write`.
4. **Copy the 13 bytes from user space** (`copy_from_user`, with the access checked against the user address range and SMAP temporarily disabled via `stac`/`clac`) into a kernel buffer. The kernel never trusts a user pointer.
5. The **tty layer** runs the line discipline's output processing (e.g. `\n` → `\r\n` if `ONLCR` is set), then pushes the bytes into the pty *master*'s input buffer and **wakes up** anything waiting on the master — the terminal emulator, blocked in `poll()`/`read()`.
6. Return 13 in `RAX`. On the way out: check for pending **signals** (deliver by rewriting the user frame to run the handler), check `need_resched` (the wakeup may have made a higher-priority task runnable — if so, context-switch *now* before returning), restore KPTI page tables, `swapgs`, **`SYSRET`** (restores `RIP` from `RCX`, `RFLAGS` from `R11`, CPL back to 3).

Total: a few hundred nanoseconds to a microsecond. `strace ./hello` shows every syscall the process makes; the full list for hello is roughly: `execve, brk, arch_prctl, mmap, access, openat, newfstatat, mmap, read, pread64, mmap, mmap, mmap, close, mprotect, mprotect, mprotect, set_tid_address, set_robust_list, rseq, mprotect, prlimit64, munmap, getrandom, newfstatat, write, exit_group` — about 30, of which only `write` and `exit_group` were on behalf of your code. Everything else was the loader.

## 10.4 Blocking calls

Had this been `read(0, …)` from an empty terminal, `sys_read` would have reached the tty layer, found no data, and called `wait_event_interruptible` → mark the task `TASK_INTERRUPTIBLE`, put it on the tty's **wait queue**, call `schedule()`. The CPU goes to another task or to idle (`HLT`/`MWAIT`). When a keypress later arrives, the tty layer's wakeup walks the wait queue, marks the task runnable, and eventually the scheduler resumes it *inside* `sys_read`, which now finds data, copies it out, and returns. Blocking is not "the program waits" — the program has no CPU at all; a kernel data structure remembers to resume it.

---

# 11 — Stage 10 — Output Reaches Your Eyes

The 13 bytes are now in the pty master's buffer. The rest of the path is another program and another stack of hardware.

1. The **terminal emulator** wakes from `poll`, `read`s the bytes, and runs them through its **VT escape-sequence parser** (the same state machine that handles colours and cursor movement). Plain text → append glyphs to the current line, advance the cursor.
2. It **renders**: for each character, look up the glyph in a font atlas (rasterised by FreeType/CoreText/DirectWrite, cached as a texture), and issue draw commands via a **GPU API** (OpenGL/Vulkan/Metal/Direct3D) or a CPU blit into a pixel buffer.
3. The **display server / compositor** (Wayland compositor, X server, Quartz, DWM) receives the new buffer, composites it with other windows into the **framebuffer** for the next frame, on a **vblank** boundary (every 16.7 ms at 60 Hz, 6.9 ms at 144 Hz).
4. The **GPU driver** (kernel DRM/KMS on Linux) programs the **display controller** to scan out the framebuffer: it walks memory row by row, converting pixels to a **DisplayPort/HDMI** signal.
5. The **monitor** decodes the stream, its **timing controller** drives row and column drivers, and the liquid-crystal cells (or OLED pixels) change state over ~1–5 ms. Photons leave the panel.

Latency from `write` to photons: dominated by waiting for the next frame — typically 10–30 ms, which is why this stage, not the CPU, sets the floor for perceived responsiveness. The program itself finished long before you could see its output.

---

# 12 — Stage 11 — Interruptions: Timers, Faults, and Sharing the CPU

Our program never had the CPU to itself. **Asynchronous events** arrive constantly, each one forcing a hardware control transfer regardless of what user code was doing.

## 12.1 The interrupt path

```text
device asserts IRQ ──▶ IOAPIC / MSI-X message ──▶ Local APIC of a chosen CPU ──▶
CPU finishes current instruction (or stops at a precise boundary) ──▶
saves RIP/CS/RFLAGS/RSP/SS on the kernel stack (switching stacks via TSS if in user mode) ──▶
looks up vector in the IDT ──▶ jumps to the handler in ring 0 ──▶
handler: acknowledge (EOI), run the device driver's ISR (short — "top half") ──▶
schedule deferred work (softirq / threaded IRQ / workqueue — "bottom half") ──▶
on return: check need_resched, pending signals ──▶ IRET back to whatever was interrupted
```

The interrupted program **cannot tell**, except by measuring time; the kernel saves and restores every register. The only architecturally visible effect is that instructions took longer.

## 12.2 Sources that interrupted hello

| Source | Frequency | Effect on the running task |
|---|---|---|
| **Local APIC timer** (scheduler tick) | every 1–4 ms (`CONFIG_HZ`), or only when needed on tickless kernels | accounting; may trigger preemption if the task's slice is used up |
| **Inter-processor interrupts** (IPI) | on TLB shootdowns, wakeups of remote CPUs, function calls | brief |
| **NIC, NVMe, USB, GPU** | per packet batch / I/O completion / keystroke / vblank | the driver runs on *your* CPU's time; napi/irq affinity spread it |
| **Page faults** (synchronous) | ~50–100 for hello | handled in Stage 6 |
| **Signals** (software) | `SIGCHLD` to bash when hello exits; `SIGINT` if you press Ctrl-C | kernel rewrites the user register frame so the handler runs next, then `sigreturn` restores the original frame |

## 12.3 Preemption

When the tick handler or a wakeup finds that another task deserves the CPU (`need_resched`), the kernel doesn't switch immediately inside the interrupt; it sets a flag and, on the **return-to-user path**, calls `schedule()`. From `hello`'s perspective, it was executing `puts` and then, ~µs to ms later, it was still executing `puts` — with cold caches. Timeslices are typically 1–6 ms under load; when the machine is otherwise idle, `hello` runs to completion uninterrupted.

---

# 13 — Stage 12 — Exit: Unwinding Everything

`main` returns 0 into `__libc_start_main`, which calls `exit(0)`.

## 13.1 User-space teardown

1. Run **`atexit`** handlers and `.fini_array` destructors in reverse registration order (C++ static destructors here).
2. **Flush and close all `stdio` streams** — this is when a redirected `stdout` buffer actually gets written to the file.
3. Call `_exit(0)` → the `exit_group` syscall (kills all threads; plain `exit` would end only the calling thread).

## 13.2 Kernel-space teardown (`do_exit`)

4. Set the task's exit code; mark `PF_EXITING`.
5. **`exit_mm`**: drop the reference on the `mm_struct`. If it hits zero, unmap every VMA: for each page, decrement its refcount; free anonymous pages back to the buddy allocator; **do not** touch file-backed pages in the page cache — they stay cached (this is why the *second* run of a program is faster). Free the page tables themselves.
6. **`exit_files`**: close every file descriptor. For the pty, this drops a reference; the terminal stays open because bash still holds it. For a socket, this begins TCP `FIN`. For a file opened with `O_TMPFILE` or already unlinked, this is when its blocks are freed.
7. **`exit_sighand`, `exit_fs`, `exit_thread`**: release signal handlers, cwd/root references, FPU state.
8. **Reparent children** to `init`/a subreaper if any exist.
9. **Notify the parent**: send `SIGCHLD` to bash and wake any `wait4` on the parent's wait queue. The task becomes a **zombie** (`EXIT_ZOMBIE`): everything freed except the `task_struct` holding the exit status — the parent has to collect it.
10. `schedule()` — the task never returns from this; the scheduler picks another task and the dead one's kernel stack is freed by the *next* context switch away from it (you can't free the stack you're standing on).

## 13.3 Back in bash

bash's `waitpid` returns with `WIFEXITED(status) && WEXITSTATUS(status) == 0`. The kernel frees the zombie's `task_struct` (`release_task`); the PID is now reusable. bash sets `$?` to 0, runs any `PROMPT_COMMAND`, `write`s the prompt string to fd 1 (the same pty path as Stage 9), and calls `read` on fd 0 — blocking again exactly where it was when you first pressed Enter.

---

# 14 — Variations: Interpreted, JIT-Compiled, and Managed Programs

Everything above is the same for `python3 hello.py`, `node hello.js`, `java Hello`, or `go run hello.go` up to the point where `main` starts — because in every case the kernel `exec`s a native ELF binary (the interpreter or runtime). What differs is what that native binary then does with *your* code.

| Model | Native binary the kernel exec's | What happens to your code | Extra machinery in the process |
|---|---|---|---|
| **AOT compiled** (C, C++, Rust, Go) | your program | already machine code; runs directly | Go: its own runtime for goroutine scheduler (M:N threads), GC, netpoller |
| **Bytecode interpreter** (CPython, Ruby MRI) | `python3` | parse → AST → bytecode (`.pyc` cached) → a `switch`/computed-goto loop that dispatches one opcode at a time; every value is a heap object; the GIL serialises threads | refcounting + cycle GC; the interpreter loop is itself the code the CPU executes, and your program is *data* it reads |
| **JIT + interpreter** (JVM HotSpot, V8, .NET CoreCLR, PyPy) | `java`, `node` | start interpreting bytecode; profile; compile hot methods to machine code in an **executable heap** (`mmap PROT_READ|PROT_WRITE|PROT_EXEC` or W^X flipping); patch call sites; **deoptimise** back if speculation (type assumptions, inlining) fails | tiered compilers (C1/C2, Ignition/TurboFan), a generational GC with its own threads and **safepoints**, a compiler thread, code cache management |
| **Shell script** | `bash` via `binfmt_script` | bash reads the file and forks/execs each external command as in Stage 2 | none |
| **WebAssembly** | the browser or `wasmtime` | validated, then compiled (baseline + optimising) to native; runs in a sandboxed linear memory | bounds checks or guard pages for memory safety |

Three consequences worth internalising:

- **Startup cost** is the runtime's, not yours: JVM startup loads and links thousands of classes (~50–100 ms); CPython imports and unmarshals modules (~15–30 ms); node initialises V8 and its snapshot (~30–40 ms). `hello` in C is ~0.5 ms. This is the whole motivation for GraalVM native-image, CDS archives, and V8 snapshots.
- **Garbage collection** is user-space memory management on top of the kernel's: the runtime `mmap`s large arenas from the kernel once, then hands out objects itself. GC pauses are the runtime stopping *your* threads at safepoints, not the kernel doing anything.
- **JIT code is just memory that happens to be executable**: it goes through the same page tables, the same L1i cache, the same branch predictor. The reason JITs can beat AOT compilers on some workloads is that they see the actual types and branch outcomes at run time and specialise for them.

---

# 15 — Variations: Windows, macOS, Containers, and VMs

## 15.1 Windows

| Linux | Windows | Notes |
|---|---|---|
| bash `fork` + `execve` | `CreateProcess` (→ `NtCreateUserProcess`) | one call creates the process *and* its first thread with the new image; no fork/exec split |
| ELF | PE/COFF (`.exe`, `.dll`) | import table ≈ DT_NEEDED; IAT ≈ GOT; relocations via `.reloc` |
| ld.so | `ntdll.dll`'s loader (`LdrInitializeThunk`) | runs in the new process on first thread start; loads `kernel32.dll`, etc.; `DllMain` ≈ `DT_INIT` |
| `SYSCALL` → syscall table | `SYSCALL` → `KiSystemCall64` → SSDT | Win32 API is a user-mode layer (`kernel32` → `ntdll` → syscall); numbers change between builds so nobody calls them directly |
| `task_struct` | `EPROCESS` + `ETHREAD` | threads are the schedulable unit; processes are containers |
| pty + line discipline | ConPTY + `conhost.exe`/Windows Terminal | historically consoles were a special IPC to `csrss.exe` |
| `exit_group` | `NtTerminateProcess` | |

## 15.2 macOS

XNU = Mach microkernel (tasks, threads, ports, VM) + BSD layer (processes, files, signals, POSIX syscalls). `fork`/`execve` work as on Linux (with `posix_spawn` strongly preferred); binaries are **Mach-O** with **dyld** as the loader, using a **dyld shared cache** that pre-links all system frameworks into one giant mapping so startup is fast. Syscalls go via `SYSCALL` with the class of the call (Mach trap vs. BSD syscall) encoded in the high bits of the number. **Code signing** is checked at `exec` and on page-in: a page whose hash doesn't match its signature faults with `SIGKILL`. On Apple Silicon, Rosetta 2 handles x86-64 binaries by AOT-translating them at first launch, so even a "foreign" program becomes a native ARM64 process before Stage 5.

## 15.3 Containers

A container changes *nothing* in Stages 3–12 at the hardware level — it is the same kernel, same page tables, same scheduler. What changes is the **view**: `execve` happens inside a set of **namespaces** (pid: the process sees itself as PID 1 or a small number; mnt: `/` is the image's root filesystem via `pivot_root`/overlayfs; net: its own interfaces; uts, ipc, user), the task is charged to a **cgroup** (CPU shares/quotas enforced by the scheduler via bandwidth control; memory limits enforced by reclaim and the OOM killer at the cgroup level), and a **seccomp** filter plus dropped **capabilities** restrict which syscalls succeed. The "container runtime" (`runc`) is itself just a program that sets these up and then `execve`s your entrypoint. `docker run` ≈ Stage 2 with a very elaborate between-fork-and-exec section.

## 15.4 Virtual machines

Inside a VM, the *guest* kernel does everything in this document believing it owns the hardware. The differences are one layer down:

- **Two-level address translation**: guest virtual → guest physical (guest page tables) → host physical (EPT/NPT, the hypervisor's page tables). A TLB miss can require up to 24 memory accesses instead of 4. The TLB caches the *combined* translation.
- **Privileged operations trap to the hypervisor** (`VM exit`): writing CR3, `HLT`, `CPUID`, some MSRs, and I/O to emulated devices. Each exit costs ~1–2 µs. Paravirtualised devices (**virtio**) minimise exits by batching through shared rings; hardware passthrough (**SR-IOV**, VFIO) eliminates them for I/O.
- **Interrupts** are posted into the guest via APIC virtualisation so most don't need an exit.
- The guest's timer tick, scheduler, and page-cache all still run — they're just now scheduled *by* the host's scheduler as ordinary threads (KVM's vCPU threads). A VM is, to the host, a process whose main loop is `ioctl(KVM_RUN)`.

---

# 16 — The Numbers: Where the Time Actually Goes

Approximate wall-clock budget for `./hello` on an idle modern Linux desktop, warm page cache:

| Stage | Approx. time | Notes |
|---|---|---|
| Keystroke → bash has the line | 1–5 ms | dominated by USB polling interval (1–8 ms) and terminal emulator |
| bash parse + `fork` | 50–150 µs | COW page-table copy of bash's ~10 MB |
| `execve` in kernel | 100–300 µs | path lookup, VMA setup, mapping ld.so |
| ld.so: load libc, relocate, init | 200–500 µs | ~30 syscalls, ~60 page faults, ~1,500 relocations |
| `main` + `puts` + `write` | 5–20 µs | the part you wrote |
| `exit` + teardown | 50–100 µs | unmapping, notifying bash |
| bash wakes, prints prompt | 50–100 µs | |
| **Total CPU-side** | **≈ 0.5–1.2 ms** | `perf stat ./hello` reports ~1M instructions, ~0.4 ms task-clock |
| write → pixels on screen | 10–30 ms | frame timing, GPU, monitor; **you** are the bottleneck |

Cold cache (first run after boot, binary and libc on NVMe): add ~100 µs–1 ms per major fault, so a few extra ms. On spinning disk, tens of ms. Over NFS, hundreds.

Where a *non-trivial* program spends its time is a different question — but it decomposes into exactly the same primitives: instructions retired, cache misses, branch mispredicts, page faults, syscalls, context switches, and blocked-on-I/O time. `perf stat` reports every one of these counters.

---

# 17 — Observing Every Stage Yourself

Every claim in this document is checkable on a Linux box:

| What you want to see | Command |
|---|---|
| The ELF structure (Stage 0) | `readelf -hlSd hello`, `objdump -d hello`, `ldd hello`, `nm -D hello` |
| bash's classification and path lookup (Stage 2) | `type hello`, `hash`, `bash -x -c ./hello`, `strace -f -e trace=process bash -c ./hello` |
| Every syscall, with timing (Stages 3, 4, 9, 12) | `strace -tt -T ./hello`; `strace -c ./hello` for a summary |
| The loader's work (Stage 4) | `LD_DEBUG=libs,reloc,bindings ./hello`; `LD_BIND_NOW=1` to force eager binding |
| The address space (Stage 3/6) | `cat /proc/$PID/maps`, `pmap -x $PID`, `/proc/$PID/smaps` for RSS per mapping |
| Page faults and their kind (Stage 6) | `perf stat -e page-faults,minor-faults,major-faults ./hello`; `perf trace -F all ./hello` |
| Scheduler decisions (Stage 5, 11) | `perf sched record ./hello && perf sched latency`; `/proc/$PID/sched`; `cat /proc/$PID/status` (voluntary/nonvoluntary ctxt switches) |
| Micro-architecture (Stage 7, 8) | `perf stat -e cycles,instructions,branches,branch-misses,L1-dcache-load-misses,LLC-load-misses ./hello`; `perf record` + `perf annotate` for per-instruction hot spots; Intel VTune / AMD uProf for pipeline slot breakdown (top-down analysis) |
| The tty path (Stage 9/10) | `tty`, `stty -a`, `ls -l /proc/$$/fd`, `script` to record a pty session |
| Interrupts (Stage 11) | `watch -n1 cat /proc/interrupts`, `perf stat -e irq:*`, `/proc/softirqs` |
| The kernel's side of a syscall or fault | `perf trace`, `bpftrace -e 'tracepoint:syscalls:sys_enter_write { printf("%s %d\n", comm, args->count); }'`, `ftrace` via `trace-cmd record -p function_graph -g __x64_sys_execve` |
| Single-step the whole thing | `gdb ./hello`, `starti` (stops at ld.so's first instruction), `info proc mappings`, `catch syscall`, `layout asm`, `si` |

Try in particular: `gdb -q ./hello`, then `starti`, then `x/5i $pc` — you are looking at the first instructions of ld.so, exactly where Stage 4 begins, before a single byte of libc has been mapped. `info proc mappings` at that point shows only `hello`, `ld.so`, the stack, and the vDSO. Then `break main`, `continue`, `info proc mappings` again — libc has appeared, and the heap.

---

# 18 — Mental Models to Keep

1. **A process is a kernel data structure, not a thing the CPU knows about.** The CPU knows: current privilege level, CR3, RIP, and the registers. "Which process is running" is the kernel's bookkeeping about which set of those it has installed.
2. **Nothing is loaded until it is touched.** `exec` and `mmap` create promises (VMAs); page faults fulfil them. Memory usage numbers (RSS vs. VSZ) only make sense with this in mind.
3. **Every layer caches to avoid the layer below**: registers ← L1 ← L2 ← L3 ← DRAM ← page cache ← disk; TLB ← page tables; µop cache ← decoders; BTB ← actual branch outcomes; ld.so.cache ← filesystem search; bash's hash table ← `$PATH` walk. Performance work is mostly the study of miss rates.
4. **Control transfers are the expensive events**: syscall, page fault, interrupt, context switch, branch mispredict, VM exit. Each costs 10²–10⁴ cycles; each is designed around (batching, vDSO, huge pages, prefetching, predictors, virtio).
5. **The kernel never trusts user space**: every pointer is validated, every copy goes through `copy_from_user`, every privilege change is by hardware trap, not by convention.
6. **Speculation is everywhere and (almost) invisible**: the branch predictor, out-of-order execution, prefetchers, lazy binding, copy-on-write, and demand paging are all bets that the common case will happen. The program only sees the timing.
7. **"Blocked" means absent.** A waiting program consumes no CPU; a kernel wait-queue entry is the only thing that remembers it exists. Wakeups, not polling, drive everything from keystrokes to network packets.
8. **The same path serves everything.** The keystroke that started the program and the bytes it printed went through the identical pty/tty machinery; the page fault that loaded `main` and the one that backs `malloc` are the same handler; a container, a JIT, and a VM are the same stages with a different actor performing one of them.

---

# 19 — Further Reading

- **Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective*** — chapters 7 (linking), 8 (exceptional control flow), 9 (virtual memory) cover Stages 3–7 at exactly this level.
- **Love, *Linux Kernel Development*** and **Bovet & Cesati, *Understanding the Linux Kernel*** — process creation, scheduling, memory management, syscall path.
- **Drepper, *How To Write Shared Libraries*** and **Levine, *Linkers and Loaders*** — everything ld.so does and why.
- **Drepper, *What Every Programmer Should Know About Memory*** — caches, DRAM, NUMA, in depth.
- **Hennessy & Patterson, *Computer Architecture: A Quantitative Approach*** — the core pipeline and memory hierarchy.
- **Intel SDM Vol. 3 / ARM Architecture Reference Manual** — the authoritative definitions of `SYSCALL`, paging, interrupts, and privilege levels.
- **Linux source**: `fs/exec.c`, `fs/binfmt_elf.c`, `arch/x86/entry/entry_64.S`, `arch/x86/mm/fault.c`, `mm/memory.c`, `kernel/sched/`, `kernel/exit.c`, `drivers/tty/n_tty.c`.
- **glibc source**: `elf/rtld.c`, `elf/dl-load.c`, `elf/dl-reloc.c`, `csu/libc-start.c`, `sysdeps/x86_64/start.S`.
- Gregg, *Systems Performance* — the observability tools in Section 17 and how to reason with them.
