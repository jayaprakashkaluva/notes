# 05 — Production Best Practices

> Evals, cost, reliability, security, observability. The parts that don't demo
> well and determine whether the thing survives contact with users.

## 1. Evaluation — the actual foundation

The uncomfortable truth of LLM engineering: **you cannot improve what you don't
measure, and vibes don't survive a prompt change.** Every serious guide
(Anthropic's included) puts empirical evals ahead of prompt cleverness.

### 1.1 Building the eval set

- Source cases from **real traffic** first (logged inputs + failures), synthetic
  generation second (LLM-generated cases, human-spot-checked), designed edge
  cases third.
- 20–50 well-chosen cases catch most regressions; hundreds are needed for
  measuring small deltas. Start small and grow from production failures — every
  incident becomes a test case (same discipline as regression tests).
- Version the eval set with the prompts. A prompt PR without eval results is an
  unreviewable PR.

### 1.2 Graders, in order of preference

1. **Code-based** (exact match, schema validation, contains/regex, tests pass,
   numeric tolerance) — cheap, deterministic, use whenever the task allows.
2. **LLM-as-judge** with a rubric — for open-ended outputs. Known failure modes
   to engineer around: position bias (swap A/B order and average), verbosity
   bias, self-preference; mitigate with explicit per-criterion rubrics
   (independently gradeable statements, not "is this good?"), forced
   choice-with-reasoning, and periodic human calibration of the judge itself.
3. **Human review** — expensive; reserve for calibrating the judge and for
   high-stakes samples.

### 1.3 What to measure per archetype

| Archetype | Core metrics |
|---|---|
| Classification/extraction | accuracy/F1 per class, schema-validity rate |
| RAG | retrieval recall@K *separately from* generation faithfulness + citation validity (doc 03 §8) |
| Agents | task success rate, steps/tokens-to-success, unsafe-action rate, regression on a frozen task suite |
| Chat | rubric scores (helpfulness, tone), refusal correctness both directions |

Run evals: on every prompt/model/parameter change (CI), and continuously against
production samples (drift detection). Model upgrades are *migrations* with eval
gates, not dependency bumps — instruction-following behavior shifts between
generations are documented and material (see the migration guidance in doc 02 §2).

## 2. Cost engineering

Levers in descending order of typical impact:

1. **Prompt caching** (doc 02 §7). Reads ≈ 0.1× input price. For chat and agents
   this is routinely a 5–10× input-cost reduction. Audit `cache_read_input_tokens`
   in production — a zero means a silent invalidator, which is a bug with a
   dollar cost.
2. **Model routing.** Price spread across a current model family is large
   (mid-2026 snapshot of Claude list prices per MTok in/out: Haiku 4.5 $1/$5,
   Sonnet 5 $3/$15, Opus 4.8 $5/$25, Fable 5 $10/$50). Route by measured
   difficulty — with an eval per route proving the cheap model clears the bar.
3. **Batch API** — flat 50% off everything that isn't latency-sensitive.
4. **Effort/thinking tuning** — sweep `effort` per route; lower effort means
   fewer/more-consolidated tool calls and terser output. Not monotonic: on
   agentic work, higher effort up front can reduce total turns and net cost.
5. **Context diet** — retrieved-context budgets, context editing for agents,
   image downsampling when high-res fidelity isn't needed, `max_tokens`
   discipline per route.

Track **cost per task** (not per request) as the product metric — an agent that
spends 3× tokens but succeeds in one attempt beats a cheap one that retries.

## 3. Latency engineering

- **Stream everything user-facing**; time-to-first-token is the perceived-latency
  metric. Account for the thinking gap (doc 02 §5).
- **Parallelize** independent calls (sectioning) and tool executions.
- **Route latency-critical paths to small models** with effort `low`.
- **Cache-warm**: a cold cache write adds full-price prefill latency to the first
  request; the Claude API supports `max_tokens: 0` pre-warm requests at startup
  or before scheduled traffic windows. Only worth it when the prefix is large,
  first-request latency is user-visible, and traffic has gaps > TTL.
- Budget for **long tails**: current frontier models on hard agentic tasks can
  legitimately run many minutes in a single request — set client timeouts
  accordingly (SDK default is 10 min), and design UX for async check-ins rather
  than a blocking spinner.

## 4. Reliability

### 4.1 Errors and retries

The Claude SDKs auto-retry 408/409/429/5xx and connection errors with
exponential backoff (default 2 retries, configurable). On top of that:

- Catch **typed exceptions, most-specific first**, separating retryable
  (429, ≥500, network) from non-retryable (400/401/403/404) — a 400 retried in
  a loop is a bug, not resilience.
- On 429, honor `retry-after`; on 529 (overloaded), back off and consider
  failing over to another model tier.
- **Idempotency**: agents retrying a turn must not re-execute side-effecting
  tools; key tool executions by `tool_use_id`.

### 4.2 Stop-reason handling (the silent-failure catalog)

Every response handler must branch on `stop_reason`:

- `max_tokens` → output truncated; raise the cap or stream, never parse the
  fragment as complete (especially JSON).
- `pause_turn` → resend to resume; SDK tool runners do **not** auto-resume this,
  so unhandled it becomes a silently truncated answer.
- `refusal` → surface, don't blind-retry the same prompt; on models with safety
  classifiers, wire a fallback model path for false positives (the API supports
  server-side `fallbacks` for this on Claude).
- `model_context_window_exceeded` → compact or split; distinct from `max_tokens`.

### 4.3 Output validation

Trust nothing structurally: schema-validate even "guaranteed" structured outputs
at the boundary (truncation and refusals can still yield non-conforming output);
parse tool inputs with a JSON parser (never string-match serialized JSON — escaping
varies across model versions); clamp/validate model-supplied identifiers before
using them (IDs, paths, amounts).

### 4.4 Graceful degradation

Design the ladder before the incident: primary model → fallback model →
cached/deterministic response → honest error. Multi-provider failover is an
option but costs prompt-portability (prompts are model-tuned; a failover target
you never eval against is not a real fallback).

## 5. Security

Frame: **the model is an untrusted interpreter running attacker-influenceable
input with your credentials.** The OWASP LLM Top 10 headliners, with concrete
mitigations:

### 5.1 Prompt injection (the unsolved one)

Any text the model reads — user input, retrieved docs, web pages, tool results,
emails — can contain instructions. There is no reliable prompt-level defense;
mitigation is *architectural*:

- **Least-privilege tools per route**: the summarizer of untrusted web content
  gets no mutating tools. Capability follows trust level of the input.
- **Human gates on irreversible actions** (send, delete, pay, push) when
  untrusted content is in context.
- **Privileged instruction channels**: keep operator instructions in the system
  role (spoofable-free channel), not interpolated into user-visible text.
- Treat "the model will refuse injected instructions" as defense-in-depth, never
  as the control.

### 5.2 Tool/code execution

- Model-generated shell commands and code run **sandboxed**: container/VM,
  non-root, restricted egress, resource limits, allowlisted executables (a
  blocklist is insufficient — documented explicitly in Anthropic's bash-tool
  guidance).
- **Path confinement**: resolve every model-supplied path to canonical form and
  verify it stays under the project root (reject `..`, symlink escapes,
  encoded traversal) before any file operation.
- **Confused deputy**: tools execute with the *end user's* authorization, not a
  broad service account — enforced server-side in the tool implementation, not
  by prompt.

### 5.3 Secrets and data

- Secrets never enter the prompt or the sandbox: conversation history is stored,
  logged, replayed, and compacted — a key pasted into a system prompt persists.
  Inject credentials outside the model path (egress-proxy substitution, host-side
  tool execution — the patterns Anthropic's managed platform productizes as
  vaults).
- Memory/persistence surfaces need the same discipline: no credentials in memory
  files (they replay into every future session), redaction paths for PII, and
  per-user isolation of memory directories.
- Log redaction for prompts/outputs containing PII; retention policy aligned
  with your data-processing obligations.

## 6. Observability

Minimum viable telemetry for any LLM product:

- **Full request/response traces** (prompts, completions, tool calls+results,
  `stop_reason`, latency, `usage` including cache fields), sampled at 100% for
  agents (you debug trajectories, not stack traces) and heavily for high-QPS
  single calls. Include the provider `request-id` for support escalation.
- **Dashboards on**: token cost per task/route, cache hit rate, p50/p95/p99
  latency and TTFT, error/refusal/truncation rates by `stop_reason`, eval scores
  over time.
- **Alert on**: cache hit rate drops (someone broke the prefix), refusal-rate
  spikes (model change or attack), cost-per-task regressions, `max_tokens`
  truncation rates.
- **Version everything**: prompts, tool schemas, model IDs, eval sets — a
  production trace should be replayable against the exact configuration that
  produced it.

## 7. Shipping discipline — a checklist

- [ ] Eval set exists, versioned, wired into CI; baseline recorded before launch
- [ ] `stop_reason` fully branched; truncated output can't be parsed as complete
- [ ] Retry/backoff configured; non-retryable errors not retried
- [ ] Prompt cache verified live (`cache_read_input_tokens > 0` on steady state)
- [ ] Cost per task and latency budgets defined with alerts
- [ ] Tool execution sandboxed; irreversible actions gated; paths confined
- [ ] Untrusted-content routes run with least-privilege tool sets
- [ ] Secrets provably absent from prompts, logs, and memory surfaces
- [ ] Traces capture full trajectories with usage and request IDs
- [ ] Model upgrades treated as migrations: eval gate + prompt re-baseline
- [ ] Degradation ladder implemented and actually tested

---

## Sources

- Anthropic platform docs: error codes & SDK retry behavior, stop reasons,
  prompt caching (economics, verification, pre-warming), batch pricing, bash/
  text-editor tool security guidance (sandboxing, allowlists, path confinement),
  memory-tool security notes, structured-outputs caveats (truncation, refusals),
  server-side fallbacks.
- Anthropic pricing page — model list prices (mid-2026 snapshot; verify live).
- OWASP *Top 10 for LLM Applications* — prompt injection, insecure output
  handling, excessive agency taxonomy.
- Anthropic model migration guides — instruction-following shifts across
  generations, long-turn latency planning, eval-gated upgrades.

---

## 8. Production reference architecture

Keep deterministic responsibilities outside the model boundary:

1. **Edge/API:** authentication, tenant resolution, quotas, size limits.
2. **Orchestrator:** routing, deadlines, state machine, budgets, provider adapter.
3. **Context service:** dialogue state, retrieval, memory, token budget, provenance.
4. **Policy/tool gateway:** authorization, validation, approvals, idempotency, egress
   controls, and audit.
5. **Model gateway:** allowlists, regional/data routing, rate limits, retries, circuit
   breaking, normalized telemetry, and configuration versions.
6. **Evaluation plane:** traces, quality sampling, red-team cases, cost allocation,
   and release gates.

The model proposes content or actions. Trusted services authenticate, authorize,
enforce invariants, and commit side effects.

## 9. SLOs and failure engineering

Define separate SLOs for availability, latency, and quality. HTTP 200 can still be
factually unsupported or contain a wrong tool call. Track task success, groundedness,
correct abstention, tool-call correctness, policy violations, p95 latency, and cost
per successful task.

Use deadline-aware retries with exponential backoff and jitter. Retry only known-safe
operations; replaying a turn after a side effect requires idempotency/reconciliation.
Use per-route/tenant bulkheads, circuit breakers, bounded queues, admission control,
and load shedding.

## 10. Evaluation, release, and threat governance

Maintain capability, regression, adversarial/safety, and operational-failure suites.
Segment results by language, use case, input length, tenant class, and risk. Release
prompts, models, retrieval, and tool schemas through shadow traffic or canaries;
compare quality, latency, cost, and safety, with automatic rollback criteria and
emergency model/tool disable switches.

Threat-model sensitive disclosure, insecure output handling, supply-chain compromise,
data/model poisoning, prompt injection, token/tool-amplification denial of service,
excessive agency, and vector/embedding weaknesses. Map each to preventive controls,
detection, response playbook, and owner. Red-team the composed application, not only
the base model.

### Cross-provider and standards sources

- [NIST, *AI RMF 1.0*](https://doi.org/10.6028/NIST.AI.100-1) and [*Generative AI Profile*](https://doi.org/10.6028/NIST.AI.600-1).
- [OWASP, *Top 10 for LLM Applications*](https://genai.owasp.org/llm-top-10/).
- [AWS, *Generative AI Lens*](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/generative-ai-lens.html).
- [Microsoft Foundry, *Observability in generative AI*](https://learn.microsoft.com/en-us/azure/foundry/concepts/observability).
- [Microsoft, *Planning red teaming for LLM applications*](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/red-teaming).
