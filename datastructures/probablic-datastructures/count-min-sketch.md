# Count-Min Sketch & Heavy Hitters — Frequency Estimation

Part of the [Probabilistic Data Structures](probabilistic-data-structures.md) series.

## 1. What it is

The **Count-Min sketch** (Cormode & Muthukrishnan, 2005) is a `d × w` matrix of counters
that answers *"approximately how many times has element x occurred?"* over a stream, in
space that depends only on the error target — not on the number of distinct elements.

```
update(x, c):  for each row i:  C[i][hᵢ(x) mod w] += c
estimate(x):   min over rows i of C[i][hᵢ(x) mod w]
```

Guarantee, with `w = ⌈e/ε⌉` columns and `d = ⌈ln(1/δ)⌉` rows, over a stream of total
count `N`:

```
true(x) ≤ estimate(x) ≤ true(x) + εN     with probability ≥ 1 − δ
```

The estimate **never undercounts** — collisions only add. Taking the row-wise *min* picks
the row where x suffered the least collision noise; each row independently overshoots by
more than εN with probability ≤ 1/e (Markov's inequality on the expected collision mass
N/w), so d independent rows fail *simultaneously* with probability ≤ e^{−d} = δ.

Concretely: ε = 0.01%, δ = 0.01% → `w = 27,183`, `d = 10` → ~270k counters ≈ **2 MB** with
64-bit counters, regardless of whether the stream has 10⁵ or 10⁹ distinct keys. Like other
sketches it is **mergeable** — element-wise matrix addition — so per-shard sketches combine
losslessly.

## 2. The problem it solves

Exact per-key frequency over a high-cardinality stream is a hash map that grows with
distinct keys — unbounded and mostly wasted, because in the Zipfian distributions real
traffic follows, almost all keys occur a handful of times and you only care about the big
ones. The questions that actually get asked are:

- *"Which IPs / queries / items are hot right now?"* (heavy hitters, top-K)
- *"Has this key been accessed frequently enough to deserve cache space?"* (admission)
- *"Is this sender's volume anomalous?"* (rate estimation for abuse detection)

All of these tolerate the CM sketch's error profile: overestimates bounded by a fraction of
*total* traffic, which means **heavy keys are estimated accurately in relative terms, while
rare keys may be wildly overestimated relative to their tiny true counts**. That's the right
error shape when you only act on heavy keys — and the wrong shape if rare-key accuracy
matters (see §5).

## 3. Worked example

Rate-limit-ish abuse detection at an API gateway: flag any client sending > 0.1% of total
traffic, across ~50M distinct API keys/IPs per day.

