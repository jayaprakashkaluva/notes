# Building Applications with LLMs — Engineering Notes

A documentation set on designing, building, and operating LLM-powered applications,
written for a staff/principal-engineer audience. Concepts are provider-neutral;
concrete API examples use the Anthropic Claude API (Messages API), with facts
grounded in Anthropic's published documentation and engineering posts (cited per
document). Where a number or behavior is provider-specific, it is called out as such.

## Contents

| Doc | Topic |
|---|---|
| [01 — Application Archetypes](01-application-archetypes.md) | The taxonomy of LLM applications: single-call, workflows, RAG, agents, multi-agent. When each is appropriate, and the decision framework for escalating complexity. |
| [02 — Core Building Blocks](02-core-building-blocks.md) | The API primitives every application composes: messages, system prompts, tool use, structured outputs, streaming, extended thinking/effort, context windows, prompt caching, batch processing. |
| [03 — Retrieval-Augmented Generation](03-rag-and-retrieval.md) | RAG architecture in depth: chunking, embeddings, hybrid search, reranking, contextual retrieval, citations, agentic search, and when *not* to build RAG. |
| [04 — Agents](04-agents.md) | Agent architecture: the loop, tool-surface design (ACI), context management over long horizons (compaction, context editing, memory), orchestration patterns, and the harness. |
| [05 — Production Best Practices](05-production-best-practices.md) | Evals, cost and latency engineering, reliability (retries, stop reasons, timeouts), security (prompt injection, tool sandboxing), and observability. |

## The one-paragraph summary

Most LLM applications fail not because the model is weak but because the system
around it is over- or under-engineered. The single most repeated finding from teams
shipping LLM products — stated explicitly in Anthropic's *Building Effective Agents*
— is to **find the simplest solution possible, and only increase complexity when
demonstrably needed**. A single well-prompted API call with retrieval and few-shot
examples covers a surprising share of production use cases. Workflows (code-orchestrated
multi-step LLM calls) cover most of the rest. Agents — where the model itself directs
the control flow — are reserved for open-ended problems where the number of steps
cannot be predicted and the value justifies latency, cost, and the possibility of
compounding errors. Every document in this set follows that gradient.

## Primary sources

- Anthropic, *Building Effective Agents* (engineering blog, Dec 2024) — the
  workflow/agent taxonomy used throughout.
- Anthropic, *Introducing Contextual Retrieval* (engineering blog, Sept 2024).
- Anthropic platform documentation (platform.claude.com/docs) — Messages API, tool
  use, prompt caching, structured outputs, batch processing, context editing,
  compaction, memory tool.
- OWASP, *Top 10 for LLM Applications* — security taxonomy referenced in doc 05.

Facts that are cached snapshots (model IDs, prices) are dated where stated; verify
against the live Models API / pricing page before relying on them for decisions.

## Expanded source base

The original notes are retained. The added sections introduce corroborating and
complementary guidance from OpenAI's agent/API guides, Microsoft Azure Architecture
Center's RAG guidance, AWS Well-Architected Generative AI and Agentic AI lenses,
NIST AI RMF 1.0 and its Generative AI Profile, OWASP, and the foundational RAG and
ReAct papers. Each chapter links directly to its new sources.

Doc 04 additionally incorporates four reference works in depth:

- Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models*
  (ICLR 2023) — the research origin of the agent loop; grounding vs.
  hallucination findings and failure-mode analysis (doc 04 §11.1).
- Schick et al., *Toolformer: Language Models Can Teach Themselves to Use
  Tools* (2023) — self-supervised tool-use learning and the origin of the
  inference-time tool-call loop (doc 04 §11.2, doc 02 §3).
- Park et al., *Generative Agents: Interactive Simulacra of Human Behavior*
  (UIST 2023) — the memory stream, recency/importance/relevance retrieval,
  reflection, and recursive planning (doc 04 §12.2).
- LangChain, *Memory overview* (conceptual docs) — short/long-term memory,
  the semantic/episodic/procedural taxonomy, profile vs. collection, and
  hot-path vs. background memory writes (doc 04 §12.1).

## Using the set in a design review

Start with the archetype, then review the API boundary, retrieval design (when
knowledge is external), agent design (when control flow is model-directed), and
production controls. Require measurable acceptance criteria, a non-LLM or simpler-
tier baseline, data and threat models, latency/cost budgets, evaluation suites,
rollout/rollback, and an owner for production quality.

Vendor-specific model IDs, prices, limits, quotas, and beta features remain dated
snapshots and must be checked against current provider documentation before adoption.
