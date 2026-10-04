# Transformer Architecture: Mechanisms, Cost Models, and Engineering Decisions

**Audience:** staff and principal engineers designing, implementing, or evaluating Transformer systems.  
**Sources reviewed:** October 4, 2026.  
**Scope:** text Transformers, with the original encoder-decoder architecture as the reference and dense causal decoders as the principal systems example.

This guide separates **source-backed mechanisms**, **mathematical derivations under stated assumptions**, and **engineering judgments**. Citations sit next to the mechanisms or findings they support. Derived numerical examples are hypothetical, not benchmarks. Architecture names refer to the cited papers or implementations; they do not imply that every model in a family uses the same configuration. Framework-specific statements use PyTorch **2.9** and Hugging Face Transformers **4.57.1** documentation.

## Contents

1. [Architecture and information flow](#1-architecture-and-information-flow)
2. [Notation and the tensor contract](#2-notation-and-the-tensor-contract)
3. [Attention, precisely](#3-attention-precisely)
4. [Masks are correctness boundaries](#4-masks-are-correctness-boundaries)
5. [Feed-forward layers and conditional computation](#5-feed-forward-layers-and-conditional-computation)
6. [Residual paths and normalization](#6-residual-paths-and-normalization)
7. [Position representations](#7-position-representations)
8. [Objectives and training execution](#8-objectives-and-training-execution)
9. [Autoregressive inference and persistent state](#9-autoregressive-inference-and-persistent-state)
10. [Parameter, compute, and memory models](#10-parameter-compute-and-memory-models)
11. [Execution kernels and distributed decomposition](#11-execution-kernels-and-distributed-decomposition)
12. [Architecture decisions and their evidence](#12-architecture-decisions-and-their-evidence)
13. [Implementation verification and design review](#13-implementation-verification-and-design-review)
14. [Primary sources](#14-primary-sources)

## 1. Architecture and information flow

The original Transformer has an encoder stack and an autoregressive decoder stack. Encoder layers contain self-attention and a position-wise feed-forward network; decoder layers add attention over encoder outputs. Residual connections and normalization surround the sublayers. Decoder self-attention masks future positions. These are the structural facts in Vaswani et al., section 3.1. [Original Transformer][original]

Separate three contracts when reviewing a model:

- **Visibility:** which positions can influence a representation?
- **Transformation:** how do attention, nonlinear layers, and normalization transform that representation?
- **Execution:** how are those operations scheduled, partitioned, and stored?

This separation is an engineering framing used throughout this guide. A mask changes visibility; replacing a feed-forward layer changes transformation; an equivalent fused kernel changes execution. They require different validation evidence.

| Family | Information flow | Representative training setup |
|---|---|---|
| Encoder-only | Bidirectional self-attention over the supplied sequence | BERT uses masked language modeling and next-sentence prediction. |
| Decoder-only | Each position sees itself and preceding positions | GPT-style autoregressive language modeling. |
| Encoder-decoder | Encoder sees source context; decoder sees its target prefix and encoder representations | Original translation Transformer; T5 uses a text-to-text setup with span corruption among the objectives it studies. |

These rows describe the cited examples, not mandatory objectives for all architectures. [BERT, sections 3.1 and A.1][bert]; [GPT-3, section 2][gpt3]; [T5, sections 2.1 and 3.3][t5]

For the causal decoder example, the data path is:

```text
token IDs [B,T]
    |
embedding lookup -> hidden states [B,T,D]
    |
L Transformer blocks
    |   attention: exchange information between allowed positions
    |   feed-forward: transform features at each position
    |   residual and normalization: compose the sublayers
    |
final normalization, when specified by the architecture
    |
output projection -> logits [B,T,Vocab]
    |
training loss OR inference token-selection policy
```

This is a schematic of the decoder implementation discussed below, not a universal block ordering. GPT-2's released implementation provides a concrete example with learned token/position embeddings, pre-normalized blocks, final normalization, and an output projection tied to the token embedding table. [GPT-2, `model` and `block`][gpt2-code]

## 2. Notation and the tensor contract

| Symbol | Meaning |
|---|---|
| `B` | Batch size; for inference, number of active sequences in a simple batch |
| `T` | Sequence length in full-sequence self-attention |
| `Tq`, `Tk` | Query and key/value sequence lengths |
| `D` | Hidden/residual-stream width |
| `L` | Number of Transformer blocks |
| `Hq`, `Hkv` | Query-head and key/value-head counts |
| `dh` | Per-head query/key width; also value width in this guide's simplified model |
| `F` | Feed-forward intermediate width |
| `Vocab` | Vocabulary size, distinct from the value tensor `V` |
| `s` | Bytes per stored scalar in a particular tensor |

**Assumptions for subsequent derivations:** row-vector activations; dense linear projections; equal query/key/value head widths; `D = Hq * dh`; equal K and V head counts; `Hq` divisible by `Hkv`; no biases in parameter/FLOP estimates. These assumptions are a tractable model, not constraints on every Transformer.

For hidden states `Xq:[B,Tq,D]` and `Xkv:[B,Tk,D]`, the projection shapes are:

```text
Wq: [D, Hq*dh]          Q: [B,Hq,Tq,dh]
Wk: [D, Hkv*dh]         K: [B,Hkv,Tk,dh]
Wv: [D, Hkv*dh]         V: [B,Hkv,Tk,dh]
Wo: [Hq*dh, D]          attention output after Wo: [B,Tq,D]
```

The reshaping follows the projection dimensions. For ordinary multi-head attention, `Hkv = Hq`. PyTorch's multi-head module documents splitting the embedding width across heads, while its lower-level attention function documents separate query and KV head axes. [MultiheadAttention][pytorch-mha]; [scaled dot-product attention][pytorch-sdpa]

An embedding is a lookup in a learned table `E:[Vocab,D]`: `X[b,t,:] = E[token_id[b,t],:]`. Looking up a token returns a vector; it does not return a context-aware representation. [Embedding API][pytorch-embedding]

**Consequence of the computation graph:** contextualization occurs in later layers, so identical token IDs can produce different final vectors when their allowed contexts differ. Tensor shape alone cannot establish that a model is correct: head ordering, mask orientation, and position conventions must also agree.

## 3. Attention, precisely

### 3.1 The operator

GQA shares one K/V head within each query-head group. MQA and MHA are limiting cases. [GQA, section 2.2][gqa]

For query head `h`, let `g(h)` select its KV head. Using contiguous, uniformly sized groups gives `g(h)=floor(h / (Hq/Hkv))`; ordinary MHA gives `g(h)=h`. This indexing convention implements the grouping in the equations below.

```text
S[b,h,i,j] = dot(Q[b,h,i,:], K[b,g(h),j,:]) / sqrt(dh)
              + positional_bias[h,i,j] + mask[b,h,i,j]

A[b,h,i,:] = softmax(S[b,h,i,:])       # over key positions j
O[b,h,i,:] = sum_j A[b,h,i,j] * V[b,g(h),j,:]
Y = merge_query_heads(O) @ Wo
```

The scaled score, additive mask, row softmax, and weighted value aggregation are the standard attention computation. `positional_bias` is optional; RoPE instead transforms Q and K before scoring. [Attention operator][pytorch-sdpa]; [RoFormer, section 3.2][rope]

**Derived interpretation:** Q and K choose mixture coefficients; V supplies the mixed vectors. The raw attention output for each head lies in the convex hull of the allowed value vectors when dropout is disabled. The subsequent output projection and residual addition need not preserve that convex-hull property. Q, K, and V are learned projections of representations; calling them search queries, database keys, and records is an analogy, not an implementation contract.

### 3.2 Why scaling matters

Under the illustrative assumption that independent components of Q and K have zero mean and unit variance, a dot product of width `dh` has variance `dh`. Division by `sqrt(dh)` normalizes that variance. This is the original paper's motivation for avoiding large scores that saturate softmax. Actual learned activations need not satisfy those independence assumptions. [Vaswani et al., section 3.2.1 and footnote 1][original]

**Derivative-based consequence:** for one row, the softmax Jacobian is `diag(a) - a*a^T`. As one coefficient approaches one and the others approach zero, these derivatives shrink. Stable numerical evaluation uses:

```text
m = max_j S[j]
a[j] = exp(S[j] - m) / sum_k exp(S[k] - m)
```

Subtracting the same constant preserves the distribution because the exponential factor cancels. It improves numerical range; it does not cure saturation caused by the distribution itself.

### 3.3 Multi-head attention is independent routing

Heads use distinct projections and softmax distributions; their outputs are concatenated and projected. [MultiheadAttention definition][pytorch-mha]

**Derived consequence:** this differs from splitting the result of a single softmax into pieces. Each head can mix positions differently. Holding `D` fixed while increasing `Hq` decreases `dh` in our model, so it changes routing granularity rather than multiplying the projection parameter count by the head count. Sections 10 and 12 quantify the exceptions introduced by changing KV sharing or widths.

There is no architectural rule assigning a head to syntax, coreference, or any other human concept. Those assignments would require evidence for the particular trained model. The equations specify an information-processing mechanism, not a guaranteed semantic decomposition.

### 3.4 Self-attention and cross-attention

In self-attention, projections originate from the same sequence. In encoder-decoder cross-attention, Q originates in the decoder and K/V originate in encoder outputs. [Original Transformer, section 3.2.3][original]

**Shape consequence:** cross-attention scores have shape `[B,Hq,Ttarget,Tsource]`, not necessarily a square shape. Attention therefore does not require query and key sequence lengths to match. A decoder can cache cross-attention K/V derived from fixed encoder outputs; those tensors differ from its growing causal self-attention cache.

## 4. Masks are correctness boundaries

### 4.1 Causal, padding, and document boundaries

Use an additive mask with zero for allowed pairs and negative infinity for forbidden pairs. For full-sequence causal attention at zero-based positions:

```text
allowed(i,j) = (j <= i)
```

For a new query chunk following a prefix of length `P`:

```text
query absolute position = P+i
allowed(i,j) = (j <= P+i)
```

The second formula is a **derivation from causality**, not a different model. In single-token decoding, the new query normally sees all valid keys in its prefix plus itself.

Padding masks suppress non-content key positions. PyTorch's MHA API explicitly defines a key-padding mask for that purpose. [Key-padding mask][pytorch-mha]

**Engineering judgment for independently packed documents:** if multiple independent examples share a tensor, require both causality and matching document IDs. Reset position IDs only if the chosen architecture/training convention requires it. Loss masking alone cannot prevent cross-document attention because it changes which predictions are scored, not which inputs influence them.

### 4.2 Boolean-mask APIs are not interchangeable

In PyTorch 2.9, `True` means **allowed** in functional scaled dot-product attention, but **blocked** in `MultiheadAttention.attn_mask`. Functional attention also applies its requested dropout even during evaluation unless the caller passes `dropout_p=0.0`. Its non-square `is_causal=True` mask uses upper-left alignment. These are API-specific facts. [Functional attention parameters and warnings][pytorch-sdpa]; [MHA mask parameters][pytorch-mha]

**Derived failure example:** an upper-left triangle with one query and four keys permits only key zero. A query representing absolute position three should see keys zero through three. Therefore a cached decode call cannot safely substitute a generic non-square causal flag for a mask whose absolute-position semantics have been verified.

### 4.3 Fully masked rows need a policy

**Mathematical edge case:** softmax over a row containing only negative infinity is undefined by the stable formula: subtracting the maximum produces `-inf - (-inf)`. Kernels may handle that case differently. An implementation should avoid such rows for scored query positions or explicitly define and verify their output. Do not depend on incidental NaN suppression.

## 5. Feed-forward layers and conditional computation

### 5.1 Dense feed-forward layers

The position-wise network transforms each token independently with shared weights across positions:

```text
FFN(x) = activation(x @ Wup + bup) @ Wdown + bdown
Wup: [D,F]                    Wdown: [F,D]
```

The original form uses ReLU; other activations and gated forms are documented in Shazeer's comparison. [GLU variants, sections 1-2][glu]

**Derived division of labor:** attention mixes positions; the dense FFN mixes features at each position. Because attention has already introduced contextual information, a position-wise FFN can still compute a context-dependent result. Removing the nonlinear transformation would change the functions the network can represent; the title of the original paper does not imply an attention-only computation graph.

For a bias-free SwiGLU layer:

```text
FFN(x) = (SiLU(x @ Wgate) * (x @ Wup)) @ Wdown
SiLU(z) = z * sigmoid(z)
Wgate, Wup: [D,F]              Wdown: [F,D]
```

Here `*` is element-wise multiplication. Shazeer evaluates gated alternatives with matched parameter/compute budgets by reducing the intermediate width, and reports favorable results for GEGLU and SwiGLU in that setup. This supports a scoped comparison, not a universal guarantee. [GLU variants, sections 2-3][glu]

**Parameter derivation:** two projections have `2DF` weights; three have `3DF`. Matching a conventional `F=4D` layer requires gated `F=8D/3`, before hardware-friendly rounding. Comparing activations at an unchanged `F` would also change the parameter and arithmetic budget.

### 5.2 Mixture of experts changes the workload

Switch Transformer routes each token to one selected feed-forward expert. Its routing introduces load balancing and finite expert capacity; overflow can cause expert computation to be skipped via the residual path in its described implementation. [Switch Transformer, sections 2.1-2.2][switch]

**Engineering consequences:** total parameter count, active parameters per token, and communication volume become separate quantities. Routing skew can leave some devices idle while others are overloaded. An MoE comparison should include dispatch/combine time, capacity policy, utilization, and quality, rather than inferring latency from active arithmetic alone. This discussion concerns Switch-style routing; other MoE designs may select multiple experts or use different overflow policies.

## 6. Residual paths and normalization

### 6.1 Ordering changes optimization

Post-normalized and pre-normalized sublayers have different graphs:

```text
Post-LN: y = Norm(x + Sublayer(x))
Pre-LN:  y = x + Sublayer(Norm(x))
```

A representative sequential pre-normalized decoder block is:

```text
u = x + Attention(Norm1(x), positions, visibility)
y = u + FFN(Norm2(u))
```

Dropout and bias terms are omitted here. GPT-2's `block` implements this sequential residual ordering, using LayerNorm and a GELU MLP. [Released implementation][gpt2-code]

Xiong et al. analyze normalization placement and show, under their mean-field assumptions, large initialization gradients near the output in Post-LN and better-behaved initialization gradients in Pre-LN. Their experiments demonstrate Pre-LN training without warmup in the studied settings. They do not establish that all pre-normalized large models should omit warmup. [Normalization analysis, sections 3-4][preln]

**Local derivative:** in Pre-LN, `dy/dx = I + J_sublayer * J_norm`. Post-LN instead has `dy/dx = J_norm * (I + J_sublayer)`. Thus only the first has an explicit identity term outside normalization. This explains the structural difference, while convergence still depends on initialization, depth, optimizer, and data. A final norm in a Pre-LN model is an additional architectural choice; inspect the actual checkpoint specification.

### 6.2 LayerNorm and RMSNorm

For a single token vector of width `D`, LayerNorm is:

```text
mu = mean_i x[i]
variance = mean_i (x[i]-mu)^2
LN(x)[i] = gamma[i] * (x[i]-mu) / sqrt(variance+epsilon) + beta[i]
```

Configured with normalized shape `D`, it reduces over features per token, not over the sequence or batch. PyTorch uses input statistics in both training and evaluation. [LayerNorm API][pytorch-ln]

RMSNorm removes mean subtraction and normalizes by the root mean square. A common stabilized, gain-only form is:

```text
RMSNorm(x)[i] = gamma[i] * x[i] / sqrt(mean_j x[j]^2 + epsilon)
```

The RMSNorm paper defines the mechanism and evaluates it; it is not algebraically identical to general LayerNorm. [RMSNorm, section 4][rmsnorm]

**Engineering judgment:** norm type, epsilon, accumulation dtype, and placement belong in the model's execution contract. Substituting one norm for another changes outputs and normally requires a trained-model evaluation, not merely a shape check. LLaMA's original paper specifies pre-normalization with RMSNorm, SwiGLU, and RoPE; that is one documented combination. [LLaMA, section 2.2][llama]

## 7. Position representations

### 7.1 Why position information matters

**Algebraic result:** for unmasked self-attention without positional information, consistently permuting the input positions permutes the output positions. If `P` is a permutation matrix, then `Q'K'^T = P(QK^T)P^T`; row softmax and value aggregation preserve that relationship. Position-wise FFNs and per-token norms also preserve it. Such a stack cannot distinguish order through token features alone.

A causal mask changes that symmetry by imposing a directional visibility pattern. Consequently, the stronger statement that *every* Transformer without explicit position embeddings is blind to order would be unjustified by this argument.

### 7.2 Absolute and rotary representations

The original model adds sinusoidal position vectors to embeddings; GPT-2's code instead looks up learned position vectors. [Original, section 3.5][original]; [GPT-2, `positions_for` and `model`][gpt2-code]

RoPE rotates pairs of projected Q/K coordinates using position-dependent angles. For one pair:

```text
R(p*theta) = [[cos(p*theta), -sin(p*theta)],
              [sin(p*theta),  cos(p*theta)]]
q_rot[p] = R(p*theta) q[p]
k_rot[r] = R(r*theta) k[r]
```

With column-vector coordinates in this local equation:

```text
q_rot[p]^T k_rot[r] = q[p]^T R((r-p)*theta) k[r]
```

The explicit rotary factor therefore depends on relative displacement. This follows the construction in RoFormer; the full representation can still depend on contextual content and masking. [RoFormer, equations 13-16][rope]

**Engineering judgment:** preserve the checkpoint's coordinate pairing, rotary width, frequencies, scaling, and position IDs. Mathematical evaluability at a longer position does not establish useful behavior at that length. Changing a serving limit is not evidence that the model learned to use the enlarged context reliably. Evaluate relevant tasks at the target lengths and placements.

## 8. Objectives and training execution

### 8.1 Autoregressive likelihood and target shifting

GPT-3 is trained as an autoregressive language model. [GPT-3, section 2][gpt3]

For a causal model, the probability chain rule and negative log-likelihood give:

```text
p(x[0:T]) = product_t p(x[t] | x[0:t])
loss = -sum_t log p(x[t+1] | x[0:t+1])
```

Here slicing uses an exclusive end; the second line scores next-token predictions where a next token exists. A start token or equivalent convention supplies the initial context if the first token is scored. In a simple training tensor:

```text
inputs:  [x0, x1, x2]
labels:  [x1, x2, x3]
```

**Graph consequence:** all provided input positions can be evaluated in parallel within a layer because causal masking restricts dependencies to previous-layer states at allowed positions. Ground-truth preceding tokens are available during training. Generation must obtain newly selected tokens before processing their successors. Parallel teacher-forced training and sequential ordinary decoding therefore coexist without contradiction.

For final hidden states `H`:

```text
logits = H @ Wlm                  Wlm: [D,Vocab]
p(token=v) = softmax(logits)[v]
```

GPT-2 uses the transpose of its input embedding table for the output projection. Tying is a documented configuration, not a requirement of attention. [GPT-2, `model`][gpt2-code]

**Engineering judgment:** specify exactly which positions contribute to loss and its denominator. Ignoring padding or prompt labels changes the learning signal; it does not automatically change visibility. A token-normalized loss and a sequence-normalized loss weight variable-length examples differently. Verify both target alignment and masking at packed-document boundaries.

### 8.2 Other objectives and training state

BERT predicts selected masked tokens using bidirectional context; T5 reconstructs corrupted spans in an encoder-decoder formulation. Neither should be described as ordinary causal next-token training merely because it contains Transformer blocks. [BERT, section 3.1][bert]; [T5, sections 3.1.4 and 3.3.4][t5]

Training stores state for differentiation and optimization in addition to weights. Activation checkpointing retains selected intermediates and recomputes others during backward, trading additional computation for lower memory. [Training with sublinear memory cost][checkpoint]

**Budget judgment:** state an optimizer and dtype policy before estimating training memory. For example, an assumed setup with 2-byte weights, 2-byte gradients, a 4-byte master weight copy, and two 4-byte optimizer moments has a nominal persistent cost of `16 * parameter_count` bytes before activations and temporary buffers. This is an accounting example; many implementations use other policies or shard/offload those states.

Training compute also competes between model size and data volume. Hoffmann et al. find approximately equal scaling of parameters and training tokens for compute-optimal training in their experimental regime. That finding does not prescribe a universal tokens-per-parameter ratio or a serving-optimal model size. [Chinchilla study][chinchilla]

## 9. Autoregressive inference and persistent state

### 9.1 Prefill and decode

During **prefill**, process known prompt tokens and construct their per-layer K/V states. The final prompt position supplies logits for the first generated token. During ordinary **decode**, process the selected token, append its K/V at each layer, and produce logits for the following token. Cached K/V avoids recomputing prior representations. [Caching mechanics][hf-cache]

**Causality proof sketch:** with fixed weights, positions, and inference behavior, appending future tokens cannot alter an earlier position's hidden state. Induct over layers: its attention sees only earlier/current positions, and its FFN and norm act locally. Consequently, earlier K/V remains reusable. This reasoning assumes the architecture respects those locality and visibility contracts.

The cache is layer-specific model state, not token IDs or stored answers. For a request with `T` valid positions, its logical K/V shape under our assumptions is `[L,2,Hkv,T,dh]`, with a batch axis added for a simple batched layout. The physical implementation need not use this contiguous layout. [Per-layer caching][hf-cache]; [PagedAttention, sections 3-4][paged]

**Engineering judgment for reuse:** identify the exact prefix tokens, model weights/adapters, position convention, attention visibility, and representation policy that produced a cache. Reusing the same text after tokenization or model configuration changes is not established as valid by textual equality alone.

### 9.2 MHA, MQA, and GQA

| Attention form | KV heads | Architectural consequence |
|---|---:|---|
| MHA | `Hkv=Hq` | Each query head has its own K/V projections. |
| MQA | `Hkv=1` | All query heads share K/V. |
| GQA | Typically `1 < Hkv < Hq` | Query groups share K/V. |

GQA reports quality near MHA and speed near MQA for evaluated uptrained T5 configurations; conversion includes additional training. [GQA, sections 2-3][gqa]

**Evidence boundary:** this result does not establish that averaging a checkpoint's K/V heads preserves quality without adaptation.

**Derived consequence:** fewer KV heads directly reduce cache storage, but query-head attention still computes separate distributions. Cache-size reduction therefore does not imply an equal reduction in total model arithmetic or latency.

### 9.3 Logical state versus allocation

PagedAttention stores KV in physical blocks addressed by logical block tables, permitting non-contiguous storage and sharing. The paper describes copy-on-write for shared blocks and allocation as requests grow. [PagedAttention, sections 4.2-4.4][paged]

**Engineering judgment:** include partially filled blocks, workspace, replicated KV, and allocator overhead in capacity planning. A scheduler's maximum request count should follow its token/state budget and latency targets, not a single batch-size setting. Releasing completed requests is as important to sustainable capacity as computing attention efficiently.

## 10. Parameter, compute, and memory models

This section contains **derivations from the stated shapes**, not measured performance or paper benchmark reproductions.

### 10.1 Parameter counts

With `D=Hq*dh`, the attention projection weights per block are:

```text
Nattention = D^2 + 2D(Hkv*dh) + D^2
           = 2D^2 + 2D(Hkv*dh)
```

For MHA this is `4D^2`. A two-projection FFN has `2DF` weights; a three-projection gated FFN has `3DF`. Thus:

```text
NMHA block, two-projection FFN = 4D^2 + 2DF
NMHA block, F=4D              = 12D^2
```

Add `Vocab*D` for a token embedding table and another `Vocab*D` for an untied output projection. Add position tables, biases, and norm parameters when present. An encoder-decoder needs separate stack accounting and an extra attention module per decoder layer. MoE needs separate total/active expert accounting.

**Example:** `D=4096`, `Hq=32`, `dh=128`, `Hkv=8`, `F=11008`, `L=32`, and `Vocab=32000`, with a three-projection gated FFN and tied embeddings:

```text
attention/block = 41,943,040
FFN/block       = 135,266,304
blocks total    = 5,670,699,008
embedding       = 131,072,000
subtotal        = 5,801,771,008 parameters
```

This hypothetical subtotal omits norms/biases; it does not identify a particular released model. Notice that the FFN has more parameters than attention in this configuration.

### 10.2 Forward arithmetic

Count one multiply plus one addition as **two FLOPs**. For a full-sequence batch and one block, projection arithmetic is approximately:

```text
FLOPs attention projections = 4BTD^2 + 4BTD(Hkv*dh)
FLOPs two-projection FFN    = 4BTDF
FLOPs gated FFN            = 6BTDF
```

A full rectangular score product and weighted value product contribute:

```text
FLOPs QK^T + AV = 4B * Hq * Tq * Tk * dh
```

For self-attention this becomes `4BT^2D`. These estimates omit softmax, activations, norms, dropout, and bias operations. A causal kernel that skips forbidden pairs uses approximately half the pairwise arithmetic for large `T`; one that computes a full square matrix and then masks it does not.

For MHA and `F=4D`, the simplified full-square block cost is:

```text
24BTD^2 + 4BT^2D
```

Equating the two terms gives `T=6D`. This is an algebraic crossover within this particular cost model. It is not the sequence length at which attention becomes the wall-clock bottleneck: data movement, parallelism, precision, and skipped causal work change that answer.

The output projection costs approximately `2BTD*Vocab` if evaluated at every position. Prefill that only needs last-position logits can avoid projecting every position, subject to the implementation. During training, `[B,T,Vocab]` logits can also be a substantial memory term unless the loss/projection implementation avoids full materialization.

### 10.3 Decode is a different shape regime

For single-token decode with prefix length `T`, use `Tq=1`, `Tk=T`. Pairwise attention arithmetic is then `4BTHq*dh = 4BTD`; per-token projections and FFN do not gain a factor of `T`.

Generating `G` tokens after a prompt of length `P` requires pairwise work proportional to:

```text
sum_{g=0}^{G-1} (P+g+1) = GP + G(G+1)/2
```

The cache removes repeated prefix evaluation, but each new query still interacts with its visible history. Long-context decoding therefore remains more expensive even with perfect cache reuse.

### 10.4 Cache and activation memory

For equal-length requests, full-history self-attention KV storage is:

```text
Mkv = 2 * L * B * T * Hkv * dh * s
```

For unequal lengths, replace `BT` by `sum_r T[r]`. Different layer widths or windows require a sum over layers. Quantization metadata and allocator overhead are excluded.

In the hypothetical configuration above, with `s=2`:

```text
KV bytes/token/sequence = 2*32*8*128*2 = 131,072 = 128 KiB
KV at T=8192           = 1 GiB per sequence
KV at B=16             = 16 GiB
```

Using `Hkv=32` at otherwise unchanged dimensions makes those KV quantities four times larger. Two-byte weights for the approximately 5.80B-parameter subtotal occupy about 10.81 GiB, excluding the omitted parameters. Cache can consequently be comparable to weights at useful concurrency.

A naive materialized attention matrix needs `B*Hq*Tq*Tk*s` bytes. With `B=1`, `Hq=32`, `T=8192`, `s=2`, **one** full-square score/probability tensor is 4 GiB. This is distinct from KV storage and from total training activation memory.

### 10.5 A latency model needs bytes and communication

As an engineering lower-bound model for a kernel:

```text
time >= max(FLOPs / achievable_compute_rate,
            bytes_transferred / achievable_memory_bandwidth)
```

End-to-end latency additionally includes launches, dependencies, synchronization, communication, scheduling, and token selection. Use measured achievable rates for the relevant shapes, not advertised peak throughput.

The GQA paper discusses decoder weight/KV bandwidth as an inference bottleneck. [GQA, introduction][gqa]

**Engineering inference:** small decode batches provide limited reuse of weights across simultaneous tokens, while larger prefill matrices can provide more reuse. KV traffic grows with history. These tendencies explain why one model can have different bottlenecks in prefill and decode; they do not establish a universal bandwidth/compute classification.

## 11. Execution kernels and distributed decomposition

### 11.1 FlashAttention preserves the dense operator

FlashAttention tiles Q/K/V into fast on-chip memory, maintains softmax normalization statistics across tiles, and avoids writing full attention matrices to GPU high-bandwidth memory. It recomputes relevant intermediates during backward. Its dense algorithm computes exact attention rather than approximating sparsity. [FlashAttention, sections 2-3][flash]

The pairwise dense arithmetic remains quadratic in sequence length; the eliminated quadratic intermediate is a different quantity. Exactness here concerns the mathematical operator, not bitwise identity across floating-point evaluation orders. Framework documentation explicitly notes backend-dependent numerical results. [FlashAttention][flash]; [PyTorch attention numerical behavior][pytorch-sdpa]

**Engineering judgment:** record the selected backend and supported shapes/dtypes when benchmarking. Requiring full attention weights for inspection may force materialization or prevent a faster path. Returning an attention matrix is an observability decision with a potential memory cost, not a free diagnostic.

### 11.2 Tensor parallelism follows the algebra

Megatron-LM partitions the first MLP projection by output columns and the second by input rows. It also partitions attention heads so their attention computation is local, followed by a partitioned output projection and reduction. [Megatron-LM, section 3][megatron]

**Derivation for two shards:**

```text
Wup = [Wup_1 | Wup_2]
z_r = activation(x @ Wup_r)
Wdown = vertical_stack(Wdown_1, Wdown_2)
y = z_1 @ Wdown_1 + z_2 @ Wdown_2
```

The nonlinear activation is local because it acts element-wise. Combining partial outputs requires a sum before subsequent computation that assumes the complete hidden state. Gated FFNs additionally require aligned partitions of the gate and up projections.

The GQA paper discusses KV-head replication under model partitioning. [GQA, section 2.2][gqa]

**Engineering inference:** matching arithmetic partitions to communication boundaries matters more than merely distributing parameter bytes. Very small per-device matrices can reduce kernel efficiency; cross-device sums can become significant at low batch sizes. KV replication means logical cache savings need not equal per-device savings.

### 11.3 Other parallel axes

Large-scale Megatron work studies data, tensor, and pipeline parallelism together. Data parallelism processes different examples with model replicas; tensor parallelism divides operations within layers; pipeline parallelism divides layers into stages. Their efficient combination depends on communication and pipeline scheduling. [Distributed Megatron study][megatron-scale]

**Engineering judgment:** choose partitions from the workload and interconnect. Tensor parallelism has frequent within-layer dependencies. Pipeline stages exchange activations and can incur idle periods when insufficient work is in flight. Additional serving replicas increase aggregate capacity but also consume additional weight memory. A design review should explain why the chosen split meets both capacity and latency requirements.

## 12. Architecture decisions and their evidence

The following table is **engineering synthesis**, not a ranking or a claim that a listed modification always improves quality.

| Decision | What it changes | Evidence needed for the target system |
|---|---|---|
| Encoder, decoder, or encoder-decoder | Visibility and conditioning graph | Objective fit, task quality, input/output length distribution |
| Hidden width and depth | Parameter budget and sequential block work | Matched-budget quality; measured latency and training stability |
| Query-head count/head width | Number and width of routing distributions | Checkpoint compatibility, kernel support, quality |
| MHA versus GQA/MQA | KV capacity and sharing | Cache budget, quality after training/adaptation, decode profile |
| Norm placement/type | Numerical transformation and gradient graph | Optimization behavior and checkpoint-faithful outputs |
| FFN activation and width | Nonlinear features and arithmetic | Comparisons matched for parameters/compute |
| Dense FFN versus MoE | Conditional parameter activation and routing | Utilization, dispatch cost, capacity behavior, quality |
| Positional method/configuration | Position-dependent scoring or embeddings | Long-context task performance and cache-position correctness |
| Dense versus restricted visibility | Reachability between positions | Information-flow analysis and task evaluation |
| Fused attention backend | Intermediate storage and execution | Numerical agreement, actual dispatch, workload-specific performance |
| Quantized weights or cache | Storage and numeric representation | End-to-end quality and latency at the chosen precision |

The generalization boundary is important: adding compute capacity, allowing longer input tensors, or changing a kernel does not itself demonstrate improved task performance. A result in one paper is evidence for its tested model/data/hardware regime; transfer to a new regime is a hypothesis to evaluate.

## 13. Implementation verification and design review

These are **recommended checks derived from the contracts above**, not claims that a cited paper prescribes this checklist.

### 13.1 Checks that catch plausible-looking wrong outputs

1. **Causality:** change a suffix and verify that earlier logits do not change, with dropout disabled and tolerance appropriate to the backend.
2. **Cached/full-forward equivalence:** compare logits from a full causal pass with stepwise cached inference on the same tokens, positions, weights, and masks. Include multiple prefix lengths.
3. **Chunked prefill equivalence:** compare one full prefill with several chunk partitions, including a one-token last chunk. This exposes cache-offset mask errors.
4. **Padding invariance:** compare valid-token outputs for an example alone and in padded batches, preserving its logical position IDs. Compare cached paths separately.
5. **Packed isolation:** alter one independently packed document and verify another document's scored outputs remain unchanged under the isolation contract.
6. **Head mapping:** compare GQA against a small reference that explicitly repeats each KV head for its query group. Keep physical repetition out of the performance conclusion.
7. **Loss alignment:** inspect an explicit short example, including start/end tokens, ignored labels, and document boundaries. Verify the denominator as well as labels.
8. **Backend and gradients:** compare an optimized kernel with a small higher-precision reference for outputs and, when training is in scope, gradients. Use tolerances; do not assume bitwise equality.

A full-forward comparison is a strong cache/mask check but can miss a bug shared by both paths. Independent small references and behavioral invariants provide complementary evidence.

### 13.2 Questions a principal-level review should resolve

- What exact architecture/configuration and tokenizer define the checkpoint? Include head counts, widths, norm settings, position rules, tying, and bias choices.
- Which information-flow constraints define correctness? Are there padding, independent documents, windows, or cross-attention sources?
- What are the separate prefill and decode workloads? Specify prompt length, generation length, concurrency, and latency percentiles.
- Does the capacity budget include weights, growing KV, temporary buffers, replication, and allocation overhead?
- What does profiling show about compute, memory traffic, communication, and scheduling at representative shapes?
- Which claims are published findings, which are algebraic estimates, and which are measurements on this deployment?
- How are quality and numerical fidelity evaluated after an architectural or precision change?

An architecture description is useful at this level when it connects the computation graph to failure modes, capacity, and decisions. Equations establish what should happen; implementation tests and representative measurements establish what actually happens in the selected system.

## 14. Primary sources

Section references above refer to these papers or official implementation/documentation pages. Paper versions are pinned where specified. The GPT-2 `master` source URL is mutable; claims here describe the code reviewed on the date at the top, not a permanent commit pin.

| Source | What this guide uses it for |
|---|---|
| [Vaswani et al. (2017), Attention Is All You Need, v7][original] | Original architecture, scaling motivation, self/cross-attention, sinusoidal positions |
| [Devlin et al. (2019), BERT][bert] | Bidirectional encoder and masked-token objective |
| [Brown et al. (2020), Language Models are Few-Shot Learners, v4][gpt3] | Autoregressive GPT example |
| [Raffel et al. (2020), T5][t5] | Encoder-decoder setup and span-corruption objective |
| [GPT-2 released implementation, `src/model.py`][gpt2-code] | Concrete block, embedding, normalization, and tied output graph |
| [PyTorch 2.9, Embedding][pytorch-embedding] | Embedding lookup semantics |
| [PyTorch 2.9, MultiheadAttention][pytorch-mha] | Head composition, padding and boolean-mask contracts |
| [PyTorch 2.9, functional scaled dot-product attention][pytorch-sdpa] | Operator, mask alignment, dropout, and numerical/backend behavior |
| [Shazeer (2020), GLU Variants Improve Transformer, v1][glu] | Dense/gated FFN definitions and matched-budget study |
| [Xiong et al. (2020), On Layer Normalization in the Transformer Architecture, v2][preln] | Pre-LN/Post-LN initialization and optimization analysis |
| [PyTorch 2.9, LayerNorm][pytorch-ln] | Feature-axis statistics and train/eval semantics |
| [Zhang and Sennrich (2019), Root Mean Square Layer Normalization, v1][rmsnorm] | RMSNorm mechanism |
| [Touvron et al. (2023), LLaMA, v1][llama] | Documented combination of pre-normalization, RMSNorm, SwiGLU, and RoPE |
| [Su et al., RoFormer, v5][rope] | Rotary construction and relative-displacement identity |
| [Chen et al. (2016), Training Deep Nets with Sublinear Memory Cost][checkpoint] | Recomputation for training-memory reduction |
| [Hoffmann et al. (2022), Training Compute-Optimal Large Language Models][chinchilla] | Scoped parameter/data compute tradeoff |
| [Transformers 4.57.1, Caching][hf-cache] | Per-layer KV reuse, generation, and position handling |
| [Ainslie et al. (2023), GQA, v3][gqa] | Head sharing, checkpoint adaptation, and inference tradeoffs |
| [Kwon et al. (2023), PagedAttention, v1][paged] | Logical/physical KV storage and sharing |
| [Dao et al. (2022), FlashAttention, v2][flash] | Exact tiled attention, IO reduction, and backward recomputation |
| [Shoeybi et al., Megatron-LM, v4][megatron] | Column/row partitioning of MLP and attention |
| [Narayanan et al. (2021), Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM][megatron-scale] | Interaction of tensor, pipeline, and data parallelism |
| [Fedus et al., Switch Transformers, v3][switch] | Sparse expert routing, capacity, and load balancing |

[original]: https://arxiv.org/html/1706.03762v7
[bert]: https://aclanthology.org/N19-1423.pdf
[gpt3]: https://arxiv.org/html/2005.14165v4
[t5]: https://www.jmlr.org/papers/volume21/20-074/20-074.pdf
[gpt2-code]: https://raw.githubusercontent.com/openai/gpt-2/master/src/model.py
[pytorch-embedding]: https://docs.pytorch.org/docs/2.9/generated/torch.nn.Embedding.html
[pytorch-mha]: https://docs.pytorch.org/docs/2.9/generated/torch.nn.MultiheadAttention.html
[pytorch-sdpa]: https://docs.pytorch.org/docs/2.9/generated/torch.nn.functional.scaled_dot_product_attention.html
[glu]: https://arxiv.org/html/2002.05202v1
[preln]: https://arxiv.org/html/2002.04745v2
[pytorch-ln]: https://docs.pytorch.org/docs/2.9/generated/torch.nn.LayerNorm.html
[rmsnorm]: https://arxiv.org/html/1910.07467v1
[llama]: https://arxiv.org/html/2302.13971v1
[rope]: https://arxiv.org/html/2104.09864v5
[checkpoint]: https://arxiv.org/abs/1604.06174
[chinchilla]: https://arxiv.org/abs/2203.15556
[hf-cache]: https://huggingface.co/docs/transformers/v4.57.1/cache_explanation
[gqa]: https://arxiv.org/html/2305.13245v3
[paged]: https://arxiv.org/html/2309.06180v1
[flash]: https://arxiv.org/html/2205.14135v2
[megatron]: https://arxiv.org/html/1909.08053v4
[megatron-scale]: https://arxiv.org/abs/2104.04473
[switch]: https://arxiv.org/html/2101.03961v3
