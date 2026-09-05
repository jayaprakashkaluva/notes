# Skip Lists — Probabilistic Balancing, Exact Answers

Part of the [Probabilistic Data Structures](probabilistic-data-structures.md) series.

## 1. What it is — and how it differs from the sketches

A **skip list** (William Pugh, 1990 — *Skip Lists: A Probabilistic Alternative to Balanced
Trees*, CACM) is an ordered map/set built from a sorted linked list plus a tower of
progressively sparser "express lane" lists above it. Unlike everything else in this series,
it is **not approximate**: lookups, inserts, deletes, and range scans return exact answers.
The probability lives entirely in the *shape* of the structure — each node's height is drawn
from a geometric distribution at insert time, and balance emerges statistically instead of
being enforced by rotation logic.

```
L3:  head ────────────────────────▶ 47 ─────────────────▶ nil
L2:  head ─────────▶ 21 ──────────▶ 47 ─────────▶ 89 ───▶ nil
L1:  head ──▶ 9 ───▶ 21 ──▶ 33 ───▶ 47 ──▶ 60 ──▶ 89 ───▶ nil
L0:  head ─▶ 5 ▶ 9 ▶ 14 ▶ 21 ▶ 33 ▶ 47 ▶ 52 ▶ 60 ▶ 89 ──▶ nil
```

Search starts at the top-left and moves right until the next key would overshoot, then
drops a level — expected **O(log n)** with O(n) space; a node gets height ≥ h with
probability `p^{h−1}` (p = 1/2 or 1/4). The performance guarantee is probabilistic
(*with high probability*, not worst-case), but it depends only on internal coin flips —
**no input sequence, adversarial or otherwise, can produce the degenerate cases** that a
naive BST suffers, because the input never influences the randomness. (Contrast with
family-A sketches, where an adversary who learns the *hash seed* can break the bounds.)

## 2. The problem it solves

Balanced search trees (red-black, AVL) achieve worst-case O(log n) at a price that is
invisible in textbooks and dominant in systems:

1. **Implementation complexity.** Rebalancing logic (rotations, color fix-ups, ~double-digit
   cases with symmetric variants) is notoriously error-prone. A skip list is a few dozen
   lines; Pugh's paper leads with exactly this argument.
2. **Concurrency.** Rotations restructure multiple nodes atomically, forcing coarse locks or
   heroic lock-free tree designs. Skip-list inserts touch a handful of *forward pointers*
   at independent levels — and a reader that misses a node's upper levels mid-insert still
   finds it via level 0, so the structure tolerates relaxed visibility. This makes
   lock-free/fine-grained-concurrent skip lists practical where equivalent trees are
   research projects. The two flagship results:
   - **Java's `ConcurrentSkipListMap`** (Doug Lea): the JDK's concurrent ordered map is a
     CAS-based lock-free skip list — chosen precisely because "there are no known efficient
     lock-free insertion and deletion algorithms for search trees" (as its javadoc/source
     commentary explains).
   - **RocksDB's default memtable** (`InlineSkipList`): supports **concurrent writes with
     lock-free reads** — readers never block on writers, essential when every read path
     (including iterators pinned by long scans) traverses the memtable.
3. **Ordered iteration is native.** The base level *is* a sorted linked list: range scans
   and ordered cursors are pointer-chasing, no in-order traversal state, no parent pointers.

The honest costs: worse cache locality than B-trees (pointer chasing vs. node-packed
arrays — why on-disk structures and in-memory stores optimized for scan throughput still
prefer B-trees/ART), ~1.33–2 pointers per node of overhead depending on p, and
probabilistic rather than hard worst-case bounds.

## 3. Worked example — a memtable

An LSM engine needs an in-memory buffer absorbing random-order writes that (a) supports
concurrent writers, (b) supports point reads *and* ordered scans (for flush and for
read-path merging), (c) never blocks readers.

Insert `key=47` into the skip list above:

1. Walk from the top, recording at each level the last node whose key < 47 (the "update
   vector") — this is the same walk as a search.
2. Flip coins: height = 1 + (number of consecutive heads), capped (RocksDB/Redis cap at
   ~12–64 depending on expected n). Say height = 3.
3. Splice the node into levels 0–2 by pointer swaps against the update vector. In the
   concurrent version each level's splice is a CAS; a concurrent reader either sees the new
   node (fine) or doesn't yet (fine — it wasn't visible before either). No global lock, no
   rebalancing storm.

At flush time, iterate level 0 → keys emerge sorted → write the SSTable sequentially. The
structure did the sort incrementally, concurrently, with no rebalancing.

## 4. Who uses it, for what (grounded)

- **Redis sorted sets (ZSET)** — implemented as a skip list (`zskiplist`, p = 1/4) paired
  with a hash map (O(1) member→score). The skip list carries **spans** (number of level-0
  links each forward pointer jumps), turning rank queries — `ZRANK`, `ZRANGEBYRANK` — into
  O(log n) by summing spans along the search path: a clean example of augmenting the
  structure. Antirez's stated reasons for choosing skip lists over balanced trees:
  comparable performance, simpler implementation, and natural range operations.
- **RocksDB / LevelDB memtables** — LevelDB introduced the skip-list memtable
  (single-writer, lock-free readers); RocksDB's `InlineSkipList` extended it to concurrent
  writers (`allow_concurrent_memtable_write`), keys inlined into nodes for cache behavior.
  ([RocksDB wiki: MemTable](https://github.com/facebook/rocksdb/wiki/MemTable))
- **Java `ConcurrentSkipListMap` / `ConcurrentSkipListSet`** — the JDK's only concurrent
  *ordered* collections; every JVM service using a concurrent ordered index (schedulers,
  time-ordered queues, order books) runs on them.
- **Apache Lucene** — multi-level skip structures over postings lists let query evaluation
  jump over non-matching document ranges (`advance()`), central to conjunction scoring;
  the same idea appears in every serious inverted index.
- **MemSQL/SingleStore** — chose lock-free skip lists as the in-memory rowstore index,
  arguing the concurrency profile beats B-trees under mixed workloads
  ([SingleStore engineering blog on skiplists](https://www.singlestore.com/blog/what-is-skiplist-why-skiplist-index-for-memsql/)).

## 5. Design notes

- **p trades depth for space**: p = 1/2 → shallower towers, ~2 pointers/node; p = 1/4
  (Redis, common) → ~1.33 pointers/node, slightly deeper searches. Cap max level at
  ~log₁ᵤₚ(N_max).
- **Determinism-adjacent variants exist** (deterministic skip lists, skip graphs for P2P
  overlays), but the plain randomized version's simplicity is the point.
- **When a B-tree/ART beats it**: read-heavy, scan-heavy, cache-sensitive workloads with
  low write concurrency — pointer chasing loses to packed nodes. The skip list's sweet spot
  is *high write concurrency + ordered semantics + in-memory*, which is exactly the
  memtable/ZSET/scheduler profile above.
