# 02 — Core Building Blocks

> The API primitives every LLM application composes, with the engineering
> properties that actually matter at scale. Examples use the Anthropic Messages
> API (Python SDK); the concepts map to any provider, but the numbers and
> parameter names here are Claude-specific and sourced from Anthropic's docs
> (snapshot noted where relevant).

## 1. The Messages API mental model

Everything goes through one endpoint: `POST /v1/messages`. Tools, structured
outputs, thinking, and caching are all *features of this single endpoint*, not
separate APIs. Supporting endpoints (batches, files, token counting, models)
feed into it.

```python
import anthropic
client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from env

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    system="You are a precise technical assistant.",
    messages=[{"role": "user", "content": "..."}],
)
for block in response.content:
    if block.type == "text":
        print(block.text)
```

Non-obvious properties:

- **Stateless.** Multi-turn = resend full history every request. Conversation
  "memory" is an application-layer concern (and the reason prompt caching exists).
- **Content is a list of typed blocks**, not a string: `text`, `thinking`,
  `tool_use`, `tool_result`, `document`, `image`, plus server-tool result types.
  Code that reads `response.content[0].text` unconditionally is a latent bug —
  a thinking block or tool_use block can be first. Always branch on `block.type`.
- **`stop_reason` is part of the contract.** `end_turn`, `max_tokens` (output cap
  hit — the response is truncated), `tool_use` (execute and continue),
  `pause_turn` (server-side tool loop paused; resend to resume), `refusal`
  (safety decline — check `stop_details`), `model_context_window_exceeded`.
  Robust apps branch on all of these; see doc 05.
- **Current context windows** (snapshot mid-2026): 1M tokens on Claude
  Opus/Sonnet tiers, 200K on Haiku 4.5; max output up to 128K tokens (64K for
  Haiku) — but outputs above ~16K require streaming to avoid HTTP timeouts.

## 2. System prompts

The system prompt is the highest-authority instruction channel and — critically —
the *front of the cacheable prefix*. Two engineering consequences:

1. **Keep it frozen.** Interpolating timestamps, user names, or feature flags into
   the system prompt invalidates the prompt cache for the entire conversation on
   every request. Dynamic context belongs later in `messages`.
2. **Instruction style is model-generation-sensitive.** Recent Claude models
   follow instructions more literally than older ones; prompts written to
   *overcome* older models' reluctance ("CRITICAL: YOU MUST use this tool")
   now overtrigger. Anthropic's migration guidance is to state conditions plainly
   ("Use this tool when...") and re-baseline prompts on each model upgrade.

Structure that works for large system prompts: identity/role → hard constraints →
tool-usage guidance → output format → few-shot examples. Put the most stable
content first (cache prefix), the most volatile last.

## 3. Tool use (function calling)

You declare tools as JSON-schema'd functions; the model returns `tool_use` blocks
naming a tool and arguments; you execute and return `tool_result` blocks; repeat.

```python
tools = [{
    "name": "get_order",
    "description": "Look up an order by ID. Call this whenever the user "
                   "references a specific order number.",
    "input_schema": {
        "type": "object",
        "properties": {"order_id": {"type": "string"}},
        "required": ["order_id"],
        "additionalProperties": False,
    },
    "strict": True,   # guarantees input validates against the schema exactly
}]
```

Facts that matter in production (from the tool-use docs):

- **Descriptions are load-bearing.** The model decides *whether* to call from the
  description. Be prescriptive about when to call, not just what it does — on
  recent Claude models (which reach for tools more conservatively), trigger
  conditions in the description measurably lift should-call rate.
- **Parallel tool use is on by default.** One assistant turn may contain multiple
  `tool_use` blocks. Execute them concurrently and return *all* results in a
  *single* user message — splitting results across messages trains the model to
  stop parallelizing.
- **`tool_choice`** controls the mode: `auto` (default), `any` (must use some
  tool), `{type:"tool", name}` (forced — the classification trick), `none`.
- **Errors are results.** A failed tool returns
  `{"type":"tool_result", "tool_use_id":..., "content":"<error>", "is_error": true}`
  — never drop it; the model recovers or adapts.
- **Server-side tools** (web search, web fetch, code execution) execute on the
  provider's infrastructure — declared in `tools` but no client loop. Their
  errors also arrive as result blocks with error objects, not exceptions.
- **Tool runner vs manual loop.** SDKs ship a tool runner that owns the
  call→execute→resend loop, with per-turn hooks for human approval, result
  modification, and retries. "I need control" is rarely a reason to hand-roll
  the loop; do so only for custom transports or control flow the hooks can't
  express.

