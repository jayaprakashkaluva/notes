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
