# 01 — LLM Application Archetypes

> How to classify what you're building, and the discipline of choosing the *least*
> agentic architecture that solves the problem.

## The complexity gradient

Anthropic's *Building Effective Agents* (Dec 2024) draws the load-bearing distinction
for this whole field:

- **Workflows** — systems where LLMs and tools are orchestrated through
  *predefined code paths*. Your code owns the control flow; the model fills in
  steps.
- **Agents** — systems where the LLM *dynamically directs its own processes and
  tool usage*, maintaining control over how it accomplishes a task.

Everything below is arranged along that gradient. The engineering rule: start at
the top of this list and move down only when the current tier demonstrably fails.
Each step down buys capability and pays for it in latency, cost, variance, and
debuggability.

```
Single call  →  Augmented call (retrieval / tools / few-shot)
             →  Workflow (chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer)
             →  Agent (model-driven loop over tools)
             →  Multi-agent (coordinator + subagents)
```

---

## Tier 1: Single LLM call

One request, one response. This tier covers a large fraction of real production
value, and it is where you should prototype *everything* first.

**Application types:**

| Type | Shape | Notes |
|---|---|---|
| Classification | text → label | Use structured outputs or an enum-typed tool to force valid labels. Small/fast models (Haiku-tier) are usually sufficient; validate with an eval set before assuming you need a bigger model. |
| Extraction | unstructured text/PDF/image → structured record | The canonical structured-outputs use case: `output_config.format` with a JSON schema, or `strict: true` tool schemas, guarantees parseable output. |
| Summarization | long doc(s) → summary | Quality is dominated by the instruction spec (audience, length, what to preserve/drop), not the model. Long-context models (1M-token context on current Claude Opus/Sonnet tiers) have mostly removed the need for map-reduce summarization except at extreme scale. |
| Q&A / chat over provided context | context + question → answer | Stuff the docs in the prompt when they fit; add citations support if provenance matters. RAG (doc 03) only becomes necessary when the corpus exceeds what's practical to send. |
| Transformation / rewriting | text → text | Style transfer, translation, redaction, normalization. |
| Content generation | brief → artifact | Marketing copy, code snippets, emails. Variance matters here; on models that removed sampling parameters, variance is elicited by prompt (e.g., "propose N distinct directions"). |

**Key engineering facts for this tier:**

- The API is **stateless** — there is no server-side conversation memory in the
  base Messages API; multi-turn means resending history.
- For high-volume, latency-insensitive work, the **Batch API** processes requests
  asynchronously at **50% of standard prices** (up to 100K requests or 256 MB per
  batch; most complete within an hour, 24h max). Classification/extraction
  pipelines over large corpora should default to batches.
- **Prompt caching** makes the "same big context, many questions" pattern cheap:
  cache reads cost ~0.1× base input price (see doc 02).

**When this tier fails:** the task needs information the model doesn't have
(→ add retrieval or tools), the output of one step feeds another (→ workflow), or
the number of steps is unpredictable (→ agent).

---

## Tier 2: The augmented LLM

Still one logical call, but the model is augmented with **retrieval, tools, and
memory** — what *Building Effective Agents* calls the basic building block of
agentic systems. In practice this means:

- **Retrieval**: relevant context fetched (by your code or by a search tool) and
  placed in the prompt. See doc 03.
- **Tools**: the model can request actions — your functions (user-defined tools),
  or server-side tools the provider executes (web search, code execution).
- **Memory**: state persisted outside the context window — from a simple
  per-user profile string you inject, up to a model-managed memory directory.

The design guidance from the same source: focus on two things — tailoring the
capabilities to your use case, and giving the model an **easy, well-documented
interface** to them. Tool definitions deserve the same care as a public API:
precise descriptions (including *when* to call the tool, not just what it does),
enums for closed sets, examples for tricky formats.

---

## Tier 3: Workflows

Multiple LLM calls orchestrated by *your code*. The five canonical patterns
(taxonomy from *Building Effective Agents*):

### 3.1 Prompt chaining

Decompose a task into a fixed sequence of steps, each call consuming the previous
output, optionally with programmatic gates/validation between steps.

- **Use when** the decomposition is known and stable: outline → draft → polish;
  extract → validate → transform; generate → translate.
- **Why it works**: each call has a narrower, better-specified job, trading latency
  for accuracy. Gates between steps are where you enforce invariants cheaply in
  code (schema checks, length limits, banned content) instead of prompting for them.

### 3.2 Routing

A classifier (LLM or traditional) dispatches the input to a specialized downstream
prompt/model.

