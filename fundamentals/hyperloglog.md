# HyperLogLog — Cardinality Estimation

Part of the [Probabilistic Data Structures](probabilistic-data-structures.md) series.

## 1. What it is

**HyperLogLog** (Flajolet, Fusy, Gandouet, Meunier, 2007) estimates the number of *distinct*
elements in a stream using `m = 2^p` small registers — kilobytes of state regardless of
whether the true cardinality is thousands or billions. The standard error is:

```
σ ≈ 1.04 / √m
```

With `p = 14` (16,384 registers): ±0.81% typical error in **12 KB** (6 bits per register).
That configuration is exactly what Redis ships. HLLs are also **mergeable**: the
register-wise `max` of two HLLs is the HLL of the union of their streams, with no loss.

## 2. The problem it solves

`COUNT(DISTINCT …)` is the most expensive innocent-looking query in analytics. Exact
distinct counting requires remembering every element seen (a hash set), which means:

- memory proportional to cardinality — GBs for billions of users/IPs/query-strings;
- no cheap pre-aggregation: distinct counts **don't add**. `distinct(Mon) + distinct(Tue)`
  double-counts returning users, so exact rollups must retain raw identifiers at every
  granularity you might later query.

HLL solves both: fixed tiny state, and union-mergeability means you can store one 12 KB
sketch per (dimension, hour) and later combine any subset — days, weeks, arbitrary dimension
slices — while keeping the same error bound. This turns "distinct users per {country ×
device × day}" from a raw-data re-scan into an aggregation of kilobyte blobs.

## 3. How it works

### Intuition: rare hash patterns witness large cardinalities

Hash every element to a uniform 64-bit string. Among the hashes, look at the position of the
leftmost 1-bit (the "rank"). A rank of `r` occurs with probability `2^{−r}` — so if the
largest rank you've *ever seen* is 20, you've plausibly seen on the order of `2^20` distinct
elements. Duplicates hash identically, so they change nothing: the estimator is inherently
duplicate-insensitive. That single-max estimator (essentially Flajolet–Martin, 1985) is
correct in expectation but wildly high-variance — one lucky hash ruins it.

### Variance reduction: many registers + harmonic mean

HLL fixes the variance with **stochastic averaging**: split the hash into two parts —

```
h(x) = [ p bits: register index j ][ remaining bits: compute rank ρ ]
M[j] = max(M[j], ρ)
```

— so the stream is implicitly partitioned into `m` substreams, each with its own max-rank
register. Each register estimates its substream's cardinality as `~2^{M[j]}`; combining them
uses the **harmonic mean**, which is dominated by the *smaller* estimates and therefore
suppresses the high outliers that plague the arithmetic mean:

```
E = α_m · m² · ( Σⱼ 2^{−M[j]} )⁻¹        α_m ≈ 0.7213/(1 + 1.079/m)
```

The 2007 paper's delicate analysis of this estimator yields the `1.04/√m` error. Raw HLL
also needs range corrections: at very low cardinality (many registers still zero) it
switches to **linear counting** (`m·ln(m/V)` where `V` = zero registers), and the original
32-bit version needed a correction near `2^32`.

### HyperLogLog++ (Google, 2013)

