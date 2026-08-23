# Compilers & Programming Languages: How Code Becomes Execution

Assumes [How Computers Work](01-how-computers-work.md) (the target: machine code), [Programming Fundamentals](02-programming-fundamentals.md) (the source: languages), and [Theory of Computation](07-theory-of-computation.md) §2–3 (regular languages and grammars — here they earn their keep). This document demystifies the layer between the code you write and the instructions the CPU runs: compilers, interpreters, JITs, linkers, and runtimes. Few topics pay a bigger everyday dividend — after this, error messages, performance mysteries, and build failures all become legible.

---

## Table of Contents

1. [The Pipeline at a Glance](#1--the-pipeline-at-a-glance)
2. [Lexing: Characters to Tokens](#2--lexing-characters-to-tokens)
3. [Parsing: Tokens to Trees](#3--parsing-tokens-to-trees)
4. [Semantic Analysis: Meaning and Types](#4--semantic-analysis-meaning-and-types)
5. [Intermediate Representations and Optimization](#5--intermediate-representations-and-optimization)
6. [Code Generation, Linking, and Loading](#6--code-generation-linking-and-loading)
7. [Interpreters, VMs, and JITs](#7--interpreters-vms-and-jits)
8. [Runtimes: Garbage Collection and Friends](#8--runtimes-garbage-collection-and-friends)
9. [What This Explains in Daily Life](#9--what-this-explains-in-daily-life)
10. [The Road to Expertise](#10--the-road-to-expertise)

---

# 1 — The Pipeline at a Glance

A compiler is a translator: source language in, target language out, *meaning preserved*. The classic architecture splits into a **front end** (understand the source), a **middle end** (improve it), and a **back end** (emit the target):

```
 source text
     │  lexer            "characters → tokens"           §2
     ▼
 token stream
     │  parser           "tokens → syntax tree"          §3
     ▼
 AST (abstract syntax tree)
     │  semantic analysis "names, types, meaning"        §4
     ▼
 IR (intermediate representation)
     │  optimizer        "same meaning, better code"     §5
     ▼
 optimized IR
     │  code generator   "IR → machine instructions"     §6
     ▼
 object file ──linker──► executable ──loader/OS──► running process
```

The front/back split is a **narrow waist** (the same design as IP in [doc 05 §1](05-computer-networks.md)): N languages × M CPU architectures needs N+M components, not N×M. This is literally how **LLVM** works — Clang (C/C++), Rust, Swift, and Julia are all front ends emitting LLVM IR; every CPU target is a shared back end. GCC has the same shape internally.

Interpreters (§7) share the front half and then *execute* instead of translating — the pipeline is universal; only the last step varies. This is also why "compiled vs. interpreted" is a property of *implementations*, not languages.

# 2 — Lexing: Characters to Tokens

The **lexer** (scanner/tokenizer) chops the character stream into **tokens** — the words of the language:

```
position = initial + rate * 60;
   │
   ▼
IDENT(position) EQUALS IDENT(initial) PLUS IDENT(rate) STAR NUMBER(60) SEMI
```

Token shapes (identifiers, numbers, strings, operators) are **regular languages** — so lexers are DFAs ([doc 07 §2](07-theory-of-computation.md) in production; lexer generators literally compile regexes to automata). The lexer also discards whitespace/comments and records line/column numbers — the coordinates in your error messages. Small but real subtleties live here: maximal munch (`>=` is one token, not two), string escapes, and the historically painful (Python's indentation becomes INDENT/DEDENT tokens via a small stack — a neat trick worth knowing).

# 3 — Parsing: Tokens to Trees

The **parser** assembles tokens into a tree according to the language's **context-free grammar** ([doc 07 §3](07-theory-of-computation.md) — nesting needs a stack). For `initial + rate * 60`:

```
        (+)
       /   \
  initial   (*)
           /   \
        rate    60
```

The tree *is* the meaning-structure: precedence (why * binds tighter) and associativity are encoded in the grammar, and once the tree exists, the flat text's ambiguities are gone. The **AST** (abstract syntax tree) drops ceremonial tokens (parens, semicolons) and keeps structure — the data structure every later stage, plus formatters (Black, Prettier), linters (ESLint), refactoring tools, and syntax highlighters, actually operates on. "Programs manipulating programs" (docs 01 §5.1, 07 §4) is concretely: *programs manipulating ASTs*.

The one technique worth knowing by name: **recursive descent** — one function per grammar rule, each consuming its part of the token stream and calling the others; the grammar's recursion becomes code recursion (doc 02 §5's "recursive shapes want recursive code," perfected). Most production compilers (Clang, Go, Rust, TypeScript) are hand-written recursive descent, usually with **Pratt/precedence-climbing** for expressions — an elegant loop that handles all binary-operator precedence in ~30 lines. (Parser *generators* — yacc/ANTLR — compile grammars to table-driven LR/LL parsers; important history and still used, but hand-written won in practice for error messages' sake.) A **syntax error** is precisely: the parser holds a token that no grammar rule allows next. Good compilers then *recover* (resynchronize at the next `;` or `}`) to report many errors per run — why one missing brace can produce a cascade of bogus follow-on errors: the recovery guessed wrong.

# 4 — Semantic Analysis: Meaning and Types

Grammar can't express "declared before use" or "types must match" (context-sensitive — [doc 07 §3](07-theory-of-computation.md) said grammars stop here; this pass is the consequence). So the compiler walks the AST and:

- **Resolves names**: builds **symbol tables** mapping each identifier to its declaration, respecting **scope** (doc 02 §2.2's rules, implemented as a stack of tables — enter a block, push; leave, pop). Errors born here: "undefined variable," "duplicate declaration."
- **Checks types**: computes every expression's type bottom-up from the tree, verifying each operation's operands (doc 02 §9's static typing, mechanically realized). **Type inference** (Hindley-Milner in ML/Haskell/Rust; local inference in Go/Kotlin/TS) runs the same computation with unification solving for unknowns — the compiler deduces the annotations you didn't write. Errors born here: "expected int, found string" — and the notorious multi-page generic-type errors are this pass showing you its work.
- Enforces the language's other static rules: definite assignment (Java), exhaustive matches (Rust/Kotlin), and — the state of the art — Rust's **borrow checker**: ownership/lifetime analysis proving memory safety at this stage (doc 02 §11's third regime, located precisely: it's semantic analysis).

After this pass the program is *known meaningful*; everything later is translation and improvement.

# 5 — Intermediate Representations and Optimization

The AST lowers to an **IR** — typically flat, explicit, three-address-style instructions in **SSA form** (static single assignment: every variable assigned exactly once; data flow becomes explicit edges), because SSA makes optimizations simple to state and compose. Then the middle end applies dozens of passes, each a meaning-preserving rewrite. The greatest hits — worth knowing because they explain observed behavior:

- **Constant folding/propagation** (`60 * 60 * 24` → `86400` at compile time) and **dead-code elimination** (unreachable or unused → gone; why your debugger says a variable is "optimized out").
- **Common subexpression elimination**, **strength reduction** (`x*8` → `x<<3`), **loop-invariant code motion** (hoist computations out of loops).
- **Inlining** — replace a call with the callee's body. The *keystone* optimization: it erases call overhead *and exposes the callee to all the other passes in context*. This is why doc 02 could say "small functions cost nothing": the abstraction is dissolved right here.
- **Loop transformations**: unrolling, and **vectorization** (compile the loop to SIMD instructions — doc 01 §7.4, delivered automatically when the compiler can prove iterations independent).
- **Register allocation** (technically back end): map unbounded IR temporaries onto the CPU's ~16 registers — graph coloring on the interference graph, an NP-hard problem ([doc 07 §7](07-theory-of-computation.md)) solved heuristically in every compilation, billions of times a day.

Two professional truths: optimization is why **`-O0` vs `-O2` can be 10×+** (and why you benchmark optimized builds only), and why **optimized code debugs weirdly** (reordered, merged, vanished lines). And the compiler may only transform within the language's rules — in C/C++, **undefined behavior** (overflow, out-of-bounds) is assumed *not to happen*, so the optimizer can and does delete your overflow check if the check itself relies on UB: a genuinely important, widely-misunderstood source of bugs ("the compiler broke my code" — no, the contract did).

# 6 — Code Generation, Linking, and Loading

- **Codegen**: IR → target instructions (doc 01 §6's assembly), via instruction selection + scheduling + the register allocation above, emitted into an **object file** (`.o`) — machine code with holes: unresolved references to symbols defined elsewhere (`printf`, your other files' functions).
- **Linking** patches the holes. **Static**: copy everything into one self-contained executable. **Dynamic**: leave references to shared libraries (`.so`/`.dll`), resolved at load time — one libc on disk and in RAM for all processes ([OS doc §5.4](operating-systems.md)'s shared mappings). This layer is the home of famous pain: "undefined symbol/unresolved external" (you declared but never linked the definition), "DLL not found," version skew ("DLL hell" — solved-ish by versioned sonames, vendoring, static linking's comeback in Go/Rust, and ultimately containers, [../infra/docker-deep-dive.md](../infra/docker-deep-dive.md)).
- **Loading**: the OS `exec` maps the executable and its libraries into a fresh address space ([OS doc Parts 3, 5](operating-systems.md)) and jumps to the entry point. Build systems (make, Gradle, Cargo...) orchestrate all of the above over many files — their core is a dependency DAG + topological sort ([doc 04 §6.1](04-algorithms.md)) with timestamps/hashes for incremental rebuilds.

# 7 — Interpreters, VMs, and JITs

The other half of the execution spectrum — a ladder of increasingly clever "execute now" strategies:

1. **Tree-walking interpreter**: recursively evaluate the AST directly. Simplest possible; slow (pointer-chasing per node — doc 01 §7.3); how Ruby began and how most course projects start.
2. **Bytecode VM**: compile the AST to a compact instruction set for an imaginary machine (**bytecode**), then run a dispatch loop over it. This is **CPython** (`.pyc` files are cached bytecode), Java (`.class` for the **JVM**), C#/.NET, Lua. Portability falls out: compile once, run on any machine with the VM — "write once, run anywhere" is this architecture.
3. **JIT (just-in-time) compilation**: the VM *watches* execution, finds hot code, and compiles it to real machine code *at runtime* — with a superpower ahead-of-time compilers lack: **runtime information**. It compiles for the types actually flowing ("this `+` has only ever seen integers"), inlines through dynamic dispatch, and **speculates**: emit fast code guarded by cheap checks, **deoptimize** (fall back to bytecode) if an assumption breaks. This is V8 (JavaScript), the JVM's HotSpot (tiered: interpret → quick JIT → optimizing JIT), PyPy. It's why JavaScript went from 100× slower than C to often within 2–3× — and why JIT'd code needs **warm-up** (benchmarks that ignore it measure the interpreter), and why microbenchmarking JITs is famously treacherous (the JIT may delete your unused benchmark loop entirely — dead code elimination, §5, at runtime).
4. **AOT native compilation** (C, C++, Go, Rust): all decisions at build time; fastest startup, no runtime surprises; profile-guided optimization (PGO) recovers some of the JIT's runtime-information advantage by feeding a profiling run back into the compiler.

CPython specifically: bytecode VM, historically no JIT (plus the GIL — [doc 02 §6](02-programming-fundamentals.md)); the traditional escape hatch is dropping to C (NumPy et al.) — "Python is fast" usually means "Python orchestrates fast C" — with recent CPython versions now adding a JIT and free-threading of their own.

# 8 — Runtimes: Garbage Collection and Friends

The **runtime** is the standing machinery under your running program: memory allocator, garbage collector, scheduler for green threads/goroutines, exception unwinding, reflection metadata. The big one is **GC** (doc 02 §11's regime 2, now mechanized):

- **Reachability** is the definition of "garbage": start from **roots** (stacks, globals, registers) and trace pointers; anything unreached is unreachable *forever* (no pointer to it exists → doc 05's "can't even name it" logic) and reclaimable.
- **Mark-and-sweep**: trace and mark live objects, sweep the rest. **Copying/compacting** variants move survivors together — defragmenting the heap and making allocation a pointer bump (faster than malloc).
- **Generational GC** — the key optimization, from the empirical *generational hypothesis*: most objects die young. Allocate into a small **nursery**, collect it frequently and cheaply (survivors are few — copying cost scales with *live* data, not garbage); promote long-survivors to an old generation collected rarely. This is the JVM's and V8's architecture.
- **Reference counting** (CPython, Swift ARC): each object counts its referrers; zero → free immediately. Prompt and simple; can't collect **cycles** (a↔b forever count 1 — CPython runs a cycle detector on top), and count updates cost on every pointer write.
- The engineering axis: **pause times**. Naive GC stops the world during collection ("GC pause" — the latency spike in your server's p99, [../distributed-systems/07-tail-latency-and-ha-architecture.md](../distributed-systems/07-tail-latency-and-ha-architecture.md)); modern collectors (Go's concurrent GC, JVM's ZGC/Shenandoah) run concurrently with the program using write barriers, trading throughput for sub-millisecond pauses. Tuning GC = choosing a point on the throughput/latency/memory triangle.
- What GC does *not* solve (worth repeating from doc 02): reachable-but-useless data still accumulates — the growing cache, the forgotten listener. "Managed language memory leak" almost always means *semantic* leak, found with a heap profiler, not a GC bug.

# 9 — What This Explains in Daily Life

The payoff section — phenomena you've seen, now with mechanisms:

| You observe | The mechanism |
|---|---|
| "SyntaxError: unexpected token" at a weird location | parser held a token no rule allows (§3); the *cause* may be lines earlier (unclosed bracket) — recovery guessed wrong |
| One missing brace → 50 errors | error recovery cascade (§3) |
| "undefined variable" vs "type mismatch" — different error flavors | different passes: name resolution vs type checking (§4) |
| Debug build slow, release build fast; debugger acts drunk in release | `-O0` vs `-O2`; optimizations reorder/delete (§5) |
| "undefined symbol" at build time; "DLL not found" at run time | linking vs loading (§6) |
| Java/JS server fast only after warm-up | JIT tiers compiling hot paths (§7) |
| p99 latency spikes in a GC'd service | GC pauses (§8) |
| Python slow in loops, fast with NumPy | bytecode dispatch per operation vs one call into compiled C (§7) |
| Formatter/linter "understands" your code | it parsed to the same AST the compiler uses (§3) |
| Rust compile times vs Go's | how much §4–5 work each language buys per compile (borrow checking, monomorphized generics, LLVM optimization) |

# 10 — The Road to Expertise

## 10.1 Books

1. **Crafting Interpreters** (Nystrom) — *free online* (craftinginterpreters.com). Build a language twice: tree-walking interpreter in Java, then a bytecode VM with GC in C. The best-written technical book of its generation; the canonical path into this field. **If you do one thing from this document, do this book.**
2. **Writing an Interpreter in Go** / **Writing a Compiler in Go** (Ball) — the same journey, terser, in Go.
3. **Engineering a Compiler** (Cooper & Torczon) — the modern serious textbook (friendlier than the venerable "Dragon Book" — *Compilers: Principles, Techniques & Tools* — which remains the classic reference).
4. **Types and Programming Languages** (Pierce, "TAPL") — the type-theory canon, when §4 hooks you.
5. LLVM's **Kaleidoscope tutorial** (free) — build a JIT'd language on LLVM in an afternoon; see the narrow waist from inside.

## 10.2 Doing

1. **Calculator interpreter** (a weekend): lexer by hand, Pratt parser for `+ - * / ( )` precedence, tree-walk evaluation. Touches §2–3–7 and demystifies the whole front end.
2. **Do Crafting Interpreters end to end** (the spine project — a functioning language with closures, classes, and a mark-sweep GC, from scratch).
3. Inspect the real thing: Python's `dis.dis(f)` (bytecode of your function — §7 made visible), `ast.dump(ast.parse(src))` (§3), godbolt.org at `-O0` vs `-O2` (§5's passes, diffed live).
4. Write one **optimization pass** over your toy language's AST/IR: constant folding — 50 lines, the middle end demystified.
5. Cause and then fix each classic build error on purpose: undefined symbol, duplicate symbol, missing DLL/so (§6).
6. Expert tier: the LLVM Kaleidoscope tutorial; then a tracing JIT experiment, or contribute a diagnostics improvement to a real compiler (they famously welcome them).

## 10.3 Ideas to retain forever

1. **One pipeline, everywhere**: lex → parse → analyze → (optimize) → run-or-emit; every language tool is a stop on it.
2. **The theory is load-bearing**: tokens are regular, syntax is context-free, meaning is neither — that's *why* lexer, parser, and semantic analysis are separate passes ([doc 07](07-theory-of-computation.md) cashed in).
3. **The AST is the program** — text is a serialization; all serious tools (compilers, formatters, linters, refactorers) work on the tree.
4. **The narrow-waist IR** (LLVM) turns N×M into N+M — the internet's trick, applied to compilation.
5. **Optimizers are meaning-preserving rewriters** — inlining is the keystone; UB is a *contract* the optimizer exploits; benchmark only optimized builds.
6. **Compiled vs interpreted is an implementation spectrum** (tree-walk → bytecode → JIT → AOT), not a language property — and JITs buy speed with runtime knowledge, paid for in warm-up and deopts.
7. **GC = reachability tracing**, made fast by "most objects die young" (generational), made low-latency by concurrency; reachable-but-useless is still a leak.
8. **Errors are pass-shaped**: syntax vs name vs type vs link vs load — reading the *kind* of error tells you which machinery objected, which tells you where to look.
9. **Register allocation is NP-hard and runs a billion times a day** — heuristics in production, theory as diagnosis (doc 07's lesson, incarnate).
10. **Building a toy language is the highest-ROI project in CS** — it simultaneously exercises recursion, trees, hash tables (symbol tables), automata, memory management, and design.

---

*Next: [Software Engineering](09-software-engineering.md) — from writing programs to building systems that survive contact with time, teams, and users.*
