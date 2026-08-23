# Algorithms: Recipes and Their Cost

Assumes [Data Structures](03-data-structures.md). An **algorithm** is a precise, finite procedure for solving a class of problems. This document covers the science of comparing them (complexity analysis), the canon everyone shares (searching, sorting, graph algorithms), and — most valuable — the **design paradigms**: reusable strategies for inventing algorithms for problems you've never seen.

---

## Table of Contents

1. [Analyzing Cost: Big-O for Real](#1--analyzing-cost-big-o-for-real)
2. [Searching and the Power of Halving](#2--searching-and-the-power-of-halving)
3. [Sorting](#3--sorting)
4. [Design Paradigm: Divide and Conquer](#4--design-paradigm-divide-and-conquer)
5. [Design Paradigm: Greedy](#5--design-paradigm-greedy)
6. [Graph Algorithms](#6--graph-algorithms)
7. [Design Paradigm: Dynamic Programming](#7--design-paradigm-dynamic-programming)
8. [Design Paradigm: Backtracking and Search](#8--design-paradigm-backtracking-and-search)
9. [Randomized and Approximate](#9--randomized-and-approximate)
10. [Strings](#10--strings)
11. [Practical Algorithm Engineering](#11--practical-algorithm-engineering)
12. [The Road to Expertise](#12--the-road-to-expertise)

---

# 1 — Analyzing Cost: Big-O for Real

## 1.1 Why not just time it?

Timing measures *one input on one machine on one day*. Analysis asks the durable question: **how does cost scale as input grows?** A program that's fine at n=1,000 and dead at n=10,000,000 has a scaling problem no faster machine fixes — because if cost is n², 10× the input is 100× the work.

## 1.2 The rules of the notation

**O(f(n))** = "for large n, cost grows no faster than proportional to f(n)." Mechanics:

- **Drop constants**: 3n and n/2 are both O(n). (Rationale: constants are machine-dependent; growth *shape* isn't.)
- **Drop dominated terms**: n² + 10n + 500 is O(n²) — for large n the n² term is essentially all of it.
- Reading code: a simple loop over n items → O(n); a loop *inside* a loop → O(n²); halving each step → O(log n); doing O(log n) work n times → O(n log n). Sequential phases *add* (keep the max); nested work *multiplies*.

Also in the family: **Ω** (lower bound — grows *at least* this fast), **Θ** (tight — both). Colloquial "big-O" usually means Θ.

## 1.3 The zoo, with canonical inhabitants

| Class | Canonical algorithm | n=10⁶ at 10⁹ ops/s |
|---|---|---|
| O(1) | hash lookup, array index | instant |
| O(log n) | binary search | instant (~20 ops) |
| O(n) | scan for max, count matches | ~1 ms |
| O(n log n) | good sorts; often provably optimal for comparison problems | ~20 ms |
| O(n²) | naive pairwise comparison, bubble sort | ~17 minutes |
| O(2ⁿ) | trying all subsets | n=50 already outlives you |
| O(n!) | trying all orderings | n=15 is ~10¹² |

The cliff between polynomial and exponential is the subject of [Theory of Computation §5](07-theory-of-computation.md).

## 1.4 Three refinements experts actually use

- **Worst vs. average vs. amortized**: quicksort is O(n²) worst but O(n log n) average (and randomization makes the worst case vanishingly unlikely); hash tables are O(1) *average*, O(n) worst; dynamic-array append is O(1) *amortized* (doc 03 §3). Say which one you mean.
- **Space complexity**: memory scales too. Often tradeable against time — the theme of §7.
- **Constants matter at real sizes** (doc 03's drumbeat): an O(n log n) algorithm with heavy constants loses to O(n²) insertion sort below n≈30 — which is why production sorts *hybridize* (§3.4). Big-O picks the neighborhood; benchmarks pick the house.

---

# 2 — Searching and the Power of Halving

**Linear search** — check each element — O(n), and the best possible on unsorted data.

**Binary search** — on *sorted* data, compare with the middle; discard half; repeat. O(log n): a billion elements in 30 comparisons. The generalization is the real lesson: binary search works on anything **monotonic** — not just arrays, but answers. "First version where the test fails" (`git bisect`), "minimum machines needed to finish by deadline" (binary-search the answer, check feasibility), square roots to precision. Whenever you can ask "is X too high or too low?" you can binary search.

Off-by-one honesty: binary search was published in 1946; the first *correct-in-all-cases* published version took until 1962, and Java's standard library carried an overflow bug in it for 9 years (`(lo+hi)/2` overflows; use `lo + (hi-lo)/2`). Boundary discipline (half-open intervals, doc 02 §3.2) is not pedantry.

---

# 3 — Sorting

The most-studied problem in CS — because sorted data unlocks binary search, deduplication, merging, median-finding, and databases' entire ORDER BY/GROUP BY/join machinery ([doc 06](06-databases.md)).

## 3.1 The simple three (know *why* they're slow)

**Bubble** (swap adjacent out-of-order pairs, repeat), **selection** (repeatedly pick the minimum), **insertion** (grow a sorted prefix, inserting each next element into place) — all O(n²): they move elements one position per comparison. Insertion sort earns its keep anyway: on *nearly-sorted* or *tiny* inputs it's excellent (adaptive, low constants) — which is why it appears inside production sorts.

## 3.2 Merge sort — O(n log n) by divide and conquer

Split in half; recursively sort each half; **merge** two sorted lists (repeatedly take the smaller head — O(n)). log n levels × O(n) merging = **O(n log n) guaranteed**. Also **stable** (equal elements keep their relative order — essential for "sort by name, then by department" layering) — but needs O(n) extra space. Merge sort's pattern scales beyond RAM: external sorting (sort chunks, merge files) is how databases and Spark sort data bigger than memory.

## 3.3 Quicksort — the in-place gambler

Pick a **pivot**; **partition** the array in place (smaller-than-pivot left, larger right — O(n), no extra memory); recurse on both sides. Average **O(n log n)** with the best constants of the family (in-place, cache-friendly, tight inner loop). Worst case O(n²) when pivots split terribly (already-sorted input + naive first-element pivot — the classic self-own); cures: random or median-of-three pivots. Not stable.

## 3.4 What languages actually ship

Hybrids, because engineering: **Timsort** (Python, Java objects) = merge sort + insertion sort + exploitation of pre-existing sorted runs — brilliant on real-world semi-sorted data; **introsort** (C++) = quicksort that switches to heapsort if recursion gets suspiciously deep (capping the worst case) and to insertion sort on small ranges; **pattern-defeating quicksort** (Rust unstable sort). Lesson: real algorithms are *layered portfolios* with fallbacks.

## 3.5 Two boundaries worth knowing

- **Comparison sorts cannot beat O(n log n)** — provable (n! orderings need log₂(n!) ≈ n log n yes/no questions to distinguish; a decision-tree argument — a rare and beautiful *lower bound*).
- **Non-comparison sorts escape the bound** when keys have structure: counting sort (O(n+k) for small integer ranges), radix sort (digit-by-digit, O(n·digits)) — how you sort a billion 32-bit integers fast.

---

# 4 — Design Paradigm: Divide and Conquer

The pattern behind merge sort, quicksort, binary search: **split the problem, solve subproblems (recursively), combine**. It wins when splitting is cheap, subproblems are independent, and combining is easy. Other trophies: Karatsuba multiplication (big-number multiply in O(n^1.585) — the first hint that "obvious" algorithms aren't optimal), the Fast Fourier Transform (O(n log n) — signal processing, and the actual engine of fast big-integer arithmetic), closest-pair-of-points. Analysis tool: recurrence relations — T(n) = 2T(n/2) + O(n) ⇒ O(n log n) (the "Master Theorem" mechanizes these).

Independence of subproblems is also what makes divide-and-conquer *parallelize* beautifully — each half on a different core ([doc 01 §7.4](01-how-computers-work.md)), or a different machine ([../analytics/spark-core-internals.md](../analytics/spark-core-internals.md)).

# 5 — Design Paradigm: Greedy

**At each step take the locally best option; never reconsider.** Fast and simple — and *usually wrong*, so the paradigm is really about recognizing the problems where local-best provably yields global-best:

- **Coin change** with 25/10/5/1: greedy works. With coins 25/10/1, making 30: greedy gives 25+1×5 (6 coins); optimal is 10×3. *Same problem, different data, greedy breaks* — the cautionary tale in one line.
- Genuine greedy wins: **interval scheduling** (most non-overlapping meetings: always take the one ending earliest), **Huffman coding** (optimal prefix codes — the core of zip/JPEG entropy coding: merge the two rarest symbols, repeat), **Dijkstra** (§6.3) and **minimum spanning trees** (§6.4) — both greedy with correctness proofs (typically "exchange arguments": any optimal solution can be massaged into the greedy one without loss).

Expert reflex: greedy is the hypothesis to try first (it's the fastest if true) — but demand a proof or a counterexample before trusting it. When greedy fails because choices interact, dynamic programming (§7) is usually the answer.

# 6 — Graph Algorithms

The payoff of doc 03 §11. Throughout: V vertices, E edges, adjacency lists.

## 6.1 BFS and DFS — the two ways to explore

- **Breadth-first search**: explore in rings via a **queue**; visit-marking prevents revisits. O(V+E). Byproduct: **shortest paths by edge count** — fewest hops, degrees of separation, word ladders, and (on grid mazes) shortest routes.
- **Depth-first search**: dive deep via **stack/recursion**; backtrack when stuck. O(V+E). Byproducts: connectivity, cycle detection, and **topological sort** — ordering a DAG (directed acyclic graph) so every edge points forward. Topological sort is quietly one of the most-run algorithms on your machine: build systems, package/dependency resolution, spreadsheet recalculation, task schedulers (Airflow, Spark's DAG scheduler) all do "order these tasks respecting dependencies"; a cycle = "circular dependency" error, detected by the same DFS.

## 6.2 Connected components, bipartiteness

Run BFS/DFS from every unvisited vertex → components (who's connected to whom). Two-color while traversing → bipartite test (can these be split into two non-conflicting groups?). Cheap, constant workhorses.

## 6.3 Shortest paths with weights

Edge weights (distance, cost, latency) break plain BFS. **Dijkstra's algorithm**: grow a "settled" set outward, always settling the unsettled vertex with the smallest known distance (greedy — provably safe when weights are non-negative), relaxing its neighbors. With a binary heap (doc 03 §9): **O((V+E) log V)**. This is the skeleton inside GPS routing, network routing protocols (OSPF — [doc 05](05-computer-networks.md)), and game pathfinding. Variants: **A\*** = Dijkstra + an admissible goal-ward heuristic (games, maps — dramatically faster toward a known target); **Bellman-Ford** = handles negative weights, detects negative cycles, O(VE) (and is the shape of the internet's BGP-style distance-vector routing); **Floyd-Warshall** = all pairs, O(V³), a 5-line triple loop (and secretly dynamic programming).

## 6.4 Minimum spanning trees

Cheapest set of edges connecting everything (network/utility design, clustering): **Kruskal** (take edges cheapest-first unless they form a cycle — needs the *union-find* structure, a gem: near-O(1) merge/find of disjoint sets) or **Prim** (grow one tree, always by the cheapest outgoing edge — Dijkstra's twin). Both greedy, both provably optimal.

## 6.5 Beyond

Strongly connected components, max-flow/min-cut (matching, image segmentation, scheduling), PageRank (random walks — how Google originally ranked the web). Graph algorithms at cluster scale: Spark GraphX / Pregel-style systems ([../analytics/](../analytics/)).

# 7 — Design Paradigm: Dynamic Programming

The most powerful — and most feared — paradigm. The fear is unwarranted; DP is one idea: **recursion + not solving the same subproblem twice**.

## 7.1 The canonical demonstration

Naive recursive Fibonacci recomputes fib(k) exponentially many times — O(1.6ⁿ). Two fixes:

- **Memoization** (top-down): cache each result the first time; every later call is a lookup. O(n).
- **Tabulation** (bottom-up): fill an array from base cases upward; often reveals you only need the last two values — O(1) space.

That's the entire mechanism. What makes DP an art is *finding the subproblem structure*: (1) **optimal substructure** — optimal solutions built from optimal sub-solutions; (2) **overlapping subproblems** — the recursion revisits the same states (else plain divide-and-conquer suffices).

## 7.2 The method (works on every DP problem)

1. **Define the state precisely in words**: "dp[i][w] = the best value using the first i items within weight w." This sentence *is* the solution; everything else is bookkeeping.
2. **Recurrence**: express dp[state] via smaller states — usually a max/min over the last choice ("take item i or don't": `dp[i][w] = max(dp[i-1][w], dp[i-1][w-wt[i]] + val[i])`).
3. **Base cases**, **evaluation order** (or just memoize), and **where the answer sits**.
4. Complexity = (#states) × (work per state).

## 7.3 The classics (each a reusable template)

- **Knapsack** (above) — resource allocation under a budget; appears everywhere in disguise.
- **Edit distance** — min insert/delete/substitute operations between strings; dp over prefix pairs. Powers spell-check, diff, and DNA alignment (bioinformatics' Needleman-Wunsch *is* this).
- **Longest common subsequence** — the core of `git diff`.
- **Longest increasing subsequence**, **coin change** (the fix for §5's greedy failure), **matrix-chain / interval DP**, **DP on grids** (path counting/costs), **DP on trees** (house robber on a tree; independent sets), **DP + bitmask** (traveling salesman on ≤ ~20 cities: O(2ⁿ·n²) — exponential, but astronomically better than n!).
- In the wild: sequence alignment, speech recognition (Viterbi on HMMs), spreadsheet-like incremental computation, query optimizers choosing join orders ([doc 06 §6](06-databases.md)), reinforcement learning's Bellman equations.

# 8 — Design Paradigm: Backtracking and Search

When no polynomial structure exists (see [doc 07](07-theory-of-computation.md) on NP), search the possibility tree — but intelligently:

- **Backtracking**: extend a partial solution one choice at a time; the moment it violates constraints, **prune** — abandon the whole subtree and step back. Sudoku, N-queens, crosswords, parsing ambiguity, SAT. Worst case exponential; pruning makes real instances tractable.
- Pruning upgrades: constraint propagation (deduce forced moves before guessing), ordered choices (try most-constrained variable first — fail fast), **branch and bound** (for optimization: prune any branch whose optimistic bound can't beat the best-so-far).
- Industrial descendants: **SAT/SMT solvers** — ferociously engineered backtracking (conflict-driven clause learning) that routinely solves million-variable industrial instances despite exponential worst cases; used for chip verification, dependency resolution, program analysis. Knowing "encode it and hand it to a solver" is a legitimate expert move.

# 9 — Randomized and Approximate

Two ways to trade certainty for speed/space, both respectable:

- **Randomized algorithms**: quicksort's random pivot (defeats adversarial inputs); hash functions themselves; reservoir sampling (uniform sample from a stream of unknown length); Miller-Rabin primality (the reason RSA keys can be generated — [doc 10](10-security-and-cryptography.md)); skip lists (doc 03). Las Vegas (always right, time random) vs. Monte Carlo (fast, tiny error probability — driven below hardware-failure rates by repetition).
- **Approximation & sketches**: for NP-hard optimization, provable near-optimality (e.g., 2-approximate vertex cover, Christofides for TSP); for massive streams, tiny-memory approximate answers — distinct counts (HyperLogLog), frequencies (count-min sketch), quantiles: see [../fundamentals/](../fundamentals/), where these have their own deep dives.

# 10 — Strings

Text is data structure + algorithm territory of its own: substring search (naive O(nm); Knuth-Morris-Pratt and Boyer-Moore reach O(n+m) — grep's engine; Rabin-Karp's rolling hash generalizes to plagiarism detection and chunking), suffix arrays/trees (all substrings, indexed — bioinformatics, full-text search), edit distance (§7.3), and compression (Huffman §5 + LZ77 dictionary methods = DEFLATE/zip/PNG; grasping LZ's "point backward at what repeated" is a genuine aha). Regular expressions — the everyday string tool — are automata in disguise: [doc 07 §2](07-theory-of-computation.md).

# 11 — Practical Algorithm Engineering

How the canon meets production:

1. **The performance food chain**: better algorithm ≫ better data structure ≫ micro-optimization. An O(n²)→O(n log n) fix beats any amount of code tuning; conversely don't tune what profiling hasn't indicted ([OS doc Part 14](operating-systems.md)).
2. **Know your n**. n ≤ 1,000 → O(n²) is fine and simple wins. n ~ 10⁶ → need O(n log n). n ~ 10⁹+ → O(n) passes, streaming, or sketches. Competitive programmers reverse-engineer the intended algorithm from the stated limits; you can too.
3. **Reuse before invention**: your language's sort, hash map, and `bisect`; graph libraries (NetworkX); solvers (SAT/LP). The skill is *recognition* — "this is topological sort," "this is knapsack," "this is a max-flow" — then reaching for the tuned implementation.
4. **The memory hierarchy has veto power** (doc 01 §7.3, doc 03 passim): sequential beats clever-but-scattered; that's why B-trees, Timsort's runs, and external merge sort look the way they do.
5. **Amdahl and parallelism**: independent subproblems (divide-and-conquer, map/reduce) parallelize; sequential dependencies (DP's later-states-need-earlier) resist. Paradigm choice today is also a scalability choice — see [../distributed-systems/](../distributed-systems/).

# 12 — The Road to Expertise

## 12.1 Books

1. **Grokking Algorithms** (Bhargava) — the gentle illustrated on-ramp.
2. **Algorithms** (Sedgewick & Wayne) + Coursera Parts I–II — the standard working-programmer text.
3. **Algorithm Design Manual** (Skiena) — the *practitioner's* book: war stories + a catalog of "your problem is actually X" mappings. The best second book.
4. **CLRS** — the rigorous reference (proofs, recurrences); consult, don't march through.
5. **Algorithm Design** (Kleinberg & Tardos) — the best *paradigm-first* treatment (greedy exchange arguments, DP, flows).
6. **Competitive Programmer's Handbook** (free PDF) — dense recipes, if you enjoy contests.

## 12.2 Doing

1. **Implement the canon** from this document: binary search (get the boundaries right *without* looking), the three fast sorts, BFS/DFS + topo sort, Dijkstra with a heap, union-find, and five DP classics (knapsack, edit distance, LCS, LIS, coin change). ~two weeks of evenings, permanent ownership.
2. **Structured problem practice**: LeetCode/NeetCode-150 organized by pattern, or Advent of Code (more fun, same muscles). Target: recognize *which paradigm* within two minutes; 150–300 problems makes recognition automatic. Always state the complexity before coding.
3. **Trace real systems to the algorithm**: `git bisect` (binary search), `git diff` (LCS), your package manager (topo sort + SAT), GPS (A\*), zip (Huffman+LZ), database EXPLAIN (join-order DP — [doc 06](06-databases.md)).
4. Then read [Theory of Computation](07-theory-of-computation.md) for the limits — what no algorithm can do, and what P vs NP really claims.

## 12.3 Ideas to retain forever

1. **Scaling shape beats constant speed** — big-O first, benchmarks second, both eventually.
2. **Halving is magic**: anything monotonic can be binary searched; log n ≈ free.
3. **n log n is sorting's floor** (for comparisons) — and hybrid portfolios, not pure algorithms, are what actually ships.
4. **Divide & conquer** when subproblems are independent; it's also the parallelism paradigm.
5. **Greedy needs a proof** — locally-best is globally-best only for special structure; carry the coin counterexample.
6. **Graphs are everywhere once you look** — and BFS/DFS/topo-sort/Dijkstra cover 90% of encounters.
7. **DP = recursion minus repetition**: define the state *in words*, write the recurrence, count states × work.
8. **When structure runs out, search with pruning** — and modern solvers make "encode it as SAT" a real strategy.
9. **Randomness and approximation are tools, not cheating** — tiny error bounds buy huge speed/space wins.
10. **Recognition is the expert skill**: most problems are a classic in disguise.

---

*Next: [Operating Systems](operating-systems.md) — how one machine runs many programs — then [Computer Networks](05-computer-networks.md).*
