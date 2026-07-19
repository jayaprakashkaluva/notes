# llama.cpp: Inference Architecture and Internals

> Grounded against the source tree at `C:\workspace\ai\llama.cpp` (master, July 2026).
> All `file:line` references point into that tree. Line numbers drift as the project moves
> fast; the function names are the stable anchors.

---

## 1. What "inference" means here

Inference is the *forward-only* execution of a trained transformer: given a sequence of
tokens, compute the probability distribution over the next token, pick one, append it, and
repeat. No gradients, no weight updates (llama.cpp does have a small training/finetune path
in `llama_context::opt_epoch`, but the entire codebase is optimized around the forward pass).

One decode step conceptually:

```
text --tokenize--> [t0 t1 ... tn]
                       |
                       v
     token embeddings lookup (tok_embd)          src/llama-graph.cpp:2151  build_inp_embd
                       |
                       v
     N x transformer layer:
        RMSNorm -> QKV proj -> RoPE -> attention (uses KV cache) -> out proj
        residual add
        RMSNorm -> FFN (SwiGLU or MoE)
        residual add
                       |
                       v
     final norm -> lm_head matmul  => logits [n_vocab]
                       |
                       v
     sampling (greedy / top-k / top-p / temperature / grammar ...)
                       |
                       v
     next token  --> appended to the sequence, loop again
```

Two properties drive the whole design:

1. **Autoregression**: each new token depends on all previous ones. Recomputing attention
   over the full history every step would be O(n^2) per token, so the per-layer attention
   keys/values of past tokens are stored in a **KV cache** and only the *new* token's
   Q/K/V are computed each step.
2. **Memory-bound generation**: for batch-of-1 decoding, the bottleneck is streaming the
   weights from RAM/VRAM, not FLOPs. Hence llama.cpp's emphasis on quantized weights
   (`ggml/src/ggml-quants.c`), `mmap` weight loading, and fused kernels.

---

## 2. Repository layout (what lives where)

| Path | Role |
|---|---|
| `include/llama.h` | The single public C API (models, contexts, batches, KV, sampling). |
| `src/llama.cpp` | API glue: backend init, device setup, model load entry points (`llama_model_load_from_file` at `src/llama.cpp:426`, `llama_backend_init` at `src/llama.cpp:89`). |
| `src/llama-model.{h,cpp}` | Model weights container, hyperparameters, tensor creation/offload policy, `llama_model::build_graph` (`src/llama-model.cpp:2288`). |
| `src/models/*.cpp` | ~138 per-architecture graph builders (llama, qwen3, gemma3, deepseek2, mamba, rwkv, bert, ...). One file = one architecture's forward pass. |
| `src/llama-context.{h,cpp}` | The inference session: `decode()`/`encode()`, output buffers, scheduler ownership, state save/load. This is the heart of runtime inference. |
| `src/llama-batch.{h,cpp}` | Batch sanitation and splitting into micro-batches (`llama_batch_allocr`). |
| `src/llama-graph.{h,cpp}` | Reusable graph-building blocks shared by all architectures: `build_attn`, `build_ffn`, `build_moe_ffn`, `build_norm`, input tensors, graph result/reuse logic. |
| `src/llama-kv-cache*.{h,cpp}`, `src/llama-memory-*.{h,cpp}` | The "memory module" family: unified KV cache, sliding-window (iSWA) cache, recurrent state (Mamba/RWKV), hybrid, DeepSeek MLA/DSA variants. |
| `src/llama-vocab.cpp`, `src/unicode.cpp` | Tokenizers (SPM/BPE/WPM/UGM/RWKV) and detokenization. |
| `src/llama-sampler.cpp` | Sampler chain (temperature, top-k, top-p, min-p, mirostat, grammar, ...). |
| `src/llama-model-loader.{h,cpp}` | GGUF parsing, mmap-based weight loading, tensor validation. |
| `src/llama-arch.cpp` | Architecture enum + GGUF key/tensor-name tables. |
| `ggml/` | The tensor library: `ggml.c` (ops/graphs), `ggml-backend.cpp` (device abstraction + scheduler), `ggml-alloc.c` (graph memory planner), `ggml-quants.c` (quant formats), plus one backend dir per device (`ggml-cuda`, `ggml-metal`, `ggml-vulkan`, `ggml-cpu`, ...). |
| `common/`, `tools/` | CLI/server/perplexity/quantize tools built on the public API (`tools/server` implements slots, continuous batching, OpenAI-compatible HTTP). |