Heule, Nunkesser & Hall's production-hardening at Google
([*HyperLogLog in Practice*, EDBT 2013](https://research.google/pubs/pub40671/)), the de
facto standard implementation today:

1. **64-bit hash** — eliminates the high-range correction entirely (collisions at 2^32 scale
   were the issue; √2^64 is beyond practical cardinalities).
2. **Empirical bias correction** — raw HLL is biased in the awkward zone between linear
   counting and the asymptotic regime (~2.5m–5m); HLL++ subtracts a pre-computed,
   interpolated bias table, measurably improving accuracy in exactly the range many real
   counts live.
3. **Sparse representation** — below a few thousand distinct values, store (index, rank)
   pairs varint-encoded instead of the dense register array, giving near-exact counts at a
   fraction of 12 KB, upgrading to dense lazily. Most real-world sketches (per-dimension
   slices) never leave sparse mode, so this dominates aggregate storage cost.

### Worked example

Count distinct IPs per minute on a load balancer fleet, `p = 14`:

1. Each LB hashes each source IP with a 64-bit hash. For hash
   `0x2A7F3C0000000000…`: top 14 bits → register index `j = 10879`; remaining 50 bits have
   leading-zero count 3 → rank `ρ = 4`; set `M[10879] = max(M[10879], 4)`.
2. Every minute each LB ships its 12 KB register array to the aggregator and resets.
3. The aggregator takes element-wise max across LBs → the fleet-wide minute sketch; a
   rolling max over 60 minute-sketches → the hourly distinct count. All exact-union
   semantics, all in kilobytes, and a DDoS spike from 10⁴ to 10⁷ distinct spoofed IPs
   changes memory usage not at all.

## 4. What HLL cannot do

- **No deletions.** Registers only go up; "distinct users except those who churned" is not
  expressible. (Sliding windows are done by keeping per-bucket sketches and merging the
  window, not by deleting.)
- **No membership or frequency** — it never knows *which* elements it saw. Pair with a Bloom
  filter or Count-Min if you need those (see the [index](probabilistic-data-structures.md)).
- **Intersections are second-class.** Union is lossless, so
  `|A∩B| = |A| + |B| − |A∪B|` works — but the absolute error is proportional to the
  *union's* size, which is catastrophic for small intersections of large sets. If audience
  overlap is the actual product requirement, use **Theta sketches**
  (see [MinHash & friends](minhash-simhash-lsh.md)), which support intersection natively.

## 5. Who uses it, for what (grounded)

- **Redis** — `PFADD` / `PFCOUNT` / `PFMERGE` in core since 2.8.9: dense mode is 2¹⁴
  six-bit registers = 12 KB with ~0.81% standard error, with a sparse encoding for small
  cardinalities. The `PF` prefix honors Philippe Flajolet.
  ([Redis docs: HyperLogLog](https://redis.io/docs/latest/develop/data-types/probabilistic/hyperloglogs/))
- **Reddit** — post view counts are HLLs: requirements were near-real-time counts, each user
  counted once per window, accuracy within a few percent, at full production event rate —
  implemented with Redis HLL (later moving the counting into their streaming pipeline).
  ([View Counting at Reddit, 2017](https://www.redditinc.com/blog/view-counting-at-reddit/))
- **Google BigQuery** — `APPROX_COUNT_DISTINCT` and the `HLL_COUNT.INIT / .MERGE /
  .EXTRACT` function family expose HLL++ sketches as first-class SQL values, letting users
  materialize per-partition sketches and merge at query time — the mergeability pattern from
  §2 productized. ([BigQuery docs: HLL functions](https://cloud.google.com/bigquery/docs/reference/standard-sql/hll_functions))
- **Presto / Trino** — `approx_distinct()` (HLL-based, default standard error 2.3%,
  tunable), plus explicit `HyperLogLog` types with `merge()` for sketch columns. Meta has
  described `approx_distinct` as the workhorse for interactive distinct counts at Facebook
  scale. ([Trino docs: HyperLogLog](https://trino.io/docs/current/functions/hyperloglog.html))
- **ClickHouse** — `uniq` (adaptive combined sketch) and `uniqHLL12` aggregate functions,
  usable in materialized views with `AggregatingMergeTree` so sketches are pre-merged at
  ingest. ([ClickHouse docs: uniq functions](https://clickhouse.com/docs/en/sql-reference/aggregate-functions/reference/uniq))
- **Apache Druid / Pinot** — distinct-count columns via
  [Apache DataSketches](https://datasketches.apache.org/) HLL sketches (a library
  open-sourced by Yahoo for exactly this pre-aggregation pattern).

## 6. Operational notes

- **Pick `p` from the error budget:** error ≈ `1.04/√(2^p)` → p=12: 1.6% / 4 KB; p=14:
  0.81% / 12 KB; p=16: 0.41% / 48 KB (per sketch — remember you'll store thousands of them).
- **Merging requires identical (p, hash).** Standardize the sketch format across the
  pipeline up front; you cannot merge a Redis HLL with a DataSketches HLL. Formats are
  library-specific — treat the sketch binary as a long-lived schema commitment if you
  persist it.
- **The error is a distribution, not a cap.** ±0.81% is one standard deviation; ~32% of
  readings are outside it, ~5% outside 2σ. Don't alert on distinct-count deltas tighter
  than a few σ.
- **Counts are not monotonic minute-to-minute** — an estimate can go slightly *down* as
  registers update. UIs displaying HLL-backed counters should round aggressively (Reddit
  displays rounded counts partly for this reason).
