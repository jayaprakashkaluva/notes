# Theory of Computation: What Computers Can and Cannot Do

The mathematical heart of computer science. Assumes [Algorithms](04-algorithms.md). This document covers the discrete math you need, then the three great questions: *what is a computer, formally?* (automata), *what can be computed at all?* (computability), and *what can be computed efficiently?* (complexity, P vs NP). Unlike the other documents, the payoff here is not a tool but a worldview — plus several results you will bump into professionally (regex limits, undecidability behind compiler warnings, NP-hardness behind "we use a heuristic").

---

## Table of Contents

1. [The Discrete Math Toolkit](#1--the-discrete-math-toolkit)
2. [Finite Automata and Regular Languages](#2--finite-automata-and-regular-languages)
3. [Grammars and Context-Free Languages](#3--grammars-and-context-free-languages)
4. [Turing Machines and the Church-Turing Thesis](#4--turing-machines-and-the-church-turing-thesis)
5. [Undecidability: Real Limits](#5--undecidability-real-limits)
6. [Complexity: P, NP, and the Great Open Question](#6--complexity-p-np-and-the-great-open-question)
7. [Living with NP-Hardness](#7--living-with-np-hardness)
8. [Information Theory in One Section](#8--information-theory-in-one-section)
9. [The Road to Expertise](#9--the-road-to-expertise)

---

# 1 — The Discrete Math Toolkit

The minimum mathematics the rest of CS stands on — collected here because every other document quietly uses it.

## 1.1 Logic

**Propositions** (statements that are true or false) combined with AND (∧), OR (∨), NOT (¬), and **implication** (→). The two facts people get wrong:

- `A → B` ("if A then B") is *true whenever A is false* — "if it rains, I bring an umbrella" makes no claim about sunny days. In code: a guard clause `if (!valid) return;` makes everything after it carry the implicit "valid →" for free.
- The **contrapositive** `¬B → ¬A` is *equivalent* to `A → B`; the **converse** `B → A` is *not*. ("All bugs cause failures" ≠ "all failures are caused by bugs.") De Morgan's laws (¬(A∧B) = ¬A∨¬B) you've already used to refactor conditionals (doc 02 §3.1).
- **Quantifiers**: ∀ ("for all") and ∃ ("there exists"). Negating swaps them: ¬∀x P(x) = ∃x ¬P(x) — "not all inputs work" means "some input fails." Every correctness claim about code is a quantified statement; every bug report is an ∃.

## 1.2 Proof techniques (the ones programmers actually use)

- **Direct** and **by cases**: mirror structured code.
- **Contradiction**: assume the opposite, derive absurdity. (Used twice below, for √2-style classics: Cantor §5.1 and Halting §5.2.)
- **Induction**: prove the base case; prove "if true for n, true for n+1"; conclude for all n. *This is recursion's proof-shadow* (doc 02 §5's "leap of faith," justified): a recursive function is correct exactly when its base case is right and its inductive step preserves correctness. **Loop invariants** are induction for loops: a property true before the loop, preserved by each iteration, is true after — the honest way to *know* your binary search is right (doc 04 §2's decades of off-by-one bugs are missing-invariant bugs).
- The **pigeonhole principle**: n+1 items in n boxes → some box has two. Trivial-sounding; proves hash collisions must exist (doc 03 §6.2), lossless compression can't shrink everything (§8), and more.

## 1.3 Sets, relations, functions, counting

**Sets** (unordered collections; ∪ ∩ ⊆, complement), **relations** (equivalence relations partition a set — the math behind "group by"; partial orders — the math behind dependency graphs and topological sort, doc 04 §6.1), **functions** (injective/surjective/bijective — bijections are why "same size" makes sense for infinite sets, §5.1). **Counting**: product rule (independent choices multiply — why 2ⁿ bit-strings, why password length beats complexity), permutations n!, combinations C(n,k). **Probability** at the working level: independence multiplies, expectation is linear (even for dependent things — the slick trick behind average-case analyses), and the birthday paradox (collisions among n random values from N appear around n ≈ √N — governs hash-table sizing and the 128-bit-ID safety argument, and returns in [doc 10](10-security-and-cryptography.md) as the birthday attack on hashes).

## 1.4 Modular arithmetic

Clock arithmetic: a ≡ b (mod n) if n divides a−b. It's why hash(key) % buckets works, how ring buffers wrap, why two's complement overflow wraps (doc 01 §2.1 — the hardware *is* mod 2⁶⁴), and — via Fermat's little theorem and friends — the engine of RSA and Diffie-Hellman ([doc 10](10-security-and-cryptography.md)).

---

# 2 — Finite Automata and Regular Languages

## 2.1 The machine

Strip a computer to its minimum: no memory except *which state you're in*. A **finite automaton (FA)**: finitely many states, transitions labeled by input symbols, a start state, accepting states. Feed it a string; follow transitions; accept if you end in an accepting state.

```
  Accepts binary strings with an even number of 1s:

        0                 0
       ┌─┐               ┌─┐
       ▼ │      1        ▼ │
     ((EVEN)) ────────► (ODD)
         ▲               │
         └───────────────┘
                 1
   ((double circle)) = accepting
```

A **language** is a set of strings; the languages FAs can recognize are the **regular languages**. Nondeterministic FAs (multiple simultaneous transitions — "try all paths") turn out to recognize *exactly the same* languages (subset construction converts NFA→DFA, possibly with exponentially many states) — the first of theory's recurring "more power that isn't" results.

## 2.2 Regular expressions are the same thing

**Kleene's theorem**: regular expressions (concatenation, `|`, `*`) and finite automata define *exactly* the same languages. Every regex engine is an automaton factory: compile the pattern to an NFA/DFA, run it over the string in O(n). Two professional consequences:

- **Regex can't count unboundedly**: matching balanced parentheses (or nested HTML tags) requires remembering *how deep you are* — unbounded memory, which finite states can't hold. Provable via the **pumping lemma** (long-enough accepted strings must revisit a state, so a loop can be "pumped"). This is the theory behind the famous Stack Overflow answer: **don't parse HTML with regex** — it's not advice, it's a theorem. (Real "regex" engines with backreferences exceed regular languages — and pay for it with exponential-time backtracking: catastrophic backtracking/ReDoS outages, e.g. Cloudflare 2019. Engines like RE2/Rust's regex stay truly regular precisely to guarantee linear time.)
- **State machines are an engineering pattern**, not just theory: TCP connection states ([doc 05 §4](05-computer-networks.md)), UI flows, lexers (doc 08 §2), protocol parsers, game AI — "enumerate the states, label the transitions" is a design tool that eliminates whole bug classes ("can't happen" states become unrepresentable).

# 3 — Grammars and Context-Free Languages

Add one memory device — a single **stack** — and you can count nesting: **pushdown automata**, equivalently **context-free grammars (CFGs)**: recursive production rules like

```
Expr → Expr "+" Term | Term
Term → Term "*" Factor | Factor
Factor → NUMBER | "(" Expr ")"
```

This grammar *is* arithmetic's nesting and precedence, stated declaratively — and CFGs describe essentially every programming language's syntax. The **Chomsky hierarchy** organizes the whole ladder: regular ⊂ context-free ⊂ context-sensitive ⊂ recursively enumerable — each rung a machine with more memory (finite states → +stack → +bounded tape → +unbounded tape), each the formal home of real artifacts (regex/lexers → parsers/ASTs → — → general programs). Parsing algorithms for CFGs are [doc 08 §3](08-compilers-and-languages.md)'s subject; the theory says *why* the lexer/parser split exists: tokens are regular (cheap DFA), structure is context-free (needs the stack).

Limits again: CFGs can't express "this variable was declared before use" or "these two counts match" (context-sensitivity) — which is exactly why compilers bolt a separate *semantic analysis* pass onto parsing rather than growing the grammar.

# 4 — Turing Machines and the Church-Turing Thesis

## 4.1 The machine

Turing's 1936 abstraction of "a person computing with pencil and paper": a finite-state control + an **unbounded tape** with a read/write head. Each step: read the symbol, and per the current state — write, move left/right, change state. That's everything.

Why this humble machine is *the* definition of computation:

- **Robustness**: every strengthening tried — more tapes, 2D tapes, random access, nondeterminism — computes exactly the same set of functions. Your laptop (doc 01) is a Turing machine with a finite-but-extendable tape; every programming language that has loops/recursion + unbounded memory is **Turing-complete** — equivalent in *capability* (not speed) to every other. (Which is why "which language is more powerful?" is, in the computability sense, settled: they're all the same, and even accidental systems — C++ templates, PowerPoint animations, Magic: The Gathering — keep turning out Turing-complete. Also why perfect static analysis of Turing-complete configuration languages is impossible — a reason to prefer *non*-Turing-complete DSLs when you can.)
- **The universal machine**: one specific TM can simulate *any* TM given its description as input. Programs-as-data (doc 01 §5.1's von Neumann insight), interpreters ([doc 08](08-compilers-and-languages.md)), and the very idea of software all live in this theorem.
- **The Church-Turing thesis**: every function computable by *any* mechanical procedure is computable by a TM. Unprovable (it defines "mechanical"), universally believed, unbroken for 90 years — quantum computers included (they threaten *speed*, §6, not computability).

# 5 — Undecidability: Real Limits

## 5.1 Warm-up: there aren't enough programs

Programs are finite strings → countably many. Problems (sets of naturals / functions ℕ→{0,1}) are uncountably many — **Cantor's diagonal argument**: given any claimed complete list of infinite bit-sequences, build one differing from the k-th sequence at position k; it's on no list. So *almost every* problem has no program solving it. Existence settled; now for a problem we actually care about:

## 5.2 The halting problem

**Can a program `halts(P, x)` decide whether program P halts on input x?** No — by diagonalization-as-self-reference:

```
def troll(P):
    if halts(P, P): loop_forever()
    else:           return
```

Does `troll(troll)` halt? If it halts, then `halts(troll, troll)` was true, so it loops — contradiction. If it loops, `halts` said false, so it returns — contradiction. Hence `halts` cannot exist. (Same self-referential knife as Cantor's diagonal and Gödel's incompleteness — the 20th century's three great limit theorems are one trick in three costumes.)

## 5.3 Rice's theorem and what it means at work

The halting problem *spreads*: **Rice's theorem** — every non-trivial question about a program's *behavior* (not its text) is undecidable. Does it ever crash? Print "hello"? Equal this other program? Leak memory? All undecidable in general. Professional consequences, so you recognize them in the wild:

- **Perfect static analysis is impossible** — every real analyzer/compiler warning system over-approximates (false positives) or under-approximates (missed bugs). "Sound *and* complete and terminating" is off the menu — pick two.
- **Antivirus can't perfectly decide "is this malicious"**; **compilers can't perfectly detect dead code or infinite loops**; **verifiers restrict the language** (loop bounds, decidable fragments, or human-supplied invariants — which is how real formal verification like seL4 and CompCert threads the needle: undecidability bars *automation for all programs*, not *proof for your program*).
- Undecidability is about **all programs, in general**: specific programs are analyzed successfully every day. The theorem forbids the universal tool, not the practice.

# 6 — Complexity: P, NP, and the Great Open Question

Computability asks *possible?*; complexity asks *feasible?* — and doc 04 §1's polynomial-vs-exponential cliff becomes a theory.

- **P**: problems solvable in polynomial time (nᶜ) — the formal stand-in for "efficiently solvable." Sorting, shortest paths, matching, linear programming, primality (proved in P only in 2002!).
- **NP**: problems whose solutions can be **verified** in polynomial time, given a certificate. Sudoku: solving seems hard, *checking* a filled grid is trivial. Likewise: SAT (does a boolean formula have a satisfying assignment?), traveling salesman (is there a tour under budget B?), subset-sum, graph coloring, protein folding models, course timetabling. NP = "easy to grade," P = "easy to solve"; **P vs NP asks whether grading-easy implies solving-easy** — whether verification and discovery are fundamentally the same difficulty. It is *the* open problem of the field ($1M Clay prize, open since 1971).
- **NP-completeness** (Cook-Levin 1971; Karp's 21, 1972): SAT is NP's hardest problem — *every* NP problem reduces to it in polynomial time (a **reduction** = "an efficient solver for B would give one for A"; the field's master tool, and a practical skill: recognizing *your* scheduling problem is graph-coloring in disguise). Thousands of natural problems from every industry are NP-complete, all polynomially equivalent: crack one in P and P=NP — all fall. Fifty years of the world's best minds failing to crack any of them is the evidence (not proof) that **P ≠ NP**: discovery really is harder than verification.
- Worth knowing exists: **NP-hard** (at least as hard as NP; may not be in NP — e.g. optimization versions, or worse), **PSPACE** (polynomial memory — generalized chess lives here), **EXPTIME**, and **BQP** (quantum polynomial time: factoring falls — Shor — but NP-complete problems are believed *not* to; quantum computers are not exponential-search machines, a correction worth carrying).
- If P=NP with a practical algorithm, modern cryptography dies (all of [doc 10](10-security-and-cryptography.md) rests on problems being hard to solve but easy to verify) — along with much of the distinction between checking proofs and finding them.

# 7 — Living with NP-Hardness

NP-hardness is a *daily engineering reality* (scheduling, routing, packing, layout, register allocation are all there), and the response is a mature playbook — mostly cross-references now, because you've met the tools:

1. **Exact but exponential, tamed**: backtracking with fierce pruning, branch & bound (doc 04 §8) — fine when instances are small or structured. **SAT/SMT and ILP solvers** industrialize this: model your problem, hand it over; they routinely eat million-variable *real* instances (worst case ≠ typical case — real instances have structure; complexity is a worst-case theory, its most important asterisk).
2. **Approximation algorithms**: provable near-optimality bounds (doc 04 §9).
3. **Heuristics/metaheuristics**: local search, simulated annealing, genetic algorithms — no guarantees, strong practice.
4. **Special cases**: NP-hard *in general* often ⊃ polynomial *for your case* (trees, small treewidth, restricted structures — independent set is trivial on trees; check before surrendering).
5. **Fixed-parameter tractability**: exponential only in a small parameter k, polynomial in n.

The professional value of the theory is the *diagnosis*: proving (by reduction) that your problem is NP-hard ends the search for a perfect fast algorithm honestly, redirects effort into this playbook, and inoculates you against both false hope and vendors claiming to have squared the circle.

# 8 — Information Theory in One Section

Shannon, 1948 — the other mathematical pillar, in brief. **Entropy** H = the true information content of a source in bits: a fair coin carries 1 bit/flip; a biased or predictable source carries less (English text: ~1–1.5 bits/character, not the 8 of ASCII — the gap *is* compressibility). Consequences: entropy is the **hard floor on lossless compression** (no algorithm shrinks all inputs — pigeonhole, §1.2; compressors win exactly by modeling predictability — Huffman and LZ, doc 04 §10, approach the floor); **channel capacity** bounds communication over noisy links, and error-correcting codes (checksums' constructive cousins, from RAM ECC to QR codes to deep-space probes) approach *that* floor; entropy also quantifies password strength and randomness quality ([doc 10](10-security-and-cryptography.md)), and cross-entropy is the loss function training essentially every neural network ([../ai/](../ai/)). One concept, five fields.

# 9 — The Road to Expertise

## 9.1 Books

1. **Discrete math foundation**: *Discrete Mathematics and Its Applications* (Rosen) — the standard; or the free *Mathematics for Computer Science* (Lehman, Leighton, Meyer — MIT 6.042 text, superb and rigorous).
2. **Introduction to the Theory of Computation** (Sipser) — *the* theory text: automata → computability → complexity, famously clear proofs. This document is a map of it.
3. **The Annotated Turing** (Petzold) — Turing's actual 1936 paper, walked through gently; wonderful supplementary read.
4. **Computers and Intractability** (Garey & Johnson) — the classic NP-completeness field guide (its problem catalog is still the reference).
5. **Quantum Computing Since Democritus** (Aaronson) — complexity theory's worldview, entertainingly; includes the honest quantum story.
6. Gödel, Escher, Bach (Hofstadter) — the self-reference trilogy (Cantor/Gödel/Turing) as a cultural odyssey; optional, beloved.

## 9.2 Doing

Theory's "doing" is proofs and constructions — different muscles, same principle:

1. Do an induction proof (sum formula), a contradiction proof (√2 irrational), and write a **loop invariant** for your binary search from doc 04 — then never fear "prove it" again.
2. **Build a regex engine**: parse a small regex syntax → NFA (Thompson's construction) → simulate it. ~150 lines; makes §2 permanent, previews doc 08.
3. Design DFAs by hand: strings ending in "01"; divisible-by-3 binary numbers (states = remainders — modular arithmetic meets automata).
4. Use the pumping lemma once: prove {aⁿbⁿ} isn't regular. Feel *why* the parenthesis theorem holds.
5. Write the halting-problem contradiction from memory, in code comments.
6. **Do one reduction**: 3-SAT → independent set (any theory course/text has it); then spot an NP-hard problem in your own work and name its disguise.
7. Play with a **SAT solver** (MiniSat, Z3): encode Sudoku or N-queens; watch "intractable" fall in milliseconds — both lessons of §7 at once.

## 9.3 Ideas to retain forever

1. **Induction is recursion is loop invariants** — one idea; it's how you *know* code is right.
2. **Machines form a ladder by memory** (finite → stack → tape); regex/parsers/programs live on its rungs, and each rung's *limits* are theorems, not opinions (don't parse HTML with regex).
3. **State machines are a design tool** — enumerate states, label transitions, outlaw the impossible.
4. **Turing-complete = universally capable** — languages differ in ergonomics and speed, never in what's computable; universality is also why programs can process programs.
5. **Some problems no program solves** (halting; via Rice: all behavioral questions) — so every analyzer approximates; the limit is on universal tools, not on verifying *your* program.
6. **P vs NP = discovery vs. verification** — NP-complete problems are one problem in a thousand costumes; reductions are the master tool.
7. **NP-hard has a playbook** (solvers, approximation, heuristics, special cases) — worst-case theory, structured-instance practice.
8. **Entropy floors compression and crypto**: information is a measurable quantity; nothing shrinks everything.
9. **Self-reference is the knife** — Cantor, Gödel, Turing: three theorems, one diagonal.
10. **Complexity is worst-case**: real instances have structure — the theory tells you when to stop seeking perfection, not when to stop engineering.

---

*Next: [Compilers & Programming Languages](08-compilers-and-languages.md) — the theory of §2–3 built into the tools you use every day.*