Layering: **tools -> llama API (`src/`) -> ggml graph/scheduler -> device backends.**
The `src/` layer never launches a kernel itself; it only *describes* computation as a ggml
graph and hands it to the ggml backend scheduler.

---

## 3. The ggml execution model

### 3.1 Tensors and graphs

`ggml` (`ggml/src/ggml.c`, `ggml/include/ggml.h`) is a C tensor library with:

- **`ggml_context`**: an arena allocator holding tensor *metadata* (shape `ne[4]`,
  strides `nb[4]`, op type, source pointers). Creating an op like `ggml_mul_mat(ctx, A, B)`
  allocates no data and computes nothing - it records a node.
- **`ggml_cgraph`**: a topologically-ordered DAG of those nodes, produced by
  `ggml_build_forward_expand(gf, tensor)` walking back from the requested output.
- **Deferred execution**: llama.cpp builds a fresh (or reused) graph per micro-batch and
  submits it. Graph *data* memory is planned by `ggml-alloc.c`, which reuses buffers
  between non-overlapping node lifetimes, so activation memory is far smaller than
  (nodes x tensor size).

### 3.2 Quantization

Weights are stored in block-quantized formats (Q4_K, Q6_K, Q8_0, IQ-series, ...) defined in
`ggml/src/ggml-quants.c` and `ggml-common.h`. Blocks of 32/256 weights share fp16 scale(s)
plus packed low-bit integers. Backends implement matmuls *directly on the quantized data*
(e.g. dot products against a Q8-quantized activation), so weights are never fully
dequantized to fp32 in memory. This is what lets a 70B model run in ~40 GB.

### 3.3 Backends and the scheduler

`ggml-backend.cpp` abstracts devices (`ggml_backend_t`) and buffers. Backends are compiled
as pluggable modules and discovered at runtime by `ggml_backend_load_all()`
(called from `llama_backend_init`, `src/llama.cpp:89-101`).

The **scheduler** (`ggml_backend_sched`) executes one graph across multiple devices:

- `ggml_backend_sched_split_graph` (`ggml/src/ggml-backend.cpp:1014`) assigns every node a
  backend (weights pin nodes to the device holding them; user hints via
  `ggml_backend_sched_set_tensor_backend` override) and cuts the graph into contiguous
  **splits**, each executed on one backend.
- `ggml_backend_sched_compute_splits` (`ggml/src/ggml-backend.cpp:1541`) runs splits in
  order, asynchronously copying each split's inputs to its backend first. Events allow
  **pipeline parallelism** across multiple GPUs (multiple in-flight copies of the graph
  inputs, `sched->cur_copy`).
- Notable optimization: when MoE expert weights live in host memory and the split's first
  node is `GGML_OP_MUL_MAT_ID`, the scheduler reads the routing ids and uploads **only the
  experts actually used** by this batch instead of the whole expert tensor
  (`ggml/src/ggml-backend.cpp:1576-1660`).

This is how partial offload works: layers whose weights sit in VRAM produce GPU splits,
the rest run on the CPU backend, with automatic tensor copies at the boundaries.
`llama_context::graph_get_cb` (`src/llama-context.cpp:2467`) additionally pins small
`norm` nodes to the layer's device to avoid pathological split boundaries.

---

## 4. Model loading

Entry: `llama_model_load_from_file` (`src/llama.cpp:426`).

1. **GGUF parsing** (`ggml/src/gguf.cpp`, `src/llama-model-loader.cpp:525` ctor):
   GGUF is a single-file container = key/value metadata (architecture, hparams, tokenizer,
   RoPE config, quant type) + a tensor directory (name, shape, type, offset) + aligned
   tensor data. Split/sharded files (`-00001-of-000NN.gguf`) are all opened up front.
2. **Architecture dispatch**: `general.architecture` selects the `llama_model_*` subclass;
   its `load_arch_hparams`/`load_arch_tensors` run (e.g. `src/models/llama.cpp:3` and
   `:34`). `create_tensor` declares each expected tensor with its shape and looks it up in
   the GGUF index; flags mark optional (`TENSOR_NOT_REQUIRED`) or shared
   (`TENSOR_DUPLICATED`, e.g. tied `output`/`tok_embd`, `src/models/llama.cpp:44-46`).
