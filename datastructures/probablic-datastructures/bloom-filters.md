# Bloom Filters & Cuckoo Filters — Approximate Membership

Part of the [Probabilistic Data Structures](probabilistic-data-structures.md) series.

## 1. What it is

A **Bloom filter** (Burton Bloom, 1970) is a bit array of `m` bits plus `k` hash functions
that represents a set and answers one question — *"is x in the set?"* — with an asymmetric
guarantee:

- **"No" is always correct** (zero false negatives).
- **"Yes" may be wrong** with a tunable false-positive probability `p`.

There is no way to enumerate members, no way to delete (in the base structure), and the keys
themselves are never stored — only the bits their hashes touch.

```
insert(x):  for i in 1..k: bits[hᵢ(x) mod m] = 1
query(x):   AND over i in 1..k of bits[hᵢ(x) mod m]     # any 0 ⇒ definitely absent
```

## 2. The problem it solves

The pattern is universal: **a cheap, memory-resident gatekeeper in front of an expensive
lookup.** The expensive thing might be a disk read, a network hop, a database query, or a
scan. A false positive costs one wasted expensive lookup; a true negative saves one. Because
false negatives are impossible, the filter can never cause you to *miss* data — it can only
fail to save you work. That asymmetry is exactly what makes it safe to bolt onto read paths.

Canonical shapes of the problem:

- *"Does this SSTable contain key k?"* — LSM-tree point reads (RocksDB, Cassandra, HBase).
- *"Has this URL/object been requested before?"* — cache admission (Akamai).
- *"Has this user already been shown this item?"* — feed/recommendation dedup.
- *"Is this row group relevant to the predicate?"* — Parquet page skipping.

## 3. The math you actually need

With `n` inserted elements, `m` bits, `k` hash functions, the probability a given bit is
still 0 is `(1 − 1/m)^{kn} ≈ e^{−kn/m}`. A false positive requires all `k` probed bits to be
1, so:

```
p ≈ (1 − e^{−kn/m})^k
```

Optimizing over `k` gives the two formulas worth memorizing:

```
k*   = (m/n) · ln 2  ≈ 0.693 · m/n            (optimal number of hashes)
m/n  = −log₂(p) / ln 2 ≈ 1.44 · log₂(1/p)     (bits per element for target p)
```

Consequences:

| Target FP rate | bits/element | optimal k |
|---|---|---|
| 10% | 4.8 | 3 |
| 1% | 9.6 | 7 |
| 0.1% | 14.4 | 10 |

Note what is *absent* from the formula: the size of the keys. A Bloom filter for 1 KB URLs
costs the same 9.6 bits/element as one for 8-byte integers. Also note the information-
theoretic lower bound for approximate membership is `log₂(1/p) ≈ 1.44× fewer` bits — a Bloom
filter is ~44% above optimal, which is the gap newer filters (cuckoo, Ribbon) close.

**Two more practical facts:**

- **Filters saturate, not degrade gracefully.** `p` is a function of `n/m`; overfill a filter
  2× past its design point and the FP rate doesn't double — it can rise by an order of
  magnitude. Size for peak `n`, and rebuild (or use a scalable-Bloom scheme of stacked
  filters) when you overrun.
- **You don't need k independent hash functions.** Kirsch & Mitzenmacher (2006) proved
  `gᵢ(x) = h₁(x) + i·h₂(x) mod m` preserves the asymptotic FP rate. Every serious
  implementation (Guava, RocksDB) derives all probes from one or two 64-bit hashes.

## 4. Worked example

Design a filter to protect a user-profile database from lookups of nonexistent users
(cache-penetration protection — a real pattern for handling scrapers and typo traffic):

- Expected users: `n = 100M`
- Acceptable wasted-DB-read rate: `p = 1%`

Then `m = 100M × 9.6 bits ≈ 120 MB` and `k = 7`. Compare: a hash set of 100M 16-byte user
IDs plus overhead is ≥ 3–4 GB. The filter fits comfortably in RAM on every API node.

Insert `"user:alice"`: compute `h₁ = 0x9d3f…`, `h₂ = 0x5b21…`, derive 7 probe positions,
set those 7 bits. Query `"user:bob"` (never inserted): 7 probes; the moment any probe hits a
0 bit, return "definitely absent" — the DB is never touched. Roughly 1 in 100 absent users
will have all 7 bits collide with bits set by real users; those pay one wasted DB read.

The subtle operational detail: on `INSERT user`, you must also add to the filter on every
node (or rebuild/broadcast periodically), and since base Bloom filters can't delete, deleted
users remain as false-positive-ish "yes" answers until the next rebuild — harmless here,
because "yes" only means "go check the DB."

## 5. Engineering variants

### Counting Bloom filter (deletable, 4–8× larger)
Replace each bit with a small counter (4 bits is standard); insert increments, delete
decrements, query checks all counters > 0. Introduced in the *Summary Cache* work (Fan, Cao,
Almeida, Broder, 1998/2000) for web-proxy cache sharing. Counter overflow must be handled
(saturate, never wrap — a wrapped counter creates false negatives).

### Blocked Bloom filter (cache-line friendly)
A standard filter with k=7 costs up to 7 cache misses per query. Blocked filters (Putze,
Sanders, Singler, 2007) first hash to one 64-byte block, then set/probe all k bits *inside
that cache line*: one memory access per query, SIMD-friendly, at the cost of a slightly
higher FP rate for the same m/n. RocksDB's modern full filters and Parquet's
**split-block Bloom filter** (part of the Parquet format spec) use this design.

