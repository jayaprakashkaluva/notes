# Quantile Sketches — t-digest, DDSketch, KLL

Part of the [Probabilistic Data Structures](probabilistic-data-structures.md) series.

## 1. What they are

Quantile sketches summarize a stream of numeric values in kilobytes such that any quantile
(p50, p99, p99.9…) can be extracted later with bounded error — and, critically, sketches
from many hosts/shards/time-buckets can be **merged** and still yield correct quantiles for
the combined population.

Exact single-pass quantiles require Ω(n) memory (Munro & Paterson, 1980), so every
monitoring system that shows you a p99 across a fleet is running one of these. The three
that matter in practice differ in *which error they bound*:

| Sketch | Error guarantee | Character |
|---|---|---|
| **t-digest** (Dunning & Ertl) | rank error, tightest at the tails | empirically excellent, no formal worst-case bound |
| **DDSketch** (Datadog, VLDB 2019) | **relative value error** (e.g., p99 within ±1% of true *value*) | formal guarantee, trivially mergeable |
| **KLL** (Karnin–Lang–Liberty, 2016) | uniform **rank error** ε with prob 1−δ | provably optimal space for the model |

## 2. The problem they solve

Averages hide exactly the behavior you're paid to notice: a mean latency of 40 ms is
compatible with a p99 of 45 ms or 4 s. But percentiles are rank statistics, and rank
statistics have a brutal operational property: **they don't aggregate**. The average of
per-host p99s is not the fleet p99 — it isn't even a bound on it. The only correct ways to
get a fleet-wide p99 are (a) centralize all raw samples, or (b) have each host maintain a
*mergeable* summary and combine those. Quantile sketches are (b).

Same story across time: you cannot roll hourly p99s into a daily p99, but you can merge 24
hourly sketches. This is why sketches — not precomputed percentile numbers — are what should
be stored in a metrics pipeline. (A precomputed percentile is a dead end; a sketch is a
reusable aggregate.)

## 3. How each one works

### t-digest

