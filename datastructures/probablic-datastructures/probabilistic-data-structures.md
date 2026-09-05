# Probabilistic Data Structures — Overview & Index

Probabilistic data structures trade a small, mathematically bounded amount of accuracy for
dramatic — often orders-of-magnitude — savings in space and time. They are the reason a
12 KB Redis key can count a billion distinct users, and the reason Cassandra can decide an
SSTable *definitely does not* contain a key without touching disk.

This is the index document. Each structure has its own deep-dive:

| Note | Structure | Question it answers |
|---|---|---|
| [Bloom Filters & Cuckoo Filters](bloom-filters.md) | Bloom filter, counting BF, blocked BF, Ribbon, cuckoo filter | "Have I seen this element before?" (approximate membership) |
| [HyperLogLog](hyperloglog.md) | Flajolet–Martin → LogLog → HLL → HLL++ | "How many *distinct* elements have I seen?" (cardinality) |
| [Count-Min Sketch & Heavy Hitters](count-min-sketch.md) | Count-Min, Count Sketch, SpaceSaving, TinyLFU | "How many times has X occurred? What are the top-K?" (frequency) |
| [Quantile Sketches](quantile-sketches.md) | t-digest, DDSketch, KLL, GK | "What is my p99 latency?" (rank/quantile estimation) |
| [MinHash, SimHash & LSH](minhash-simhash-lsh.md) | MinHash, SimHash, LSH banding, Theta sketch | "How similar are these two sets/documents?" (similarity) |
| [Skip Lists](skip-lists.md) | Skip list | Ordered map/set with probabilistic *balancing* (exact answers) |

---

## 1. Why "probabilistic" at all?

Exact answers to some questions have provable lower bounds on memory:

- **Exact distinct count** of a stream drawn from a universe of size `U` requires Ω(U) bits
  in the worst case (you must be able to distinguish every possible subset you might have seen).
- **Exact membership** for a set of `n` arbitrary keys requires storing information
  proportional to the keys themselves.
- **Exact quantiles** over a stream require Ω(n) space in a single pass (Munro & Paterson, 1980).

At the scale of a CDN edge server, an ad-analytics pipeline, or an LSM-tree storage engine,
"store everything" is not an option — or is an option whose cost is dominated by RAM and
cache-line misses. Probabilistic structures sidestep the lower bounds by relaxing the
guarantee: the answer is *approximately* correct, or correct *with high probability*, and the
error is a tunable parameter you pay for in bits.

The recurring bargain, concretely:

| Task | Exact cost | Sketch cost | Typical error |
|---|---|---|---|
| Membership over 1B keys | tens of GB (hash set of keys) | ~1.2 GB @ 1% FP (Bloom, 9.6 bits/key) | 1% false positives, 0% false negatives |
| Distinct count of 1B items | tens of GB (hash set) | 12 KB (HLL, 2¹⁴ registers) | ±0.81% standard error |
| Per-item frequency, 1B events | GBs (hash map) | a few MB (Count-Min) | overestimate ≤ εN with prob 1−δ |
| p99 over 1B samples | 8 GB (all samples) | ~10 KB (t-digest/DDSketch) | ~1% relative error at tails |

## 2. A taxonomy worth internalizing

Not all "probabilistic data structures" are probabilistic in the same way. Two distinct
families get conflated, and the distinction matters when you reason about failure modes:

**A. Approximate-answer structures (sketches).**
The structure is deterministic once its hash functions are fixed, but the *answer* carries
bounded error: Bloom filters (false positives), HyperLogLog (relative error on cardinality),
Count-Min (overestimation), t-digest (rank error). These are lossy compressions of a
multiset, built on hashing.

**B. Randomized structures with exact answers.**
The *answers* are always exact; randomness is used internally to keep the structure balanced
without coordination. Skip lists and treaps are the canonical examples: expected O(log n)
operations, with the "probabilistic" part being the running time distribution, never
correctness. These compete with red-black/B-trees, not with sketches.

Family A is where the interesting distributed-systems properties live, and three of those
properties explain almost all production adoption:

1. **Sublinear space.** Size depends on the error target (and sometimes log log of the
   cardinality), not on the data size. An HLL is 12 KB whether it has seen 10³ or 10⁹ items.
2. **One-pass, streaming updates.** O(1) or O(k) work per element, no re-reads. This is what
   makes them fit stream processors (Flink, Kafka Streams) and hot write paths (LSM memtables).
3. **Mergeability.** Most sketches form a commutative monoid: `merge(sketch(A), sketch(B)) =
   sketch(A ∪ B)`. Bloom filters merge by bitwise OR, HLLs by register-wise max, Count-Min by
   matrix addition. This is the killer feature for distributed systems — each shard/mapper
   sketches locally, a coordinator merges, and the error bounds still hold. It is why
   `approx_distinct` parallelizes in Presto/Trino and why pre-aggregated HLL columns work in
   Druid and BigQuery.

## 3. The shared machinery: hashing as a randomness source

Every family-A structure follows the same recipe:

```
element ──hash──▶ pseudo-random point(s) in a small space ──▶ update tiny state
```

The analysis then treats hash outputs as uniform random variables and applies concentration
inequalities (Markov's inequality for Count-Min's one-sided bound; Chernoff-style bounds when
independent rows are combined; the delicate analysis of order statistics of geometric
variables for HLL). Practical implications a senior engineer should carry around:

- **Hash quality matters, cryptographic strength does not.** You need good avalanche behavior
  and independence, not preimage resistance. The industry defaults are MurmurHash3,
  xxHash/XXH3, and WyHash. (Theoretical papers assume k-wise independent families; production
  code uses fast hashes that empirically behave close enough.)
- **One 64-bit hash is usually enough.** Kirsch & Mitzenmacher (2006) showed two hash values
  `h1, h2` can synthesize k values as `gᵢ = h1 + i·h2` with no asymptotic loss for Bloom
  filters; HLL splits a single hash into "register index" bits and "rank" bits.
- **Adversarial inputs break the model.** All error bounds assume inputs are independent of
  the hash function. If an attacker knows your hash seed they can manufacture false positives
  (Bloom), inflate a victim key's Count-Min estimate, or degrade a skip list's balance is not
  an issue (levels are random, not hash-derived) but hash-flooding a plain hash table is.
  Structures exposed to untrusted input should use per-instance random seeds or a keyed hash
  (SipHash exists precisely because of this class of attack).
- **64-bit hashes postpone birthday problems.** With 32-bit hashes, collisions become
  material around ~10⁵ items (√2³² ≈ 65k); this is why HyperLogLog++ moved to 64-bit hashing
  and why serious sketch libraries (Apache DataSketches) are 64-bit throughout.

## 4. Choosing a structure — decision table

| You need… | Reach for | Do NOT reach for |
|---|---|---|
| "Skip the expensive lookup if the key can't exist" | Bloom filter (or Ribbon if RAM-bound, cuckoo if you need deletes) | Anything requiring false-negative-freedom *and* deletes → counting BF/cuckoo only |
| Distinct users/IPs/queries per dimension, mergeable across time & shards | HyperLogLog (HLL++ / DataSketches HLL) | Bloom filter (it answers membership, not cardinality) |
| Distinct counts with set *intersection/difference* (audience overlap) | Theta sketch (see [MinHash doc](minhash-simhash-lsh.md)) | Plain HLL — intersections via inclusion–exclusion blow up the error |
| Approximate per-key frequency / top-K in a stream | Count-Min + heap, SpaceSaving, HeavyKeeper | HLL (no frequencies), exact map (memory) |
| Latency percentiles, mergeable across hosts | t-digest or DDSketch; KLL for worst-case guarantees | Averages (they lie), storing raw samples |
| Near-duplicate detection / Jaccard similarity at scale | MinHash + LSH banding, SimHash | Pairwise comparison (O(n²)) |
| Ordered map with lock-friendly concurrency | Skip list | — (exact structure; different trade-off space) |

## 5. When *not* to use them

