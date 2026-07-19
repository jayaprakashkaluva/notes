# 03 — Retrieval-Augmented Generation

> RAG is a systems problem wearing an ML costume: 80% of quality comes from
> ingestion, chunking, and retrieval; the LLM call at the end is the easy part.

## 0. First: do you need RAG at all?

Anthropic's contextual-retrieval post makes this point explicitly: **if your
knowledge base fits in the context window, just include it in the prompt.**
With 200K–1M-token context windows and prompt caching (cache reads ≈ 0.1× input
price), "stuff the corpus, cache the prefix, ask many questions" beats a
retrieval pipeline on quality *and* engineering cost for corpora up to roughly
the low hundreds of thousands of tokens. RAG earns its complexity only when the
corpus is genuinely larger, changes frequently, or must be permission-filtered
per user.

Also distinguish RAG from **agentic search** (§7): letting a model iteratively
search/grep/read is increasingly competitive with embedding pipelines for
codebases and well-structured document sets, with far less infrastructure.

## 1. Reference architecture

```
Ingestion (offline)                        Query (online)
──────────────────                         ─────────────
sources → parse → chunk                    query → [rewrite/expand]
       → contextualize (optional)                → retrieve (vector + BM25)
       → embed + index (vector)                  → rank fusion
       → index (BM25)                            → rerank (optional)
                                                 → assemble context (budget, dedupe, order)
                                                 → generate (with citations)
```

Every box is an independent quality lever with its own failure modes. Debug RAG
by isolating stages: log the retrieved set and its scores *before* blaming the
generator.

## 2. Parsing and chunking

- Parsing is the silent killer: PDFs with multi-column layouts, tables flattened
  into word soup, headers/footers polluting chunks. Inspect parsed output on your
  ugliest real documents before tuning anything downstream.
- **Chunk along semantic boundaries** (headings, sections, functions/classes for
  code), not fixed character windows, when structure exists. Fixed-size with
  overlap is the fallback, not the default.
- Chunk size trades recall precision against context: smaller chunks embed more
  precisely but lose surrounding meaning. A few hundred tokens is the common
  operating range; there is no universal number — Anthropic's guidance is that
  chunk-boundary choice can significantly affect performance and should be
  evaluated per corpus.
- Store rich metadata per chunk (source doc, section path, timestamps, ACLs).
  Metadata filtering at query time (tenant, recency, permissions) is a hard
  functional requirement in most enterprise deployments, and it must be applied
  *in the retriever*, not post-hoc — post-hoc filtering silently starves the
  context of results.

## 3. The context-loss problem and Contextual Retrieval

Classic failure: a chunk reading "The company's revenue grew by 3% over the
previous quarter" embeds and matches poorly because it doesn't say *which
company* or *which quarter* — that information lived elsewhere in the document.

**Contextual Retrieval** (Anthropic, Sept 2024) fixes this at ingestion: for each
chunk, an LLM call generates a short (~50–100 token) chunk-situating context
("This chunk is from ACME Corp's Q2 2023 SEC filing; previous-quarter revenue
was $314M...") which is *prepended to the chunk* before embedding and before
BM25 indexing.

Published results (on their internal benchmark, top-20 retrieval):

- Contextual embeddings alone: **35%** reduction in retrieval failure rate
  (5.7% → 3.7%).
- Contextual embeddings + contextual BM25: **49%** reduction (5.7% → 2.9%).
- Adding reranking on top: **67%** reduction (5.7% → 1.9%).

Cost engineering: the contextualization pass sends the *full document* plus each
chunk — prompt caching makes this viable (the doc is cached across its chunks;
Anthropic's estimate was ~$1.02 per million document tokens at then-current
Sonnet pricing), and the Batch API halves it again. This is a canonical
"caching + batches turn an impractical pipeline into a cheap one" case.

## 4. Hybrid retrieval and rank fusion

Embeddings capture semantics but miss exact identifiers; BM25 (lexical, TF-IDF
family) nails exact matches — error codes, function names, part numbers, legal
citations — but misses paraphrase. Production systems run **both** and fuse:

1. Vector search → top-N by cosine similarity.
2. BM25 → top-N by lexical score.
3. Fuse (reciprocal rank fusion or weighted merge), dedupe, take top-K.

Anthropic's published pipeline used exactly this shape (their experiments fetched
top-150 fused candidates ahead of reranking). Skipping BM25 because "embeddings
are semantic" is the most common self-inflicted RAG wound in technical domains.

## 5. Reranking

A cross-encoder reranker scores (query, chunk) pairs jointly — far more accurate
than bi-encoder similarity — applied to the top-50/150 fused candidates to pick
the final top-K. Adds tens to hundreds of ms and per-query cost; the published
contextual-retrieval numbers above show it delivering the single largest
incremental gain (49% → 67% failure-rate reduction). Skip it only for
latency-critical paths, and know what recall you're giving up.

A related decision: retrieve-more-then-rerank changes where your precision comes
from. With a strong reranker, you can afford high-recall first-stage retrieval
(looser thresholds, larger N) — tune the stages together, not independently.

## 6. Context assembly and generation

- **Budget explicitly.** Fixed token budget for retrieved context; fill by rank;
  drop below-threshold chunks rather than padding — irrelevant chunks actively
  degrade answers (the model attends to them).
- **Attach provenance.** Wrap each chunk with source metadata in the prompt, and
  instruct the model to cite. On the Claude API, `document` content blocks with
  `citations: {enabled: true}` return structured span-level citations
  (`cited_text`, document index, char/page location) — strictly better than
  asking the model to interpolate footnotes as free text, which it can hallucinate.
