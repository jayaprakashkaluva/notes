# Programming Fundamentals: The Craft of Instructing Machines

How humans express computation. Assumes [How Computers Work](01-how-computers-work.md) — you know the machine underneath runs dumb instructions very fast. This document is about the humane layers we build on top, and it doubles as a map of programming *concepts across all languages*, not a tutorial for one language (pick Python to practice with; every concept here appears in it).

---

## Table of Contents

1. [What a Program Is](#1--what-a-program-is)
2. [Values, Types, and Variables](#2--values-types-and-variables)
3. [Control Flow](#3--control-flow)
4. [Functions and Abstraction](#4--functions-and-abstraction)
5. [Recursion](#5--recursion)
6. [State, Mutation, and Why They're Dangerous](#6--state-mutation-and-why-theyre-dangerous)
7. [Object-Oriented Programming](#7--object-oriented-programming)
8. [Functional Programming](#8--functional-programming)
9. [Type Systems](#9--type-systems)
10. [Errors and How Languages Handle Them](#10--errors-and-how-languages-handle-them)
11. [Memory Models Across Languages](#11--memory-models-across-languages)
12. [The Language Landscape](#12--the-language-landscape)
13. [The Road to Expertise](#13--the-road-to-expertise)

---

# 1 — What a Program Is

A program is a precise description of a computation, written in a **programming language** — a formal notation with exact rules, translated (by a compiler or interpreter — [document 08](08-compilers-and-languages.md)) into the machine instructions of document 01.

The defining difficulty of programming is that **computers do exactly what you say, not what you mean**. Human instructions ("make me a sandwich") lean on shared context and goodwill; programs get neither. The skill being trained, more than any syntax, is *decomposition*: breaking a fuzzy goal into steps so explicit that an obedient idiot executing billions of steps per second produces the outcome you want.

Two mindsets you'll alternate between forever:

- **Building**: composing small, understood pieces into larger behavior.
- **Debugging**: the program did something surprising; your mental model and reality disagree; find the divergence. Debugging is *the* core skill — expect to spend more time doing it than writing new code, and know that this ratio holds for experts too (experts just debug faster, by forming better hypotheses).

---

# 2 — Values, Types, and Variables

## 2.1 Values and types

A **value** is a piece of data: `42`, `3.14`, `"hello"`, `true`, a list, a date. Every value has a **type** — the set it belongs to plus the operations that make sense on it. Core types in nearly every language:

- **Integers** and **floats** (with all the caveats from doc 01 §2.1–2.2 — overflow, `0.1 + 0.2` ≠ `0.3`)
- **Booleans**: `true`/`false` — the values conditions produce
- **Strings**: text (Unicode, per doc 01 §2.3)
- **Collections**: lists/arrays, key-value maps, sets — previewed here, dissected in [Data Structures](03-data-structures.md)
- A "nothing" value: `null`/`None`/`nil` — useful and notorious (§10.3)

Types are the first line of defense against nonsense: `"hello" / 7` is rejected rather than computed. *When* it's rejected — before running or mid-run — is the static/dynamic divide of §9.

## 2.2 Variables

A **variable** is a *name bound to a value* — at machine level, a name for a memory location (doc 01 §6 showed that at bottom there are only addresses; names are the first great fiction compilers maintain for us).

```python
count = 0            # bind the name
count = count + 1    # read it, compute, rebind
```

The single most important habit: **names are for humans**. `days_until_expiry` vs. `d` costs nothing at runtime and everything in comprehension. Programs are read far more often than written — most "hard to work with" code is hard because of naming and structure, not cleverness.

**Scope** — where a name is visible: names created inside a function exist only there (each call gets fresh ones — this is what makes functions reusable); names at the top level ("globals") are visible everywhere, which is exactly why they're dangerous (§6).

---

# 3 — Control Flow

Programs run top to bottom until a control-flow construct redirects them — each of these compiles to the compare-and-jump instructions of doc 01 §6.

## 3.1 Conditionals

```python
if temperature > 30:
    advice = "stay hydrated"
elif temperature < 0:
    advice = "wear layers"
else:
    advice = "enjoy"
```

Conditions are boolean expressions built with comparisons (`==`, `<`, `>=`...) and boolean operators (`and`, `or`, `not` — Boole's algebra from doc 01 §3.2, now in code; De Morgan's laws are how you simplify a gnarly `not (a and b)`). Most languages **short-circuit**: `x != 0 and 10/x > 2` never divides by zero because a false left side skips the right side entirely — an idiom, not an accident.

## 3.2 Loops

```python
while attempts < 3:          # repeat while condition holds
    ...

for item in cart:            # once per element of a collection
    total += item.price

for i in range(10):          # counted: i = 0,1,...,9
    ...
```

Plus `break` (leave the loop now) and `continue` (skip to the next iteration). Two loop bugs account for a comic share of all bugs ever written: **off-by-one errors** (looping one time too many/few — why CS counts from 0, and why half-open ranges like `range(0, n)` — includes 0, excludes n — are the convention: the length is just `n`, and adjacent ranges butt together without overlap) and **infinite loops** (the condition never becomes false — and per doc 01, the machine will happily comply forever).

## 3.3 Iteration vs. the collection being iterated

A rule you'll rediscover painfully otherwise: **don't modify a collection while looping over it** (languages either corrupt the iteration or throw). Build a new collection, or iterate over a copy.

---

# 4 — Functions and Abstraction

## 4.1 The mechanics

A **function** packages a computation behind a name and a parameter list:

```python
def price_with_tax(price, tax_rate=0.20):   # parameters (one with a default)
    return price * (1 + tax_rate)           # return sends a value back

total = price_with_tax(100)                 # call it; total = 120.0
```

Calls can nest and chain; each call's locals live in its own stack frame (doc 01 §6 — this is *why* recursion and reentrancy work at all).

## 4.2 Why functions are the most important idea in this document

Functions are the primary unit of **abstraction** — the theme of the entire curriculum. A good function:

- **Names an idea**: `is_valid_email(s)` lets readers think in the problem's vocabulary rather than in string operations.
- **Hides its implementation**: callers depend only on the *contract* (inputs → output), so the inside can be rewritten freely. This independence is what lets millions of programmers build on each other's work via **libraries** — functions someone else wrote (`import math`, `npm install ...`); modern programming is mostly *composing* libraries, and knowing what exists is real expertise.
- **Enables Don't Repeat Yourself (DRY)**: logic that exists once has one place to be fixed. (Balanced by its counter-principle: duplication is cheaper than the *wrong* abstraction — merging two things that merely look similar couples things that later need to diverge.)
- **Decomposes problems**: the universal expert method for any large task is writing the top-level story as calls to functions that don't exist yet — `data = load(path); cleaned = clean(data); report = summarize(cleaned)` — then implementing each, recursively, until the pieces are trivial. This is "top-down design"; it turns a mountain into stairs.

Rules of thumb: a function should do *one thing* at *one level of detail*; if you can't name it without "and," split it; if it takes six parameters, some of them probably form an object (§7).

## 4.3 Functions as values

In most modern languages, functions are themselves values — assignable, passable, returnable ("first-class"). Passing behavior as an argument is everywhere:

```python
sorted(people, key=lambda p: p.age)     # 'how to rank' passed into 'sort'
button.on_click(handle_click)           # 'what to do later' — a callback
```

This is the doorway to functional programming (§8) and to event-driven code (GUIs, servers), where the program is largely *functions registered to run when things happen*.

---

# 5 — Recursion

A function that calls itself, on a smaller piece of the problem:

```python
def factorial(n):
    if n <= 1:              # BASE CASE: answer known directly
        return 1
    return n * factorial(n - 1)   # RECURSIVE CASE: shrink toward the base
```

Every correct recursion has (1) base case(s), (2) recursive calls on *strictly smaller* inputs, so the base case is inevitably reached. Miss either and the frames pile up until **stack overflow** (doc 01 §6 — now you know the mechanism).

Why bother, when loops exist? Because some structures are *recursively shaped*, and on them recursion is dramatically clearer: trees and nested data ("a folder contains files and folders" — the definition recurses, so the traversal should too), divide-and-conquer algorithms (merge sort, quicksort — [doc 04](04-algorithms.md)), and parsers (expressions contain sub-expressions — [doc 08](08-compilers-and-languages.md)). The skill is the **recursive leap of faith**: *assume* the recursive call handles the smaller case correctly, and just write the step that combines results. (Induction from [doc 07](07-theory-of-computation.md) is why the faith is justified.)

Iteration and recursion are formally interchangeable; choose whichever fits the problem's shape. Caveat: naive recursion can redo work catastrophically (`fib(n)` recomputes subproblems exponentially many times) — the fix, *memoization/dynamic programming*, is a centerpiece of [doc 04](04-algorithms.md).

---

# 6 — State, Mutation, and Why They're Dangerous

**State** = everything that can change as a program runs (variable values, object contents, files...). **Mutation** = changing it in place.

State is unavoidable (programs exist to have effects) but it's the primary source of complexity in software:

- A function whose result depends on *hidden* state (globals, object fields, files) can't be understood — or tested — in isolation: calling it twice with the same arguments may do different things.
- **Aliasing**: two names refer to the *same* mutable object, so a change through one name surprises code holding the other. In most languages, variables hold *references* to objects, and assignment copies the reference, not the object:

```python
a = [1, 2, 3]
b = a            # b is the SAME list, not a copy
b.append(4)
print(a)         # [1, 2, 3, 4]  — "spooky action at a distance"
```

  This — reference vs. value, shallow vs. deep copy — is the #1 misconception-generator for new programmers. Related trap: mutable default arguments (Python's `def f(x, acc=[])` shares one list across all calls).

- Shared *mutable* state across threads is the root of race conditions ([OS doc Part 6](operating-systems.md#part-6--concurrency-and-synchronization)).

The discipline that follows (and previews §8): **minimize scope of mutation**. Prefer local over global, prefer creating new values over modifying old ones, make data immutable unless there's a reason not to, and confine necessary mutation behind small interfaces. Much of "good design" in any paradigm is exactly this.

---

# 7 — Object-Oriented Programming

## 7.1 The idea

Bundle **state and the operations on it** into one unit — an **object** — and let the object guard its own consistency. A **class** is the blueprint; objects are its **instances**.

```python
class BankAccount:
    def __init__(self, owner):        # constructor
        self.owner = owner
        self._balance = 0             # convention: _ = "internal, hands off"

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("deposit must be positive")
        self._balance += amount

    def balance(self):
        return self._balance
```

**Encapsulation** is the load-bearing principle: outside code *cannot* put the account into a nonsense state (negative deposit), because the only door in is `deposit()`, which checks. The class defines an **invariant** ("balance reflects exactly the validated deposits") and its methods preserve it. Objects are §6's discipline made structural: mutation still exists but is *confined* behind an interface.

## 7.2 Polymorphism — the actually-important part

Different classes can implement the *same interface*, and calling code neither knows nor cares which it has:

```python
def total_area(shapes):           # works for ANY mix of shapes,
    return sum(s.area() for s in shapes)   # including ones invented later
```

Each shape (`Circle`, `Square`, ...) computes `area()` its own way; `total_area` is *closed for modification, open for extension*. This — replacing "switch on the kind of thing" with "ask the thing" — is the heart of OOP's value, and of every plugin system, driver interface ([OS doc Part 8](operating-systems.md#part-8--inputoutput-and-device-drivers)), and framework you'll meet. Statically-typed languages name the contract explicitly (`interface`/`trait`/abstract class); Python just checks at call time ("duck typing" — if it quacks, it's a duck).

## 7.3 Inheritance — the overrated part

Classes can also *extend* other classes (`class SavingsAccount(BankAccount)`), inheriting their code. Sometimes right (genuine "is-a" hierarchies), but decades of experience produced a firm consensus: **prefer composition over inheritance** — build objects *containing* the pieces they need rather than deep family trees. Inheritance couples children to parents' internals ("fragile base class"), and hierarchies ossify around yesterday's taxonomy. Modern languages (Go, Rust) omit implementation inheritance entirely and nobody misses it; interfaces + composition cover the real needs.

The frequently-cited SOLID principles boil down to the same forces: small single-purpose units, contracts over concretions, extension without modification.

---

# 8 — Functional Programming

The opposite bet from OOP: instead of *organizing* state, **minimize it**.

- **Pure functions**: output depends only on inputs; no side effects (no mutation, no I/O). Pure functions are trivially testable, cacheable, parallelizable (no shared state → no races), and understandable in isolation — every property §6 said state destroys.
- **Immutable data**: "changing" a value means producing a new one. (Efficiently, via structure sharing — persistent data structures reuse the unchanged parts.)
- **Expressions over statements**: programs as compositions of value-producing expressions, not sequences of state changes.

The everyday toolkit (now absorbed into *every* mainstream language):

```python
adults  = [p for p in people if p.age >= 18]          # filter
names   = [p.name for p in adults]                    # map
total   = sum(p.income for p in adults)               # reduce/fold
```

`map` (transform each element), `filter` (keep some), `reduce` (combine all into one value) replace hand-rolled loops with declarations of *intent* — and because they don't dictate order, they scale to parallel and distributed execution unchanged: Spark and MapReduce (see [../analytics/](../analytics/)) are literally this idiom at datacenter scale, and that's *why* they're designed this way.

Also from FP, now everywhere: **closures** (a function value captures the variables around it), higher-order functions, and **Option/Result types** (§10.3). Fully-committed FP languages (Haskell, OCaml, Elixir, Clojure) push further — laziness, algebraic data types, effects tracked in types — and are the best mind-expanders in language-land even if you never deploy them.

**The pragmatic synthesis** most experts land on: a *functional core* (pure logic on immutable data — easy to test) inside an *imperative shell* (a thin layer doing the I/O and mutation at the edges). Paradigms are tools, not religions.

---

# 9 — Type Systems

When is `"hello" / 7` caught?

- **Static typing** (C, Java, Go, Rust, TypeScript): types checked **before the program runs**; the error never ships. Costs annotation effort (much reduced by *type inference* — the compiler deduces types you don't write); pays off increasingly with codebase size: types are machine-checked documentation, make renames/refactors safe, and power IDE autocompletion.
- **Dynamic typing** (Python, JavaScript, Ruby): values carry types at runtime; checks happen **at each operation**. Faster to start, superb for scripts and exploration; in large codebases the missing guarantees return as runtime crashes and fear of refactoring — which is why the industry's biggest dynamic codebases retrofitted static checkers (TypeScript over JavaScript; type hints + mypy over Python).
- Orthogonal axis — **strong vs. weak**: does the language quietly coerce nonsense? JavaScript's `"5" - 1 == 4` and `"5" + 1 == "51"` are weak typing's greatest hits; Python (strongly typed though dynamic) raises an error instead.

The expert view: a type system is a *lightweight formal proof* about your program ("this can never be a string here"), verified on every compile. Advanced systems prove more: **generics** ("a list of T, for any T" — one implementation, all types, still checked); sum types/enums with exhaustive `match` ("a response is Success(data) or Failure(reason), and the compiler insists you handle both"); Rust's ownership types prove memory and thread safety (§11). The design frontier of programming languages is mostly "which properties can we move from runtime failure to compile-time impossibility, at acceptable annotation cost."

---

# 10 — Errors and How Languages Handle Them

Things go wrong: files are missing, networks drop, inputs are garbage, code has bugs. Error *handling strategy* is a defining trait of a language.

## 10.1 Exceptions (Python, Java, C#, JS)

An error **raises/throws** an exception, which abandons normal execution and *propagates up the call stack* until a matching `try/catch` **handles** it (or none does — crash + stack trace: read it top-down as "who called whom into this mess"; the deepest frame is where it detonated, the frames above are the path there).

```python
try:
    config = parse(open(path).read())
except FileNotFoundError:
    config = defaults()
finally:
    ...   # runs either way — cleanup (or use with/using blocks: RAII-style)
```

Virtues: errors can't be silently ignored; handling code sits away from the happy path. Vices: invisible control flow (any call might jump away — every line has a hidden exit), and "catch-all" handlers that swallow bugs. Discipline: catch *specific* exceptions, *where you can actually do something about them*; let the rest propagate.

## 10.2 Errors as return values (Go, Rust, C)

The other school: errors are ordinary values the caller must confront at the call site. Go returns `result, err` pairs and you check `if err != nil` (explicit, verbose, nothing hidden). Rust returns `Result<T, E>` — a sum type you *must* unpack, with the `?` operator making propagation one character. The type system guarantees no error goes unnoticed — arguably the best of both worlds, and newer languages keep converging on it.

## 10.3 The null problem

`null`/`None` — "no value here" — is useful and catastrophic: it inhabits *every* reference type silently, so any dereference might explode (`NullPointerException`, `AttributeError: 'NoneType'...`). Its inventor Tony Hoare calls it his "billion-dollar mistake." The modern fix: make absence **explicit in the type** — `Optional[T]` / `T?` / Rust's `Option<T>` — so the compiler forces a "what if it's absent?" branch exactly where absence is possible, and *nowhere else*. Kotlin, Swift, Rust, and TypeScript (strict mode) build this in.

## 10.4 Defensive craft

Validate inputs at trust boundaries (user input, network, files) and *fail fast* — a loud early error beats a quiet corrupted result by miles; use **assertions** for "this should be impossible" conditions so impossibilities announce themselves; when debugging, **read the error message** — verbatim, all of it. (Half of practical debugging skill is genuinely this, plus binary-searching the divergence point with prints/debugger/git bisect, plus writing a minimal reproduction — which usually reveals the bug by itself.)

---

# 11 — Memory Models Across Languages

Doc 01 gave you memory as hardware; languages differ in who manages it. Three regimes:

1. **Manual (C, C++)**: you call `malloc`/`free` (or `new`/`delete`). Total control, zero safety: forget to free → **leak**; free twice or use after free → corruption and security holes (§ [doc 10](10-security-and-cryptography.md) — memory-unsafety is behind ~70% of serious vulnerabilities in C/C++ codebases per Microsoft/Google data). C++ mitigates with RAII and smart pointers, but the language can't *prove* safety.
2. **Garbage collection (Python, Java, Go, JS, C#)**: the runtime periodically finds unreachable objects and reclaims them (see [doc 08 §7](08-compilers-and-languages.md) for how). You can't corrupt memory; you pay in runtime overhead and (in some collectors) pauses. Leaks still possible in spirit: anything *reachable* — a growing global cache, a forgotten listener registration — is never collected.
3. **Ownership (Rust)**: the compiler *statically tracks* which variable owns each value and enforces "one owner; either many readers or one writer" — memory freed automatically when the owner goes out of scope, with **no GC and no unsafety**, and data races banned at compile time. The cost is learning to satisfy the borrow checker. Rust's rise is the industry concluding this trade is worth it for systems code.

Related vocabulary worth cementing now: **stack vs. heap allocation** (locals die with the frame — fast; objects with unpredictable lifetimes go to the heap — managed by one of the regimes above), and **value vs. reference semantics** (§6's aliasing, formalized).

---

# 12 — The Language Landscape

A map, not a ranking — languages are points in a design space, chosen per problem:

| Language | Typing | Memory | Sweet spot |
|---|---|---|---|
| **Python** | dynamic (+opt-in hints) | GC | learning, scripting, data science/ML, automation |
| **JavaScript/TypeScript** | dynamic / static | GC | everything in a browser; much of the server (Node) |
| **Java / C#** | static | GC | large enterprise systems, Android / Windows ecosystems |
| **Go** | static | GC | servers, networked services, cloud tooling (Docker, Kubernetes are written in it) |
| **Rust** | static | ownership | systems code needing C-speed with safety; increasingly OS kernels, browsers |
| **C** | static (weak) | manual | OS kernels, embedded, the lingua franca every system speaks ([doc on OSes assumes it](operating-systems.md)) |
| **C++** | static | manual/RAII | games, HFT, browsers, ML runtimes — maximum control at maximum complexity |
| **SQL** | — | — | declarative data queries — not general-purpose, universally required ([doc 06](06-databases.md)) |
| **Haskell/OCaml/Clojure/Elixir** | static/static/dyn/dyn | GC | FP heartland; mind-expanders and niche powerhouses |

Deeper distinctions than syntax: compiled vs. interpreted (and JITs — [doc 08](08-compilers-and-languages.md)), imperative vs. declarative (say *how* vs. say *what* — SQL, HTML, and build systems are declarative), and concurrency models (threads vs. async/event-loop vs. goroutines/channels vs. actors — grounded in [OS doc Parts 4, 6, 9](operating-systems.md)).

Advice: learn **Python** deeply first (lowest ceremony between thought and running code). Add **C** alongside the OS document (it *is* the machine model made portable). Add **one static language** (Go is the gentlest; Rust the most educational) with the Software Engineering document. After three, the fourth is a weekend — you'll recognize everything here in new clothes, because languages are recombinations of this document.

---

# 13 — The Road to Expertise

## 13.1 Books

1. **Python Crash Course** (Matthes) or the official Python tutorial — mechanics, fast.
2. **Structure and Interpretation of Computer Programs (SICP)** — Abelson & Sussman. *Free online.* The legendary "programming as ideas" book: abstraction, state, streams, interpreters. Life-changing for those who persist. (Gentler modern sibling: *Composing Programs*, composingprograms.com.)
3. **A Philosophy of Software Design** — Ousterhout. Short, superb: deep modules, information hiding, complexity as *the* enemy. Read it early, reread it yearly.
4. **The Pragmatic Programmer** — Hunt & Thomas. The craft's collected folklore: DRY, orthogonality, tracer bullets.
5. **Effective Python / Effective Java / etc.** — per-language idiom, once fluent.
6. **Learn You a Haskell** or an OCaml course — the FP immersion, when you're ready for it.

## 13.2 Doing (nothing below substitutes for this)

1. **Weeks 1–4**: Python basics; solve dozens of tiny problems (Exercism, Advent of Code early days). Typing, running, erroring, fixing — daily.
2. **Build a text game or a to-do CLI** — first taste of state + design.
3. **Decomposition practice**: pick something real (expense tracker, flashcards app); write the top-level function calls first, then implement downward (§4.2's method).
4. **Refactoring practice**: take your own month-old code; make it readable without changing behavior. (Painful, transformative.)
5. **Read excellent code**: small well-loved libraries (e.g., Python's `pathlib`, `requests` internals). Reading is half the craft.
6. **Do SICP's first two chapters** once loops and functions are second nature.
7. Then continue to [Data Structures](03-data-structures.md) — with running code as your laboratory.

## 13.3 Ideas to retain forever

1. **Programs are read more than written** — optimize for the reader; naming is design.
2. **Abstraction = a name + a contract + a hidden inside** — functions, then classes, then modules, then services; same idea at every scale.
3. **Decompose top-down; build bottom-up.**
4. **State is the main source of complexity** — minimize it, localize it, make the rest immutable; aliasing of mutable data is the classic trap.
5. **Recursion mirrors recursive structure** — base case, smaller subproblem, leap of faith.
6. **Polymorphism beats switch-statements; composition beats inheritance.**
7. **Pure core, imperative shell** — the paradigm synthesis that actually ships.
8. **Types are proofs**: every property moved to compile time is a bug class deleted.
9. **Errors are part of the design**, not an afterthought: explicit, specific, fail-fast; absence (`null`) should be visible in types.
10. **Languages are design points, not tribes** — concepts transfer; collect concepts, not syntax.

---

*Next: [Data Structures](03-data-structures.md) — the shapes data takes, and why the choice dominates program performance.*