Bounded error is still error. Avoid sketches when:

- **Money or compliance is attached to the number.** Billing, financial reconciliation,
  inventory, anything auditable — use exact counts, or use sketches only as a cross-check.
- **The result feeds a decision with an exactness contract** ("has this idempotency key been
  used?"). A Bloom filter's false positive here means silently dropping valid work. Bloom
  filters are safe only when a false positive costs *extra work* (a wasted disk read), never
  when it costs *correctness*.
- **Cardinalities are tiny.** A hash set of 10k entries is small, exact, and simpler to debug.
  Sketches earn their complexity at scale. (Good libraries handle this internally: Redis HLL
  and HLL++ both use an exact/sparse mode at low cardinality and switch to the dense sketch
  only when it becomes cheaper.)
- **You need to enumerate the elements.** Sketches are one-way compressions; nothing can be
  listed or recovered from them. (This is occasionally a *feature* — sketches are used in
  privacy-conscious telemetry precisely because raw identifiers are not retained.)

## 6. Production adoption at a glance

Each deep-dive has details and citations; the pattern to notice is that these structures are
*infrastructure primitives*, embedded in the databases you already run:

- **LSM storage engines** — RocksDB, LevelDB, Cassandra, HBase, ScyllaDB: Bloom/Ribbon
  filters per SSTable to skip files during point reads; skip lists as memtables.
- **Caches/CDNs** — Akamai: Bloom filter cache-admission ("cache on second hit") roughly
  doubled hit rates and halved disk IO ([Maggs & Sitaraman, *Algorithmic Nuggets in Content
  Delivery*, SIGCOMM CCR 2015](https://people.cs.umass.edu/~ramesh/Site/PUBLICATIONS_files/CCRpaper_1.pdf));
  Caffeine's W-TinyLFU admission policy is built on a Count-Min-style frequency sketch.
- **Analytics engines** — BigQuery (`APPROX_COUNT_DISTINCT` / `HLL_COUNT.*` on HLL++),
  Presto/Trino (`approx_distinct`, `approx_percentile`), ClickHouse (`uniq*`, `topK`,
  `quantileTDigest`), Druid & Pinot (Apache DataSketches: HLL, Theta, KLL).
- **Data stores** — Redis: HyperLogLog (`PFADD`/`PFCOUNT`/`PFMERGE`) in core, sorted sets on
  skip lists; RedisBloom module adds Bloom, cuckoo, Count-Min, top-K.
- **Consumer products** — Reddit's view counts are HLLs in Redis
  ([View Counting at Reddit, 2017](https://www.redditinc.com/blog/view-counting-at-reddit/));
  monitoring vendors ship quantile sketches (Datadog's DDSketch, Elasticsearch's t-digest).

## 7. Foundational papers

| Year | Paper | Structure |
|---|---|---|
| 1970 | Bloom, *Space/Time Trade-offs in Hash Coding with Allowable Errors* | Bloom filter |
| 1985 | Flajolet & Martin, *Probabilistic Counting Algorithms for Data Base Applications* | FM sketch |
| 1990 | Pugh, *Skip Lists: A Probabilistic Alternative to Balanced Trees* | Skip list |
| 1997 | Broder, *On the Resemblance and Containment of Documents* | MinHash |
| 2002 | Charikar, *Similarity Estimation Techniques from Rounding Algorithms* | SimHash |
| 2005 | Cormode & Muthukrishnan, *An Improved Data Stream Summary: The Count-Min Sketch* | Count-Min |
| 2007 | Flajolet, Fusy, Gandouet, Meunier, *HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm* | HLL |
| 2013 | Heule, Nunkesser, Hall (Google), *HyperLogLog in Practice* | HLL++ |
| 2014 | Fan, Andersen, Kaminsky, Mitzenmacher, *Cuckoo Filter: Practically Better Than Bloom* | Cuckoo filter |
| 2019 | Masson, Rim, Lee (Datadog), *DDSketch: A Fast and Fully-Mergeable Quantile Sketch* | DDSketch |