Choose ε = 0.01% (10× tighter than the threshold, so noise can't push a benign key over) and
δ = 10⁻⁶: `w = ⌈e/0.0001⌉ ≈ 27,183`, `d = ⌈ln 10⁶⌉ = 14`. Sketch: 14 × 27,183 ≈ 380k
counters ≈ 3 MB. An exact map for 50M keys would be several GB per node.

Per request: hash the key 14 times (or derive 14 probes from one 128-bit hash), increment
14 counters. To check a client: read the same 14 counters, take the min, compare against
0.1% of the running total N. Per-shard sketches are summed every window to get the global
view. False accusations are bounded: a key estimated over threshold truly has ≥
(0.1% − 0.01%)·N traffic with probability 1−δ.

**Finding heavy hitters without knowing who to ask about:** the sketch alone can't
enumerate keys (queries require the key). The standard composition is CM sketch + a small
min-heap: on each update, estimate the updated key; if the estimate exceeds the current
heap minimum, insert/update it in a K-sized heap. The heap holds the candidate top-K with
their approximate counts; the sketch bounds the error.

## 4. The design space around it

### Conservative update
On `update(x, 1)`, increment only the counters that equal the current min (the others are
already inflated by collisions and don't need to grow to preserve the ≥-true invariant).
Cuts overestimation substantially on skewed streams at the cost of not supporting weighted
merges cleanly. Widely used in practice (e.g., network measurement literature).

### Count Sketch (the unbiased sibling)
Charikar, Chen, Farach-Colton (2002) — predates CM. Each row also has a sign hash
`sᵢ(x) ∈ {±1}`; update adds `sᵢ(x)·c`, estimate takes the **median** of `sᵢ(x)·C[i][hᵢ(x)]`.
Collisions now cancel in expectation instead of accumulating: the estimator is *unbiased*,
with error `ε‖f‖₂` (L2 norm of the frequency vector) instead of CM's `ε‖f‖₁ = εN`. On
skewed data L2 ≪ L1, so Count Sketch is more accurate per bit — but it can *under*estimate,
which disqualifies it where one-sided error is load-bearing. It is also the core of the
AMS/L2-estimation and the "CountSketch" operator used for randomized linear algebra.

### Counter-based alternatives: Misra–Gries & SpaceSaving
For pure top-K, counter-based algorithms beat sketches on accuracy per byte.
**SpaceSaving** (Metwally, Agrawal, El Abbadi, 2005) keeps exactly `k` (key, count, error)
entries; a new key evicts the minimum-count entry and *inherits its count* (recording that
count as the max possible overestimate). Guarantees: any key with true frequency > N/k is
in the table, and each count overestimates by at most the smallest counter.
**Misra–Gries** (1982) is the decrement-based ancestor with the same flavor. These store
actual keys (so they can enumerate!), and are what analytics engines mostly ship:
ClickHouse's `topK` implements Filtered Space-Saving.

### HeavyKeeper
A 2018 refinement (Gong et al., ATC '18) using exponential-decay eviction of fingerprint
counters, markedly more accurate for top-K on skewed streams; this is the algorithm behind
RedisBloom's `TOPK.*` commands.

## 5. Failure modes

- **Rare-key queries are garbage.** A key seen once can be estimated at εN — millions of
  times its true count. Never use CM estimates as per-key truth for the long tail (e.g.,
  billing, per-user quotas for small users).
- **Adversarial inflation:** anyone who knows the hash seeds can craft keys colliding with
  a victim's counters in all d rows, framing them as a heavy hitter. Seed per-deployment;
  use keyed hashes at trust boundaries.
- **Decay must be designed in.** Counters only grow; "hot *right now*" needs either
  windowed sketch rotation (keep sketches per time bucket, sum the window) or periodic
  halving of all counters (the aging trick TinyLFU uses).
- **ε is relative to N, not per-key.** Doubling traffic doubles absolute error everywhere.
  Thresholds expressed as fractions of N stay sound; absolute-count thresholds silently
  degrade.

## 6. Who uses it, for what (grounded)

- **Caffeine (Java caching library) — W-TinyLFU admission.** The highest-leverage deployment
  of CM sketches most engineers already run. TinyLFU (Einziger, Friedman, Manes, 2017)
  answers "is the incoming key historically more frequent than the eviction victim?" using a
  4-bit-counter Count-Min sketch with periodic halving ("reset") for freshness. This
  admission gate is why Caffeine tops LRU-family hit rates across trace benchmarks; Caffeine
  in turn backs caching inside Cassandra, Kafka brokers, Spring's cache abstraction, and
  much of the JVM ecosystem.
  ([TinyLFU: A Highly Efficient Cache Admission Policy](https://arxiv.org/abs/1512.00727),
  [Caffeine design docs](https://github.com/ben-manes/caffeine/wiki/Design))
- **RedisBloom** — `CMS.INCRBY` / `CMS.QUERY` / `CMS.MERGE` expose a raw Count-Min sketch;
  `TOPK.*` exposes HeavyKeeper-based top-K.
  ([Redis docs: Count-min sketch](https://redis.io/docs/latest/develop/data-types/probabilistic/count-min-sketch/))
- **ClickHouse** — `topK(N)` / `topKWeighted` aggregate functions (Filtered Space-Saving)
  for approximate most-frequent-values in SQL.
  ([ClickHouse docs: topK](https://clickhouse.com/docs/en/sql-reference/aggregate-functions/reference/topk))
- **Twitter's Algebird** — open-sourced `CMS` and `TopCMS` as monoids for Summingbird/Scalding
  streaming jobs — trend detection expressed as a mergeable aggregation, the canonical
  "sketches as monoids" codebase. ([Algebird docs: CountMinSketch](https://twitter.github.io/algebird/datatypes/approx/countminsketch.html))
- **Apache DataSketches** — ships frequency sketches (Misra–Gries-family "Frequent Items")
  used from Druid for approximate top-N over pre-aggregated data.
- **Network measurement** — the sketch literature's home turf: switch/router heavy-hitter
  detection (flow monitoring, DDoS detection) in software (sFlow-style collectors) and in
  programmable data planes, where a few MB of SRAM counters per pipeline stage is exactly
  the CM sketch's shape.