- **Instruct the failure mode.** Tell the model what to do when the context
  doesn't contain the answer ("say you don't know; do not answer from general
  knowledge") — un-instructed models fall back to parametric knowledge, which is
  precisely the hallucination RAG was built to prevent.
- **Cache-aware prompt layout.** Static instructions and stable corpus prefixes
  before the last cache breakpoint; the per-query retrieved set and question
  after it.

## 7. Agentic / iterative retrieval

Single-shot retrieval fails on multi-hop questions ("compare X's approach in
2023 vs 2025") and on queries phrased nothing like the corpus. Two escalations:

- **Query rewriting/expansion** (workflow-tier): an LLM call rewrites the user
  query into one or more retrieval queries; cheap, often a large win for
  conversational inputs where the question depends on chat history.
- **Search-as-tool** (agent-tier): expose retrieval as a tool and let the model
  iterate — search, read, refine query, search again. This subsumes rewriting,
  handles multi-hop naturally, and for corpora with good native structure
  (codebases via grep/glob, wikis via title search) can replace the embedding
  pipeline entirely. Cost: multiple model round-trips per question; mitigate
  with an effort/model-tier appropriate to the route.

The practical spectrum: FAQ bot → single-shot RAG; research assistant →
agentic search over hybrid retrieval tools.

## 8. Evaluating RAG

Evaluate retrieval and generation **separately**; end-to-end evals can't tell
you which stage broke.

- **Retrieval metrics:** recall@K against a labeled set of (query → gold chunks/
  docs). Anthropic's published benchmark used failure rate = 1 − recall@20.
  Build the labeled set from real user queries plus LLM-generated question/chunk
  pairs, human-spot-checked.
- **Generation metrics:** faithfulness (is every claim supported by the provided
  context?) and answer relevance — typically LLM-as-judge with a rubric, plus
  citation-validity checks (do cited spans exist and support the sentence?).
- **Regression discipline:** every chunking/embedding/reranker change reruns the
  retrieval eval; every prompt change reruns the generation eval. Index rebuilds
  are deployments — version them.

## 9. Operational concerns

- **Index freshness:** decide staleness tolerance per source; incremental upserts
  keyed on content hash beat full rebuilds; delete propagation (doc removed →
  chunks removed) is legally load-bearing under retention/erasure requirements.
- **Permissions:** ACL filtering at query time in the retriever (metadata
  filters), evaluated per-request against the *end user's* identity — retrieval
  running with a service account's broad permissions is a classic data-leak
  pattern (confused deputy).
- **Cost model:** ingestion (embedding + optional contextualization, batchable at
  50% off) is per-corpus-change; query cost is per-request (embed query + rerank
  + generation tokens). For high-QPS products, generation tokens dominate —
  retrieved-context budget is your cost dial.

---

## Sources

- Anthropic, *Introducing Contextual Retrieval* (engineering blog, Sept 2024) —
  contextual embeddings/BM25 technique, published failure-rate reductions
  (35%/49%/67%), cost estimate via prompt caching, "just use long context for
  small corpora" guidance, hybrid + rerank pipeline shape.
- Anthropic platform docs: citations (document blocks), prompt caching, Batch
  API, tool use (search-as-tool pattern).
- BM25/hybrid-search and cross-encoder reranking: standard IR literature
  (Robertson & Zaragoza on BM25; bi- vs cross-encoder distinction from the
  sentence-transformers line of work).

---

## 10. Production retrieval architecture

Separate ingestion from serving. The **control plane** owns sources, connectors,
parser/embedding/index versions, ACL mappings, re-index jobs, and quality reports.
The **data plane** owns query normalization, authorization filters, candidate
retrieval, fusion/reranking, context packing, generation, and citations.

Every chunk needs an immutable source and chunk ID, content hash/version,
tenant/security labels, canonical URI/title, effective timestamps, parser/chunker/
embedding versions, and source offsets for citations. This lineage enables deletion,
re-indexing, incident analysis, and citation repair.

Apply the caller's authorization **during retrieval**, not after generation. Preserve
tenant boundaries in indexes, caches, eval data, and traces. Retrieved text is
untrusted and can contain prompt injection even when it came from an internal source.

### Evaluate the layers separately

| Layer | Measures | Failure isolated |
|---|---|---|
| Ingestion | parse coverage, freshness lag, ACL accuracy | missing/stale knowledge |
| Retrieval | recall@k, MRR, nDCG | evidence not found or ranked |
| Packing | evidence coverage, duplication, token use | evidence lost before generation |
| Generation | groundedness, relevance, completeness | evidence ignored or distorted |
| Citation | entailment, source/span validity | citation does not support claim |
| End to end | success, correct abstention, latency, cost | product outcome |

A correct answer does not prove retrieval worked: the model may know it from
training. Include corpus-specific, recently changed, and intentionally unanswerable
questions. Require abstention when evidence is absent.

Use blue/green indexes when changing embeddings, chunking, or schema. Build, run
offline and end-to-end evals, canary traffic, then promote. Trace the exact index
version. Deletion must be verifiable across raw data, chunks, vectors, caches, and
replicas.

### Cross-provider and research sources

- [Lewis et al., *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*](https://arxiv.org/abs/2005.11401).
- [Microsoft, *RAG solution design and evaluation guide*](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide).
- [Azure AI Search, *RAG overview*](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview).
- [NIST, *Generative AI Profile (AI 600-1)*](https://doi.org/10.6028/NIST.AI.600-1).