### Ribbon filter (space-optimal-ish, static)
RocksDB's Ribbon filter (Dillinger & Walzer, 2021) abandons the Bloom construction entirely
in favor of solving a linear system over the keys, landing much closer to the
information-theoretic bound — RocksDB reports roughly **30% space savings vs. its Bloom
filters at the same FP rate**, paying ~4× more CPU at *construction* time with comparable
query cost. Only viable because SSTable filters are built once at flush/compaction and never
mutated — a good example of exploiting immutability. Configured via
`NewRibbonFilterPolicy` as a drop-in for `NewBloomFilterPolicy`.

### Cuckoo filter (deletable *and* smaller, at low FP rates)
Fan, Andersen, Kaminsky, Mitzenmacher (CoNEXT 2014). Instead of bits, store a short
**fingerprint** (e.g., 8–16 bits) of each key in a cuckoo hash table with buckets of 4 slots
and two candidate buckets per key. The trick making cuckoo *hashing* work without the
original keys is **partial-key cuckoo hashing**: the alternate bucket is computed from the
fingerprint itself,

```
i₂ = i₁ XOR hash(fingerprint)
```

so an entry can be kicked between its two buckets knowing only (bucket, fingerprint).
Properties vs. Bloom:

- Supports **deletion** (remove one matching fingerprint — only safe if the key was
  actually inserted).
- **Better space than Bloom below ~3% FP** (the paper's crossover) and ~95% achievable load
  factor with 4-way buckets.
- Two cache misses worst case per query; inserts can fail when the table is near-full
  (needs a small eviction-loop bound and an overflow strategy).

RedisBloom ships both (`BF.*` and `CF.*` commands); cuckoo is the choice when you need
deletes or count-limited membership.

## 6. Who uses it, for what (grounded)

- **RocksDB / LevelDB** — per-SSTable filters consulted on every point read (`Get`) to skip
  files whose key range matches but which don't contain the key; default ~10 bits/key ≈ 1%
  FP. Without filters, a read in a leveled LSM touches one file per level; with them, almost
  only the file that has the key. Ribbon offered as the space-optimized alternative.
  ([RocksDB wiki: Bloom Filter](https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter),
  [RocksDB blog: Ribbon Filter](https://rocksdb.org/blog/2021/12/29/ribbon-filter.html))
- **Apache Cassandra / ScyllaDB / HBase** — same LSM role. Cassandra exposes
  `bloom_filter_fp_chance` per table (default 0.01; leveled compaction defaults to 0.1
  because LCS already bounds the number of SSTables per read) — a rare example of the
  FP-rate/memory trade-off surfaced as a user-facing schema knob.
  ([Cassandra docs: bloom filters](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/bloom_filters.html))
- **Akamai** — Bloom filter as **cache admission**: an object is cached only on its second
  request within a window, because ~75% of requested objects are "one-hit wonders" never
  requested again. Reported effect: roughly doubled hit rate, halved disk writes.
  ([Maggs & Sitaraman, SIGCOMM CCR 2015](https://people.cs.umass.edu/~ramesh/Site/PUBLICATIONS_files/CCRpaper_1.pdf))
- **PostgreSQL** — the `bloom` extension provides a Bloom-filter *index access method* for
  multi-column equality (any subset of columns, one small index), and PG14+ added a `bloom`
  opclass for BRIN indexes. ([PostgreSQL docs: bloom](https://www.postgresql.org/docs/current/bloom.html))
- **Apache Parquet / ORC** — optional per-column Bloom filters in file metadata let engines
  (Spark, Trino, Impala) skip row groups for selective predicates on high-cardinality
  columns where min/max statistics are useless.
- **Bitcoin** — BIP-37 had SPV light clients send Bloom filters of their addresses to full
  nodes to receive only relevant transactions. Instructive *failure* case study: the
  filters leaked wallet privacy (an FP rate low enough to be useful is high enough to
  fingerprint) and enabled DoS on full nodes; superseded by BIP-157/158 "compact block
  filters" (Golomb-coded sets, filters computed by the *server*, membership tested by the
  client — inverting who reveals what).
- **Google Chrome** — historically used a local Bloom filter for Safe Browsing URL checks
  before replacing it with a more compact exact prefix-set structure; still the textbook
  example of "filter locally, confirm remotely."

## 7. Failure modes & operational notes

- **Never use where a false positive breaks correctness** — e.g., "if the filter says the
  idempotency key exists, reject the request" silently drops ~p of valid requests. Filters
  gate *optimizations*, not *decisions*.
- **Deletes without counting/cuckoo create false negatives** — the one guarantee you had is
  gone. If you need deletes, change structure, don't clear bits.
- **Plan the rebuild path.** Filters drift from truth (saturation, deletions, config
  changes). LSM engines get rebuilds for free at compaction; standalone filters need an
  explicit rebuild-from-source-of-truth job.
- **Adversarial inputs:** with a known hash seed, collisions can be manufactured. Seed
  per-instance, or use a keyed hash when input is untrusted.
- **Sizing rule of thumb:** `m = 1.44 · n · log₂(1/p)` bits, `k = round(0.693·m/n)`, and
  budget for peak `n`, not average.