Maintains ~100–500 weighted centroids (mean, count) approximating the distribution. The
core idea is a **scale function**: a centroid is allowed to absorb points only while its
weight stays under a cap that *shrinks near the extremes* — centroids near the median may
hold thousands of points, centroids near q=0.999 hold a handful. Error is therefore
proportional to `q(1−q)`: sharpest exactly where SLOs live. Compression parameter δ
(typically 100–200) bounds centroid count; typical size is a few KB. Merging concatenates
centroid lists and re-compresses. Caveats a staff engineer should know: accuracy is
empirical rather than worst-case-proven, merge order can slightly affect results
(non-strict determinism), and pathological orderings degrade accuracy — in exchange it's
compact, fast, and battle-tested for a decade.
([Computing Extremely Accurate Quantiles Using t-Digests](https://arxiv.org/abs/1902.04023))

### DDSketch

Radically simpler: logarithmically spaced buckets. For relative accuracy α, bucket
boundaries are powers of `γ = (1+α)/(1−α)`; a value v lands in bucket `⌈log_γ v⌉`. Any value
reported from a bucket is within α *relative error* of the true value — a guarantee on the
*value* axis ("p99 = 812 ms ± 1%"), which is what humans reading latency dashboards actually
assume they're getting. Rank-error sketches can be arbitrarily wrong in value terms on
heavy-tailed data (the gap between p98.9 and p99 might be 2×); DDSketch cannot. Merging is
index-wise bucket addition — fully deterministic and commutative. Memory is bounded by
collapsing the lowest buckets when a bucket limit is hit (the paper's variant), keeping tail
accuracy intact. ([DDSketch paper, VLDB 2019](https://arxiv.org/abs/1908.10693))

The same log-bucket idea, independently standardized, underlies **OpenTelemetry's
exponential histograms** and **Prometheus native histograms** — the industry converging on
"mergeable log-bucketed histograms" as the wire format for latency distributions.

### KLL

The theoretician's choice: a hierarchy of compactors that randomly downsample pairs
upward, achieving uniform rank error ε with space `O((1/ε)·√log log(1/δ))` — proved optimal
(Karnin, Lang, Liberty, FOCS 2016). Unlike t-digest its guarantee is worst-case (no data
distribution can break it, only bad luck bounded by δ), and unlike DDSketch it's
comparison-based, so it works on any ordered domain, not just floats — at the cost of
randomized answers and rank-not-value error. This is the quantile sketch in
[Apache DataSketches](https://datasketches.apache.org/docs/KLL/KLLSketch.html) (Druid,
Pinot, Hive integrations). Its predecessor **GK** (Greenwald–Khanna, 2001) is deterministic
but non-mergeable in general — the reason it appears in single-node contexts (Spark's
`approxQuantile` uses a GK variant) but not in merge-heavy pipelines.

## 4. Worked example

Fleet latency SLO: "p99 of checkout requests < 500 ms, evaluated over 5-minute windows,
across 800 pods."

- Each pod maintains one DDSketch (α = 1%) per 5-minute window and ships it (a few KB of
  bucket counts) to the metrics backend instead of raw samples (~10⁶ samples/pod/window).
- Backend merges 800 sketches by bucket-wise addition → fleet sketch → `quantile(0.99)`.
  Result is guaranteed within 1% of the true fleet p99 *value*.
- The same stored sketches answer later questions no precomputed number could: daily p99
  (merge 288 windows), p99.9 during an incident, "what fraction of requests exceeded
  500 ms" (rank query of a value — the CDF direction), per-region p99 (merge a subset).

The anti-pattern this replaces: each pod exports `p99` as a gauge and a dashboard averages
them — a number with no defined meaning that goes *down* when one unloaded pod reports fast.

## 5. Who uses what (grounded)

- **Elasticsearch** — the `percentiles` aggregation runs on **t-digest** by default
  (`compression` exposed as a parameter), with HDRHistogram as an opt-in alternative for
  latency-style data; its docs are unusually candid that percentiles are estimates.
  ([Elasticsearch docs: percentiles aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-metrics-percentile-aggregation.html))
- **Datadog** — built and published **DDSketch** for its distribution metrics, so
  percentiles can be re-aggregated across arbitrary tag sets server-side; open-sourced
  implementations in Go/Java/Python. ([DDSketch repo](https://github.com/DataDog/sketches-java))
- **Apache Druid / Pinot / Hive** — quantile columns via DataSketches **KLL** (and the
  older Quantiles sketch), pre-aggregated at ingestion, merged at query time.
- **Apache Spark** — `DataFrame.approxQuantile` implements the **Greenwald–Khanna**
  algorithm (documented in its API docs) for single-pass quantiles.
- **Prometheus / OpenTelemetry** — native/exponential histograms: log-bucketed, mergeable
  distribution aggregates replacing both fixed-bucket histograms and client-side quantile
  summaries (whose non-aggregatability was a long-standing Prometheus foot-gun).
- **ClickHouse** — `quantileTDigest`, `quantileTDigestWeighted` alongside sampling-based
  `quantile` variants, selectable per query.
  ([ClickHouse docs: quantileTDigest](https://clickhouse.com/docs/en/sql-reference/aggregate-functions/reference/quantiletdigest))
- **PostgreSQL** — the `tdigest` extension (Tomáš Vondra) provides t-digest as an aggregate
  with partial-aggregation support, bringing the merge pattern into plain SQL.
  ([github.com/tvondra/tdigest](https://github.com/tvondra/tdigest))

## 6. Choosing, and operational notes

- **Dashboarding latencies with SLOs on values** → DDSketch / exponential histograms.
  The relative-value guarantee matches how SLOs are written, and deterministic merging
  makes backfills reproducible.
- **General-purpose analytics quantiles in a database** → t-digest (compact, tail-accurate)
  or KLL (worst-case guarantees, arbitrary comparable types).
- **Never store bare percentile numbers as the system of record.** Store sketches (or
  log-bucketed histograms); derive percentiles at read time. Every aggregation flexibility
  you'll ever want falls out of that one decision.
- **Fixed-boundary histograms are the hidden failure case:** classic Prometheus histograms
  give quantiles by linear interpolation inside human-chosen buckets — cross a bucket
  boundary and your "p99" can jump 2× with no change in traffic. Log-bucketed sketches
  exist to end that class of incident.
- **Mind the value domain:** DDSketch's relative-error scheme needs special handling around
  zero and negatives (separate stores); t-digest and KLL don't care.