- **Use when** inputs fall into distinct categories that are handled better
  separately — mixing them into one mega-prompt degrades all of them.
- **The economic version**: route easy/common queries to a small fast model and
  hard ones to a frontier model. This is the highest-leverage cost control after
  caching, but it *requires an eval set per route* to know the small model is
  actually adequate.

### 3.3 Parallelization

Two sub-forms:

- **Sectioning** — independent subtasks run concurrently (e.g., one call screens
  a query for policy while another answers it; review N files simultaneously).
- **Voting** — the same task run multiple times for diverse answers, aggregated
  (majority vote for labels, union for defect-finding, threshold for flagging).

Parallelization buys latency (sectioning) or confidence (voting) at linear token
cost. Voting is a legitimate, simple accuracy lever for high-stakes classification
before reaching for fine-tuning or bigger models.

### 3.4 Orchestrator-workers

A central LLM call *dynamically decomposes* the task into subtasks, delegates each
to worker calls, and synthesizes results. Differs from parallelization in that the
subtasks aren't predefined — the orchestrator decides them per input (e.g., "which
files does this change touch?" → per-file edit workers; multi-source research →
per-source search workers).

This is the bridge pattern: control flow is still bounded (one decomposition, one
synthesis) but the decomposition itself is model-driven.

### 3.5 Evaluator-optimizer

A generator call produces output; an evaluator call critiques it against criteria;
loop until accepted or budget exhausted.

- **Use when** you have *clear evaluation criteria* and iterative refinement
  measurably helps — literary translation, complex search with completeness
  criteria, code that must pass review criteria.
- **Failure mode**: vague rubrics produce noisy accept/reject loops that burn
  tokens without converging. Write the rubric as independently gradeable criteria
  ("the CSV has a numeric `price` column"), not vibes ("data looks good").

**Workflow-tier engineering notes:**

- Workflows are **debuggable and testable per-step** — you can unit-test each
  prompt with fixtures, snapshot intermediate outputs, and attribute regressions
  to a specific step. This is the property you give up when you move to agents.
- Chain state lives in your code, so ordinary engineering applies: idempotency,
  retries per step, checkpointing between steps for long pipelines.
- Mixed-model chains are normal (cheap model drafts, expensive model verifies),
  but note caches are model-scoped — a model switch never reads the other model's
  prompt cache.

---

## Tier 4: Agents

The model runs a loop: plan → call tool(s) → observe results → repeat, until it
decides the task is done. Your code executes tools and feeds results back; the
*model* owns control flow. Covered in depth in doc 04. The research origin of
this loop is the **ReAct** pattern (Yao et al., ICLR 2023): interleave
free-form *reasoning traces* with environment *actions* so that reasoning
creates and adjusts the plan while actions ground the reasoning in retrieved
fact — the paper showed this beats acting without reasoning and hallucinates
far less than reasoning without acting (doc 04 §11.1).