3. **Placement**: `-ngl N` (n_gpu_layers) decides which layers' tensors are created in GPU
   buffer types vs CPU/host buffers (`llama_prepare_model_devices`, `src/llama.cpp:125`;
   split modes: none / layer / row / tensor-parallel meta-device).
4. **Data loading** (`src/llama-model-loader.cpp:1343-1570`): with `use_mmap` the file is
   memory-mapped and CPU tensors simply point into the mapping - zero-copy, lazily paged,
   sharable between processes. GPU tensors are streamed from the mapping into device
   buffers. `check_tensors` optionally validates row data (`ggml_validate_row_data`,
   `src/llama-model-loader.cpp:1410`).

Result: a `llama_model` = hparams + vocab + a vector of per-layer tensor structs, ready to
be referenced by graph builders. Models are immutable and can be shared by many contexts.

---

## 5. The inference session: `llama_context`

`llama_init_from_model` (`src/llama-context.cpp:3507`) creates the mutable session state:

- context params (`n_ctx`, `n_batch`, `n_ubatch`, `n_seq_max`, flash-attn, offload policy);
- one `ggml_backend_t` per participating device + the `ggml_backend_sched`;
- the **memory module** (KV cache variant chosen by the model architecture - see §8);
- host output buffers for logits/embeddings (`output_reserve`);
- `llama_batch_allocr` for batch splitting;
- optionally per-sequence **backend samplers** (sampling on GPU, §9).

The public API surface for one step (`include/llama.h`):

```c
llama_batch batch = ...;             // tokens, positions, seq ids, output flags
llama_decode(ctx, batch);            // src/llama-context.cpp:4071 -> llama_context::decode
float * logits = llama_get_logits_ith(ctx, i);   // src/llama-context.cpp:3683
llama_token t = llama_sampler_sample(smpl, ctx, i); // src/llama-sampler.cpp:806
```

