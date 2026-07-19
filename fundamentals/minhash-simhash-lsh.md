# MinHash, SimHash & LSH — Similarity Sketches

Part of the [Probabilistic Data Structures](probabilistic-data-structures.md) series.

## 1. What they are

Similarity sketches compress a large set (or document, or vector) into a short signature
such that **the similarity of two signatures estimates the similarity of the originals** —
without ever comparing the originals. Combined with locality-sensitive hashing (LSH), they
also solve the harder problem: *finding* similar pairs in a corpus without O(n²) comparison.

- **MinHash** (Broder, 1997) estimates **Jaccard similarity** of sets.
- **SimHash** (Charikar, 2002) estimates **cosine similarity** of weighted feature vectors.
- **Theta sketches** (Apache DataSketches) extend the same sampling idea to full **set
  algebra** (union/intersection/difference) on distinct counts.

## 2. The problem they solve

"Are these two things near-duplicates?" appears everywhere: crawlers must not index the
same article syndicated across 500 URLs; plagiarism and license-scanning tools compare
documents against millions; dedup in storage; "substantially similar" spam campaigns.
Two structural obstacles:

1. **Pairwise cost.** Exact Jaccard between two documents' shingle sets means intersecting
   large sets; doing it for all pairs of n documents is O(n²) — 10⁹ documents → 10¹⁸ pairs.
2. **Storage.** Keeping every document's full shingle set online for comparison is its own
   scaling problem.

MinHash kills obstacle 2 (a set of any size → e.g. 128 × 64-bit signature), and LSH banding
kills obstacle 1 (only candidate pairs that collide in a hash bucket are ever compared).

## 3. How MinHash works

Jaccard similarity: `J(A,B) = |A∩B| / |A∪B|`.

Apply a random permutation (in practice: a random hash) `h` to every element and keep only
the **minimum** hash value of each set. The founding observation:

```
P[ min h(A) == min h(B) ] = J(A, B)
```

because the minimum over `A∪B` is equally likely to be *any* element of the union, and the
minima coincide exactly when that element lies in the intersection. One hash gives a single
biased coin flip whose bias *is* the Jaccard similarity; `k` independent hashes give `k`
flips, and the fraction of matching components estimates J with standard error
`√(J(1−J)/k)` — k=128 gives ±~4.4% worst case, k=256 ±~3%.

Signatures are **mergeable** (component-wise min = signature of the union) and support the
same streaming/pre-aggregation patterns as other sketches. A practical refinement, *one
permutation hashing*, gets a k-component signature from a single pass with one hash
function by bucketing, at equal accuracy for large sets.

### LSH banding: search without pairwise comparison

Split each k-component signature into `b` bands of `r` rows (`k = b·r`). Hash each band to
a bucket table; any two documents sharing *any* band bucket become a **candidate pair**.
Probability two documents with similarity `s` become candidates:

```
P(candidate) = 1 − (1 − sʳ)ᵇ
```

an S-curve with threshold ≈ `(1/b)^{1/r}`. With k=128 as b=32 bands × r=4: threshold
≈ 0.42 — pairs at J=0.8 are caught with probability >99.9%, pairs at J=0.2 rarely collide.
Tuning b and r moves the threshold and trades false positives (verified cheaply against
full signatures) against false negatives (silent misses — choose conservatively).

## 4. SimHash — the other geometry

MinHash sees sets; SimHash sees **weighted vectors** under cosine similarity (random
hyperplane rounding, Charikar 2002). For a 64-bit fingerprint: for each feature, take a
64-bit hash; for each bit position, add the feature's weight if the bit is 1 else subtract;
the fingerprint's bit i is the sign of accumulator i. Then

```
P[bit agrees] = 1 − θ/π        (θ = angle between the vectors)
```

