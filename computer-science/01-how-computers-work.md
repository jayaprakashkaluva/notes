# How Computers Work: From Electricity to Executable

The bottom layer of the whole computer science stack. No prior knowledge assumed. By the end you'll understand how switches become logic, logic becomes arithmetic, arithmetic becomes a CPU, and a CPU runs programs.

---

## Table of Contents

1. [Information and Binary](#1--information-and-binary)
2. [Representing Everything in Bits](#2--representing-everything-in-bits)
3. [From Switches to Logic Gates](#3--from-switches-to-logic-gates)
4. [From Gates to Arithmetic and Memory](#4--from-gates-to-arithmetic-and-memory)
5. [The CPU: A Machine That Follows Instructions](#5--the-cpu-a-machine-that-follows-instructions)
6. [Assembly Language](#6--assembly-language)
7. [Making CPUs Fast](#7--making-cpus-fast-pipelines-caches-prediction)
8. [The Memory System](#8--the-memory-system)
9. [Storage, Buses, Peripherals](#9--storage-buses-peripherals)
10. [The Road to Expertise](#10--the-road-to-expertise)

---

# 1 — Information and Binary

## 1.1 Why two symbols

A computer is built from billions of microscopic switches (**transistors**), each either conducting electricity or not. Two reliably distinguishable states → two symbols: **0** and **1**. One such symbol is a **bit** (binary digit).

Why not ten states per switch to match decimal? Reliability. Distinguishing "on vs. off" survives noise, heat, and manufacturing variation; distinguishing ten voltage levels doesn't. Everything digital is built on this bet: *two states, perfectly copied, billions of times*.

## 1.2 Counting in binary

Decimal is positional: 342 = 3×100 + 4×10 + 2×1 — each position worth 10× the one to its right. Binary is identical with 2 instead of 10:

```
binary 1 0 1 1
       | | | +-- 1 × 1   = 1
       | | +---- 1 × 2   = 2
       | +------ 0 × 4   = 0
       +-------- 1 × 8   = 8
                           -- = 11 (decimal)
```

With *n* bits you can represent 2ⁿ distinct values: 8 bits → 256, 32 bits → ~4.3 billion, 64 bits → ~1.8 × 10¹⁹.

- A **byte** = 8 bits, the standard addressable unit of memory.
- **Hexadecimal** (base 16, digits 0–9 then A–F) is a compact human notation for binary — one hex digit = exactly 4 bits. `0xFF` = `11111111` = 255. When you see `0x7fff5a8b`, that's just bits written for humans.
- KB/MB/GB/TB: powers of ten (10³, 10⁶, ...) in marketing, powers of two (KiB = 1024, MiB = 1024², ...) in most software — the discrepancy is why a "1 TB" disk shows up as ~931 "GB."

## 1.3 Binary arithmetic

Same grade-school algorithms, base 2. 0+0=0, 0+1=1, 1+1=10 (write 0, carry 1):

```
    1 0 1 1   (11)
  + 0 1 1 0   ( 6)
  ---------
  1 0 0 0 1   (17)
```

This is worth doing by hand once — Section 4 builds a circuit that does exactly this.

---

# 2 — Representing Everything in Bits

Bits have no inherent meaning. **Meaning is assigned by convention** — the same 8 bits `01000001` are the number 65, the letter `A`, or a bluish pixel component, depending on what the program treats them as. The major conventions:

## 2.1 Integers

- **Unsigned**: straight binary. 32 bits → 0 to 4,294,967,295.
- **Signed — two's complement** (universal): the top bit's weight is *negative* (for 8 bits: −128). So `11111111` = −128+64+32+16+8+4+2+1 = −1. Why this scheme won: the *same* addition circuit works for signed and unsigned; there's a single zero; negation is "flip all bits, add 1."
- **Overflow**: fixed bits → wraparound. 8-bit unsigned: 255 + 1 = 0. Signed: 127 + 1 = −128. Real consequences: the Ariane 5 rocket exploded (1996) partly due to a numeric conversion overflow; a YouTube video (Gangnam Style) once exceeded the 32-bit view counter. Languages either wrap (C — silently!), trap, or auto-grow integers (Python).

## 2.2 Real numbers: floating point (IEEE 754)

Scientific notation in binary: a number = sign × 1.fraction × 2^exponent, packed into 32 bits (`float`) or 64 bits (`double`: 1 sign + 11 exponent + 52 fraction bits).

The essential facts every programmer eventually learns the hard way:

- Floats are **samples, not the real line**: most decimals aren't exactly representable. 0.1 in binary is infinite (like 1/3 in decimal), so `0.1 + 0.2 == 0.3` is **false** in virtually every language (it's 0.30000000000000004). Compare with a tolerance; never use floats for money (use integer cents or decimal types).
- Doubles have ~15–16 significant decimal digits; adding a tiny number to a huge one loses the tiny one entirely; subtracting nearly-equal numbers destroys precision ("catastrophic cancellation").
- Special values: `+∞`, `−∞`, and `NaN` (not-a-number, from 0/0 etc.; NaN ≠ NaN by definition — a famous interview trap).

## 2.3 Text

- **ASCII** (1963): 7 bits, 128 characters — English letters, digits, punctuation, control codes. `A` = 65, `a` = 97, `0` = 48.
- **Unicode**: one number (*code point*) for every character in every language, plus emoji — over 150,000 assigned (`U+1F600` = 😀).
- **UTF-8**: the dominant *encoding* of code points into bytes. Brilliant design: ASCII characters take 1 byte unchanged (perfect backward compatibility); other characters take 2–4 bytes, self-synchronizing. The web is >98% UTF-8. When you see garbled text (`Ã©` for `é`), some program decoded bytes with the wrong convention — "mojibake."
- Consequence: "length of a string" is ambiguous (bytes? code points? on-screen characters — which may combine several code points, like é = e + ́ , or family emoji glued from multiple emoji?). Expert answer: know which one your language's `len()` counts.

## 2.4 Images, sound, and everything else

- **Images**: a grid of pixels; each pixel typically 3 bytes — red, green, blue intensities 0–255 (plus alpha for transparency). Raw is huge (a 12-megapixel photo ≈ 36 MB), so compression: **PNG** (lossless — exact bits back), **JPEG** (lossy — discards detail human eyes miss, ~10× smaller).
- **Sound**: measure air-pressure ~44,100 times/second (sampling), store each measurement as a 16-bit number. That's a WAV/CD; MP3/AAC compress lossily by dropping what psychoacoustics says you can't hear.
- **Video**: images over time + motion-based compression (store *differences* between frames).
- The generalization: **digitization** = sample the analog world at sufficient resolution, then it's all just bits, all copyable perfectly, all processable by the same machine. This is why one device replaced your camera, walkman, TV, mail, and library.

---

# 3 — From Switches to Logic Gates

## 3.1 The transistor

A **transistor** is an electrically controlled switch: voltage on its *gate* terminal determines whether current can flow between its other two terminals. That's all you need to know for CS purposes (the physics — doped silicon, field effects — belongs to electrical engineering). A modern CPU packs >10 billion of them, each a few dozen atoms wide, printed by photolithography.

## 3.2 Logic gates

Wire a few transistors together and you get **gates** — circuits computing tiny functions of bits:

| Gate | Output is 1 when... | |
|---|---|---|
| **NOT** | input is 0 | inverts |
| **AND** | both inputs are 1 | |
| **OR** | at least one input is 1 | |
| **XOR** | inputs differ | "exclusive or" |
| **NAND** | not both 1 | NOT(AND) — the universal gate |

**NAND alone can build every other gate** (NOT is NAND of a signal with itself; AND is NAND then NOT; etc.). So in principle: *the entire computer is NAND gates* — billions of copies of one simple part. This is the first great abstraction collapse in the stack.

**Boolean algebra** (George Boole, 1847 — a century before computers) is the math of these operations: laws like De Morgan's (`NOT(A AND B) = NOT A OR NOT B`) let engineers simplify circuits, and reappear verbatim when you refactor `if` conditions in code.

## 3.3 Combinational circuits

Feed gate outputs into gate inputs (no loops) and you compute any fixed boolean function: comparators (are these two 8-bit numbers equal? just XOR each bit pair and NOR the results), multiplexers (select 1 of N inputs — hardware's `if`), decoders (turn a binary number into "activate line #n" — how memory addresses select a physical row).

---

# 4 — From Gates to Arithmetic and Memory

## 4.1 The adder

Adding two bits: sum = A XOR B, carry = A AND B (check the table: 1+1 = sum 0 carry 1 ✓). That's a **half adder**; handle an incoming carry too and it's a **full adder** (about 5 gates). Chain 64 of them, carry rippling left like grade-school addition, and you have a 64-bit adder. Subtraction is free thanks to two's complement (add the negation). Multipliers and dividers are bigger compositions of the same ideas.

The **ALU** (arithmetic logic unit) bundles adder, logic ops, shifter, comparator — with control bits choosing which result to output. This is the "compute" core of the CPU.

## 4.2 Memory: circuits that remember

Everything so far is stateless. Feed a gate's output *back toward its input* and you get bistability: a **latch/flip-flop** — a circuit that stays in whichever of two states you put it, i.e., **stores one bit**. Add a clock signal so it only updates at ticks, and you can build:

- **Registers**: a row of 64 flip-flops = one 64-bit storage cell inside the CPU. Blazing fast, tiny capacity.
- **SRAM** (~6 transistors/bit): CPU caches.
- **DRAM** (1 transistor + 1 capacitor/bit): main memory (RAM). Denser and cheaper, but slower, and the capacitor leaks — every cell must be re-read and rewritten thousands of times a second ("refresh"). That's the D: *dynamic*.

## 4.3 The clock

A crystal oscillator ticks billions of times per second (3 GHz = 3 billion ticks/s). On each tick, all flip-flops simultaneously capture their inputs; between ticks, signals ripple through gate networks. The clock is the metronome that turns a soup of gates into an orderly sequence of steps — and "clock speed" measures exactly this.

---

# 5 — The CPU: A Machine That Follows Instructions

## 5.1 The von Neumann architecture

The design (1945) behind virtually every computer: **programs are data**. Instructions live in the same memory as the data they operate on, as bytes. Consequences: computers are reprogrammable by loading different bytes (no rewiring — this is what makes software *soft*); programs can manipulate programs (compilers, OS loaders — see [Compilers](08-compilers-and-languages.md)); and code/data confusion becomes possible (the root of many security exploits — see [Security](10-security-and-cryptography.md)).

Components: the **ALU** (§4.1), **registers** (§4.2) including the special **program counter (PC)** holding the address of the next instruction, the **control unit** (decodes instructions and orchestrates everything), and **memory**.

## 5.2 The fetch-decode-execute cycle

Forever:

1. **Fetch** the bytes at address PC from memory.
2. **Decode** them: the bit pattern encodes an operation (add? load? jump?) and its operands (which registers? what address?).
3. **Execute**: route values through the ALU or memory.
4. Advance PC to the next instruction — *unless* the instruction was a **jump**, which sets PC somewhere else.

That's the whole machine. Every program ever run — spreadsheets, games, neural networks — is this loop grinding through instructions at billions per second.

## 5.3 The instruction set (ISA)

The **instruction set architecture** is the CPU's vocabulary — the contract between hardware and software. Categories:

- **Data movement**: load from memory to register; store back; move between registers.
- **Arithmetic/logic**: add, subtract, multiply, AND, OR, shift... (registers in, register out).
- **Control flow**: unconditional jump; **conditional branch** ("jump if the last comparison said equal") — this is how `if` and loops exist; **call/return** for functions.

Two ISAs matter today: **x86-64** (Intel/AMD — PCs, most servers; a sprawling vocabulary accumulated since 1978) and **ARM64** (phones, Apple Silicon, growing server share; a cleaner *RISC* design — fewer, simpler, fixed-size instructions, betting that compilers compose simple ops better than hardware implements complex ones. History judged RISC right: modern x86 chips internally *translate* x86 instructions into RISC-like micro-ops).

## 5.4 Why conditional branching = universality

A machine that can compute, remember, and *branch on results* can express any computation expressible at all (a claim made precise in [Theory of Computation](07-theory-of-computation.md) — Turing completeness). Loops are just "branch backward"; `if/else` is "branch forward." Everything above — every language, every app — compiles down to straight-line arithmetic plus conditional jumps.

---

# 6 — Assembly Language

Machine code is raw bytes; **assembly** is its human-readable notation, one line per instruction. You don't need to write it fluently — but *reading* a little permanently demystifies what programs are. Here's `x = a + b` and a loop, in simplified x86-64:

```asm
; x = a + b
mov  rax, [a]        ; load memory at address 'a' into register rax
add  rax, [b]        ; rax = rax + memory at 'b'
mov  [x], rax        ; store rax to memory at 'x'

; for (i = 0; i < 10; i++) total += i;
     mov  rcx, 0          ; i = 0        (rcx holds i)
     mov  rax, 0          ; total = 0    (rax holds total)
loop_top:
     cmp  rcx, 10         ; compare i with 10
     jge  done            ; jump to 'done' if i >= 10
     add  rax, rcx        ; total += i
     inc  rcx             ; i++
     jmp  loop_top        ; back to the top
done:
```

Note what vanished: there are no variables (only registers and addresses), no loops (only compare-and-jump), no types (only bit patterns). All of those are fictions maintained by compilers ([document 9](08-compilers-and-languages.md)).

**Function calls** use the **stack** (a region of memory + the `rsp` register): `call` pushes the return address and jumps; the function pushes its local variables; `ret` pops the return address and jumps back. Deeply recursive calls stack frames until they overflow — this is the literal mechanics behind the "stack" in "stack trace" and "Stack Overflow." Calling conventions fix which registers pass arguments (so code compiled separately can interoperate).

Try it: paste a tiny C function into **godbolt.org** (Compiler Explorer) and read the assembly it becomes. Watching `if` turn into `cmp`/`jne` closes the loop between this document and real tools.

---

# 7 — Making CPUs Fast: Pipelines, Caches, Prediction

A naive CPU finishes one instruction before starting the next. Modern CPUs are ~within 10× of physics' limits through three big ideas — which also *leak upward* into how you must write fast software.

## 7.1 Pipelining and superscalar execution

Like an assembly line: while instruction 1 executes, instruction 2 decodes and instruction 3 fetches — stages overlap (modern chips: 14–20 stages). **Superscalar**: multiple parallel pipelines — 4–8 instructions can *complete every cycle*. **Out-of-order execution**: the CPU dynamically reorders independent instructions to keep pipelines full, while preserving the *illusion* of sequential order.

## 7.2 Branch prediction and speculation

A pipeline is poison for branches: which instruction is next isn't known until the compare resolves, stalling everything. So CPUs **predict** the branch direction (learning from history — modern predictors exceed 95% accuracy) and **speculatively execute** down the guessed path, discarding the work if wrong. A mispredict costs ~15–20 cycles.

Software consequence: predictable branches are nearly free; random ones are expensive (famous StackOverflow question: sorting an array first made summing its large elements 6× faster — the branch became predictable). Security consequence: speculative side effects leak secrets — Spectre/Meltdown (2018), covered in the OS and Security documents.

## 7.3 Caches

RAM is ~100–200 cycles away — an eternity when you retire 4 instructions/cycle. So CPUs keep small, fast SRAM **caches** of recently used memory: L1 (~32–64 KB, ~4 cycles), L2 (~1 MB, ~14), L3 (tens of MB, shared, ~40–70). Caches work because programs exhibit **locality**: *temporal* (recently used → soon reused) and *spatial* (used address → neighbors next; caches fetch 64-byte **lines** at a time, so touching one byte prefetches its 63 neighbors).

Software consequences (the deepest performance lessons in this document):

- **Sequential array traversal is enormously faster than pointer-chasing** (linked lists, scattered objects) — every element is a spatial-locality hit, and the hardware prefetcher streams ahead of you.
- Row-major vs. column-major traversal of a 2D array can differ 10×+ — same arithmetic, different locality.
- This is *the* reason the [Data Structures](03-data-structures.md) document keeps saying "arrays win in practice despite big-O ties."

## 7.4 Multicore and SIMD

Around 2005, clock speeds hit a power/heat wall (~3–5 GHz ever since). The transistor budget went instead to **multiple cores** — full CPUs on one chip, sharing L3 and RAM. Free-lunch sequential speedups ended; software must now be parallel to use the hardware, which is why concurrency (OS doc, Part 6) went from niche to mandatory. Also on the chip: **SIMD** vector units (one instruction operates on 8–16 values at once — the workhorse of multimedia and numerical code), and increasingly, specialized accelerators. **GPUs** take parallelism to the extreme: thousands of simple cores running the same operation across huge data arrays — built for graphics, now the engine of deep learning.

---

# 8 — The Memory System

Assembling the full hierarchy (each level ~10–100× bigger and slower than the one above):

| Level | Size | Latency | Notes |
|---|---|---|---|
| Registers | ~1 KB | 0 cycles | inside the pipelines |
| L1 cache | 32–64 KB/core | ~4 cycles | split instruction/data |
| L2 cache | 0.5–2 MB/core | ~14 cycles | |
| L3 cache | 8–96 MB shared | ~40–70 cycles | |
| RAM (DRAM) | 8–512 GB | ~100–200 cycles | volatile |
| SSD | 0.5–8 TB | ~100 µs | persistent |
| HDD | up to ~20 TB | ~10 ms | mechanical |

Two more facts complete the picture:

- **Virtual memory**: addresses your program uses are translated per-access by the MMU via page tables — the mechanism belongs to hardware, the *policy* to the OS. Covered thoroughly in [Operating Systems Part 5](operating-systems.md#part-5--memory-management-and-virtual-memory).
- **Cache coherence**: on multicore, each core's private caches could hold stale copies of the same line. The **MESI** protocol keeps them consistent by messaging between caches — invisible to correctness, very visible to performance: two cores repeatedly writing the same (or even *adjacent* — "false sharing") data ping-pong the cache line between them, and a "parallel" program crawls. Root cause of many real multicore mysteries.

---

# 9 — Storage, Buses, Peripherals

- **HDD**: spinning magnetic platters + moving arm. Sequential reads fine; each random access costs a ~10 ms mechanical seek. Shaped decades of software design (databases, file systems minimize seeks).
- **SSD**: NAND flash, no moving parts, ~100 µs access, massive internal parallelism. Quirks that leak upward: writes go page-at-a-time but erases only block-at-a-time, so drives run a background garbage-collecting **flash translation layer**; cells wear out (wear leveling spreads writes); sustained write performance can fall off a cliff when the drive fills.
- **Buses** connect everything: PCIe (GPUs, NVMe SSDs — point-to-point lanes, ~4 GB/s per lane in v5), SATA (older disks), USB (peripherals), memory channels (RAM). "NVMe" is the protocol letting SSDs speak PCIe directly with deep command queues — built for flash's parallelism rather than inherited from disks.
- **How devices and CPUs interact** — memory-mapped I/O, interrupts, DMA — is covered in [Operating Systems Part 8](operating-systems.md#part-8--inputoutput-and-device-drivers).

---

# 10 — The Road to Expertise

## 10.1 The layer map, restated

```
  transistor  -> gate (NAND)  -> adder/latch -> ALU/registers/RAM
  -> CPU (fetch-decode-execute) -> ISA/assembly
  -> [compilers translate languages down to this]      -> doc 08
  -> [the OS multiplexes and protects this]            -> operating-systems.md
```

## 10.2 Books and courses

1. **Code: The Hidden Language of Computer Hardware and Software** — Petzold. The gentlest, most delightful bottom-up build (relays to CPUs). Ideal absolute-first book.
2. **The Elements of Computing Systems** (a.k.a. **Nand2Tetris**, nand2tetris.org) — *build* a computer from NAND gates in a simulator, then an assembler, VM, compiler, and OS for it. The single most transformative hands-on course for this material. Free.
3. **Computer Systems: A Programmer's Perspective (CS:APP)** — Bryant & O'Hallaron. The hardware/software boundary from the programmer's side: data representation, assembly, caches, linking. The standard serious text; pairs with C.
4. **Computer Organization and Design** — Patterson & Hennessy. The classic architecture textbook (RISC-V edition recommended); deeper hardware than CS:APP.
5. **Computer Architecture: A Quantitative Approach** — Hennessy & Patterson. Graduate/expert tier: how architects actually evaluate designs.

## 10.3 Hands-on milestones

1. Convert numbers by hand: decimal ↔ binary ↔ hex. Add two binary numbers. Negate via two's complement.
2. Explain to someone why `0.1 + 0.2 != 0.3`, and why `é` sometimes renders as `Ã©`.
3. Do **Nand2Tetris Part I** (gates → ALU → CPU). ~40 hours, permanently changes how you see computers.
4. On **godbolt.org**, write a 5-line C function; identify the loop, the branch, the register allocation in its assembly output.
5. Write the two-loop-orders matrix traversal benchmark; measure the cache effect yourself.
6. Expert tier: write a simple 8-bit CPU emulator (CHIP-8 is the classic starter project — a weekend well spent), or an emulator for the Game Boy (a month).

## 10.4 Ideas to retain forever

1. **It's all bits; meaning is convention** — integers, text, pixels, and code are interpretations, not substances.
2. **NAND is enough** — the entire machine is one simple part, composed.
3. **Programs are data** (von Neumann) — hence compilers, loaders, and code-injection attacks.
4. **The CPU is dumb and fast**: fetch-decode-execute plus conditional jumps = everything.
5. **Fixed-width numbers overflow; floats approximate** — silent numeric surprises are a permanent fact of life.
6. **The memory hierarchy and locality govern real performance** — caches reward arrays and sequential access, punish pointer-chasing and false sharing.
7. **Branch prediction rewards predictability**; speculation has correctness *and* security consequences.
8. **The free lunch is over**: clocks stopped scaling in ~2005; performance now comes from cores, vectors, and cache-friendly code.

---

*Next: [Programming Fundamentals](02-programming-fundamentals.md) — the craft of instructing this machine at a humane level of abstraction.*