### Scaling the tool set

Two documented mechanisms avoid stuffing hundreds of schemas into context:

- **Tool search** — mark tools `defer_loading: true` and expose a search tool;
  the model discovers and loads only relevant schemas. Discovered schemas are
  *appended*, preserving the prompt cache.
- **Programmatic tool calling** — the model writes a script (run in the code-
  execution sandbox) that calls your tools as functions; intermediate results
  stay in the sandbox and never hit the context window. Use when chaining many
  calls or when intermediate payloads are large.

## 4. Structured outputs

Two distinct features, often confused:

1. **Output format** (`output_config.format` with a JSON schema, or SDK
   `messages.parse()` with a Pydantic/Zod model): constrains the *response body*
   to schema-valid JSON. Use for extraction, classification, any machine-consumed
   output.
2. **Strict tool use** (`strict: true` on a tool definition): guarantees
   `tool_use.input` validates against the schema exactly. Requires
   `additionalProperties: false` and `required`.

```python
from pydantic import BaseModel

class Contact(BaseModel):
    name: str
    email: str
    plan: str

resp = client.messages.parse(
    model="claude-opus-4-8", max_tokens=1024,
    messages=[{"role": "user", "content": raw_text}],
    output_format=Contact,
)
contact = resp.parsed_output  # validated Contact instance
```

Documented limitations worth designing around: no recursive schemas, no numeric
range constraints (`minimum`/`maximum`) or string length constraints enforced
server-side (Python/TS SDKs strip these and validate client-side); new schemas
pay a one-time compilation latency (then cached ~24h); incompatible with
citations; `stop_reason: "max_tokens"` can still truncate mid-JSON — check it.

Historical note: assistant-message **prefilling** (seeding the reply with
`{"role":"assistant","content":"{"}`) was the old way to force formats; it
returns a 400 on current Claude models. Structured outputs is the replacement.

## 5. Streaming

Server-sent events: `message_start` → per-block `content_block_start/delta/stop`
→ `message_delta` (carries `stop_reason` + usage) → `message_stop`.

- **Stream by default** for anything user-facing or with large `max_tokens`
  (>~16K non-streaming risks SDK HTTP timeouts; SDKs will refuse or warn).
- SDK helpers (`client.messages.stream(...)` + `get_final_message()` /
  `finalMessage()`) give you the streaming transport *and* the fully accumulated
  message — you rarely need to hand-process events unless rendering token-by-token.
- Design UIs for the **thinking gap**: with reasoning enabled and thinking display
  omitted, the stream can be silent for a while before visible text; either
  surface summarized thinking or show an activity indicator.

## 6. Extended thinking and effort

Current Claude models use **adaptive thinking** (`thinking: {type:"adaptive"}`)
— the model decides when/how much to reason — plus an **effort** dial
(`output_config: {effort: "low"|"medium"|"high"|"xhigh"|"max"}`) controlling
depth and overall token spend. (The older fixed `budget_tokens` thinking budget
is removed on current models.)

Engineering guidance from the docs:

- Effort is the primary intelligence↔latency↔cost control. `high` is the default;
  `xhigh` for the hardest coding/agentic work; `low`/`medium` for routine or
  latency-sensitive routes and subagents. Sweep it on your own evals — the
  relationship isn't monotonic in cost (higher effort up front can *reduce* total
  cost on agentic tasks via fewer, better tool calls).
- Thinking blocks must be passed back **unchanged** in multi-turn tool loops.
- Thinking is billed as output tokens regardless of whether you display it.

## 7. Prompt caching — the economics of context

The single most important cost lever. Mechanics (Anthropic-specific):

- **Prefix match.** The cache key is the exact bytes of the rendered prompt up to
  each `cache_control` breakpoint. Render order is `tools` → `system` →
  `messages`. One changed byte at position N invalidates everything ≥ N.
- **Pricing:** cache reads ≈ **0.1×** base input price; writes cost **1.25×**
  (5-minute TTL) or **2×** (1-hour TTL). Break-even at 2 requests (5m) / 3
  requests (1h).
- Up to **4 breakpoints** per request; minimum cacheable prefix is model-dependent
  (roughly 1–4K tokens) — shorter prefixes silently don't cache.
- Caches are **model-scoped**: switching models mid-conversation is a full cold
  start.

Placement patterns: breakpoint on the last system block (caches tools+system);
in multi-turn chat, breakpoint on the last content block of the newest turn so
hits accrue incrementally; for shared-prefix/varying-suffix workloads, breakpoint
at the end of the *shared* portion only.