so **Hamming distance between fingerprints ∝ angular distance**. Near-duplicates cluster
within a few bits of each other, enabling an indexable exact-match problem: Google's
crawl-dedup paper (Manku, Jain, Das Sarma, WWW 2007) used 64-bit SimHash fingerprints,
declared near-duplicates at Hamming distance ≤ 3, and described the bit-rotation/table
scheme that finds all such neighbors over an 8-billion-page corpus efficiently.
([Detecting Near-Duplicates for Web Crawling](https://research.google/pubs/detecting-near-duplicates-for-web-crawling/))

Rule of thumb: **MinHash for "same content, edited a little"** (set overlap of shingles —
robust to reordering), **SimHash for "same topic-ish / templated variants"** (weighted
features, compact fingerprints, cheap Hamming search). Modern semantic near-dup (embeddings
+ ANN indexes like HNSW) is the learned-representation descendant of the same
pipeline shape.

## 5. Theta sketches — set algebra on samples

A different member of the min-hashing family, from Yahoo's
[Apache DataSketches](https://datasketches.apache.org/docs/Theta/ThetaSketches.html):
keep the `k` smallest hash values seen (a KMV/bottom-k sample) plus the threshold θ = the
k-th smallest; the sample is a uniform random sample of the distinct elements, so
`|sample|/θ` estimates cardinality, and — the part HLL cannot do — **intersections and
differences are computed directly on the samples** with error proportional to the *result*
size rather than the union size. This is the production answer to audience-overlap queries
("users who saw campaign A ∩ campaign B"), integrated in Druid and Pinot for exactly that
workload. Trade-off vs. HLL: several KB–hundreds of KB per sketch instead of 1.5–12 KB,
in exchange for set expressiveness.

## 6. Worked example — near-duplicate news articles

Dedup a crawl of ~100M articles:

1. **Shingle**: each article → set of word 5-grams (typically thousands of shingles).
2. **Sign**: 128 MinHash components per article → 1 KB signature. 100M articles → ~100 GB
   of signatures vs. tens of TB of shingle sets.
3. **Index**: LSH with b=32, r=4 (threshold ≈ 0.42). Each article inserts 32 band-hashes
   into bucket tables; syndicated copies (J ≈ 0.85–0.95 after boilerplate stripping)
   collide with near-certainty.
4. **Verify**: candidate pairs compared on full 128-component signatures (estimated J), and
   optionally on raw shingles for the pairs that matter; cluster with union-find; keep one
   canonical article per cluster.

Each stage is embarrassingly parallel and streaming-friendly; new articles query the same
band tables to be deduped at ingest time. This is, in outline, the architecture Broder
built MinHash *for* — clustering syndicated near-duplicates at AltaVista
([Broder, *On the Resemblance and Containment of Documents*, 1997](https://ieeexplore.ieee.org/document/666900)) —
and its descendants run in every large crawler and LLM-pretraining data pipeline since
(MinHash-based dedup is standard in open dataset pipelines, e.g. the dedup stages described
for RefinedWeb/FineWeb and similar corpora).

## 7. Who uses it, for what (grounded)

- **AltaVista** — MinHash's origin: syndicated near-duplicate clustering of the web index
  (Broder 1997; the technique predates and outlived the company).
- **Google** — SimHash-based near-duplicate detection in web crawling at 8B-page scale
  (Manku et al., WWW 2007).
- **Apache Spark MLlib** — ships `MinHashLSH` and `BucketedRandomProjectionLSH` as library
  operators for join-style similarity search on DataFrames.
  ([Spark docs: LSH](https://spark.apache.org/docs/latest/ml-features.html#locality-sensitive-hashing))
- **Apache Druid / Pinot via DataSketches** — Theta sketches for distinct counts with set
  operations (campaign/audience overlap analytics); Yahoo built and open-sourced the
  library for these workloads.
- **LLM training-data pipelines** — MinHash-LSH document dedup is a standard preprocessing
  stage for web-scale corpora (documented in dataset papers such as RefinedWeb and the
  tooling around them), directly affecting model quality by removing duplicated text.
- **Code/license scanning & plagiarism detection** — shingling + MinHash is the standard
  design for "find files substantially similar to known code" at repository scale.

## 8. Operational notes

- **Similarity definition drives structure choice**: Jaccard on sets → MinHash; cosine on
  weighted vectors → SimHash; learned semantic similarity → embeddings + ANN (different
  toolbox). Mixing them up produces plausible-looking garbage.
- **Shingling choices dominate quality** (w of the w-grams, word vs. character shingles,
  boilerplate stripping). The sketch faithfully estimates similarity of *whatever you
  shingled* — garbage features in, confident garbage out.
- **LSH false negatives are silent.** The S-curve tells you the miss probability at a given
  similarity — compute it for your threshold; don't guess. Add more bands (or a second,
  looser index) if misses are costly.
- **Fix k, hash functions, and seeds corpus-wide, forever.** Signatures built with
  different parameters are incomparable; treat the signing scheme as a versioned schema.