**The four-question gate** (from Anthropic's agent-design guidance) — build an
agent only if all four answers are yes:

1. **Complexity** — is the task multi-step and hard to fully specify in advance?
   ("Turn this design doc into a PR" yes; "extract the title from this PDF" no.)
2. **Value** — does the outcome justify substantially higher cost and latency?
3. **Viability** — is the model actually capable at this task type? (Prototype
   the hardest representative case first.)
4. **Cost of error** — can errors be caught and recovered from (tests, review,
   sandboxing, rollback)? Autonomy means compounding errors; if a mistake is
   expensive and undetectable, don't grant autonomy.

**Canonical agent applications** (where the pattern has proven out in production):
coding agents (test suites provide the error-correction signal), computer-use
agents, customer-support agents with tool access to refunds/orders/KB, deep
research, data analysis over warehouses, SRE/incident investigation.

The common thread: a *verifiable environment* — some ground truth (tests pass,
the page rendered, the query returned rows) that lets the agent check its own work.
Agent quality tracks the quality of the feedback signal more than anything else.

---

## Tier 5: Multi-agent systems

A coordinator agent delegates to subagents, each with its own context window,
tools, and persona. Two grounded motivations:

1. **Context isolation** — a subagent can burn 200K tokens reading files and
   return a 2K-token summary; the coordinator's context stays clean. This is the
   dominant practical reason, not "role play."
2. **Parallelism** — independent workstreams (read 30 files, run 5 test suites)
   fan out concurrently.

Grounded warnings:

- Subagents share *no conversation state* with the coordinator (they may share a
  filesystem, depending on platform). Every delegation message must carry the
  context the subagent needs, or it will rediscover it expensively — this is the
  #1 multi-agent bug.
- Coordinator-worker depth beyond one level rarely pays; platforms that support
  multi-agent natively (e.g., Anthropic Managed Agents) explicitly ignore
  delegation deeper than one level.
- Don't use subagents where a `grep` would do. Model guidance for recent Claude
  models is explicit: delegate for *parallel or independent workstreams*, work
  directly for single-file reads and sequential operations.

---

## Choosing a hosting/harness tier (Claude-specific)

Orthogonal to the application archetype is *who supplies the loop and the infra*.
For the Claude ecosystem there are four documented options (see the Claude API
docs for current details):

| Approach | You write | Who runs the loop | Who hosts |
|---|---|---|---|
| Messages API, manual loop | the `while stop_reason == "tool_use"` loop | you | you |
| Messages API, SDK Tool Runner | just tool functions | SDK (with per-turn hooks for approval/interception) | you |
| Managed Agents (beta) | agent config + custom-tool results | Anthropic (orchestration layer + per-session sandbox container) | Anthropic |
| Claude Agent SDK | a prompt + options | SDK ships the full Claude Code harness (built-in file/bash/search tools, subagents, permissions) | you |

Rule of thumb from the docs: "simplest" means the least code *you own*. For a
hosted, scheduled, or memory-backed agent, the managed offering is often simpler
than a hand-rolled loop even though it's a bigger platform. For a custom-tool
agent on your own infra, the Tool Runner is the default; drop to the manual loop
only for control the runner's hooks don't expose.

---

## Anti-patterns observed at every tier

- **Agent-washing a workflow.** If you can enumerate the steps on a whiteboard,
  it's a workflow. Encoding known steps as "the agent will figure it out" trades
  determinism for nothing.
- **Framework-first development.** Frameworks (LangChain/LangGraph, etc.) add
  abstraction layers that obscure the actual prompts and responses, making
  debugging harder — Anthropic's explicit suggestion is to start with direct API
  calls, and if you adopt a framework, make sure you understand what's under the
  hood.
- **One mega-prompt serving five use cases.** Routing exists precisely because
  specialized prompts beat a Swiss-army prompt; the mega-prompt also destroys your
  prompt-cache hit rate if per-use-case content varies early in the prompt.
- **Skipping the eval before escalating tiers.** "The single call wasn't good
  enough" is only meaningful against a measured baseline. Most tier escalations
  happen without one and buy variance instead of accuracy.

---

## Sources

- Anthropic, *Building Effective Agents*, Dec 2024 — workflow/agent definitions,
  the five workflow patterns, augmented-LLM building block, simplicity principle.
- Anthropic platform docs: agent-design guidance (complexity/value/viability/cost-of-error
  gate), Batch API limits and pricing, structured outputs, multi-agent constraints
  (Managed Agents documentation).

---

## Cross-provider extension: capability and risk taxonomy

The tiers above describe **control flow**. A principal-level review should also
classify the system along independent axes; private data implies retrieval and
authorization, for example, but does not by itself imply an agent.

| Axis | Low end | High end | Architecture consequence |
|---|---|---|---|
| Knowledge freshness | supplied input | live/private corpus | retrieval or read-only tools |
| Action authority | answer only | irreversible action | authorization, approval, idempotency, audit |
| Control-flow uncertainty | fixed transform | unknown steps | workflow first; bounded agent when measured |
| Verifiability | subjective prose | executable/testable | automatic validators and feedback loops |
| Duration | one request | hours/days | durable execution, checkpoints, resumability |
| Data sensitivity | public | regulated/tenant-private | isolation, retention, redaction, ACL filtering |

Additional product families include **copilots** (human decides; model drafts),
**knowledge assistants** (retrieval and grounded synthesis), **document
intelligence** (parse, extract to schema, validate, exception queue), **decision
support** (evidence and alternatives without silent high-impact decisions),
**content pipelines** (generate, critique, policy-check, publish), **natural-language
interfaces** (translate intent to constrained queries/API calls), and **autonomous
operators** (bounded, observable and reversible action).

Before implementation, record the business outcome, non-LLM baseline, minimum
quality, latency/cost budgets, data classification, permissible actions, evaluation
method, rollback path, and evidence that a simpler tier was insufficient.

### Cross-provider sources

- [OpenAI, *A practical guide to building agents*](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf).
- [Microsoft, *Design and develop a RAG solution*](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide).
- [AWS, *Generative AI Lens: design principles*](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/design-principles.html).
- [Yao et al., *ReAct*](https://arxiv.org/abs/2210.03629).
