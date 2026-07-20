# 04 — Agents

> An agent is a model in a loop with tools and feedback from an environment.
> The model supplies intelligence; everything else — the tool surface, the
> context strategy, the harness — is your system design, and it is where agent
> quality is actually won or lost.

## 1. The loop

The irreducible core, in Messages-API terms:

```python
messages = [{"role": "user", "content": task}]
while True:
    resp = client.messages.create(
        model="claude-opus-4-8", max_tokens=16000,
        tools=tools, messages=messages,
    )
    if resp.stop_reason == "end_turn":
        break
    if resp.stop_reason == "pause_turn":          # server-tool loop paused
        messages.append({"role": "assistant", "content": resp.content})
        continue
    messages.append({"role": "assistant", "content": resp.content})
    results = [
        {"type": "tool_result", "tool_use_id": b.id,
         "content": run_tool(b.name, b.input)}
        for b in resp.content if b.type == "tool_use"
    ]
    messages.append({"role": "user", "content": results})
```

In practice, use the SDK **tool runner** instead of hand-writing this — it owns
the loop and still exposes per-turn hooks (inspect the pending tool call before
it executes, gate on human approval, modify results, bound iterations with
`max_iterations`). Reserve the manual loop for control flow the hooks can't
express. Either way, the properties you must guarantee yourself:

- **Termination bounds** — max iterations *and* a token/cost budget. (Claude
  additionally supports a model-visible `task_budget` so the model paces itself
  and wraps up gracefully rather than being cut off.)
- **Every `tool_use` gets a `tool_result`** — including failures
  (`is_error: true`); an unmatched ID is a hard API error, a dropped failure is
  a silent hallucination invitation.
- **Full content blocks round-trip** — append `resp.content` verbatim (thinking
  blocks included, unmodified), not just extracted text.

## 2. Feedback is the product

The defining property of agents that work: **a verifiable environment**. Coding
agents work because tests and compilers provide dense, honest error signal every
iteration. Computer-use agents get screenshots. Support agents get API responses.
When you design an agent, your first question isn't "what tools?" but "how does
the model find out it was wrong *before the user does*?"

Concrete forms this takes:

- Tools that return rich errors (linter output, stack traces, HTTP bodies) rather
  than boolean success.