`llama_batch` carries multiple **sequences** (`seq_id` per token), which is how one context
serves many parallel conversations (the server's "slots") sharing one weights copy and one
KV buffer.

---

## 6. Anatomy of `llama_context::decode` (`src/llama-context.cpp:1693`)

This is the core loop; worth reading line by line.

```
decode(batch)
 |-- balloc->init(batch, vocab, memory, ...)         :1744   sanitize batch, fill defaults,
 |                                                           validate positions vs memory
 |-- memory_update(false)                            :1779   apply pending KV shifts/copies
 |-- mctx = memory->init_batch(balloc, n_ubatch)     :1784   split into ubatches + reserve
 |       (on FAILED_PREPARE: retry once after                KV slots for all of them
 |        cache optimization, else return 1)         :1799
 |-- output_reserve(n_outputs_all)                   :1827   host logits/embd buffers
 |-- do { per ubatch:                                :1835
 |     n_outputs = outputs in this ubatch            :1839
 |     res = process_ubatch(ubatch, ..., mctx)       :1856   << graph build + compute
 |     copy logits  (async D2H)                      :1903
 |     copy embeddings (per pooling type)            :1918
 |     copy backend-sampling results                 :1999
 |   } while (mctx->next())                          :2013
 |-- record output order / plan lazy reorder swaps   :2019
 `-- return 0        (no synchronize here - see below)
```

Key mechanics:

- **Batch -> ubatch splitting** (`src/llama-batch.cpp`): a `llama_batch` of up to `n_batch`
  tokens is cut into micro-batches of at most `n_ubatch` tokens (`llama_ubatch`,
  `src/llama-batch.h:15`). Strategies: `split_simple` (`:476`, plain chunks for the
  unified attention cache), `split_equal` (`:510`, equal-length sequence sets, required by
  recurrent/hybrid memory), `split_seq` (`:681`, one sequence-set per ubatch). The memory
  module chooses the strategy inside `init_batch` (`src/llama-kv-cache.cpp:698`).
- **Error handling**: if a ubatch fails/aborts mid-batch, the tokens already written into
  the memory module for that ubatch are rolled back with `memory->seq_rm`
  (`src/llama-context.cpp:1859-1879`), keeping cache state consistent.
- **Output handling**: only tokens with `logits[i] != 0` produce outputs. The mapping from
  batch position to output row is kept in `output_ids`; if ubatch-splitting reordered
  outputs, swaps are recorded and applied lazily on first access
  (`src/llama-context.cpp:2036-2062`, `output_reorder` at `:2265`).
- **Asynchrony**: logits are copied with `ggml_backend_tensor_get_async`
  (`src/llama-context.cpp:1913`) and `decode` returns *without* synchronizing; the wait
  happens inside `llama_get_logits*` / `synchronize()` (`src/llama-context.cpp:697`). This
  overlaps GPU compute of ubatch N+1 with host-side work on ubatch N.

### 6.1 `process_ubatch` (`src/llama-context.cpp:1317`) - build, allocate, run

```
process_ubatch(ubatch, gtype, mctx):
    mctx->apply()                       // commit KV slot bookkeeping for this ubatch
    gparams = graph_params(...)         :1329
    if res->can_reuse(gparams):         :1331  // same topology as last graph?
        reuse previous graph (only inputs change)   -> n_reused++
    else:
        ggml_backend_sched_reset(sched)
        gf = model.build_graph(gparams) :1350  // ~thousands of nodes
        ggml_backend_sched_alloc_graph(sched, gf)  :1360  // split + plan memory
    res->set_inputs(&ubatch)            :1372  // upload tokens/positions/KQ-mask/...
    graph_compute(gf, n_tokens > 1)     :1377  // async submit to scheduler
```

**Graph reuse** matters: during steady-state generation every step has an identical graph
topology (same n_tokens=1, same KV layout parameters), so llama.cpp skips rebuilding and
re-planning entirely - only input tensor *contents* change. `can_reuse` compares
`llm_graph_params`, which is defined to uniquely determine the topology
(`src/llama-context.cpp:1328` comment; implementation in `src/llama-graph.h/.cpp`).

`graph_compute` (`src/llama-context.cpp:2438`) sets thread counts / threadpools for CPU
backends, then calls `ggml_backend_sched_graph_compute_async`
(`ggml/src/ggml-backend.cpp:1889`) -> split -> `compute_splits` (§3.3).

---

## 7. Building the transformer graph

`llama_model::build_graph` (`src/llama-model.cpp:2288`) instantiates the architecture's
graph class (e.g. `llama_model_llama::build_arch_graph`, `src/models/llama.cpp:94`). Each
of these derives from `llm_graph_context` (`src/llama-graph.h`), which provides shared
building blocks so a new architecture is mostly composition.

Walk-through of the LLaMA builder (`src/models/llama.cpp:99-247`):

```cpp
inpL = build_inp_embd(model.tok_embd);          // token ids -> ggml_get_rows(embd table)
inp_pos  = build_inp_pos();                     // positions for RoPE
inp_attn = build_attn_inp_kv();                 // KV-cache indices + KQ mask inputs
inp_out_ids = build_inp_out_ids();              // which rows produce output

for (il = 0..n_layer):
    cur = build_norm(inpL, attn_norm, RMS)                  // llama-graph.cpp:1451
    [Q,K,V] = build_qkv(layer, cur)                         // fused or separate wq/wk/wv
    Q = ggml_rope_ext(Q, inp_pos, rope_factors, ...)        // rotary pos-emb, models/llama.cpp:146
    K = ggml_rope_ext(K, ...)
    cur = build_attn(inp_attn, wo, ..., Q, K, V, kq_scale)  // llama-graph.cpp:2629
    if (last layer) rows = ggml_get_rows(cur, inp_out_ids)  // drop non-output rows early
    ffn_inp = cur + inpSA                                   // residual
    cur = build_norm(ffn_inp, ffn_norm, RMS)
    cur = ffn_gate_inp ? build_moe_ffn(...)                 // MoE: llama-graph.cpp:1755
                       : build_ffn(..., SILU, PAR)          // SwiGLU: llama-graph.cpp:1564
    inpL = cur + ffn_inp                                    // residual
cur = build_norm(inpL, output_norm)
res->t_embd = cur;
cur = build_lora_mm(model.output, cur);                     // lm_head (+ LoRA if attached)
res->t_logits = cur;
```

Note `inp_out_ids` (`src/models/llama.cpp:174-177`): before the LM head, rows that don't
need logits are discarded with `ggml_get_rows`, so during prompt processing the expensive
`[n_embd x n_vocab]` matmul runs only for the requested output tokens (usually just the
last one).

### 7.1 Attention internals: `build_attn` + `build_attn_mha`

`build_attn` (KV-cache variant, `src/llama-graph.cpp:2629`):

1. Expands Q/V/K into the graph in a fixed order to reduce scheduler splits and to enable
   the fused RoPE-writes-into-KV-cache path (`:2653-2658`).
2. **Writes the new K/V into the cache**: `mctx->cpy_k / cpy_v`
   (`src/llama-graph.cpp:2662-2669` -> `src/llama-kv-cache.cpp:1295/1330`) scatter the
   current ubatch's K/V rows into the big cache tensors at the slot indices chosen by
   `find_slot`, via `ggml_set_rows` with an indices input tensor (so the *same graph* can
   be reused with different destination slots - indices are just input data).
3. Gets cache **views** covering cells `[0, n_kv)`: `get_k` / `get_v`
   (`src/llama-kv-cache.cpp:1243/1263`).
4. Calls `build_attn_mha` (`src/llama-graph.cpp:2384`), which has two paths:
   - **Flash attention** (`:2407-2448`): single fused `ggml_flash_attn_ext(q,k,v,mask)` -
     no materialized `[n_kv x n_tokens]` score matrix, fp16 K/V, optional attention sinks,
     forced fp32 accumulation precision.
   - **Naive path** (`:2449-2512`): `kq = ggml_mul_mat(k, q)` (fp32 precision forced,
     `:2455`), optional logit softcapping (Gemma-style tanh) or Grok scaling, then
     `ggml_soft_max_ext(kq, kq_mask, kq_scale)` - the mask encodes causality/SWA/padding
     as `-inf` biases - then `kqv = ggml_mul_mat(v, kq)`.
   - MLA models (DeepSeek) pass `v_mla` to decompress latent-space attention output back
     to full head dimension (`:2431-2446`, `:2498-2501`).
5. Output projection `wo` (+ optional bias / LoRA), `src/llama-graph.cpp:2684-2699`.

GQA (n_head_kv < n_head) falls out naturally: K/V have fewer heads and ggml matmuls
broadcast across the grouped query heads.

### 7.2 FFN and MoE

- `build_ffn` (`src/llama-graph.cpp:1564`) implements the gated MLP family:
  `down( act(gate(x)) * up(x) )` with selectable activation (SILU/GELU/RELU/SWIGLU-OAI...)
  and parallel/sequential gating.
- `build_moe_ffn` (`src/llama-graph.cpp:1755`) implements routed experts: router matmul
  `ffn_gate_inp` -> softmax/sigmoid gating -> top-k expert selection (`ggml_top_k`) ->
  batched expert matmuls with `ggml_mul_mat_id` (indexed by the selected expert ids) ->
  weighted sum, plus optional shared expert.

### 7.3 Graph inputs

Every `build_inp_*` / `build_attn_inp_*` creates an input tensor
(`ggml_set_input`) and registers an `llm_graph_input_*` object on the graph result. After
(re)using a graph, `res->set_inputs(&ubatch)` (`src/llama-context.cpp:1372`) fills them:
token ids, positions, the KQ mask built from KV cell/sequence metadata, KV slot indices,
output row ids, pooling maps, etc. This clean split (topology vs data) is what makes graph
reuse safe.

---

## 8. The memory module (KV cache and friends)

`llama_memory_i` (`src/llama-memory.h`) abstracts "state carried across decode calls".
Implementations:

| Class | File | Used for |
|---|---|---|
| `llama_kv_cache` | `src/llama-kv-cache.cpp` | Standard causal attention (unified cache). |
| `llama_kv_cache_iswa` | `src/llama-kv-cache-iswa.cpp` | Two caches: full-attn layers + sliding-window layers (Gemma-style), the SWA one only keeps the window. |
| `llama_memory_recurrent` | `src/llama-memory-recurrent.cpp` | Mamba/RWKV fixed-size recurrent state per sequence. |
| `llama_memory_hybrid(_iswa)` | `src/llama-memory-hybrid.cpp` | Mix of attention + recurrent layers (Jamba, Falcon-H1). |
| DSA/DSv4 variants | `src/llama-kv-cache-dsa.cpp`, `-dsv4.cpp` | DeepSeek sparse/MLA-family attention caches. |

### 8.1 Unified KV cache internals (`src/llama-kv-cache.cpp`)

- **Storage**: per attention layer, one K tensor `[n_embd_k_gqa, kv_size, n_stream]` and
  one V tensor (`src/llama-kv-cache.cpp:231-247`), allocated in the backend buffer of that
  layer's device (offloaded KV). Cache dtype defaults to fp16; `--cache-type-k/v q8_0`
  etc. quantize it. With flash attention off, V may be stored transposed for the
  `kqv` matmul.
- **Cells** (`src/llama-kv-cells.h`): CPU-side bookkeeping parallel to the tensors - for
  each cell: position + set of sequence ids using it. A cell can be shared by many
  sequences (prompt sharing after `llama_memory_seq_cp`), which is how the server forks a
  common system prompt without copying K/V data.
- **Slot search**: `find_slot` (`src/llama-kv-cache.cpp:894`) scans from a moving `head`
  for free (or reusable) cells for the ubatch's tokens - contiguous or scattered
  (`cont` flag, `:1014`), with a heuristic reset of `head` to 0 when enough free cells
  exist before it (`:999-1003`). `prepare` (`:747`) dry-runs slot search for *all*
  ubatches of the batch before any compute, so a batch either fully fits or fails cleanly
  upfront. `apply_ubatch` (`:1093`) then commits cell metadata.
- **Streams**: with `kv_unified=false`, each sequence gets its own stream (dim 3 of the
  cache tensors), enabling per-sequence contiguous layouts.
- **Maintenance**: `memory_update` (`src/llama-context.cpp:775`) applies deferred
  operations: position shifts (context shift applies a RoPE delta to cached K),
  defragmentation (compacting scattered cells), and SWA pruning. `llama_memory_seq_rm/
  seq_cp/seq_keep/seq_add` are the public sequence-editing APIs (`include/llama.h`).
- **`n_kv`**: per ubatch the graph only attends over `[0, used cells upper bound)` rather
  than the full `kv_size`, keeping attention cost proportional to actual context.

The attention mask input then encodes, per (query token, cache cell): same-sequence?
causal (cell.pos <= query.pos)? inside SWA window? -> 0 or -inf. Masked softmax does the
rest; no control flow exists on the device side.

---

## 9. Sampling (`src/llama-sampler.cpp`)

Logits -> token happens outside `decode`, in a user-composed **sampler chain**
(`llama_sampler_chain_init` + `llama_sampler_chain_add`; chain apply at
`src/llama-sampler.cpp:642`). Each sampler transforms a `llama_token_data_array`
(token id / logit / prob triples):

- distribution shaping: temperature, top-k, top-p (nucleus), min-p, typical-p, XTC,
  top-n-sigma, mirostat v1/v2;
- constraints: repetition/frequency/presence penalties, DRY, logit bias,
  **grammar** (GBNF, `src/llama-grammar.cpp`) which zeroes out tokens that would violate a
  context-free grammar (used for forced-JSON output);
- terminal: `dist` (seeded multinomial draw) or `greedy` (argmax).

`llama_sampler_sample` (`src/llama-sampler.cpp:806`) shows the full flow: if a **backend
sampler** already picked a token on-GPU during decode (`llama_context::set_sampler`,
`src/llama-context.cpp:1201`; results copied at `src/llama-context.cpp:1999-2009`), it is
returned directly - avoiding the D2H copy of the whole `n_vocab` logit vector. Otherwise
the host builds the candidate array from logits (`:849-855`) and runs the chain.

After sampling, `llama_sampler_accept` informs stateful samplers (penalties, grammar,
mirostat) of the chosen token.

**Detokenization** (`llama_detokenize`, `src/llama-vocab.cpp`) maps the token back to
bytes; streaming UIs must buffer partial UTF-8 sequences.

---

## 10. Tokenization (`src/llama-vocab.cpp`)

`llama_vocab::impl::tokenize` (`src/llama-vocab.cpp:3291`) dispatches on the vocab type
read from GGUF (`:3306-3456`):

- **SPM** (SentencePiece BPE, LLaMA-1/2): greedy longest-match + bigram merges.
- **BPE** (GPT-2 style byte-level BPE, LLaMA-3, Qwen, most modern models): regex
  pre-tokenization (per-model patterns, using `src/unicode.cpp` tables) + rank-based
  merges.
- **WPM** (WordPiece, BERT), **UGM** (Unigram, T5), **RWKV** trie-based.

Special tokens (BOS/EOS/EOT/control tokens) are matched by a separate pass
(`tokenizer_st_partition`) so user text can't forge them unless `parse_special` is set.
Chat formatting (`llama_chat_apply_template`, `src/llama-chat.cpp` and the Jinja engine in
`common/jinja/`) happens above tokenization.

---

## 11. End-to-end life of a generated token

Putting it all together for one step of `llama-cli -m model.gguf -p "Hello"`:

1. **Startup** (once): `llama_backend_init` loads backend DLLs; model GGUF is mmapped and
   tensors placed per `-ngl`; context created with KV cache sized `n_ctx`.
2. **Prompt processing** ("prefill"): the prompt tokens go into one `llama_batch`
   (only the last token has `logits=1`). `decode` splits it into `n_ubatch` chunks; each
   chunk's graph processes hundreds of tokens at once (compute-bound, GPU-friendly), and
   writes K/V for every token into the cache. One logit row comes back.
3. **Sampling**: sampler chain picks token `t`.
4. **Generation loop**: a batch of exactly 1 token (`t`, position n, `logits=1`) is
   decoded: graph reused from last step, KV slot found at `head`, attention reads all
   cached cells (memory-bound), one logit row copied back asynchronously; sample; repeat
   until EOS/limit. Perf counters split these phases as `prompt eval` vs `eval`
   (`llama_perf_context`).
5. **Context overflow**: when the cache fills, callers either fail (return 1 from
   `decode`), shift (`llama_memory_seq_rm` + `seq_add` -> deferred K-RoPE shift in
   `memory_update`), or restart with a truncated history.

The server (`tools/server`) runs the same loop but multiplexes many sequences into each
batch (continuous batching): while one slot is prefilling, others decode single tokens in
the same `llama_decode` call - all sharing the weights and the unified KV buffer.

---

## 12. Performance techniques worth knowing (index)

| Technique | Where |
|---|---|
| Weight quantization (K-quants, IQ, MXFP4) | `ggml/src/ggml-quants.c`, `tools/quantize` |
| mmap zero-copy weight load | `src/llama-mmap.cpp`, `src/llama-model-loader.cpp:1343` |
| KV cache (O(n) per token instead of O(n^2)) | `src/llama-kv-cache.cpp` |
| Flash attention (fused, no score matrix) | `src/llama-graph.cpp:2407`, per-backend kernels |
| Graph reuse across decode steps | `src/llama-context.cpp:1331` (`can_reuse`) |
| Fused ops (RoPE->KV-store, norm+mul, MoE fusions) | `llama_context::resolve_fused_ops`, `src/llama-context.cpp:496` |
| Async compute + async D2H logit copies | `src/llama-context.cpp:1913,2457` |
| Pipeline parallelism across GPUs | `ggml/src/ggml-backend.cpp` (sched events, graph input copies) |
| Only-used-experts upload for MoE offload | `ggml/src/ggml-backend.cpp:1576` |
| Output-row pruning before lm_head | `src/models/llama.cpp:174` (`inp_out_ids`) |
| KV-cell sharing across sequences (prompt reuse) | `src/llama-kv-cells.h`, `seq_cp` |
| Backend (on-GPU) sampling | `src/llama-context.cpp:1201`, `src/llama-sampler.cpp:806` |
| Speculative decoding / draft models | `common/speculative.*`, `tools/server` |

---

## 13. Reading path suggestion

If you want to internalize the code, read in this order:

1. `include/llama.h` - the vocabulary of the whole system.
2. `src/llama-context.cpp:1693` (`decode`) then `:1317` (`process_ubatch`).
3. `src/models/llama.cpp` - one complete, small architecture builder.
4. `src/llama-graph.cpp:2384` (`build_attn_mha`) and `:2629` (`build_attn`).
5. `src/llama-kv-cache.cpp:698` (`init_batch`), `:894` (`find_slot`), `:1295` (`cpy_k`).
6. `ggml/src/ggml-backend.cpp:1014` (`split_graph`) and `:1541` (`compute_splits`).
7. `src/llama-sampler.cpp:806` (`llama_sampler_sample`).
