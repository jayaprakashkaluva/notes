# Data Structures: How Programs Organize Data

Assumes [Programming Fundamentals](02-programming-fundamentals.md). A data structure is a way of arranging data in memory so that the operations *you* need are fast. There is no best structure — only trade-offs — and choosing well is among the highest-leverage decisions in programming: the right structure often makes the algorithm obvious.

---

## Table of Contents

1. [How to Compare Structures: Big-O in Brief](#1--how-to-compare-structures-big-o-in-brief)
2. [Arrays](#2--arrays)
3. [Dynamic Arrays](#3--dynamic-arrays)
4. [Linked Lists](#4--linked-lists)
5. [Stacks and Queues](#5--stacks-and-queues)
6. [Hash Tables](#6--hash-tables)
7. [Trees](#7--trees)
8. [Binary Search Trees and Balanced Trees](#8--binary-search-trees-and-balanced-trees)
9. [Heaps and Priority Queues](#9--heaps-and-priority-queues)
10. [Tries](#10--tries)
11. [Graphs](#11--graphs)
12. [Choosing: The Decision Guide](#12--choosing-the-decision-guide)
13. [The Road to Expertise](#13--the-road-to-expertise)

---

# 1 — How to Compare Structures: Big-O in Brief

(Fully developed in [Algorithms](04-algorithms.md); here's the working minimum.)

We describe cost by **how it grows with input size n**, ignoring constants and small terms:

| Notation | Name | Feel (n = 1,000,000) |
|---|---|---|
| O(1) | constant | instant, regardless of n |
| O(log n) | logarithmic | ~20 steps — "repeated halving" |
| O(n) | linear | 1M steps — "look at everything once" |
| O(n log n) | linearithmic | ~20M steps — good sorting |
| O(n²) | quadratic | 10¹² steps — "compare everything to everything"; death for large n |

Each structure below is summarized by the big-O of its operations — but **watch for the recurring twist**: on real hardware, constants and cache behavior (doc 01 §7.3) regularly overturn big-O ties, and the winner on paper loses in the benchmark. Both levels of analysis are part of expertise.

---

# 2 — Arrays

A contiguous block of memory holding elements of the same size:

```
index:    0     1     2     3     4
       +-----+-----+-----+-----+-----+
       | 17  | 42  |  8  | 99  | 23  |     base address = 0x1000
       +-----+-----+-----+-----+-----+
address of element i = base + i × element_size    -> one multiply+add
```

- **Index access: O(1)** — the formula above; no searching, just arithmetic. This is *the* defining virtue.
- **Search (unsorted): O(n)**; **sorted: O(log n)** via binary search (doc 04).
- **Insert/delete in the middle: O(n)** — everything after must shift.
- **Cache behavior: superb** — contiguity is exactly the spatial locality doc 01 §7.3 rewards; iterating an array streams through the prefetcher at full speed.

Zero-based indexing + the address formula is *why* `arr[0]` is the first element. Out-of-bounds access is the classic C catastrophe (buffer overflow — [doc 10](10-security-and-cryptography.md)); safe languages bounds-check every access.

# 3 — Dynamic Arrays

Fixed size is intolerable in practice, so every language wraps arrays in an auto-growing structure — Python `list`, Java `ArrayList`, C++ `vector`, JS `Array`, Go slice. Mechanism: keep a bigger backing array than needed (*capacity* ≥ *length*); append into the spare room in O(1); when full, allocate a new array **double** the size and copy everything (O(n) that once).

The accounting that justifies "double": growing to n elements costs total copies n/2 + n/4 + ... < n, so appends are **O(1) amortized** — expensive moments, cheap on average. (Amortized analysis returns in doc 04.) Growing by a *constant* instead (+10 each time) would make appends O(n) amortized — a real bug people have shipped.

**The workhorse verdict**: dynamic arrays are the default collection in all languages for good reason — O(1) access, O(1) amortized append, unbeatable cache behavior. Reach for something else only when a *specific* operation (mid-insertion, keyed lookup, min-extraction, prefix search) dominates your workload.

# 4 — Linked Lists

Each element (*node*) holds a value plus a pointer to the next node; nodes live anywhere in memory:

```
head -> [17|next] -> [42|next] -> [8|next] -> null
```

(Doubly-linked: also a `prev` pointer, allowing backward traversal and O(1) removal of a node you hold.)

- **Insert/delete at a known node: O(1)** — rewire two pointers, nothing shifts.
- **Index access / search: O(n)** — must walk from the head.
- **Cache behavior: terrible** — every `next` hop is a potential cache miss into a random address; doc 01 §7.3's pointer-chasing worst case.

The honest modern assessment: linked lists **lose to dynamic arrays for almost every real workload**, including many mid-insertion ones — O(n) shifting of contiguous memory is often faster than O(1) pointer surgery plus O(n) cache-missing traversal to *find* the spot. They remain essential (a) as a concept — "nodes + pointers" is the germ of trees and graphs, (b) inside systems where nodes are embedded in larger objects and never traversed by index (OS kernels use *intrusive* lists heavily — run queues, wait lists), (c) in interviews, and (d) as building blocks (hash-table chaining, §6; LRU caches = hash map + doubly-linked list — a classic composite worth knowing).

**Skip lists** — linked lists with probabilistic express lanes giving O(log n) search — power Redis sorted sets; see [../fundamentals/skip-lists.md](../fundamentals/skip-lists.md).

# 5 — Stacks and Queues

Not structures so much as *disciplines* imposed on a list — worth having as vocabulary because they're everywhere:

- **Stack — LIFO** (last in, first out): `push` on top, `pop` from top, O(1) each (a dynamic array does it perfectly). Appears as: the call stack (doc 01 §6), undo history, back-button, matching brackets, depth-first search (§11), expression evaluation ([doc 08](08-compilers-and-languages.md)).
- **Queue — FIFO** (first in, first out): `enqueue` at back, `dequeue` from front (ring buffer or deque for O(1) both ends). Appears as: task/print/message queues, breadth-first search (§11), OS scheduler run queues, producer/consumer pipelines ([OS doc Part 6](operating-systems.md)), every buffering layer in networking.
- **Deque**: O(1) at both ends; generalizes both.

# 6 — Hash Tables

The most important data structure in practical programming — the mechanism behind Python `dict`/`set`, Java `HashMap`, JS objects/`Map`, Go `map`, every cache, every index-by-key anywhere.

## 6.1 The idea

You want dictionary operations: `put(key, value)`, `get(key)`, `delete(key)` — by *key*, not index. Trick: a **hash function** converts any key into a number, mashing its bits into an effectively-random-looking but *deterministic* value; take it modulo the array size and you get an index:

```
index = hash("alice") % 16   ->  7      buckets[7] = ("alice", 91)
```

Lookup recomputes the hash → same index → there's your value. **O(1) average** for get/put/delete — no scanning, no comparisons but one.

## 6.2 Collisions

Two keys will inevitably hash to the same bucket (pigeonhole: more possible keys than buckets). Two standard remedies:

- **Chaining**: each bucket holds a little list of entries; scan it on lookup (short, if the table is sized right).
- **Open addressing**: on collision, probe the next slots by some rule until an empty one; modern high-performance tables (Swiss tables — Abseil, Rust's hashbrown) do this with SIMD-scanned control bytes, winning on cache behavior.

**Load factor** = entries ÷ buckets. Keep it bounded (typically < ~0.75–0.9) by **resizing** — allocate double, rehash everything (O(n) occasionally, O(1) amortized — the dynamic-array move again). Let it grow unbounded and O(1) degrades to O(n).

## 6.3 The fine print (where real engineering lives)

- **Worst case is O(n)** — if all keys collide. This is *exploitable*: attackers sending thousands of deliberately-colliding keys turned web frameworks into CPU furnaces (HashDoS, 2011) — the reason languages now seed hashes randomly per process (also why Python `set`/`dict` iteration order can't be relied on across runs, and why Go *deliberately randomizes* map iteration order to stop you depending on it).
- **Keys must be hashable and equal-consistent**: `a == b` must imply `hash(a) == hash(b)` (the contract behind Java's "override equals and hashCode together"; why mutable objects make dangerous keys — mutate a key after insertion and it's lost in the wrong bucket).
- **No order**: hash tables scatter keys by design. Need sorted traversal or range queries ("all keys between X and Y")? That's what trees are for (§8) — the single most common reason to reach past a hash map.
- Cousins: **sets** (hash table storing only keys — membership tests in O(1)); **bloom filters** (probabilistic membership in tiny space, small false-positive rate — see [../fundamentals/bloom-filters.md](../fundamentals/bloom-filters.md)).

# 7 — Trees

A hierarchy of nodes: one **root**, every other node has exactly one **parent**; nodes with no children are **leaves**. Trees are how you represent anything nested — file systems, HTML/JSON documents, organization charts, expression syntax ([doc 08](08-compilers-and-languages.md)), decision processes.

Vocabulary: **depth/height** (distance from root / to deepest leaf), **subtree** (any node is the root of its own tree — *this recursive shape is why recursion, doc 02 §5, is the natural tree idiom*), **binary tree** (≤ 2 children each).

Traversals — visiting every node systematically:

- **Depth-first (DFS)**, three flavors by when you visit the node itself: *pre-order* (node, then children — copying a tree), *in-order* (left, node, right — yields sorted order in a BST, §8), *post-order* (children, then node — deleting a tree, computing sizes bottom-up). Naturally recursive, or a loop + explicit stack.
- **Breadth-first (BFS)**: level by level, via a queue. (Both return, generalized, in §11 — a tree is just a graph without cycles.)

# 8 — Binary Search Trees and Balanced Trees

## 8.1 BST: ordering made structural

**Invariant**: every node's left subtree holds only smaller keys; the right, only larger. Search is then twenty-questions: compare at the root, go left or right, repeat — discarding half the (balanced) tree per step. **O(log n)** search/insert/delete, *plus* everything hash tables can't do: **in-order = sorted**, min/max, floor/ceiling, and **range queries**.

## 8.2 The balance problem

Insert keys in sorted order into a naive BST and every node chains rightward — height n, everything O(n). **Self-balancing** trees restructure on the fly to pin height at O(log n):

- **Red-black trees**: color-based rules bounding imbalance ≤ 2×; rebalancing via local *rotations* (O(1) pointer surgeries that preserve BST order). The standard: Java `TreeMap`, C++ `std::map`, the Linux CFS scheduler's run queue ([OS doc §4.4](operating-systems.md)).
- **AVL trees**: stricter balance — slightly faster lookups, more rotation work on writes.
- Don't memorize the rules; know *what* they guarantee (O(log n) worst case, sorted iteration, range queries) and *that rotations* are the mechanism.

## 8.3 B-trees: balanced trees meet the memory hierarchy

For data on **disk**, the game changes: each node visit is a ~100 µs–10 ms I/O (doc 01 §8), so you want the *shortest possible tree*. **B-trees/B+trees** give each node hundreds of keys (sized to a disk page), making the tree 3–4 levels deep for *billions* of keys. This is the data structure inside virtually every database index and most file systems — the star of [Databases §5](06-databases.md), with internals in [../datastores/](../datastores/). (Same logic, RAM edition: cache-friendly n-ary trees beat binary ones there too.) The write-optimized alternative — **LSM-trees** — appears in doc 06 and [../datastores/rocksdb-internals.md](../datastores/rocksdb-internals.md).

# 9 — Heaps and Priority Queues

A **priority queue** serves not the oldest item but the *most important*: `insert`, `peek-min`, `extract-min` (or max). The structure behind it, the **binary heap**: a complete binary tree with one weak invariant — *every parent ≤ its children* (min-heap). No left/right ordering, so far cheaper to maintain than a BST:

- The root is always the minimum → **peek O(1)**.
- Insert: place at the bottom, *bubble up* while smaller than parent — **O(log n)**.
- Extract-min: move the last element to the root, *sift down* — **O(log n)**.
- Completeness means it packs into a **plain array** with no pointers at all (children of index i live at 2i+1, 2i+2) — compact and cache-friendly.

Where it appears: OS and job schedulers, Dijkstra's algorithm (doc 04), event simulation and timer wheels, "top-K of a stream" (keep a size-K heap — a constant interview and production pattern), heapsort, merging K sorted lists.

# 10 — Tries

A tree keyed by *characters*: each root-to-node path spells a prefix; each node's children are the possible next characters. Purpose-built for **prefix operations**: autocomplete ("all words starting `pre`"), spell-checkers, IP routing tables (longest-prefix match — how routers decide where packets go, [doc 05](05-computer-networks.md)), and word games. Lookup is O(key length) — independent of how many keys are stored. Memory-hungry naive; compressed variants (radix/PATRICIA trees) fix that and appear in kernels and databases.

# 11 — Graphs

The fully general structure: **vertices** (things) and **edges** (relationships) — no root, no hierarchy, cycles allowed. Directed or undirected; weighted or not. Road maps, social networks, the web's link structure, dependency graphs (build systems, package managers, spreadsheet formulas), state machines, database query plans — "what depends on what" and "what connects to what" is always a graph, and *recognizing* the graph hiding in a problem is a core expert reflex.

Two standard representations, one classic trade-off:

- **Adjacency list** — per-vertex list of neighbors. Space O(V+E); the default, since real graphs are overwhelmingly *sparse*.
- **Adjacency matrix** — V×V grid of booleans/weights. O(1) edge test, O(V²) space; right for small dense graphs.

The algorithms that operate on graphs — BFS/DFS, shortest paths, topological sort, connected components, minimum spanning trees — are the centerpiece of [Algorithms §6](04-algorithms.md).

# 12 — Choosing: The Decision Guide

Start from the *operations your workload actually performs*, not from the data's shape:

| You mostly need... | Reach for | Why |
|---|---|---|
| Ordered items, index access, iteration | **dynamic array** | O(1) access, cache-optimal — the default |
| Lookup **by key** | **hash map** | O(1) average |
| Membership testing | **hash set** | O(1) average |
| Sorted order, range queries, floor/ceiling | **balanced BST / B-tree** | O(log n) with order preserved |
| Repeatedly take the min/max/most-urgent | **heap** | O(log n) insert/extract |
| LIFO / FIFO processing | **stack / queue** | O(1), matches the discipline |
| Prefix/autocomplete queries | **trie** | O(key length) |
| Nested/hierarchical data | **tree** | mirrors the structure |
| Arbitrary relationships | **graph** (adjacency list) | fully general |
| O(1) removal from within a scan-free sequence, kernel-style | **(intrusive) linked list** | niche but real |
| Approximate membership/counts at huge scale | **sketches** | [../fundamentals/](../fundamentals/) |

Second-order expert habits: compose structures (LRU = hash + linked list; ordered dict = hash + insertion list; graph = hash of adjacency lists); measure before trusting big-O across a factor-of-two decision (cache effects — §2 vs §4 — decide those); and remember n: below a few hundred elements, *everything* is fast and the simplest structure wins.

# 13 — The Road to Expertise

## 13.1 Build each one (the only path that works)

In your practice language, implement from scratch — no libraries: dynamic array (with doubling), singly + doubly linked list, stack, queue (ring buffer), hash map (chaining; then open addressing), BST (insert/search/delete/in-order), binary heap (on an array), trie, graph + BFS/DFS. Each is 30–150 lines. Write tests as you go ([doc 09](09-software-engineering.md) formalizes how). This project — a personal `collections` library — teaches more than any month of reading.

## 13.2 Books and resources

1. **Grokking Algorithms** (Bhargava) — friendliest illustrated introduction to structures + algorithms together.
2. **Open Data Structures** (opendatastructures.org, free) — rigorous, readable, per-structure proofs.
3. **Algorithms, 4th ed.** (Sedgewick & Wayne) + its free Coursera courses — the standard; superb visualizations.
4. **CLRS** (*Introduction to Algorithms*) — the encyclopedic reference; use per-chapter, don't read linearly.
5. VisuAlgo.net — animated operations on every structure here.

## 13.3 Ideas to retain forever

1. **Structures are operation menus with prices** — choose by the operations your workload actually needs.
2. **Contiguity is speed**: arrays and cache locality win real benchmarks; pointer-chasing loses them — big-O ties are broken by the memory hierarchy.
3. **Hash tables buy O(1) lookup by sacrificing order**; trees buy order for O(log n). This one trade-off explains half of all structure choices.
4. **Amortized ≠ average luck**: doubling makes rare O(n) moments genuinely O(1) per operation over time.
5. **Invariants are the soul of a structure** (BST ordering, heap property, load factor) — operations exist to preserve them; understand the invariant and you can re-derive the operations.
6. **Match the structure to the storage level**: binary trees for RAM, fat B-trees for disk pages, tries for prefixes, sketches for streams.
7. **Compose structures** — the best practical answers are often two structures wired together.
8. **Recursive shapes want recursive code** — trees are doc 02 §5's payoff.

---

*Next: [Algorithms](04-algorithms.md) — the recipes that run over these structures, and the science of their cost.*