- An explicit verification step in the task spec ("run the test suite; don't
  report done until it passes").
- **Fresh-context verifier subagents** — Anthropic's long-running-agent guidance
  notes that a separate verifier with clean context tends to outperform
  self-critique by the same context that produced the work.
- Rubric-graded outer loops (the evaluator-optimizer pattern; productized as
  "Outcomes" in Managed Agents: define done-criteria as a gradeable rubric, and
  the harness runs iterate → grade → revise).

## 3. Designing the tool surface (the ACI)

Anthropic's term is the **agent-computer interface** — and their stated position
is that it deserves as much design effort as a human UI. Grounded principles:

**Give the model room to think before it commits.** Tool formats that force the
model to write a perfect diff in one shot, or escape thousands of lines inside
JSON strings, fail more. Prefer formats close to what the model has seen in the
wild (markdown, plain code) and with low bookkeeping overhead (no line-counting).

**Write tool docs like you'd write for a junior engineer.** Example usage, edge
cases, input format, and *boundaries against other tools*. If a human would be
confused when to use `search_tickets` vs `list_tickets`, the model will be too.
Anthropic reports one of their own engineers' fixes during SWE-bench work was
simply changing a tool spec (requiring absolute file paths) — it eliminated a
whole error class.

**Poka-yoke: make errors structurally impossible**, not just documented.
Enums instead of free strings, absolute paths required, separate read-only vs
mutating tools, server-side validation that rejects with a corrective message.

**Bash-vs-dedicated-tools decision** (from Anthropic's agent-design doc): a bash
tool gives maximal breadth but hands your harness an opaque command string.
Promote an action to a dedicated tool when you need to:

- **gate it** (a `send_email` tool is easy to require approval for;
  `bash -c "curl -X POST ..."` is not),
- **enforce invariants** (an `edit` tool can reject writes to files changed since
  last read; bash can't),
- **render it** (custom UI for questions/options),
- **schedule it** (read-only tools can be marked parallel-safe; bash can't be
  distinguished from `git push`).

Rule of thumb: start with bash for breadth; promote when you need to gate,
render, audit, or parallelize.

**Keep the set small and focused.** Overlapping tools and long tail of
rarely-used tools degrade selection accuracy; use tool search / deferred loading
past a few dozen tools.

## 4. Context management over long horizons

An agent that runs for hours faces context-window exhaustion and context *rot*
(stale tool results diluting attention). The documented toolkit, in order of
escalation:

| Mechanism | What it does | When |
|---|---|---|
| **Context editing** | Clears old tool results / thinking blocks by threshold — prunes, doesn't summarize | Stale intermediates accumulating (file dumps, page snapshots) |
| **Compaction** | Server-side summarization of earlier history into a compaction block when nearing the limit | Conversation genuinely exceeds the window; the block must be round-tripped verbatim |
| **Memory (files)** | Model reads/writes a persistent directory (notes, learnings, state) across sessions | Cross-session persistence; also *within* long sessions as a scratchpad |
| **Subagent fan-out** | Burn context in a child; return a summary to the parent | Exploration/reading tasks with high context cost and low summary size |

Practical notes:

- Long-running agents commonly use all of these together.
- Anthropic's model guidance is explicit that recent models perform measurably
  better *when given* a memory surface plus instructions on format ("one lesson
  per file, one-line summary at top; record corrections and confirmed approaches
  with why; update rather than duplicate").
- Compaction/summarization is lossy — design the memory/notes discipline so
  load-bearing facts (decisions, file paths, invariants) live in files, not only
  in the conversation that will be compacted.
- For memory taxonomy (semantic/episodic/procedural, profile vs. collection,
  hot-path vs. background writes) and retrieval scoring over a large memory
  store, see §12.

## 5. Caching strategy for agents

Agent loops resend a growing history every iteration — without caching, cost is
quadratic in turns. The prefix-cache constraints (doc 02) become architecture:

- **Never mutate the system prompt mid-session** — append operator instructions
  as messages instead (Claude supports mid-conversation `role:"system"` messages
  on current Opus for exactly this; older models: a `<system-reminder>` block in
  the user turn).
- **Never add/remove/reorder tools mid-session** — use tool search for dynamic
  discovery (appends, doesn't swap).
- **Don't switch models mid-session** — spawn a subagent on the cheaper model
  instead; caches are model-scoped.
- Watch the **lookback window**: a cache breakpoint searches a bounded number of
  content blocks backward (20 on the Claude API); turns with many tool calls
  need intermediate breakpoints or hits silently stop.

## 6. Orchestration patterns

- **Single agent** — default. Most "multi-agent" designs are a single agent
  whose tool surface was badly factored.
- **Subagents (coordinator-workers)** — for context isolation and parallel
  fan-out (doc 01 §5). Delegation messages must carry full context; results
  return as summaries. One level deep.
- **Long-lived peer agents with async messaging** — Anthropic's recent-model
  guidance notes async communication with long-running subagents outperforms
  spawn-and-block: workers keep their context (cache-warm) across subtasks and
  the orchestrator isn't bottlenecked on the slowest child.
- **Human-in-the-loop as a tool** — model a question-to-user as a *tool call*
  (blocking, rendered as UI) rather than ending the turn with prose questions;
  and gate irreversible tools (approve/deny) at the harness. This is how Claude
  Code models it, and it's the pattern the tool-runner hooks / Managed Agents
  `always_ask` permission policies support natively.

## 7. The harness (everything around the loop)

What separates a demo from a product is almost entirely harness:

- **Permissions**: per-tool policy (auto-allow reads; confirm mutations;
  deny-by-default for destructive ops). Managed Agents exposes this as
  `always_allow`/`always_ask` per tool; in your own harness it's a gate in the
  tool-execution path.
- **Sandboxing**: model-generated commands and code are untrusted output —
  containerize execution, restrict egress, allowlist executables (see doc 05 §
  security).
- **Durability**: persist the event/message log so a crashed run resumes rather
  than restarts; checkpoint before expensive irreversible steps.
- **Steering**: an interrupt channel (stop/redirect mid-run), and message
  queuing so users can pile on follow-ups without racing the loop.
- **Observability**: log every model request/response, tool call+result,
  token usage per turn; make traces reviewable — you debug agents by reading
  trajectories, not stack traces.
- **Prompting for autonomy**: long-horizon guidance for current Claude models —
  give the *full task specification up front* in one well-specified turn, run at
  high effort, state boundaries explicitly (what not to touch), and require
  progress claims to be audited against tool results ("only report work you can
  point to evidence for").

## 8. Build-vs-platform decision

Recap of the four documented approaches (detail in doc 01): manual loop (full
ownership), SDK tool runner (loop supplied, you host), Claude Agent SDK (full
Claude Code harness with built-in file/bash/search tools, you host), Managed
Agents (Anthropic hosts loop + per-session sandbox container, versioned agent
configs, sessions, event streams, scheduled deployments). The decision axis is
*who supplies the harness* vs *who supplies the deployment* — the tool runner
and Agent SDK are harness-only; only the managed offering adds deployment.
Choose managed when you'd otherwise build sessions, sandboxes, schedulers, and
credential vaults yourself; choose self-hosted when tools must touch infra that
can't leave your network (or use its self-hosted-sandbox mode, where the loop
stays managed but tool execution polls outbound from your container).

---

## Sources

- Anthropic, *Building Effective Agents* (Dec 2024) — agent definition, ACI
  guidance (tool formats, poka-yoke, the SWE-bench absolute-paths anecdote),
  simplicity/transparency principles, coding & support agents as proven cases.
- Anthropic agent-design documentation — bash-vs-dedicated-tools criteria,
  context editing vs compaction vs memory, agent caching workarounds, tool
  search, programmatic tool calling.
- Anthropic model-behavior guidance (Claude 4.7+/Fable-era migration docs) —
  verifier subagents, async subagent communication, memory-surface prompting,
  task budgets, full-spec-up-front long-horizon guidance.
- Anthropic Managed Agents documentation — permission policies, outcomes/rubric
  grading, sessions/events, self-hosted sandboxes.

---

## 9. Durable agent execution

Implement an agent as an explicit state machine, not an opaque recursive function.
Persist an append-only run log containing messages, normalized model responses, tool
requests, admission decisions, results, checkpoints, and outcome. Useful states are
`RUNNING`, `WAITING_FOR_TOOL`, `WAITING_FOR_APPROVAL`, `WAITING_FOR_USER`,
`SUCCEEDED`, `FAILED`, `CANCELLED`, and `EXPIRED`.

Make transitions idempotent. Assign an operation key before a side effect, persist
intent, execute, and persist the result. After a crash, reconcile uncertain external
state rather than blindly repeat the action.

Enforce hard budgets outside the model: wall-clock time, model/tool calls, tokens,
cost, repeated identical actions, and consecutive errors. Trusted code—not the
model—decides whether a run is terminal.

Every tool contract should define purpose, preconditions, authorization scope,
schema, timeout, side effects, idempotency, retry class, error taxonomy, sensitivity,
and approval policy. Prefer `issue_refund(order_id, amount, reason)` to unrestricted
HTTP, SQL, or shell access.

### Autonomy levels

| Level | Capability | Control |
|---|---|---|
| 0 | advise | user executes |
| 1 | prepare | preview/diff |
| 2 | reversible low-risk action | policy gate and audit |
| 3 | consequential action | explicit approval/separation of duties |
| 4 | bounded autonomous operation | narrow scope, monitoring, kill switch |

Set autonomy per tool and situation, not once for an entire agent.

## 10. Trajectory evaluation

Grade more than the final answer: tool choice and arguments, ordering, policy
compliance, recovery, redundancy, budget, and required external state. Replay saved
tool results for deterministic regressions and use sandboxed integration tests for
end-to-end behavior.

## 11. Research foundations: ReAct and Toolformer

The tool-use loop in §1 did not appear fully formed in provider APIs; two
papers supply its intellectual grounding, and their empirical findings still
predict failure modes you will see in production agents.

### 11.1 ReAct — interleaving reasoning and acting (Yao et al., ICLR 2023)

ReAct's formal move: augment the agent's action space with a *language* action
— a **thought** — that does not touch the environment and returns no
observation; its only effect is to update the agent's own context to support
future reasoning or acting. Useful thought types the paper catalogs:
decomposing goals into action plans, injecting relevant commonsense knowledge,
extracting the important parts of observations, tracking progress and
transitioning plans, and handling exceptions ("Front Row is not found; I need
to search Front Row (software)"). The trajectory becomes interleaved
thought → action → observation steps — recognizably the modern agent loop
with visible reasoning.

Findings worth carrying into agent design:

- **Reasoning grounds acting, and acting grounds reasoning.** On HotpotQA and
  FEVER (with only a minimal Wikipedia search/lookup API), ReAct beat
  act-only prompting on both tasks — thoughts are what let the model
  synthesize across retrieved evidence rather than flail through actions. In
  the reverse direction, chain-of-thought *without* actions hallucinated
  badly: in the paper's human failure analysis, hallucination was 56% of
  CoT's failures (and 14% of its "successes" rested on hallucinated
  reasoning), versus ~0% hallucination failures for ReAct (6% false-positive
  successes).
- **ReAct has its own signature failure modes.** The structural constraint of
  alternating thought/action reduced reasoning flexibility: 47% of ReAct's
  failures were reasoning errors, including a distinctive loop where the
  model *repetitively regenerates its previous thought and action* — watch
  for exactly this in production trajectories. Another 23% were
  non-informative search results derailing the reasoning with no recovery —
  retrieval quality gates agent quality (cf. doc 03).
- **Combine internal and external knowledge.** The best configurations were
  hybrids with a back-off rule: fall back from ReAct to self-consistency CoT
  when the agent fails to finish within a step budget (7 steps on HotpotQA, 5
  on FEVER — more steps didn't help), and fall back from CoT-SC to ReAct
  when the majority vote is weak (majority answer < n/2 of samples — a
  measurable "the model isn't confident" signal). The modern reading: give
  agents an explicit budget and a defined degradation path, not an unbounded
  loop.
- **Dense vs. sparse thoughts.** For reasoning-heavy tasks the paper
  alternates thought and action every step; for long decision-making tasks
  (ALFWorld, WebShop) thoughts appear sparsely and the *model decides* when
  to think. That asynchronous-thinking design is the ancestor of today's
  adaptive thinking (doc 02 §6).
- **Small fine-tuned beats large prompted.** Fine-tuned on just 3,000
  correct ReAct trajectories, PaLM-8B beat prompted PaLM-62B, and fine-tuned
  62B beat prompted 540B — while fine-tuning models to memorize facts
  (Standard/CoT) stayed far worse than fine-tuning them to *act to retrieve*.
  On ALFWorld/WebShop, one- or two-shot ReAct prompting beat imitation and
  reinforcement learning baselines trained on 10³–10⁵ task instances by 34%
  and 10% absolute success rate.
- **Interpretability and steering.** Because the reasoning trace is explicit,
  humans can distinguish what came from the model's internal knowledge vs.
  the environment, inspect the decision basis, and even *edit a thought* to
  redirect the agent mid-run — the research antecedent of the steering
  channel in §7.

### 11.2 Toolformer — when is a tool call actually useful? (Schick et al., 2023)

Toolformer taught a 6.7B GPT-J model to decide **which** API to call, **when**
to call it, **what arguments** to pass, and **how to incorporate results** —
self-supervised, from only a handful of demonstrations per tool. The
mechanism: have the LM annotate plain text with candidate API calls
(calculator, Wikipedia search, QA system, translation, calendar), execute
them, then **keep only the calls whose inserted call-plus-result reduces the
model's loss on the subsequent tokens** (vs. no call, or a call without its
result), and fine-tune on the filtered data. Results: large zero-shot gains,
often beating a far larger GPT-3, without degrading core language-modeling
ability — because the augmented training set is the original text with only
provably-helpful calls inserted.

What survives into current practice:

- **Its inference procedure is the modern tool loop**: decode until the model
  emits a call marker, pause decoding, execute the API, insert the result
  into context, resume decoding. Today's `tool_use`/`tool_result` blocks (doc
  02 §3) are this exact shape with structure and typing added.
- **Tool-call usefulness is definable and measurable** — "did the result make
  the model's subsequent output better?" — not a matter of taste. That's the
  quantitative version of doc 02's guidance that descriptions and trigger
  conditions determine call quality, and a caution for tool-surface design:
  the paper notes explicitly that *what humans find useful may differ from
  what a model finds useful*.
- **Tools patch specific model weaknesses** (arithmetic, fresh facts,
  low-resource translation, awareness of the current date) more cheaply than
  scale does — the original argument for the augmented LLM (doc 01, Tier 2).

## 12. Memory architectures for agents

§4 covered Anthropic's file-based memory surface. Two of the references
develop memory further: LangChain's memory docs give a production taxonomy,
and *Generative Agents* (Park et al., UIST 2023) contributes the
retrieval-scored memory stream and reflection, still the reference design for
long-lived agent memory.

### 12.1 A taxonomy (LangChain memory docs)

**Short-term memory** is thread-scoped: the ongoing conversation's message
history plus other session state (uploaded files, retrieved documents,
generated artifacts), persisted via checkpoints so a thread can be resumed.
The documented pain is the one from §4: long histories overflow the window,
and even within it models get "distracted" by stale or off-topic content
while costing more and responding slower — hence deliberate trimming and
forgetting, not unbounded accumulation.

**Long-term memory** is cross-thread: stored under custom **namespaces**
(e.g., user or org ID) rather than a thread ID, as JSON documents with a key
per memory, recallable from any conversation, searchable by semantic search
and content filters.

Within long-term memory, three types (mapped from human memory research; the
LangChain docs credit the CoALA paper for the mapping):

| Type | Stores | Human analog | Agent example |
|---|---|---|---|
| **Semantic** | facts | things learned in school | facts about a user |
| **Episodic** | experiences | things I did | past agent actions, used as few-shot examples |
| **Procedural** | instructions | motor skills/instincts | the agent's own system prompt |

Design tradeoffs documented per type:

- **Semantic — profile vs. collection.** A *profile* is one continuously
  updated JSON document: easy to read whole, but updates become error-prone
  as it grows (the model must reconcile new info with the old document;
  mitigate with patch-style updates, splitting into multiple documents, or
  strict decoding against a schema). A *collection* of narrow documents is
  easier to write (generating a new object beats reconciling) and yields
  higher recall — but shifts complexity to updating (models over-insert or
  over-update existing items) and to search, and can lose the relationships
  between memories that a unified profile shows for free.
- **Episodic in practice = few-shot examples**: it's often easier to "show"
  than "tell", and the hard part is *selecting* the most relevant past
  examples for the current input, not storing them.
- **Procedural in practice = the agent rewriting its own instructions** via
  reflection/meta-prompting: prompt the agent with its current instructions
  plus recent conversations or explicit user feedback, and have it emit
  refined instructions — useful precisely where instructions are hard to
  specify up front.

**When to write memories** — two documented paths:

- **In the hot path** (agent decides mid-conversation, e.g., a
  `save_memories` tool the model may call on each user message, the
  ChatGPT-style design): memories are available immediately and the user can
  be shown what was saved, but it adds latency and the agent multitasks
  between memory curation and the actual job, which degrades both.
- **In the background** (a separate async/scheduled process generates
  memories from the transcript): no user-facing latency and memory logic
  stays out of application logic, but you must choose triggers (after a
  quiet period, on a schedule, or manual) and infrequent runs leave other
  threads without fresh context.

### 12.2 Scored retrieval and reflection (Generative Agents)

The paper ran 25 LLM-driven agents in a simulated town for days of game time;
its architecture answers "how does an agent use an experience record far too
large for the context window?" The load-bearing insight: **retrieval beats
summarization** — summarizing everything into the prompt produced generic
answers, while surfacing the *relevant* records produced specific ones.

- **Memory stream**: an append-only record of *observations* — natural-
  language descriptions of what the agent did or perceived, each with a
  creation timestamp and a last-access timestamp.
- **Retrieval score** = normalized (min-max to [0,1]) weighted sum of three
  components, all weights 1 in the paper:
  - **Recency** — exponential decay on time since last access (decay factor
    0.995 per sandbox hour), so recently *used* memories stay warm;
  - **Importance** — an LLM-assigned 1–10 poignancy score at write time
    ("brushing teeth" ≈ 1–2, "a break-up / college acceptance" ≈ 8–10),
    separating core memories from mundane ones;
  - **Relevance** — embedding cosine similarity between the memory and the
    current query context.
  Top-ranked memories that fit the context budget go into the prompt.
- **Reflection**: raw observations don't support inference (asked who to
  spend time with, an agent picks whoever it *saw most often* rather than the
  actual kindred spirit). Periodically — triggered when the summed importance
  of recent events crosses a threshold (150; roughly 2–3×/day in practice) —
  the agent takes its ~100 most recent records, asks the model for the most
  salient high-level questions they raise, retrieves memories per question,
  and synthesizes insights *with citations back to the evidence records*.
  Reflections are stored as memories themselves and can reflect on prior
  reflections, yielding trees of increasingly abstract self-knowledge.
- **Planning**: to stay coherent over hours (without a plan, an agent asked
  "what now?" at 12:00, 12:30, and 1:00 eats lunch three times), agents draft
  a broad-strokes day plan, then recursively decompose it into hour-level and
  then 5–15-minute actions. Plans live in the memory stream too, so
  observations, reflections, and plans are retrieved together; each new
  observation is checked — continue the plan, or react and re-plan from here?
- **Evidence it matters**: in the paper's ablation study (agents
  "interviewed" in natural language), removing any of observation memory,
  reflection, or planning measurably degraded believability — each component
  is causally load-bearing. The most common failures were the agent *failing
  to retrieve* the relevant memory, **fabricating embellishments** on top of
  real memories, and inheriting an overly formal style from the model —
  the same three failure classes to test for in any memory-backed assistant.

The synthesis with §4: Anthropic's file-based memory gives the *surface*
(read/write/update files); the LangChain taxonomy tells you *what kinds of
things* to store and when to write them; the generative-agents scoring tells
you *how to pick* what re-enters context when the store outgrows naive
inclusion — recency, importance, and relevance, plus periodic consolidation
of raw records into higher-level, citable lessons.

### Cross-provider and research sources

- [OpenAI, *A practical guide to building agents*](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf).
- [Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models*](https://arxiv.org/abs/2210.03629) (ICLR 2023) — §11.1: thought-as-action formalism, HotpotQA/FEVER/ALFWorld/WebShop results, failure-mode analysis, ReAct↔CoT-SC back-off, fine-tuning scaling, thought editing.
- [Schick et al., *Toolformer: Language Models Can Teach Themselves to Use Tools*](https://arxiv.org/abs/2302.04761) — §11.2: self-supervised sample→execute→filter-by-loss pipeline, five tools, inference-time call/pause/resume decoding.
- [Park et al., *Generative Agents: Interactive Simulacra of Human Behavior*](https://arxiv.org/abs/2304.03442) (UIST 2023) — §12.2: memory stream, recency/importance/relevance retrieval, reflection trees, recursive planning, ablation and failure findings.
- [LangChain, *Memory overview* (conceptual docs)](https://docs.langchain.com/oss/python/concepts/memory) — §12.1: short/long-term split, semantic/episodic/procedural types, profile vs. collection, hot-path vs. background writes, namespaced store.
- [AWS, *Agentic AI Lens*](https://docs.aws.amazon.com/wellarchitected/latest/agentic-ai-lens/agentic-ai-lens.html).
- [OWASP, *Agentic AI Security Initiative*](https://genai.owasp.org/initiatives/agentic-security-initiative/).