Silent invalidators to grep for in any prompt-assembly path: `datetime.now()` in
the system prompt, UUIDs early in content, non-deterministic JSON serialization
(unsorted keys, set iteration), per-user tool sets, conditional system sections.
Verify with `usage.cache_read_input_tokens` — zero across identical-prefix
requests means an invalidator is at work; diff the rendered bytes.

## 8. Batch processing

`POST /v1/messages/batches`: async processing at **50% of standard prices**, all
Messages features supported (tools, vision, caching). Limits: 100K requests or
256 MB per batch; most complete within an hour (24h ceiling); results retained
29 days; results stream back **in arbitrary order** — key by `custom_id`, never
by position.

Default to batches for: dataset labeling, backfills, eval runs, embedding-corpus
preprocessing (e.g., contextual retrieval), nightly report generation — anything
that doesn't need an answer in seconds.

## 9. Multimodal input and files

- **Images**: base64 or URL source blocks in user content; current Opus/Sonnet
  models support high-resolution input (long edge up to 2576px) with coordinates
  mapping 1:1 to pixels — but full-res images cost up to ~3× the image tokens of
  older caps, so downsample when fidelity isn't needed.
- **PDFs**: `document` blocks (base64 or Files API reference); enable
  `citations: {enabled: true}` per document block to get span-level citations
  back — the grounded-answer pattern for doc Q&A.
- **Files API** (beta): upload once, reference by `file_id` across many requests
  — avoids re-uploading a 50-page PDF per question.

## 10. Token counting and context budgeting

`POST /v1/messages/count_tokens` gives exact, model-specific counts — use it, not
tiktoken (OpenAI's tokenizer; undercounts Claude tokens by ~15–20% on typical
text, worse on code). Tokenizers also change *between model generations* (e.g.
the Opus 4.7-era tokenizer counts ~1–1.35× the older one), so never carry
measured budgets across a model migration without re-baselining.

Budget rule of thumb for context assembly: reserve headroom for output
(`max_tokens`) plus thinking; fill retrieval/context by priority order with a
hard cap; **never silently truncate** user-provided content — surface the
overflow and choose chunking/summarization deliberately.

---

## Sources

- Anthropic platform docs: Messages API, tool use overview, structured outputs,
  streaming, prompt caching, batch processing, PDF support & citations, token
  counting, adaptive thinking & effort.
- Anthropic model migration guide (prefill removal, tokenizer changes,
  instruction-following shifts across model generations).
- Pricing/economics figures (cache multipliers, batch discount) per Anthropic
  pricing documentation, snapshot mid-2026 — verify before cost modeling.

---

## 11. Provider-neutral request architecture

Put provider SDKs behind an internal contract containing ordered content, tool
schemas, output schema, capability requirements, deadline, token/cost budget,
tenant/trace IDs, and a normalized result: text, structured data, tool requests,
finish reason, usage, safety outcome, and provider request ID. Keep a portable core
plus explicit capability flags; do not erase meaningful provider differences.

Boundary invariants:

1. Validate tool arguments in application code even when the API enforces a schema;
   schema validity is neither authorization nor business validity.
2. Separate model **tool selection**, policy **tool admission**, and trusted-code
   **tool execution**.
3. Propagate deadlines and cancellation into tools.
4. Store large artifacts outside conversation history and pass bounded extracts or
   stable references. Context windows are not databases.
5. Version prompts, schemas, retrieval configuration, model snapshots, and policies
   as one deployable configuration.

## 12. Prompt, context, and model adaptation

Treat a production prompt as software configuration: owner, version, changelog,
eval suite, and rollback. Split stable policy from task data, delimit untrusted
content, and define the output contract. Prompts must never enforce permissions.

Budget context explicitly among instructions, dialogue, evidence, tool results, and
output. Truncate by semantic priority and record omissions. Improve systems in the
empirically tested order: prompt/context → retrieval/tools → routing/stronger model
→ fine-tuning. Fine-tuning fits stable behavior or specialization; it is not a good
store for frequently changing facts.

### Cross-provider sources

- [OpenAI, *Function calling*](https://platform.openai.com/docs/guides/function-calling) and [*Structured Outputs*](https://platform.openai.com/docs/guides/structured-outputs).
- [Google Cloud, *Generative AI architecture*](https://cloud.google.com/architecture/ai-ml/generative-ai).
- [AWS, *Generative AI Lens: performance efficiency*](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/performance-efficiency.html).
