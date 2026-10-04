# KV Cache: Mechanics, Cost Models, and Production Design

**Audience:** staff and principal engineers designing or evaluating LLM inference systems.  
**Sources reviewed:** October 4, 2026.  
**Scope:** inference in causal autoregressive Transformers, primarily decoder-only models.

This document distinguishes **source-backed mechanisms**, **derivations under stated assumptions**, and **engineering recommendations**. Numerical examples are hypothetical calculations, not measured performance. Hugging Face examples are grounded in the explicitly versioned Transformers **v4.57.1** documentation and source. Links to vLLM `latest` and repository `main` describe the material reviewed on the date above; implementation details can change.

## 1. What KV cache is

A **key-value cache** retains the key and value tensors produced for previously processed tokens at each attention layer. When the model processes the next token, it reuses those tensors instead of evaluating the entire prefix again. It still computes attention between the new query and the applicable historical keys and values. [Transformers caching][hf-caching]

The cache is inference state associated with a particular sequence and model execution. Its contents are learned representations produced by the model, rather than token IDs, model parameters, or previously generated answers. A token has different key/value representations in different layers. [Hugging Face's implementation walkthrough][hf-blog]

Three separate ideas are often called “caching”:

| Mechanism | Reused object | Work avoided |
|---|---|---|
| KV caching during generation | Past attention states within a sequence | Reevaluating the already processed prefix on every step |
| Prefix caching across requests | Compatible KV states for a shared prefix | Prefilling that shared prefix again |
| Response caching | Previously completed output | Executing a request at all, subject to the application's response-cache policy |

The first two operate on model state. Prefix caching is documented directly by vLLM; the response-cache row is an application-level distinction, not a claim about a particular serving engine. [vLLM automatic prefix caching][vllm-apc]

The architectural consequence is that generation trades redundant computation for persistent, growing state. Understanding the cache therefore requires both an attention model and a memory-management model.

## 2. The attention computation that makes caching possible

### 2.1 Queries, keys, and values

For one attention head in layer `l`, let `x[l,t]` be the representation entering that layer at token position `t`. Ignoring biases and positional transformations for the moment:

```text
q[l,t] = x[l,t] W_Q[l]
k[l,t] = x[l,t] W_K[l]
v[l,t] = x[l,t] W_V[l]

scores[t,j] = dot(q[l,t], k[l,j]) / sqrt(d_k) + mask[t,j]
weights[t,:] = softmax(scores[t,:])
output[l,t] = sum_j weights[t,j] v[l,j]
```

`mask[t,j]` is zero for an allowed position and negative infinity for a disallowed position in this conceptual additive-mask formulation. Causal self-attention excludes future positions. Keys determine relevance to a query; values supply the vectors combined using the attention weights. This follows scaled dot-product attention and decoder masking in the original Transformer paper. [Attention Is All You Need, sections 3.1–3.2][transformer]

For multiple heads, the model computes head-specific attention outputs and combines them through an output projection. GQA and MQA change how query heads share keys and values; section 5 discusses their cache implications.

### 2.2 Why old states remain valid: a dependency argument

**Derivation from causal masking.** Assume fixed weights, fixed earlier inputs and positions, deterministic inference semantics, and layer operations that do not introduce dependence on future tokens.

At the embedding layer, the representation at position `j` depends on that token and its position. At a causal attention layer, position `j` can depend only on positions at or before `j`. Token-local normalization and feed-forward operations preserve that property. By induction through the layers, appending token `t > j` cannot change the earlier position's hidden state, key, or value.

Consequently, the previously computed `k[l,j]` and `v[l,j]` can be reused. This is the dependency property described in the Transformers caching explanation, applied through the entire stack. [Transformers caching][hf-caching]

The guarantee has boundaries. Changing an earlier token invalidates dependent later states. Changing weights or an adapter invalidates affected states. A model with bidirectional attention over an expanding input does not satisfy this argument. Position-dependent changes must also be accounted for.

Past queries are unnecessary for ordinary incremental next-token inference: they were used to produce earlier outputs; the next attention calculation needs the new query and historical keys/values. [Hugging Face's implementation walkthrough][hf-blog]

### 2.3 Where the cache sits

```text
new token representation
        |
        v
  Q/K/V projection
        |
        +---- new Q ------------------------+
        |                                  |
        +---- new K/V --> layer KV cache ---+--> attention
                                                 |
                                                 v
                                  output projection, residual,
                                  feed-forward block, next layer
```

Every layer has its own cache. Caching does not eliminate the new token's projections, attention, feed-forward computation, or final vocabulary projection. The Llama implementation provides a concrete example: it projects Q/K/V, applies rotary position information, updates the layer cache, and performs attention. [Transformers v4.57.1 Llama source, `LlamaAttention.forward`][hf-llama]

## 3. Prefill and decode: the lifecycle and an off-by-one trap

**Prefill** evaluates the input prompt and populates its cache. **Decode** repeatedly evaluates newly selected tokens against the growing history. The MQA paper presents incremental attention with retained keys and values; Orca describes execution and scheduling at the granularity of generation iterations. [Fast Transformer Decoding, section 2][mqa], [Orca][orca]

For prompt tokens `[A, B, C]`, a conventional generation loop behaves as follows:

| Forward pass | Inputs processed | Cache after the pass | Output selected from logits |
|---|---|---|---|
| Prefill | `A, B, C` | `A, B, C` | `D` |
| First incremental pass | `D` | `A, B, C, D` | `E` |
| Second incremental pass | `E` | `A, B, C, D, E` | `F` |

**Derived consequence:** the first generated token comes from prefill logits. Selecting `D` does not itself insert `D` into the KV cache; the model must process it. If generation stops immediately after selecting `F`, the cache in this example ends at `E`.

This matters when storing conversation state or resuming generation. Track the number of **processed** tokens separately from the number of **selected** tokens. The documented Transformers loop likewise forwards the newly selected token on the next iteration. [Transformers caching, generation-loop example][hf-caching]

Conceptual pseudocode, with no promise of compatibility with a particular library API:

```text
cache = empty_state_for_this_sequence()
logits, cache = forward(prompt_tokens, cache, prompt_positions, causal_mask)
next_token = select(logits_at_last_prompt_position)

while next_token does not end generation:
    emit(next_token)
    logits, cache = forward([next_token], cache, next_position, causal_mask)
    next_position += 1
    next_token = select(logits_at_new_position)

release_or_retain(cache, according_to_session_policy)
```

Selection policy and cache validity are separate concerns. A different sampling temperature need not invalidate an already processed prefix, but a different selected continuation needs its own subsequent states. This follows from the dependency argument in section 2.

## 4. Exactly how much computation caching saves

### 4.1 A cost model with explicit assumptions

**Derivation.** Consider dense full causal self-attention with fixed layer/head dimensions. Let `n` be the sequence length processed so far, `D` the hidden width, `H_q` the query-head count, and `d` the head dimension. Ignore hardware constants, positional operations, and vocabulary projection for this comparison.

| Operation per generation step | Naively forward the whole length-`n` prefix | Forward one new token with KV cache |
|---|---|---|
| Dense projections and feed-forward work | Roughly `O(n D^2)` for a conventional dense block | Roughly `O(D^2)` |
| Attention work | `O(H_q n^2 d)` | `O(H_q n d)` |
| Persistent KV payload | No retained inter-step KV required | `O(n H_kv d)` per layer |

The uncached column describes a **naive full-prefix forward pass each step**, not every possible implementation without a persistent cache. GQA changes projection widths, and MoE changes feed-forward costs; neither is represented precisely by the simplified `D^2` term.

For prompt length `P` and `G` incremental one-token forward passes after prefill, cached attention evaluates:

```text
prefill:              P(P + 1) / 2 allowed query-key pairs
incremental passes:   sum_(i=1..G) (P + i)
                    = GP + G(G + 1) / 2
```

Thus cached attention work is proportional to `P^2 + PG + G^2` across the run. Repeatedly forwarding each entire prefix instead gives a sum of squared prefix lengths, proportional to `GP^2 + PG^2 + G^3` for the incremental passes. `G` counts forward passes here; the generated-token count can differ by one, as section 3 explains.

**Interpretation:** one cached dense-attention decode step is linear in retained context, and the sequence of such steps is still quadratic in the growing continuation. KV caching does not make full-context attention constant-time. The model above follows directly from the attention equation and incremental formulation. [Attention Is All You Need][transformer], [Fast Transformer Decoding][mqa]

### 4.2 Capacity and bandwidth are different constraints

The cache must fit somewhere, and its contents must be accessed quickly enough. The MQA paper identifies repeated loading of historical K/V tensors as a bandwidth cost of incremental decoding. [Fast Transformer Decoding][mqa]

**Engineering cost approximation:** if a decode iteration reads each retained KV element once from high-bandwidth memory, then:

```text
KV read bytes per iteration ≈ resident KV payload read by that iteration
KV service time            ≈ KV read bytes / effective memory bandwidth
```

This is a conditional traffic estimate, not a measured latency or a universal lower bound based on resident size. On-chip reuse, repeated loads, layout, quantization, and head sharing can change actual traffic. Different sequences sharing physical prefix blocks need not share the corresponding reads in a particular kernel.

A useful diagnostic model is:

```text
iteration time ≳ max(compute work / effective compute rate,
                     actual memory traffic / effective bandwidth)
```

The expression omits launch overhead, synchronization, communication, and scheduling. Small-batch dense decoding can be bandwidth constrained; large batches or different architectures can change the dominant cost. Use profiling rather than assigning a bottleneck solely from the presence of a KV cache.

## 5. Memory sizing: use KV heads, not parameter count

### 5.1 The logical payload equation

**Derivation from the stored tensors.** For sequence `i`, layer `l`, and retained token count `T[i,l]`:

```text
M_KV = sum_i sum_l T[i,l] × H_kv[l] ×
                     (d_k[l] × bytes_K[l] + d_v[l] × bytes_V[l])
```

For uniform full-attention layers with equal key/value dimensions and element sizes:

```text
M_KV = 2 × L × H_kv × d × s × sum_i T_i

bytes_per_token = 2 × L × H_kv × d × s
```

The factor of two accounts for keys and values. This counts logical payload before padding, allocation rounding, metadata, sharing, replication, and quantization overhead. A concrete tensor layout is `[batch, KV_heads, sequence, head_dimension]`, although kernels may store an equivalent blocked or reordered layout. [Transformers cache source][hf-cache-source]

### 5.2 MHA, GQA, and MQA

| Attention architecture | KV-head relationship | Relative payload at equal `L`, `d`, `s`, and token count |
|---|---|---|
| Multi-head attention, MHA | `H_kv = H_q` | Baseline |
| Grouped-query attention, GQA | Query heads share KV heads in groups | `H_kv / H_q` of MHA |
| Multi-query attention, MQA | `H_kv = 1` | `1 / H_q` of MHA |

These ratios are derived from the payload equation. MQA shares K/V across query heads; GQA generalizes that design to an intermediate number of KV heads. These are trained architectural choices, not a lossless runtime transformation available for any checkpoint. [MQA paper][mqa], [GQA paper][gqa]

A kernel should preserve sharing rather than persistently expanding the cache to query-head count. Temporary expansion or redundant reads can still affect peak memory and bandwidth; inspect the backend. This is an engineering implication, not a claim that all implementations avoid such costs.

### 5.3 Worked example: hypothetical full-attention model

Assume `L = 32`, `H_q = 32`, `H_kv = 8`, `d = 128`, and FP16/BF16 storage at `s = 2` bytes per element. These are illustrative dimensions, not a claim about a named checkpoint.

```text
bytes_per_token = 2 × 32 × 8 × 128 × 2
                = 131,072 bytes = 128 KiB
```

| Retained length per sequence | One sequence | 16 independent sequences |
|---|---:|---:|
| 4,096 tokens | 512 MiB | 8 GiB |
| 8,192 tokens | 1 GiB | 16 GiB |
| 32,768 tokens | 4 GiB | 64 GiB |

At 8,192 tokens, changing only `H_kv` to 32 gives 4 GiB per sequence; changing it to 1 gives 128 MiB. These comparisons isolate cache geometry; they make no claim that the resulting models have equivalent quality.

Here `KiB = 2^10`, `MiB = 2^20`, and `GiB = 2^30` bytes. All table values exclude model weights and runtime overhead.

### 5.4 Logical payload is not peak allocated memory

**Capacity-planning recommendation:** budget each device separately:

```text
usable device memory >= weights + physical KV allocation
                      + peak activations/workspaces + graph/runtime allocations
                      + operational headroom
```

For an unshared paged cache with block size `b`, homogeneous token cost `c`, and lengths `T_i`, allocation rounds to:

```text
M_pages = c × b × sum_i ceil(T_i / b)
```

Example: `b = 16`, `T = 17`, and `c = 128 KiB` allocate 32 token slots: 4 MiB allocated for 2.125 MiB of logical payload. For shared caches, count **unique physical blocks**, not the sum of logical sequence lengths. A serving engine can preallocate a large pool; used-block count can fall while process-level GPU allocation stays constant. These are accounting consequences of block allocation, not measured engine statistics. [PagedAttention, sections 4.1–4.4][paged]

## 6. Architecture boundaries: when the basic equation changes

**Sliding-window attention.** A layer restricted to a finite attention window can recycle states no longer reachable by future queries. Its retained length can be bounded by the window, subject to implementation workspace requirements. Mistral 7B describes rolling-buffer caching for its sliding-window architecture. Information can still propagate through successive layers beyond one layer's direct window. [Mistral 7B, section 2][mistral]

For a hybrid model, apply the layer-wise equation: global-attention layers can continue growing while windowed layers saturate. Do not apply a single window bound to the whole model unless all relevant layers satisfy it. This follows from the layer-wise accounting in section 5.

**Evicting tokens from full attention.** Arbitrarily retaining only recent states changes the attention inputs. StreamingLLM studies this problem and retains initial “attention sink” tokens as well as recent tokens. Its streaming results do not imply exact equivalence to full-history attention or access to every discarded token. [StreamingLLM][streaming]

**Multi-head latent attention, MLA.** DeepSeek-V2 jointly compresses K/V into a latent representation and also retains a decoupled rotary key component. Let `d_c` be the latent dimension and `d_R` the decoupled rotary-key dimension. For its compressed inference representation, stored elements per token per layer equal `d_c + d_R`, rather than `2 H_kv d`. A backend that materializes additional tensors may require more memory. Use the actual implementation's stored representation. [DeepSeek-V2, sections 2.1.2–2.1.4][deepseek]

**Encoder–decoder models.** Decoder self-attention state grows with the generated sequence; cross-attention can reuse projected keys/values from fixed encoder outputs. Size these separately. This follows from the distinction between decoder self-attention and encoder–decoder attention in the original Transformer architecture. [Attention Is All You Need, section 3.2.3][transformer]

**Recurrent and hybrid models.** The conventional two-tensor, length-growing equation is specific to attention KV state. Mamba uses selective state-space recurrence and an architecture without attention. Its recurrent state therefore needs a different sizing model. For hybrids, account separately for attention and recurrent layers rather than extrapolating the full-attention formula to every layer. [Mamba][mamba]

## 7. Storage strategies and their tradeoffs

### 7.1 Dynamic and static allocation

A dynamic cache grows with the processed sequence. A static cache reserves a maximum capacity and writes new states into predetermined slots. Transformers v4.57.1 documents static caches as compatible with compilation, with a capacity/computation tradeoff when many positions remain unused and masked. [Transformers KV cache strategies][hf-strategies]

**Engineering implication:** bounded, similar-length workloads can make static shapes attractive. Highly variable lengths make maximum-size reservation expensive. Dynamic growth does not necessarily imply a full copy every step: append behavior depends on whether the backend uses tensor concatenation, reserved capacity, or blocks. The v4.57.1 `DynamicLayer` provides one specific concatenation implementation. [Transformers cache source, `DynamicLayer.update`][hf-cache-source]

### 7.2 Paging and PagedAttention

PagedAttention divides each sequence's cache into fixed-size logical blocks mapped to physical blocks that need not be contiguous. Its attention kernel follows that mapping. The paper also describes reference counts and copy-on-write for shared prefixes and branching continuations. Paging reduces allocation waste and redundant duplication; it does not remove historical states required by attention. [PagedAttention][paged]

**Derived block-mapping example:** with four token slots per block:

```text
sequence A logical blocks: [0, 1, 2] -> physical blocks [7, 2, 9]
sequence B logical blocks: [0, 1, 2] -> physical blocks [7, 2, 5]

physical blocks 7 and 2: shared, immutable prefix
physical blocks 9 and 5: separate continuations
```

Before mutating shared storage, a branch must acquire private writable storage. Smaller blocks reduce tail waste and permit finer reuse; larger blocks reduce mapping overhead and may suit kernels better. Treat block size as a joint allocator/kernel/workload decision. [TensorRT-LLM cache reuse, block-size discussion][trt-reuse]

### 7.3 FlashAttention solves a different problem

FlashAttention uses tiling to reduce transfers between high-bandwidth memory and on-chip memory during exact attention. In particular, it avoids storing the full attention matrix in high-bandwidth memory. It does not eliminate the persistent K/V history needed for future tokens. KV caching, efficient attention kernels, and paged allocation can coexist. [FlashAttention paper][flash]

### 7.4 Quantization and offload

KV quantization lowers storage precision. KIVI finds different useful quantization granularities for keys and values: per-channel keys and per-token values. Its quality and performance results concern its evaluated models and workloads. They do not establish that any low-bit cache preserves arbitrary task quality. [KIVI][kivi]

**Derived accounting:** changing 16-bit payload to packed 4-bit payload yields an ideal 4× payload reduction. Real allocations include scales, possible zero points, grouping/padding, and any high-precision residual region. Kernel conversion work can offset bandwidth savings. Weight precision and cache precision are separate settings.

Offloading keeps some KV state in host memory and transfers it for GPU computation. Transformers describes a layer-wise mechanism that prefetches the next layer and moves other state back to the CPU. This trades device capacity for transfer work. [Transformers KV cache strategies][hf-strategies]

**Engineering distinction:** active-state offload repeatedly supplies states needed by decoding; moving idle reusable prefixes to another tier serves a different access pattern. TensorRT-LLM documents host offload of reusable blocks. Estimate transfer frequency and volume for the mechanism actually deployed. [TensorRT-LLM cache reuse][trt-reuse]

## 8. Prefix reuse: identity is a correctness contract

### 8.1 A matching passage is not enough

**Derived example:** suppose prompts are:

```text
request A: [system X] [document D] [question A]
request B: [system Y] [document D] [question B]
```

The document's later-layer states can depend on the preceding system tokens. Therefore, identical document text does not make its causal KV state interchangeable across these prompts. Exact reuse requires a matching prefix under compatible execution semantics.

vLLM's block identity incorporates a parent hash, the block's tokens, and extra identifying inputs such as LoRA IDs, multimodal hashes, and cache salts. This preserves dependence on preceding blocks; the design reviewed here caches full blocks. [vLLM prefix-cache design][vllm-prefix]

### 8.2 What a production cache namespace should express

**Engineering recommendation, derived from the dependency argument:** make compatibility explicit for:

| Identity dimension | Why it matters |
|---|---|
| Model revision and active adapter | They determine projections and hidden states |
| Exact token IDs and preceding prefix | Text similarity does not establish equality of model inputs |
| Positions, attention rules, positional configuration | They determine which states interact and how scores are computed |
| Multimodal features or supplied embeddings | Equal placeholder token IDs can represent different inputs |
| Stored dtype, quantization scales, layout, backend contract | A consumer must interpret bytes consistently |
| Tenant/trust namespace | Cross-request reuse must respect the intended isolation boundary |

Some dimensions may be fixed by the lifetime of an engine instance rather than serialized into every key. Compatibility, not the number of hash fields, is the requirement. The extra-input examples and isolation mechanism are directly supported by the vLLM design. [vLLM prefix-cache design][vllm-prefix]

Compare the actual tokenized prompt. Independently tokenizing fragments and concatenating their token IDs need not match tokenizing the combined text; reuse begins at the matching token boundary. Similarly, chat templates and special tokens are part of the effective prefix. This is an engineering requirement that follows from input identity, not a universal tokenizer-specific behavior claim.

### 8.3 What a hit saves—and what it still costs

A prefix hit avoids computing the cached prefix again. Uncached suffix tokens still attend to that prefix, and decoding still attends to retained context. vLLM explicitly distinguishes prefill savings from generation work. [vLLM automatic prefix caching, Limits][vllm-apc]

**Derived operational consequences:** routing a request to a replica holding its prefix can reduce prefill work but increase queue time if that replica is busy. Retaining many idle prefixes can compete with active continuations for memory. Evaluate end-to-end latency and physical occupancy together.

KV alone is also not a stored next-token distribution. If an identical full prompt is reused, the runtime still needs valid final logits or must perform enough forward computation to produce them, for example by recomputing a boundary token against preceding cached states. This follows from the separate attention-state and output-projection paths; it is not a statement that every engine uses the same boundary policy.

## 9. Scheduling: budget future state, not only today's batch

Orca's iteration-level scheduling allows new requests to enter and completed requests to leave between generation iterations. This makes active sequence lengths and cache demand variable over time. [Orca][orca]

**Admission-control recommendation:** use both a compute/token budget for the next iteration and a physical-block budget for its writes. An engine can have capacity for today's batch but insufficient blocks for tomorrow's continuation.

For homogeneous full-attention state, a conservative unshared reservation is:

```text
reserved_slots_for_request_i = b × ceil((P_i + G_max_i) / b)
```

This may overreserve. Incremental allocation uses memory more efficiently but needs explicit behavior when remaining output growth exceeds capacity: delay admission, preempt/recompute, transfer state, or fail under a defined policy. vLLM documents request preemption and recomputation under cache pressure. [vLLM optimization and tuning][vllm-tuning]

**Derived residency model:** if an active sequence starts with `P` processed tokens, grows approximately at constant rate `r`, and remains active for `tau` seconds:

```text
token-seconds ≈ integral_(0..tau) (P + r t) dt
             = P tau + r tau^2 / 2
```

Under stationary arrivals at rate `lambda`, no sharing, and the same simplified growth model, mean active payload is approximately `c × lambda × E[token-seconds]`. This illustrates why long generations and slow service increase both state residency and memory pressure. It is a planning model, not an engine benchmark; windowed layers, sharing, and pauses require different accounting.

Chunked prefill limits the prompt work admitted into an iteration and can interleave it with decoding. vLLM documents its latency/throughput tradeoffs. Chunking does not by itself remove full-attention states accumulated for the prompt. [vLLM optimization and tuning][vllm-tuning]

Cache occupancy is therefore a control signal, not a goal to maximize indiscriminately. Reserve growth headroom and judge throughput against latency objectives.

## 10. Distributed serving: locate and transport the state

### 10.1 Parallelism changes the per-device formula

Tensor parallelism may partition KV heads, but it does not guarantee payload divided by device count. In the reviewed vLLM Llama implementation, KV heads are partitioned when their number permits it and replicated when tensor-parallel size exceeds KV-head count. [vLLM Llama source, `LlamaAttention.__init__`][vllm-llama]

**Derived example:** with two KV heads and eight tensor-parallel ranks, that implementation assigns one local KV head to each rank. The aggregate physical head count is eight, even though the logical model has two. Per-rank payload is half the unreplicated full cache, not one eighth. Weight partitioning can nevertheless leave more device memory available for state.

For pipeline parallelism, count the attention layers owned by each stage. For context/sequence partitioning, count the locally stored token positions and the communication required by the attention algorithm. Ring Attention provides an example that distributes long sequences and communicates KV blocks while performing blockwise attention. Its overlap claims depend on its execution conditions. [Ring Attention][ring]

### 10.2 Separating prefill from decode creates a transfer boundary

DistServe assigns prefill and decoding to different GPUs to control interference and provision the two phases independently. It explicitly accounts for inter-phase communication and cluster bandwidth. [DistServe][distserve]

**Derived transfer estimate:** if a handoff transfers `M` bytes over a path with effective bandwidth `B`:

```text
bulk transfer component ≈ M / B
```

The illustrative 8,192-token cache in section 5 has 1 GiB of payload. At an assumed **effective** 25 GiB/s, its bulk-transfer component is 40 ms. This excludes queueing, startup, layout conversion, synchronization, topology effects, and overlap; it is not measured network performance.

**Engineering requirements:** define a compatible tensor/layout contract, transfer ownership safely, retain source blocks until transfer completion, and measure whether decode-pool gains exceed handoff costs. A retry must not attach a partial or stale state to a different request.

## 11. Correctness requirements beyond allocation

### 11.1 Positions and rectangular masks

For `p` cached tokens and `m` newly processed tokens, the conceptual attention matrix has `m` query rows and `p + m` key columns. New row `r`, zero-indexed, can attend through key position `p + r`, inclusive.

**Derived example:** with `p = 3` and `m = 2`, the allowed mask is:

```text
               historical keys   new keys
               0   1   2         3   4
new query 3:   1   1   1         1   0
new query 4:   1   1   1         1   1
```

A triangular mask anchored at the wrong corner is incorrect. FlashAttention's repository explicitly describes bottom-right causal alignment for unequal query/key lengths and records a change to this behavior in version 2.1. Verify the exact backend/version contract. [FlashAttention repository][flash-code]

Maintain logical positions separately from physical storage slots. Recycling slot zero in a rolling buffer does not make the incoming token position zero. RoPE encodes positional information through rotations, so position conventions are part of cache compatibility. [RoFormer][rope]

### 11.2 Branching, speculation, and cancellation

Speculative sampling verifies draft continuations with a target model and accepts only a valid continuation according to its acceptance procedure. The paper establishes preservation of the target distribution within hardware numerics for its algorithm. [Speculative sampling][speculative]

**Derived state-management requirement:** rejected candidate states must not remain visible to later attention. Track a committed length separately from tentative writes; rollback, truncate, or mask tentative entries. Separate draft and target models generally require separate state because their weights differ. Budget tentative capacity as well as committed capacity.

For branching continuations, protect shared state before mutation. The PagedAttention paper provides the reference-count/copy-on-write mechanism; applying it safely requires coordinating readers, writers, and allocation reuse. [PagedAttention][paged]

**Engineering recommendation:** request cancellation, timeout, and worker failure need explicit ownership transitions. Remove obsolete mappings and release references only after relevant asynchronous operations are complete. A freed physical block must not remain reachable through a live request's block table.

### 11.3 Mathematical equivalence is not bitwise identity

Exact reuse under unchanged computation preserves the mathematical dependency graph. Different kernels, reduction orders, dtypes, or batch shapes can change floating-point results. Lossy KV quantization or removing attention-visible tokens introduces an additional change to the represented computation. Validate numerical behavior and task behavior separately; do not equate “exact attention” with guaranteed identical output bytes.

### 11.4 Isolation

The cache is sensitive model state derived from request inputs. vLLM documents cache salts to restrict prefix reuse to a trust group and discusses timing-based inference of cached content. It also discusses hash-collision risks. [vLLM prefix-cache design][vllm-prefix]

**Engineering recommendation:** choose tenant namespaces deliberately, protect persistent/offloaded state according to its data sensitivity, and avoid exporting raw KV contents in telemetry. A performance cache key is not an authorization check.

## 12. How to validate and operate a cache implementation

This section is an **engineering validation plan**, not a claim that the cited engines implement every check.

### 12.1 Establish correctness before benchmarking

1. **Cached versus uncached logits:** compare on identical token sequences, positions, masks, weights, and precision. Evaluate every incremental step with documented numerical tolerances.
2. **Chunk boundaries:** compare one-shot prefill with several chunk sizes, including chunks crossing cache-block boundaries.
3. **Ragged batches:** compare each sequence alone and in batches with padding and different retained lengths.
4. **Branch independence:** fork a prefix, append different continuations, and confirm one branch cannot change another's results.
5. **Rollback:** append tentative candidates, reject part of them, and compare the resumed sequence with a fresh evaluation of its committed prefix.
6. **Allocation lifecycle:** repeat completion, cancellation, and failure; check block references, occupancy, and state visibility.
7. **Reuse identity:** force misses for changed model/adapters, positions, preceding tokens, or multimodal inputs, and check intended isolation boundaries.
8. **Lossy strategies:** evaluate long-context retrieval, relevant application tasks, and adversarial lengths in addition to generic perplexity.

Compare fixed-input logits before free-running output strings: a small numerical difference can change one sampled token and then cause an entirely different continuation.

### 12.2 Measure the layer that can explain the symptom

vLLM documents running/waiting requests, KV occupancy, prefix queries/hits, TTFT, inter-token latency, and per-request time per output token. These are useful observability categories even when another runtime exposes different names. [vLLM metrics design][vllm-metrics]

| Signal to collect | Question it helps answer |
|---|---|
| Unique occupied blocks, free blocks, allocation failures | Is physical cache capacity limiting progress? |
| Allocated bytes versus logical retained bytes | Is rounding, padding, replication, or pool reservation responsible? |
| Prefix tokens queried and reused | How much prefill work is actually reusable? |
| Queue time, TTFT, inter-token latency, request TPOT | Which service objective is regressing? |
| Preemptions, recomputed tokens, offload bytes | Is memory pressure causing additional work? |
| Context lengths, output lengths, request lifetimes | Which workload classes consume capacity and residency? |
| Kernel time, measured memory traffic, transfer time | Is the execution bottleneck compute, memory, or transport? |

Occupancy definitions differ: distinguish active referenced blocks, retained idle prefixes, and preallocated pool size. Likewise, distinguish request hit rate from the fraction of queried prefix tokens reused. Report the denominator.

### 12.3 Benchmark the intended workload

Record model revision, attention architecture, cache precision, backend versions, parallelism, hardware, block size, and scheduler settings. Vary prompt length, output length, concurrency, prefix-sharing distribution, and arrival bursts. Separate cold-cache and warm-cache runs.

Report throughput **within** the application's latency objectives, including tail TTFT and token latency. DistServe frames this as goodput under both prefill and decode constraints. [DistServe][distserve]

Include end-to-end serving measurements as well as isolated kernels. A faster attention kernel can coexist with worse queueing, lower admission capacity, or expensive cache transfers. No universal KV-cache speedup follows from the mechanics alone.

## 13. A practical decision sequence

The following recommendations are derived from the preceding mechanisms and cost models:

1. **Inspect actual cache geometry.** Identify layer types, KV heads, dimensions, retained windows, stored precision, and device replication.
2. **Compute logical demand and physical allocation.** Include output growth, block rounding, sharing, and temporary candidate state.
3. **Identify the limiting resource.** Profile capacity, actual traffic, kernel time, queueing, and transport separately.
4. **Match the intervention to the bottleneck.** Paging addresses allocation/sharing; prefix caching addresses repeated prefill; quantization addresses representation size; offload addresses placement; kernel optimization addresses execution.
5. **Specify the correctness contract.** Define identity, positions, mask semantics, write ownership, rollback, and isolation.
6. **Validate under overload and failure.** Check growth pressure, cancellation, recomputation, and transfer retries alongside normal throughput.

The principal design question is whether the system can retain, locate, and consume the required state efficiently while meeting its correctness and service objectives.

## 14. Primary-source index and claim map

The links below identify the papers, official documentation, and project source used above. Section references are preferable to unstable page line numbers. Mathematical calculations and explicitly labeled engineering recommendations are this document's derivations; none are presented as measured results.

| Source | Claims grounded by the source |
|---|---|
| [Vaswani et al., *Attention Is All You Need* (2017)][transformer] | Scaled dot-product attention, causal masking, encoder–decoder attention; sections 3.1–3.2 |
| [Hugging Face, *Caching*, Transformers v4.57.1][hf-caching] | Per-layer retained state, causal reuse, processed-token loop, positions and masks |
| [Hugging Face, *KV Cache from scratch in nanoVLM*][hf-blog] | Q/K/V interpretation and incremental inference walkthrough |
| [Transformers v4.57.1, `modeling_llama.py`][hf-llama] | Concrete projection, rotary, cache-update, and attention path |
| [Transformers v4.57.1, `cache_utils.py`][hf-cache-source] | Tensor state, cache classes, update/allocation behavior |
| [Shazeer, *Fast Transformer Decoding* (2019)][mqa] | Incremental attention, bandwidth analysis, MQA |
| [Ainslie et al., *GQA* (2023)][gqa] | Grouped KV heads and architectural/uptraining choices |
| [Kwon et al., *Efficient Memory Management for Large Language Model Serving with PagedAttention* (2023)][paged] | Block mapping, sharing, reference counts, copy-on-write; sections 4.1–4.4 |
| [Jiang et al., *Mistral 7B* (2023)][mistral] | Sliding-window attention and rolling-buffer cache; section 2 |
| [Xiao et al., *Efficient Streaming Language Models with Attention Sinks* (2023)][streaming] | Window eviction limitations and attention sinks |
| [DeepSeek-AI, *DeepSeek-V2* (2024)][deepseek] | Compressed latent KV and decoupled rotary state; sections 2.1.2–2.1.4 |
| [Gu and Dao, *Mamba* (2023)][mamba] | Selective state-space recurrence and an architecture without attention |
| [Hugging Face, *KV cache strategies*, Transformers v4.57.1][hf-strategies] | Static/dynamic choices and layer-wise offload |
| [Dao et al., *FlashAttention* (2022)][flash] | IO-aware exact attention and tiling |
| [Dao-AILab, FlashAttention repository][flash-code] | KV-cache kernel contract and rectangular causal-mask alignment |
| [Liu et al., *KIVI* (2024)][kivi] | Asymmetric key/value quantization and evaluated tradeoffs |
| [vLLM, *Automatic Prefix Caching* feature documentation][vllm-apc] | Shared-prefix prefill reuse and decode limitations |
| [vLLM, *Automatic Prefix Caching* design][vllm-prefix] | Parent/block identity, extra inputs, salts and isolation |
| [NVIDIA, TensorRT-LLM, *KV cache reuse*][trt-reuse] | Block-size tradeoffs and host retention of reusable blocks |
| [Yu et al., *Orca* (OSDI 2022)][orca] | Iteration-level scheduling |
| [vLLM, *Optimization and Tuning*][vllm-tuning] | Cache-pressure preemption/recompute and chunked-prefill tradeoffs |
| [vLLM, `llama.py`, repository `main`][vllm-llama] | KV-head partitioning/replication in `LlamaAttention` |
| [Liu et al., *Ring Attention* (2023)][ring] | Sequence distribution and KV-block communication |
| [Zhong et al., *DistServe* (OSDI 2024)][distserve] | Prefill/decode disaggregation and latency-constrained goodput |
| [Su et al., *RoFormer* (2021)][rope] | Rotary positional encoding |
| [Chen et al., *Accelerating Large Language Model Decoding with Speculative Sampling* (2023)][speculative] | Draft verification and target-distribution-preserving acceptance |
| [vLLM, *Metrics* design][vllm-metrics] | Cache and serving observability categories and metric semantics |

[transformer]: https://arxiv.org/html/1706.03762
[hf-caching]: https://huggingface.co/docs/transformers/v4.57.1/cache_explanation
[hf-blog]: https://huggingface.co/blog/kv-cache
[hf-llama]: https://github.com/huggingface/transformers/blob/v4.57.1/src/transformers/models/llama/modeling_llama.py
[hf-cache-source]: https://github.com/huggingface/transformers/blob/v4.57.1/src/transformers/cache_utils.py
[mqa]: https://arxiv.org/html/1911.02150
[gqa]: https://arxiv.org/abs/2305.13245
[paged]: https://arxiv.org/html/2309.06180
[mistral]: https://arxiv.org/html/2310.06825v1
[streaming]: https://arxiv.org/abs/2309.17453
[deepseek]: https://arxiv.org/html/2405.04434v5
[mamba]: https://arxiv.org/abs/2312.00752
[hf-strategies]: https://huggingface.co/docs/transformers/v4.57.1/kv_cache
[flash]: https://arxiv.org/abs/2205.14135
[flash-code]: https://github.com/Dao-AILab/flash-attention
[kivi]: https://arxiv.org/html/2402.02750
[vllm-apc]: https://docs.vllm.ai/en/latest/features/automatic_prefix_caching/
[vllm-prefix]: https://docs.vllm.ai/en/latest/design/prefix_caching/
[trt-reuse]: https://nvidia.github.io/TensorRT-LLM/advanced/kv-cache-reuse.html
[orca]: https://www.usenix.org/conference/osdi22/presentation/yu
[vllm-tuning]: https://docs.vllm.ai/en/latest/configuration/optimization/
[vllm-llama]: https://github.com/vllm-project/vllm/blob/main/vllm/model_executor/models/llama.py
[ring]: https://arxiv.org/abs/2310.01889
[distserve]: https://arxiv.org/abs/2401.09670
[rope]: https://arxiv.org/abs/2104.09864
[speculative]: https://arxiv.org/abs/2302.01318
[vllm-metrics]: https://docs.vllm.ai/en/latest/design/metrics/
