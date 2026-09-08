# Domain-Driven Design: Building Software That Matches the Business

A staff/principal-engineer's treatment of Domain-Driven Design (DDD). The goal is not to recite the pattern catalogue but to leave you with the **problem DDD exists to solve**, the **mechanisms** it uses, the **numbers and heuristics** that guide sizing decisions, the **ways it fails in production**, and the **design judgement** that separates someone who can name "Aggregate" and "Bounded Context" from someone who can walk into a 400-table monolith with 60 engineers, find the three real domains hiding inside it, draw the context map, decide which parts deserve a rich model and which should stay CRUD, and defend that decision to the CTO.

Every section is built the same way: **the problem** → **the mechanism that solves it** → **the heuristics and numbers** → **how it fails** → **what a senior engineer decides**.

Companion notes in this repo:

- [REST API Best Practices](../software-engineering/rest-api-best-practices.md) — the outer edge of a bounded context is usually an HTTP API; that note covers how to shape it, this one covers what sits behind it.
- [GraphQL Best Practices](../software-engineering/graphql-best-practices.md) — GraphQL federation is context mapping with a schema; see §6 and §15 here.
- [Exactly-Once Semantics & Stream Processing](../distributed-systems/05-exactly-once-and-stream-processing.md) — domain events cross context boundaries over Kafka; the delivery guarantees there are what make §10 and §15 here honest.
- [Realtime, High-Scale Distributed Systems](../distributed-systems/realtime-high-scale-distributed-systems.md) — aggregates are the unit of consistency; partitioning and replication in that note explain why the aggregate boundary is also a sharding boundary.
- [Consensus, Raft and Spanner](../distributed-systems/03-consensus-raft-and-spanner.md) — what a "transaction" costs when it spans machines, which is why DDD says one aggregate per transaction.

---

## Table of Contents

1. [The Problem: Software That Drifts From the Business](#1--the-problem-software-that-drifts-from-the-business)
2. [What Happens Without DDD: The Decay Timeline](#2--what-happens-without-ddd-the-decay-timeline)
3. [What DDD Actually Is](#3--what-ddd-actually-is)
4. [Ubiquitous Language: The Model Is the Vocabulary](#4--ubiquitous-language-the-model-is-the-vocabulary)
5. [Bounded Contexts: One Word, One Meaning, One Boundary](#5--bounded-contexts-one-word-one-meaning-one-boundary)
6. [Context Mapping: How Contexts Relate and Who Bends](#6--context-mapping-how-contexts-relate-and-who-bends)
7. [Subdomains and Core Domain Distillation: Where to Spend the Money](#7--subdomains-and-core-domain-distillation-where-to-spend-the-money)
8. [Tactical Building Blocks: Entities and Value Objects](#8--tactical-building-blocks-entities-and-value-objects)
9. [Aggregates: The Consistency Boundary](#9--aggregates-the-consistency-boundary)
10. [Domain Events: Making the Past Explicit](#10--domain-events-making-the-past-explicit)
11. [Repositories, Factories, Domain Services, and Application Services](#11--repositories-factories-domain-services-and-application-services)
12. [Architecture: Layers, Hexagons, and Where the Domain Lives](#12--architecture-layers-hexagons-and-where-the-domain-lives)
13. [The Modelling Process: Knowledge Crunching and EventStorming](#13--the-modelling-process-knowledge-crunching-and-eventstorming)
14. [Persistence: ORMs, Event Sourcing, and CQRS](#14--persistence-orms-event-sourcing-and-cqrs)
15. [DDD in Distributed Systems: Microservices, Sagas, and the Outbox](#15--ddd-in-distributed-systems-microservices-sagas-and-the-outbox)
16. [DDD and Legacy: Bubbles, Anti-Corruption Layers, and the Strangler](#16--ddd-and-legacy-bubbles-anti-corruption-layers-and-the-strangler)
17. [A Worked Example: Modelling Order Fulfilment End to End](#17--a-worked-example-modelling-order-fulfilment-end-to-end)
18. [Anti-Patterns: How DDD Fails in Practice](#18--anti-patterns-how-ddd-fails-in-practice)
19. [War Stories](#19--war-stories)
20. [The Staff Engineer's Decision Checklist](#20--the-staff-engineers-decision-checklist)
21. [Mental Models Worth Keeping](#21--mental-models-worth-keeping)
22. [Reading Notes: The Path to Expertise](#22--reading-notes-the-path-to-expertise)
23. [A Lab Path](#23--a-lab-path)

---

# 1 — The Problem: Software That Drifts From the Business

Every non-trivial business system is a *model* of some part of the world: a bank models accounts and ledgers, an airline models seats, fares and itineraries, a hospital models patients, encounters and orders. The software is valuable exactly to the extent that its model captures what matters to the people who run the business, and it becomes a liability exactly to the extent that the model diverges from how those people actually think.

DDD starts from three observations that Eric Evans made explicit in 2003 and that anyone who has worked on a 10-year-old system will recognise:

**1. The hard part of most software is not the technology, it is the domain.** A payments engine is not hard because of TCP or Postgres; it is hard because "settlement", "authorisation", "capture", "chargeback", "partial refund on a split-tender order with a promo code" are concepts with dozens of edge cases that the business has spent decades learning and the engineers have spent weeks. Technical complexity is at least well-documented. Domain complexity lives in the heads of a handful of people who are usually not in the room when the code is written.

**2. Knowledge is lost at every translation.** The business expert explains it to a product manager, who writes a ticket, which an engineer reads and turns into a class named `OrderProcessorServiceImpl`. Each hop drops nuance. Six months later the only people who can explain why the code does what it does are the ones who wrote it, and they have moved teams. The translation tax is not one-off; it is paid on every change forever.

**3. A single unified model of a large business does not exist and cannot be built.** "Customer" means something different to sales, billing, support, and legal. Attempting to have one `Customer` class satisfy all of them produces an object with 300 fields, 40 nullable ones, six meanings of `status`, and a change-approval process involving four teams. Large models must be *split*, and the split lines must be chosen deliberately.

DDD is the discipline of taking those three observations seriously. It gives you:

| Problem | DDD's answer |
|---|---|
| Domain knowledge is scattered and lossy | **Ubiquitous Language** — one vocabulary, used by experts and code alike, refined continuously |
| A single model cannot serve the whole business | **Bounded Contexts** — explicit boundaries inside which a model is consistent, and explicit relationships between them (**Context Maps**) |
| Effort is spread evenly over what matters and what doesn't | **Core Domain distillation** — identify the 10–20% that differentiates the business and put the best people there |
| Business rules are smeared across controllers, SQL, and UI | **Tactical patterns** — Entities, Value Objects, Aggregates, Domain Events, Repositories — that give rules a home |
| Consistency boundaries are implicit, so transactions grow until they deadlock | **Aggregates** — an explicit unit of consistency that is also the unit of locking, sharding, and ownership |
| Systems are integrated by sharing databases, so nothing can change | **Anti-Corruption Layers, Published Language, Open Host Service** — integration patterns that keep models independent |

The rest of this note goes through each mechanism. But first, it is worth being concrete about the alternative.

---

# 2 — What Happens Without DDD: The Decay Timeline

"Without DDD" does not mean chaos on day one. It means a system that starts fine and decays along a predictable curve. The following is a composite of a dozen real systems; if you have worked on anything over five years old you will recognise the stages.

## 2.1 Year 0–1: The CRUD honeymoon

The team builds from the database up. Tables map to classes; classes map to REST endpoints; the "model" is a set of getters and setters. Business logic goes into "service" classes because that is where it fits. Everything is fast because the domain is still simple and everyone who wrote it is still here.

This is Fowler's **Anemic Domain Model**: objects that are bags of data with all behaviour elsewhere. He called it an anti-pattern in 2003 because it "is contrary to the basic idea of object-oriented design; which is to combine data and process together" and "incurs all of the costs of a domain model, without yielding any of the benefits."

Nothing is visibly wrong yet. This is the dangerous part.

## 2.2 Year 1–3: The god objects and the translation tax

Requirements accumulate. The `Order` table gains `status`, then `sub_status`, then `fulfilment_state`, then `is_cancelled`, then `cancellation_reason`, then `legacy_status_do_not_use`. There are now 11 statuses and 3 booleans and only two engineers know which combinations are valid. Every service that touches orders has its own `if (order.status == "SHIPPED" || order.status == "PARTIALLY_SHIPPED" && !order.isCancelled)` and they disagree.

The business says "a *fulfilled* order can be *returned* within 30 days." Engineering asks "which status is 'fulfilled'?" The answer takes a meeting. The **translation tax** is now a recurring line item: each feature spends 20–40% of its effort figuring out what the existing code *means* before changing it.

The `OrderService` is 4,000 lines. `Customer` has 140 columns because marketing, billing, and support each added their own. Nobody can delete a column because nobody knows who reads it.

## 2.3 Year 3–5: The shared database and the integration lock

More teams. To move faster, they split into "services" — but the services share the database, because the model was never partitioned and the data is all tangled together. Now:

- Team A wants to rename `customer.tier` to `customer.loyalty_band`. Fourteen services read it. Change is blocked.
- Team B adds a column. Team C's nightly job breaks because it does `SELECT *` into a fixed struct.
- A transaction in the checkout path touches `orders`, `inventory`, `customers`, `promotions`, and `ledger`. Under load, lock contention on `inventory` rows pushes p99 checkout latency from 200 ms to 4 s. The fix is to "add an index," which does not help because the problem is that the *consistency boundary* was never designed and now spans five tables that belong to five teams.

Conway's law asserts itself: the org has five teams, but the system has one model, so every non-trivial change is a five-team negotiation. Lead time for a "simple" change is measured in weeks.

## 2.4 Year 5+: The big ball of mud and the rewrite

Foote and Yoder named it in 1997: "A Big Ball of Mud is a haphazardly structured, sprawling, sloppy, duct-tape-and-baling-wire, spaghetti-code jungle." The distinguishing symptoms:

- **Nobody can draw the system.** Ask for an architecture diagram and you get boxes labelled by *team* or by *deployment unit*, not by *concept*.
- **Every change is a regression risk.** No one knows which rules live where. The test suite is 40% mocks of other internal classes and passes even when the business logic is wrong.
- **The domain experts have given up.** They describe what they want in business terms; engineering translates it into a patch on the existing tangle; the result is 80% right; a follow-up ticket fixes the rest. The domain experts learn to speak in patches instead of intent.
- **A rewrite is proposed.** It is estimated at 12 months, takes 30, reproduces most of the same structure because the same undocumented model is copied, and is itself a big ball of mud by year 3. Vernon calls the failure to do the modelling work before a rewrite "the second-system effect with extra steps."

## 2.5 The cost, quantified

The numbers vary but the shape does not:

| Symptom | Typical measurement | Where it shows up |
|---|---|---|
| Translation tax | 20–40% of feature effort spent understanding existing meaning | Sprint velocity flat while headcount doubles |
| Coordination cost | Changes needing 3+ teams' sign-off: >30% of tickets | Lead time weeks, not days |
| Shared-DB coupling | 1 schema change → N downstream breakages | Nightly job failures, "who owns this column" threads |
| Rule duplication | Same invariant implemented in 5–15 places | Inconsistent behaviour; "it works in the app but not the API" |
| Transaction sprawl | Checkout transaction touches 5+ tables across teams | Lock contention, p99 collapse under load, deadlocks |
| Onboarding | 3–6 months to be productive | Because the model lives in people's heads, not in the code |

None of this is about the choice of language, framework, or database. Teams with Kubernetes, Kafka, and a modern stack decay the same way. The failure is that *the model was never designed*, so the system's structure is the accumulated history of whoever touched it last.

DDD is a set of practices to prevent this curve, or to bend an existing system off it.

---

# 3 — What DDD Actually Is

Evans' definition, from the *DDD Reference* (2015):

> Domain-Driven Design is an approach to the development of complex software in which we: (1) Focus on the core domain. (2) Explore models in a creative collaboration of domain practitioners and software practitioners. (3) Speak a ubiquitous language within an explicitly bounded context.

Note what is *not* in that definition: no mention of Entities, Aggregates, Repositories, microservices, event sourcing, CQRS, hexagonal architecture, or any particular language. Those are tools. The definition is about **where to focus**, **how to learn**, and **how to talk**.

## 3.1 The two halves

DDD is conventionally split into two halves, and the split matters because most teams learn the wrong one first:

**Strategic design** — the big-picture decisions:
- What are the subdomains? Which is core?
- Where are the bounded contexts? How do they relate (context map)?
- Which teams own which contexts? What is the integration contract between them?

**Tactical design** — the in-the-code patterns:
- Entities, Value Objects, Aggregates
- Domain Events, Domain Services
- Repositories, Factories, Application Services

Evans has said, repeatedly, that the tactical patterns got too much attention in the first decade of DDD and that the strategic patterns are the ones that matter. Vernon's *DDD Distilled* puts strategic design first for the same reason. The practical consequence:

> **If you only do tactical DDD — Entities and Repositories inside one undivided model — you get a slightly better-organised big ball of mud.** The wins come from bounded contexts and context maps.

## 3.2 What DDD is not

- **Not a layering scheme.** "We have a domain layer" is not DDD. Plenty of teams have a `domain` package full of anemic DTOs.
- **Not microservices.** DDD predates microservices by a decade and works fine in a monolith. Microservices *need* DDD (to find service boundaries); DDD does not need microservices.
- **Not event sourcing or CQRS.** Those are persistence and read-model techniques that combine well with DDD in some contexts and are overkill in most.
- **Not for everything.** Evans is explicit that DDD's overhead is only justified for the *complex core*. A CRUD admin panel for reference data does not need an aggregate.

## 3.3 When DDD pays off

Use this table honestly. The answer is often "not here."

| Situation | DDD value | What to do instead |
|---|---|---|
| Complex business rules, many stakeholders, long-lived system | **High** — this is what it's for | — |
| Integration of several existing systems with different models | **High** — context mapping and ACLs | — |
| Deciding service boundaries for a monolith split | **High** — bounded contexts are the answer | — |
| Simple CRUD over reference data | Low | Active Record / table-module; keep it dumb |
| Data pipeline / ETL | Low for tactical; the *language* still helps | Schema-first, dataflow design |
| Prototype / startup pre-product-market-fit | Low — the domain is not yet understood | Build fast, keep the *language* clean, plan to model later |
| Pure technical infrastructure (a cache, a proxy) | Low | Standard systems design |

The staff-level skill is not applying DDD everywhere; it is knowing which 20% of the system is the core domain and applying it *there* while keeping the rest simple.

---

# 4 — Ubiquitous Language: The Model Is the Vocabulary

## 4.1 The problem

Engineers and domain experts speak different languages. The expert says "underwriting", the engineer hears "validation step". The expert says "the policy lapses", the engineer writes `setActive(false)`. Each translation is a place where the software can be subtly wrong, and because the translation is in someone's head, no one notices until production.

Worse, engineers develop their *own* jargon over time: "the enricher", "the v2 flow", "the legacy sync". That vocabulary is meaningless to the business, and its existence proves that the code is modelling something other than the domain.

## 4.2 The mechanism

A **Ubiquitous Language** is a single vocabulary, built jointly by domain experts and engineers, and used *everywhere*: in conversation, in documents, in tickets, in class names, method names, database columns, event names, and test names. Evans:

> Use the model as the backbone of a language. Commit the team to exercising that language relentlessly in all communication within the team and in the code.

The key properties:

1. **It is the model.** The language is not documentation *about* the model; the language *is* the model. If the business says "an Order is *placed*, then *confirmed*, then *fulfilled*", the code has `Order.place()`, `Order.confirm()`, `Order.fulfil()`, events `OrderPlaced`, `OrderConfirmed`, `OrderFulfilled`, and a state machine with exactly those states.
2. **It is rigorous.** Business jargon is often ambiguous. Building the language forces the ambiguity out. "What exactly does 'confirmed' mean? Payment authorised? Or stock reserved? Or both?" That question, asked early, prevents the 11-status `Order` table.
3. **It evolves.** When the model changes, the language changes, and the code is refactored to match. A class named `OrderConfirmation` that the business now calls "acceptance" is a bug.
4. **It is scoped to a bounded context** (§5). "Product" in the catalogue context and "Product" in the warehouse context are different words that happen to be spelled the same.

## 4.3 What it looks like in practice

**A glossary, maintained like code.** Not a wiki page nobody reads; a `GLOSSARY.md` in the repo, reviewed in PRs, where each term has a definition, its context, and a link to the type that implements it. Cyrille Martraire's *Living Documentation* describes generating this from annotations so it cannot drift.

**Code that reads as domain prose.** Compare:

```java
// Before: technical language, business rule hidden
if (order.getStatus() == 3 && order.getPaymentInfo().getAuthCode() != null
    && inventory.check(order.getItems()) == 0) {
    order.setStatus(4);
    order.setUpdatedAt(now());
    eventBus.publish(new OrderStatusChangedEvent(order.getId(), 3, 4));
}

// After: the ubiquitous language, business rule explicit
if (order.isAwaitingConfirmation() && payment.isAuthorised() && stock.isReservedFor(order)) {
    order.confirm(clock);              // raises OrderConfirmed internally
}
```

The second version can be read by the domain expert. That is not a nicety; it means the expert can *review the logic* and catch the case where confirmation should also require a fraud check.

**Tests named in the language.** `shouldNotConfirmOrderWhenStockReservationExpired` is a specification the business can read. `testConfirm3` is not.

**Events named in the past tense, as the business would say them.** `OrderConfirmed`, `PaymentAuthorised`, `ShipmentDispatched`. Not `OrderUpdate`, `PaymentEvent`, `ShipmentMessage`.

## 4.4 How it fails

- **Two vocabularies.** The business speaks one language in meetings and the code speaks another. This is the default state, and the fix requires engineers to actually change class names when the business changes terms, which feels like "unnecessary refactoring" to a manager who does not understand the cost of drift.
- **The language is owned by engineers.** Engineers invent terms ("the reconciler", "the shadow order") and the business learns them. Now the business is speaking the code's accidental structure. The language must be *co-owned*, and business terms win ties.
- **Synonyms tolerated.** "Client", "customer", "account", "user", "party" used interchangeably. Each is either the same concept (pick one) or a different concept (define the difference). Tolerating synonyms is how "customer" ends up meaning six things.
- **Language without a context.** One glossary for the whole company guarantees collisions. See §5.

## 4.5 What a senior engineer decides

- Start the glossary in week one of any new domain, and make it a PR-reviewed artefact.
- When you hear an engineer explain a business rule using a technical term, ask "what does the business call that?" If the answer is "they don't have a word for it," you have found either a missing concept in the model (a good discovery) or an accidental structure in the code (a smell).
- Budget for renames. A rename that aligns code with a changed business term is a first-class change, not gold-plating.

---

# 5 — Bounded Contexts: One Word, One Meaning, One Boundary

## 5.1 The problem

In any business larger than a few people, the same word means different things to different groups, and the same thing is called by different words. Consider "Product" at a retailer:

| Group | What "Product" means to them | Key attributes |
|---|---|---|
| Catalogue / merchandising | A thing customers can browse and buy | Name, description, images, category, variants, SEO |
| Pricing | A thing with a price schedule | SKU, base price, promotions, tax class, currency |
| Warehouse | A physical unit that takes up shelf space | SKU, dimensions, weight, bin location, hazmat flag |
| Procurement | A thing bought from a supplier | Supplier SKU, lead time, MOQ, cost price |
| Support | A thing customers complain about | SKU, return policy, warranty terms, known issues |

A single `Product` class serving all five has 80+ fields, most nullable, most meaningless to most consumers. Every team's change risks every other team. The alternative is not "five copies of Product" — it is the recognition that these are **five different concepts** that share an identifier.

## 5.2 The mechanism

A **Bounded Context** is an explicit boundary — in code, in the team, in the database, in the vocabulary — inside which a particular model is defined and applies. Evans:

> Explicitly define the context within which a model applies. Explicitly set boundaries in terms of team organization, usage within specific parts of the application, and physical manifestations such as code bases and database schemas. Keep the model strictly consistent within these bounds, but don't be distracted or confused by issues outside.

Inside a context:
- Each term has exactly one meaning.
- The model is internally consistent and can be reasoned about as a whole.
- One team owns it (or at most a small, tightly coordinated set).
- The model has its own persistence; no other context reads its tables.

Between contexts:
- Communication is via explicit contracts: APIs, published events, or a shared kernel that is deliberately small.
- Translation happens at the boundary (§6 — Anti-Corruption Layer).
- Identity is shared (the SKU, the customer ID), but attributes are not.

## 5.3 How to find the boundaries

This is the hardest skill in DDD and is mostly learned by doing. The heuristics, roughly in order of reliability:

1. **Linguistic boundaries.** Where does a word change meaning? Where do two groups use different words for the same thing? Those are the seams. The "Product" table above is five contexts because it is five vocabularies.
2. **Business-capability boundaries.** What does the business *do*? "Sell", "fulfil", "bill", "support" are capabilities that existed before software and will outlive any particular system. A context per capability is a good first cut.
3. **Team / ownership boundaries** (Conway). If two teams already own two areas, the system will fracture along that line whether you plan it or not. Either plan the boundary there, or change the team structure first (the "Inverse Conway Manoeuvre" from *Team Topologies*).
4. **Rate-of-change boundaries.** Things that change together belong together; things that change at different rates for different reasons belong apart. The pricing engine changes weekly under commercial pressure; the warehouse model changes yearly under operational pressure.
5. **Consistency boundaries.** What must be transactionally consistent? (Not "what would be nice to be consistent" — that is everything.) If two things never need to be updated atomically, they can live in different contexts. See §9 on aggregates for the same question at a smaller scale.
6. **Data-ownership boundaries.** Who is the *source of truth* for this attribute? If procurement owns cost price and pricing owns sell price, those are different contexts even though both are "the price of a product".

Heuristics that are *wrong*:

- **By entity.** "Customer service", "Order service", "Product service" — one context per noun. This is the most common mistake and it reproduces the god-object problem at service scale, because every business process needs every noun. Contexts are about *capabilities and vocabularies*, not nouns.
- **By technical layer.** "Data service", "business-logic service", "API gateway". These are layers, not contexts.
- **By size.** "Each context should be N lines / one team / one sprint." Size is an outcome, not an input.

## 5.4 Numbers and heuristics

- A context is usually owned by **one team of 5–9 people**. A context needing three teams is probably three contexts. A team owning six contexts is probably fine if they are small, but check whether they are really contexts or just modules.
- A mid-sized enterprise (say 200 engineers) typically has **15–40 bounded contexts**. A company with 200 "microservices" and no context map has ~200 contexts by default, most of them accidental.
- A context should be understandable by one person in **a few days**. If it takes a month to understand, either it is too big or the model is bad.
- Contexts do not have to be deployment units. A modular monolith with clean module boundaries, separate schemas, and enforced import rules (ArchUnit, module systems) is a perfectly valid set of bounded contexts.

## 5.5 How it fails

- **Contexts on paper only.** The diagram has boundaries; the database has one schema; every "service" joins across it. The boundary is fiction if data is shared.
- **Shared "common" model.** A `common-domain` library containing `Customer`, `Product`, `Address` used by every context. This is the god object again, distributed. A small *Shared Kernel* (§6) is legitimate; a "common" library that grows without governance is the un-splitting of everything you split.
- **Too-fine contexts.** Every noun a service, every service a context. Now a single business process (place an order) spans 12 network calls, and there is no place where the *order-placement rules* live. See §15.
- **Context without a language.** A service boundary drawn for scaling reasons, with no model behind it. It is a deployment unit, not a context.

## 5.6 What a senior engineer decides

- Draw the context map (§6) before drawing the service diagram. If you cannot name the vocabulary difference between two proposed services, they are one context.
- Insist on **schema-per-context** from day one, even in a monolith. It is nearly free early and nearly impossible late.
- When someone proposes a shared library of domain classes, ask: "Which context owns the meaning of `Customer.status`?" If the answer is "all of them," the library is a coupling point, not a reuse win.

---

# 6 — Context Mapping: How Contexts Relate and Who Bends

## 6.1 The problem

Having drawn boundaries, you still need the contexts to work together — orders need prices, shipments need addresses, invoices need orders. Every integration is a relationship with a *power dynamic*: when the upstream model changes, does the downstream have to follow? Who translates? Who pays when it breaks? Leaving those questions implicit is how "we integrated via the database" happens.

## 6.2 The mechanism: the context map and its relationship patterns

A **Context Map** is a diagram of the bounded contexts and the relationships between them, annotated with the *kind* of relationship. Evans' catalogue, extended by Vernon:

| Pattern | Meaning | When to use | Cost |
|---|---|---|---|
| **Partnership** | Two teams coordinate closely; both models evolve together; mutual dependency | Two contexts on the critical path of the same feature, both actively developed | High coordination; only sustainable between two teams who talk daily |
| **Shared Kernel** | A small, explicitly shared subset of the model (code and/or schema), changed only by agreement | Genuinely identical concepts (e.g. `Money`, `CustomerId`) between closely related contexts | Every change needs both teams; keep it *tiny* |
| **Customer–Supplier** | Upstream (supplier) serves downstream (customer); downstream's needs are on the upstream's backlog | Clear provider/consumer with a healthy negotiation channel | Upstream must prioritise; downstream must express needs early |
| **Conformist** | Downstream adopts the upstream's model wholesale; no translation | Upstream is huge, stable, and won't negotiate (e.g. a SaaS platform, a legacy ERP), and its model is *good enough* | Downstream's model is now hostage to the upstream; zero freedom |
| **Anti-Corruption Layer (ACL)** | Downstream builds a translation layer so its own model stays clean | Upstream model is bad, unstable, or foreign, and downstream must protect itself | A whole translation layer to maintain; worth it far more often than teams think |
| **Open Host Service (OHS)** | Upstream exposes a well-defined protocol/API for *any* consumer | Many downstreams; the upstream cannot negotiate with each | Must be designed as a product, versioned, documented |
| **Published Language** | A shared, documented interchange format (schema, events) that OHS speaks | Whenever there is an OHS; ideally an industry standard (FHIR, ISO 20022) or a governed internal schema | Schema governance |
| **Separate Ways** | No integration at all; duplicate the small overlap | Integration cost exceeds the value | Some duplication; often correct |
| **Big Ball of Mud** | A region with no coherent model; draw a line around it and don't let it leak | Legacy you cannot fix now | Must be quarantined with an ACL |

The upstream/downstream arrow (**U → D**) is drawn on every edge. Upstream can change without asking; downstream must react.

## 6.3 Worked map

```
   ┌────────────────┐  OHS/PL   ┌────────────────┐  ACL   ┌───────────────────┐
   │   Catalogue    │──────────▶│    Ordering    │◀───────│  Legacy ERP (BBoM) │
   │  (supporting)  │   U → D   │    (core)      │  D ← U │   (external)      │
   └────────────────┘           └───────┬────────┘        └───────────────────┘
                                        │ Customer–Supplier
                                        │ U → D   (events: OrderConfirmed…)
                        ┌───────────────┼────────────────┐
                        ▼               ▼                ▼
                ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                │  Fulfilment  │ │   Billing    │ │ Notifications│
                │ (supporting) │ │ (supporting) │ │  (generic)   │
                └──────────────┘ └──────────────┘ └──────────────┘
                        │ Conformist (carrier API is what it is)
                        ▼
                ┌──────────────┐
                │ Carrier SaaS │
                └──────────────┘
```

Reading it: Ordering consumes the Catalogue's published language and *conforms* to nothing — it protects itself from the ERP with an ACL. Downstream contexts consume Ordering's events. Fulfilment just conforms to the carrier's API because fighting it is not worth it.

## 6.4 The ACL in detail

The Anti-Corruption Layer is the single most under-used pattern in the catalogue. It is a facade + adapter + translator that sits at the boundary and converts the *foreign* model into *our* model:

```java
// Foreign model (legacy ERP): flat, stringly typed, its own vocabulary
record ErpCustomerRecord(String custNo, String stat, String addrLine1, String addrLine2,
                         String cty, String pcode, String ctry, String crLimit) {}

// Our model (Ordering context)
record Customer(CustomerId id, CustomerStanding standing, Address address, Money creditLimit) {}

// The ACL: the only place that knows both
final class ErpCustomerAdapter implements CustomerLookup {
    private final ErpClient erp;

    public Optional<Customer> find(CustomerId id) {
        return erp.fetchCustomer(id.value()).map(this::translate);
    }

    private Customer translate(ErpCustomerRecord r) {
        var standing = switch (r.stat()) {
            case "A", "AC" -> CustomerStanding.GOOD;
            case "H"       -> CustomerStanding.ON_HOLD;
            case "X", "CL" -> CustomerStanding.CLOSED;
            default        -> throw new UnknownErpStatus(r.stat());   // fail loudly, don't propagate junk
        };
        return new Customer(new CustomerId(r.custNo()), standing,
                            Address.parse(r.addrLine1(), r.addrLine2(), r.cty(), r.pcode(), r.ctry()),
                            Money.of(new BigDecimal(r.crLimit()), Currency.GBP));
    }
}
```

Note the properties: the ERP's status codes never appear outside this class; unknown codes fail fast rather than becoming a mystery `null`; and if the ERP is replaced, this is the only file that changes.

Cost: for a rich integration, an ACL is 5–15% of the context's code. Benefit: the other 85–95% is not contaminated. This is almost always the right trade.

## 6.5 How it fails

- **The map is never drawn.** Relationships are implicit, so nobody knows who is upstream. When Catalogue changes a field, Ordering breaks, and the argument about whose fault it is takes a week.
- **Conformist by accident.** A downstream team uses the upstream's DTOs directly "to save time." Two years later, the downstream's entire model is shaped by an upstream it does not control.
- **Shared Kernel creep.** The kernel starts as `Money` and `Id`s, grows to include `Customer`, `Address`, `Product`, and is now a common library that couples everything.
- **Partnership between teams who do not talk.** Partnership only works with daily communication. Between teams in different time zones with different managers, it degenerates into mutual blocking.

## 6.6 What a senior engineer decides

- Every cross-context edge gets a named pattern and an upstream/downstream arrow. Put it in the repo, not a slide deck.
- Default to **ACL** for anything external or legacy, **OHS + Published Language** for anything with more than two consumers, and **Customer–Supplier** for internal pairs. Reserve Conformist for when the upstream is genuinely good and stable.
- When a Shared Kernel is proposed, cap it: a list of types, a named owner, a change process. If nobody wants to own it, it is not a kernel; it is a dumping ground.

---

# 7 — Subdomains and Core Domain Distillation: Where to Spend the Money

## 7.1 The problem

Not all parts of the business are equally important, but engineering effort spreads evenly by default. The team spends three months building a bespoke notification system while the pricing engine — the thing that actually makes the company money — is a spreadsheet uploaded nightly.

## 7.2 The mechanism

A **Subdomain** is a part of the business problem space. (A Bounded Context is a part of the *solution* space; ideally they align one-to-one, but in legacy systems they rarely do.) Evans classifies them:

| Type | Definition | Investment strategy | Example (retailer) |
|---|---|---|---|
| **Core** | What differentiates the business; the reason customers choose you; the thing competitors cannot easily copy | Best engineers, deepest modelling, in-house, iterated continuously | Personalised pricing and promotions; the recommendation engine; the fulfilment-routing optimiser |
| **Supporting** | Necessary, business-specific, but not a differentiator | Competent engineers, adequate modelling, in-house or contracted, kept simple | Order management, returns, inventory tracking |
| **Generic** | Necessary, not business-specific; every company needs it | Buy or adopt open source; do *not* build | Authentication, email sending, payments processing, accounting ledger |

The distinction is not "important vs unimportant." Authentication is critical — if it breaks, nothing works. But it is *generic*: your auth is no better than anyone else's, so you should not spend your scarce modelling talent on it. Buy Auth0/Cognito/Keycloak and move on.

Core is where the *model* is the product. A 10% better pricing model is worth millions; a 10% better auth system is worth nothing.

## 7.3 Distillation

Evans' "distillation" is the process of finding the core and making it visible:

- **Domain Vision Statement.** One page: what the core domain is and why it matters. If the team cannot write this, they do not know what they are building.
- **Highlighted Core.** In the code, the core is marked — a separate module, a `core` package, a different colour on the map — so that anyone can see which changes are to the crown jewels.
- **Segregated Core.** Refactor to pull the core out of the surrounding generic and supporting concepts, so it can be modelled deeply without dragging the rest along.
- **Generic Subdomains extracted.** Pull generic concerns out and replace with off-the-shelf where possible.
- **Abstract Core.** When the core is mature, its deepest concepts are extracted into an abstract model that the specialised parts implement.

## 7.4 Heuristics

- Ask the CEO "what would competitors have to copy to beat us?" That is the core. Ask them "what could we outsource tomorrow without customers noticing?" That is generic.
- **Core changes constantly.** It is where the business experiments. If a part of the system has not had a business-driven change in a year, it is not core, however much code it has.
- **Core is small.** Typically 10–20% of the codebase. If "everything is core," nothing is.
- **Core moves.** A startup's core is the product; at scale it might become the logistics network; later, the data. Re-evaluate yearly.
- **Supporting subdomains are where DDD-lite lives.** Reasonable model, ubiquitous language, but no need for event sourcing and six-way aggregate debates.

## 7.5 How it fails

- **Everything is core.** Every team believes their part is the differentiator; investment is flat; the actual core is starved.
- **Building generic.** The team writes its own auth, its own message queue, its own workflow engine. Each is worse than the off-the-shelf alternative and consumes the engineers who should be on the core.
- **Core mistaken for supporting.** The pricing engine is treated as a CRUD table of prices because "it's just data." The competitor with a real model wins.
- **Buying the core.** An off-the-shelf "commerce platform" implements the very thing you were supposed to be better at. Now your differentiation is limited to what the vendor's plugin API allows.

## 7.6 What a senior engineer decides

- Write the domain vision statement. Get the business to sign it. Revisit it every year.
- Staff the core with the strongest modellers and give them direct access to domain experts. Put the generic subdomains on a "buy" list.
- When a project proposal arrives, classify it: core, supporting, generic. Investment and design rigour follow the classification.

---

# 8 — Tactical Building Blocks: Entities and Value Objects

The strategic patterns decide *where* models live. The tactical patterns decide *what they are made of*. The first distinction — entity vs value — is the one most engineers get wrong first and keep getting wrong.

## 8.1 Entities: identity over time

An **Entity** is an object defined by its *identity*, not its attributes. Two entities with identical attributes are still different things; one entity whose attributes all change is still the same thing.

- A `Customer` is an entity: change their name, address, and email, and they are still the same customer. Two customers named "Jane Smith" at the same address are two customers.
- An `Order` is an entity: its lines, status, and totals change; its identity does not.

Properties:

- **Identity is assigned early and never changes.** Generate the ID *before* persistence (UUID, ULID, Snowflake, or a domain-meaningful key like an ISBN), not from a database sequence at insert time. Database-generated IDs force an insert before the entity can be referenced, which distorts the model around persistence.
- **Equality is by identity.** `equals`/`hashCode` on the ID only.
- **Entities have a life cycle**: created, mutated through well-named operations, eventually archived or deleted. The life cycle is a state machine, and it should be explicit.
- **Entities enforce their own invariants.** `order.addLine(...)` checks that the order is still open. There is no `setStatus`.

## 8.2 Value Objects: attributes without identity

A **Value Object** is defined entirely by its attributes. Two values with the same attributes are *the same value*. It has no identity, no life cycle, and — critically — it is **immutable**.

- `Money(amount=100, currency=GBP)` is a value. Any two are interchangeable.
- `Address(line1, city, postcode, country)` is a value. If the customer moves, you do not "update the address"; you *replace* it with a new one.
- `DateRange`, `EmailAddress`, `Quantity`, `Percentage`, `OrderLine` (usually), `Coordinates`, `PhoneNumber`, `CustomerId` itself.

Properties:

- **Immutable.** All operations return a new value: `money.plus(other)`, `range.extendTo(date)`.
- **Self-validating.** The constructor rejects invalid states. An `EmailAddress` that can hold `"not an email"` is a `String` with a misleading name.
- **Equality by attributes.** Structural `equals`/`hashCode`.
- **Side-effect-free functions.** Value operations are pure; that makes them trivially testable and safely shareable.
- **Rich behaviour.** `Money.allocate(ratios)` (split a total without losing pennies — Fowler's classic), `DateRange.overlaps(other)`, `Weight.inKilograms()`.

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        Objects.requireNonNull(amount); Objects.requireNonNull(currency);
        if (amount.scale() > currency.getDefaultFractionDigits())
            throw new IllegalArgumentException("Too many decimal places for " + currency);
    }
    public Money plus(Money other) {
        requireSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }
    /** Splits into n parts whose sum is exactly this amount (no lost pennies). */
    public List<Money> allocate(int n) {
        var unit = BigDecimal.ONE.movePointLeft(currency.getDefaultFractionDigits());
        var low = amount.divide(BigDecimal.valueOf(n), RoundingMode.DOWN);
        var remainder = amount.subtract(low.multiply(BigDecimal.valueOf(n)));
        var parts = new ArrayList<Money>(n);
        for (int i = 0; i < n; i++) {
            var extra = remainder.compareTo(BigDecimal.ZERO) > 0 ? unit : BigDecimal.ZERO;
            parts.add(new Money(low.add(extra), currency));
            remainder = remainder.subtract(extra);
        }
        return parts;
    }
    private void requireSameCurrency(Money o) {
        if (!currency.equals(o.currency)) throw new CurrencyMismatch(currency, o.currency);
    }
}
```

## 8.3 The rule of thumb: prefer values

Most engineers over-use entities because every ORM tutorial starts with `@Entity` and `@Id`. The DDD heuristic is the reverse:

> **Default to Value Object. Promote to Entity only if the business genuinely needs to track this thing's identity over time.**

The payoff is enormous. Values are immutable, so they are thread-safe, cacheable, trivially serialisable, and free of aliasing bugs. A model that is 80% values and 20% entities is dramatically easier to reason about than the reverse.

Classic promotion mistakes:

- `OrderLine` as an entity with its own ID and repository. Unless lines are referenced from outside the order (rare), they are values inside the `Order` aggregate.
- `Address` as an entity with an `addresses` table and a foreign key. Now "change address" is an UPDATE that silently alters the address on every historical order that referenced it. As a value, each order carries the address *as it was*.
- `Money` as a `BigDecimal` column plus a `currency` column that half the code forgets to check.

## 8.4 Primitive obsession

The most common tactical smell: domain concepts represented as `String`, `int`, `BigDecimal`, `UUID`. Symptoms:

- `void ship(UUID orderId, UUID warehouseId, UUID carrierId)` — call it with the arguments in the wrong order and the compiler is silent.
- `String email` validated in three places, differently.
- `int quantity` that is sometimes a count and sometimes a weight in grams.

Fix: wrap every domain concept in a value type — `OrderId`, `EmailAddress`, `Quantity`. The cost in Java/Kotlin/C#/TypeScript with records/data classes is one line per type. The benefit is that invalid states become unrepresentable and the compiler enforces the ubiquitous language.

## 8.5 What a senior engineer decides

- In design review, for every class ask: "Does the business track this thing's identity?" If not, make it a value.
- Ban `setX` on entities. Every mutation is a named business operation that checks invariants.
- Generate IDs in the domain, not the database.
- Wrap primitives. Treat a `String` parameter named `customerId` as a code-review finding.

---

# 9 — Aggregates: The Consistency Boundary

This is the most important tactical pattern and the one most often done wrong. It is also where DDD connects directly to distributed-systems engineering.

## 9.1 The problem

Business invariants span multiple objects. "An order's total must equal the sum of its lines." "A warehouse bin cannot hold more than its capacity." "A customer's outstanding balance cannot exceed their credit limit." Each of these must be enforced *atomically* — no interleaving of operations can leave the rule violated.

The naive approach is to enforce them in one big transaction that loads everything related. That has two failure modes:

1. **Contention.** If "place order" locks the customer, the order, every product on the order, and the warehouse, then two customers buying the same product serialise on the product row. At scale this is the p99 collapse from §2.3.
2. **Scope creep.** Transactions grow: someone adds "also update the loyalty points," then "also decrement the promotion budget." Now one transaction spans five tables owned by four teams, and the deadlock graph is a business-process diagram.

The other naive approach — no boundaries, update objects individually — means invariants are enforced nowhere and data is inconsistent.

## 9.2 The mechanism

An **Aggregate** is a cluster of entities and values that is treated as a single unit for the purpose of data changes. It has:

- **A root entity** (the Aggregate Root). External objects hold references only to the root, only by its ID.
- **A boundary.** Everything inside is owned by the root and is only reachable through it. Nothing outside can hold a reference to an internal entity.
- **Invariants** that are guaranteed to hold at the end of every operation on the aggregate.
- **A transaction scope.** One aggregate instance is loaded, modified, and saved in one transaction. Nothing else.

Evans:

> Cluster the Entities and Value Objects into Aggregates and define boundaries around each. Choose one Entity to be the root of each Aggregate, and allow external objects to hold references to the root only. Define properties and invariants for the Aggregate as a whole and give enforcement responsibility to the root.

The consequences are what make the pattern powerful:

- **The aggregate is the unit of consistency.** Invariants *inside* an aggregate are strongly consistent. Invariants *across* aggregates are eventually consistent (via domain events, §10). This is not a limitation; it is a design decision that forces the question "does this *really* need to be atomic?"
- **The aggregate is the unit of concurrency.** One aggregate, one optimistic-lock version. Two users editing different aggregates never conflict.
- **The aggregate is the unit of distribution.** An aggregate lives entirely on one node / one shard / one partition. Its ID is the partition key. This is why aggregates map cleanly to Kafka partitions, DynamoDB items, Cassandra partitions, and actor-model actors.
- **The aggregate is the unit of loading.** A repository loads whole aggregates, never fragments.

## 9.3 Vernon's four rules of aggregate design

From "Effective Aggregate Design" (Vernon, 2011), the most-cited practical guidance:

**Rule 1: Model true invariants in consistency boundaries.** Only cluster things together that *must* be transactionally consistent. "Would be nice to be consistent" does not count. Ask the domain expert: "If these two things were out of sync for two seconds, what would break?" Usually: nothing.

**Rule 2: Design small aggregates.** The default should be a root entity plus values. Add a child entity only when an invariant demands it. Large aggregates:
- Load slowly (an `Order` with 10,000 lines loads 10,000 rows to add one).
- Contend heavily (every operation on any part locks the whole).
- Fail on optimistic locking (two users editing different lines of the same order conflict).

A useful threshold: if an aggregate could hold more than ~100 child entities in normal operation, that child is probably its own aggregate.

**Rule 3: Reference other aggregates by identity only.** `Order` holds `CustomerId`, not `Customer`. This prevents accidental traversal (`order.getCustomer().getOrders().get(0).getCustomer()...`), keeps loads small, and makes the aggregate independently shardable.

**Rule 4: Use eventual consistency outside the boundary.** When a change in aggregate A requires a change in aggregate B, A raises a domain event; a handler updates B in a *separate* transaction. If the business truly cannot tolerate the delay, the boundary is wrong — merge them.

## 9.4 A worked sizing decision

Warehouse bins and stock. Naive aggregate: `Warehouse` containing `Bin`s containing `StockItem`s. Invariant: "a bin's total volume cannot exceed capacity."

- **Warehouse as aggregate:** every stock movement anywhere in the warehouse locks the entire warehouse. 500 pickers → serialised. Rejected.
- **Bin as aggregate:** a stock movement locks one bin (source) and one bin (destination). Two aggregates in one operation — a rule violation. But the invariant is *per bin*, so each bin can be updated in its own transaction: remove from source (commit), add to destination (commit), with a `StockMoveStarted` event to reconcile if the second step fails. This is a saga (§15).
- **StockItem as aggregate:** no way to enforce bin capacity. Rejected.

Bin wins. Note the reasoning was entirely about *invariants* and *contention*, not about "what is the natural noun."

## 9.5 Aggregate code shape

```java
public final class Order {                                 // aggregate root
    private final OrderId id;
    private final CustomerId customerId;                   // reference by ID, rule 3
    private OrderStatus status;
    private final List<OrderLine> lines = new ArrayList<>();   // values, not entities
    private final List<DomainEvent> pendingEvents = new ArrayList<>();
    private long version;                                  // optimistic concurrency

    public static Order place(OrderId id, CustomerId customerId, List<OrderLine> lines, Clock clock) {
        if (lines.isEmpty()) throw new EmptyOrder(id);
        var order = new Order(id, customerId, OrderStatus.PLACED, lines);
        order.raise(new OrderPlaced(id, customerId, order.total(), clock.instant()));
        return order;
    }

    public void addLine(OrderLine line) {
        requireStatus(OrderStatus.PLACED, "add a line");     // invariant: only open orders change
        lines.add(line);
        raise(new OrderLineAdded(id, line));
    }

    public void confirm(Clock clock) {
        requireStatus(OrderStatus.PLACED, "confirm");
        if (total().isZero()) throw new CannotConfirmZeroValueOrder(id);
        status = OrderStatus.CONFIRMED;
        raise(new OrderConfirmed(id, customerId, total(), clock.instant()));
    }

    public Money total() {                                   // derived, never stored inconsistently
        return lines.stream().map(OrderLine::subtotal).reduce(Money.zero(GBP), Money::plus);
    }

    private void requireStatus(OrderStatus expected, String action) {
        if (status != expected) throw new InvalidOrderTransition(id, status, action);
    }
    private void raise(DomainEvent e) { pendingEvents.add(e); }
    public List<DomainEvent> pullEvents() { var out = List.copyOf(pendingEvents); pendingEvents.clear(); return out; }
}
```

Things to notice:

- No public constructor; a named factory `place(...)` that establishes invariants and raises the creation event.
- No setters. Every mutation is a verb from the ubiquitous language with its own precondition.
- `total()` is derived. Storing it as a field is a denormalisation that must be justified by measurement.
- Events are collected and pulled by the application layer after the transaction commits (§10, §15 outbox).
- The `version` field is the optimistic-lock token. Concurrent modification of the same order → one wins, the other gets a `ConcurrentModification` and retries or reports.

## 9.6 Numbers

- **Aggregate size:** root + values + 0–a-few child entities. Loading should be one query or a small fixed number, well under 10 ms from a local database.
- **Transaction per aggregate:** exactly one aggregate modified per transaction. Reading others is fine; writing two is a design smell to be justified in review.
- **Optimistic-lock conflict rate:** if >1% of writes to an aggregate type conflict, the aggregate is too large or too hot. Split it or redesign the operation (e.g. a `Counter` that increments via a database atomic op rather than load-modify-save).
- **Event fan-out:** one aggregate operation raising >5 events is usually doing too much.

## 9.7 How it fails

- **The god aggregate.** `Customer` containing all their orders, all their addresses, all their payment methods, all their support tickets. Loading a customer to change their email pulls 40 MB. Two operations on different orders conflict on the customer's version.
- **Aggregate as ORM object graph.** `@OneToMany(fetch = EAGER)` everywhere. The "aggregate" is whatever the ORM decides to load, which is everything.
- **Cross-aggregate transactions "just this once."** Someone needs to update `Order` and `Inventory` atomically. They do it in one transaction "temporarily." Two years later there are 30 such places and the deadlocks are back.
- **Anemic aggregate.** The root has getters and setters; invariants are checked in a service. The aggregate is a boundary with nothing inside enforcing it.
- **Ignoring concurrency.** No version field; last write wins; a customer's two browser tabs silently overwrite each other's changes.

## 9.8 What a senior engineer decides

- For every proposed aggregate, write down its invariants. If there are none spanning multiple entities, it should be a root plus values.
- Enforce "one aggregate per transaction" in review. Exceptions need a written justification and a plan to remove them.
- Make the aggregate ID the partition/shard key from day one. This is the single most important scaling decision that DDD makes for free.
- Add optimistic-lock versioning to every aggregate. It costs one column.

---

# 10 — Domain Events: Making the Past Explicit

## 10.1 The problem

Things happen in a domain, and other parts of the domain — or other contexts — need to know. The naive approach is for the code that does the thing to also call everything that cares: `placeOrder()` calls `reserveStock()`, `sendConfirmationEmail()`, `updateLoyaltyPoints()`, `notifyFraudCheck()`. Each addition couples the order code to another concern. The order code now knows about email templates. A failure in loyalty points fails the order. And the domain expert, asked "what happens when an order is placed?", can answer immediately — but the code's answer is spread across a 400-line method.

## 10.2 The mechanism

A **Domain Event** is an immutable record of something that *happened* in the domain, expressed in the ubiquitous language, in the past tense, and significant to domain experts. `OrderPlaced`. `PaymentAuthorised`. `ShipmentDispatched`. `CustomerCreditLimitExceeded`.

Properties:

- **Immutable and past tense.** It is a fact. It cannot be un-happened.
- **Named in the ubiquitous language.** If a domain expert would not say it, it is not a domain event (it may be an integration or technical event).
- **Carries what happened, not everything.** `OrderConfirmed { orderId, customerId, total, occurredAt }`. Not the whole order. Consumers who need more fetch it.
- **Raised by aggregates.** The aggregate that made the decision raises the event as part of the same operation.
- **Handled asynchronously (usually), after commit.** Handlers run in their own transactions, possibly in other contexts, possibly on other machines. This is how eventual consistency across aggregates is implemented (§9.3 rule 4).

Two flavours, which must not be confused:

| | Domain event (internal) | Integration event (published) |
|---|---|---|
| Audience | Same bounded context | Other contexts |
| Schema | Can change freely with the model | Published Language; versioned; backward-compatible |
| Content | Rich, references domain types | Flat, primitives, self-contained |
| Transport | In-process dispatch, after commit | Message broker, outbox pattern (§15) |
| Coupling | Internal | Contract |

The mistake is to publish internal domain events directly onto Kafka for other contexts to consume. Now every refactor of your model is a breaking change for someone else. Translate at the boundary: an internal `OrderConfirmed` becomes a published `ordering.order-confirmed.v2` with a governed schema.

## 10.3 Event-driven invariants across aggregates

"When an order is confirmed, reserve the stock." Order and Stock are different aggregates (different contention profiles, different owners). So:

1. `Order.confirm()` raises `OrderConfirmed`. Transaction commits.
2. After commit, `OrderConfirmed` is dispatched.
3. A handler in the Inventory context loads each `StockItem` aggregate and calls `reserve(orderId, qty)`. Each in its own transaction.
4. If reservation fails (insufficient stock), the handler raises `StockReservationFailed`, and a handler in Ordering calls `order.markUnfulfillable(reason)`.

The order was "confirmed" for perhaps 50 ms before stock was reserved. The business asked "what if a customer sees a confirmed order and then it's unfulfillable?" and answered "that already happens today with the warehouse; we email them." The eventual consistency was acceptable *because we asked*.

## 10.4 Event naming discipline

| Bad | Why | Good |
|---|---|---|
| `OrderEvent` | Not a fact; a category | `OrderPlaced`, `OrderConfirmed` |
| `OrderUpdated` | Which update? Consumers must diff | `OrderLineAdded`, `ShippingAddressChanged` |
| `SendEmailCommand` | A command, not an event; tells someone what to do | `OrderConfirmed` (the email handler decides to email) |
| `OrderStatusChanged { from, to }` | Leaks the state machine as data; every consumer re-implements the transitions | One event per meaningful transition |
| `OrderPlacedV2DTO` | Technical noise | `OrderPlaced` (versioning belongs in the published schema, not the domain type) |

## 10.5 How it fails

- **Events as RPC.** `OrderPlaced` handler *synchronously* calls the payment gateway, and if it fails, the order-placement transaction rolls back. This is a synchronous call with extra indirection, and it couples availability.
- **Events published before commit.** The handler reads the database, finds no order (transaction not yet committed), and fails. Solution: dispatch after commit, or use the outbox (§15).
- **Lost events.** Publish to the broker after commit; the process crashes between commit and publish; the event is gone. Solution: outbox (§15).
- **Fat events.** The event carries the entire aggregate. Every consumer now depends on the whole aggregate's shape.
- **Event storms.** Every setter raises an event; the stream is noise; nobody can find the significant ones.

## 10.6 What a senior engineer decides

- Every aggregate operation that matters to the business raises exactly one well-named event.
- Internal events and integration events are separate types with a translation step.
- Dispatch after commit, via an outbox, always. "We'll add reliability later" means "we'll debug lost events in production."
- Keep a catalogue of events per context alongside the glossary. It is the best documentation of what the system *does*.

---

# 11 — Repositories, Factories, Domain Services, and Application Services

## 11.1 Repositories: collections of aggregates

A **Repository** presents the illusion of an in-memory collection of aggregate roots. `orders.findById(id)`, `orders.save(order)`, `orders.findAwaitingConfirmationOlderThan(cutoff)`. It hides persistence completely: the domain neither knows nor cares whether it is Postgres, DynamoDB, or an event store.

Rules:

- **One repository per aggregate root.** Not per entity, not per table. If you have an `OrderLineRepository`, `OrderLine` has been wrongly promoted to an aggregate.
- **Returns whole aggregates.** Never a partially loaded one.
- **The interface lives in the domain; the implementation lives in infrastructure.** The domain defines `interface OrderRepository`; the `PostgresOrderRepository` is injected.
- **Not a query layer.** Repositories serve the *write* side (load an aggregate to mutate it). Complex read queries — dashboards, search, reports — bypass the domain and go straight to the database or a read model (§14 CQRS). Forcing reports through aggregates is how you end up loading 10,000 aggregates to sum a column.

## 11.2 Factories: complicated creation

When creating an aggregate is non-trivial — requires other objects, validation across several inputs, or a choice of concrete type — a **Factory** encapsulates it. Often a static method on the root (`Order.place(...)`); sometimes a separate class when the creation needs collaborators (e.g. a `PolicyFactory` that consults `UnderwritingRules`). The point is that an aggregate cannot be constructed in an invalid state.

## 11.3 Domain Services: operations that belong to no one entity

Some domain operations do not naturally belong to any single entity or value: transferring money between two accounts, calculating a shipping quote that depends on the order, the warehouse, and the carrier tariffs. Forcing these onto one entity distorts it. A **Domain Service** is a stateless operation, named in the ubiquitous language, that takes domain objects and does domain work:

```java
public final class FundsTransfer {          // domain service
    public TransferResult transfer(Account from, Account to, Money amount) {
        from.debit(amount);                  // each aggregate enforces its own invariants
        to.credit(amount);
        return new TransferResult(from.id(), to.id(), amount);
    }
}
```

Warning: this is the pattern most easily abused. Every "put the logic in a service" instinct from the anemic model masquerades as a domain service. The test: **does the operation involve a rule that a domain expert would recognise, and does it genuinely not fit on any entity?** If the logic could live on `Account`, it should.

Domain services contain *domain* logic. They do not touch databases, HTTP, queues, or clocks except via injected domain-facing abstractions.

## 11.4 Application Services: use-case orchestration

An **Application Service** (also "use case", "command handler", "interactor") is the entry point for one use case. It:

1. Receives a command (from a controller, a message handler, a CLI).
2. Starts a transaction.
3. Loads the aggregate(s) via repositories.
4. Calls domain methods.
5. Saves the aggregate.
6. Commits; dispatches events.
7. Returns a result.

It contains **no business logic**. If there is an `if` about the domain in an application service, it belongs in an entity or domain service.

```java
public final class ConfirmOrderHandler {
    private final OrderRepository orders;
    private final Clock clock;
    private final UnitOfWork uow;

    public void handle(ConfirmOrder cmd) {
        uow.run(() -> {
            var order = orders.findById(cmd.orderId()).orElseThrow(() -> new OrderNotFound(cmd.orderId()));
            order.confirm(clock);                        // all the rules are in here
            orders.save(order);                          // includes outbox write of pulled events
        });
    }
}
```

Application services are thin. A 20-line application service is typical; a 200-line one is hiding domain logic.

## 11.5 The layering in one picture

```
Inbound adapters      REST controller · gRPC handler · Kafka consumer · CLI · scheduler
                            │ commands / queries
Application layer     ConfirmOrderHandler · PlaceOrderHandler · OrderSummaryQuery
                            │ uses
Domain layer          Order (aggregate) · Money · OrderConfirmed · FundsTransfer · OrderRepository (interface)
                            │ abstractions implemented by
Outbound adapters     PostgresOrderRepository · KafkaEventPublisher · StripePaymentGateway · SystemClock
```

Dependencies point *inward*. The domain layer imports nothing from the outer layers. This is the Hexagonal / Ports-and-Adapters / Clean Architecture shape, covered in §12.

## 11.6 How it fails

- **Repositories that leak the ORM.** `orders.findAll(Specification<OrderEntity>)` returning JPA-managed entities that lazy-load on access outside the transaction. The domain is now dependent on the persistence framework's session lifecycle.
- **Generic repositories.** `Repository<T>` with `findAll()`, `save()`, `delete()` for every table. It is a DAO with a fashionable name. Repositories are per-aggregate with domain-meaningful query methods.
- **Domain services as the new service layer.** All logic in `OrderDomainService`; entities are anemic. Nothing has changed except the package name.
- **Application services that know the domain.** Business rules in the handler because "it's easier to test." It is not; it is easier to *mock*, which is not the same thing.

---

# 12 — Architecture: Layers, Hexagons, and Where the Domain Lives

## 12.1 The problem

Tactical patterns need somewhere to live that keeps the domain free of infrastructure. Classic three-tier architecture (UI → business → data) makes the business layer depend on the data layer: the domain imports the ORM, the domain objects are shaped by tables, and testing the domain requires a database.

## 12.2 The mechanism: dependency inversion around the domain

Alistair Cockburn's **Hexagonal Architecture** (2005), also called Ports and Adapters, and the equivalent Onion (Palermo) and Clean (Martin) architectures all make the same move: *the domain is the centre, and everything else depends on it, never the reverse.*

- **Ports** are interfaces defined by the domain/application layer describing what it needs (`OrderRepository`, `PaymentGateway`, `Clock`) and what it offers (`ConfirmOrderUseCase`).
- **Adapters** implement ports for specific technologies: `PostgresOrderRepository`, `StripePaymentGateway`, `RestOrderController`.
- **Inbound adapters** (driving) call the application layer. **Outbound adapters** (driven) are called by it, through ports.

The result:

- The domain has zero framework imports. It can be unit-tested with plain objects at 10,000+ tests per second.
- Swapping Postgres for DynamoDB, or REST for gRPC, changes adapters only.
- The domain model's shape is determined by the *domain*, not by the table layout or the JSON payload.

## 12.3 Enforcing it

A layering that is not enforced decays in a year. Enforce it mechanically:

- **Module boundaries.** Separate compilation units: `ordering-domain` has no dependency on `ordering-infrastructure`; the build fails if someone adds one. Java modules, Gradle subprojects, .NET projects, Go internal packages, Rust crates.
- **Architecture tests.** ArchUnit (JVM), NetArchTest (.NET), dependency-cruiser (JS/TS), import-linter (Python): `classes in ..domain.. should not depend on classes in ..infrastructure..`, `..domain.. should not depend on javax.persistence`, run in CI.
- **Package-by-context, not by layer.** Top-level packages are `ordering`, `fulfilment`, `billing` — bounded contexts — and *inside* each are `domain`, `application`, `infrastructure`. Package-by-layer (`controllers/`, `services/`, `repositories/` at the top) makes contexts invisible and encourages cross-context coupling.

## 12.4 The modular monolith

A frequently overlooked option: one deployable, many bounded contexts, with the boundaries enforced by module rules and separate schemas. It gives most of the design benefit of microservices (independent models, clear ownership) without the operational cost (network, partial failure, distributed tracing, deployment orchestration). Extract a module into a service *when there is a concrete reason*: independent scaling, independent deployment cadence, different technology, or team autonomy that the monolith is actually blocking. Shopify, Stack Overflow, and many well-run companies run this way.

Rules that make it work:

- One schema (or database) per context. No cross-schema joins.
- Cross-context calls go through the context's public API (an interface in a well-named package), never through internal classes.
- Cross-context data flow uses in-process domain events — the same shape you would use with a broker, so extraction later is mechanical.

## 12.5 How it fails

- **Hexagon with an anemic centre.** All the ports and adapters, domain layer is DTOs, logic in application services. Architecture theatre.
- **Adapters bleeding in.** The domain has `@Entity`, `@JsonProperty`, `@Column` annotations. The "clean" layering is decorative.
- **Too many layers.** DTO → command → domain → entity → row, with mappers between each. Six classes per concept. The mapping cost is real; only introduce a layer when it separates things that change for different reasons.
- **Big-bang microservices.** Contexts extracted before they are understood, with the boundaries in the wrong place, and now every boundary mistake costs a network hop and a distributed transaction.

## 12.6 What a senior engineer decides

- Domain module with no framework dependencies, enforced by build and architecture tests. Non-negotiable in the core domain.
- Package by context first.
- Default to a modular monolith; extract to services on evidence, not on fashion.

---

# 13 — The Modelling Process: Knowledge Crunching and EventStorming

## 13.1 The problem

Everything above describes *what* a good model looks like. None of it says *how to get one*. The model lives in domain experts' heads, tacitly, inconsistently, and full of exceptions they forgot to mention. Extracting it via requirements documents does not work; the document is a lossy snapshot and the expert's real knowledge only surfaces when they see the model doing something wrong.

## 13.2 Knowledge crunching

Evans' term for the iterative, collaborative process of building the model: domain experts and engineers in a room, talking through scenarios, sketching, trying a model, finding where it breaks, refining. The key attitudes:

- **Engineers must learn the domain.** Not "gather requirements" — *learn it*, the way a new hire in the business would. A payments engineer should be able to explain a chargeback to a merchant.
- **Domain experts must engage with the model.** They are not just a source of requirements; they are co-designers. They need to see the model (in language, diagrams, and ideally running software) and say "no, that's not how it works."
- **Scenarios, not abstractions.** "Walk me through what happens when a customer returns one item from a three-item order where they used a 20%-off promo code" surfaces more than "describe the returns process."
- **Refactoring toward deeper insight** (Evans Part III). The first model is always wrong. Every breakthrough — "oh, a *reservation* and an *allocation* are different things" — is a refactoring of the code and the language together. Budget for it. The best models are found on the third or fourth iteration.

## 13.3 EventStorming

Alberto Brandolini's **EventStorming** (2013) is the most effective workshop format for knowledge crunching at scale. The full "big picture" form:

**Setup.** A wall 8–10 metres long covered in paper. Sticky notes in fixed colours. Everyone who knows anything — domain experts, engineers, product, ops, support. Half a day to two days. No chairs.

**Phase 1: Domain events (orange).** Everyone writes events — past tense, business language — and puts them on the wall in rough time order. "Order placed." "Payment declined." "Shipment lost by carrier." "Customer complained." Chaos is the point: in 30 minutes you have 200–500 events, and duplicates and disagreements are surfaced by the wall, not by a meeting.

**Phase 2: Enforce the timeline.** Sort. Merge duplicates. Argue about ordering ("does stock get reserved before or after payment?" — the answer is often "it depends" and the *it depends* is a business rule nobody had written down).

**Phase 3: Hotspots (pink/red).** Wherever there is disagreement, uncertainty, or someone says "that's a known problem," mark it. These are the places where the model is contested and where the most value lies.

**Phase 4: Commands (blue) and actors (yellow).** What triggered each event? Who did it? "Place order" (customer) → "Order placed." "Cancel order" (support agent) → "Order cancelled."

**Phase 5: Policies (lilac) — "whenever X, then Y".** The reactive rules. "Whenever payment is declined, then notify the customer and hold the order for 24 hours." Policies become event handlers and sagas.

**Phase 6: Read models (green).** What information does the actor need to decide to issue the command? This becomes the query side.

**Phase 7: Aggregates (yellow, large) and bounded contexts.** Group commands and events around the thing that decides: "Order" accepts `PlaceOrder`, `CancelOrder`, raises `OrderPlaced`, `OrderCancelled`. Draw lines where vocabulary changes. The lines are context boundaries.

The output is a wall that *is* the model: events (→ domain events), commands (→ application services), aggregates, policies (→ handlers/sagas), read models (→ queries), and boundaries (→ contexts). Photograph it; then build the glossary and the context map from it.

**Design-level EventStorming** repeats the process for one context at finer grain, and is the direct input to aggregate design.

## 13.4 Other techniques

- **Domain Storytelling** (Hofer & Schwentner): pictographic stories of actors and work objects, good for processes with many hand-offs.
- **Example Mapping** (Matts): for one rule, collect concrete examples until the rule is clear; feeds BDD tests directly.
- **Bounded Context Canvas** (Tune): a one-page template per context — purpose, ubiquitous language, inbound/outbound communication, business decisions, assumptions — that forces the boundary questions.
- **Context Mapping workshops**: the context map from §6, done as a group with the teams present.
- **Wardley Mapping** for the core/supporting/generic classification: where is each capability on the genesis → commodity axis?

## 13.5 How it fails

- **Modelling by engineers alone.** The model is a clean abstraction of a domain nobody in the room understood.
- **Modelling by document.** A 60-page requirements doc, read once, never updated. All the exceptions are discovered in UAT.
- **One-shot workshop.** EventStorming once, then two years of building. The model is refined only through bugs.
- **Analysis paralysis.** Six months of modelling, no software. The model is untested; running code is the fastest way to find where it is wrong.
- **Experts who are not experts.** The "domain expert" in the room is a business analyst three hops from the actual practice. Get the person who does the work.

## 13.6 What a senior engineer decides

- Start every new domain with a big-picture EventStorming. Two days with the right people replaces six months of discovery-by-incident.
- Keep domain experts in the loop continuously: a weekly modelling session, demo of running software in their language, glossary reviews.
- Treat model breakthroughs as high-priority refactors, not technical debt.

---

# 14 — Persistence: ORMs, Event Sourcing, and CQRS

## 14.1 The ORM problem

Object-relational mappers were built for the anemic model: map a class to a table, a field to a column, a reference to a foreign key. Rich aggregates fight this in predictable ways:

- **Value objects vs tables.** A `Money` field is two columns; an `Address` is five. Embeddables help; collections of values (`List<OrderLine>`) need element collections or a child table, and the ORM wants an ID on each.
- **Encapsulation vs field access.** The ORM needs to set private fields at load time. Reflection or field access mode; fine, but it means the "no-arg constructor" and mutable fields the ORM wants leak into the domain.
- **Lazy loading vs aggregate boundaries.** The ORM's lazy proxy lets `order.getCustomer()` silently load another aggregate outside its boundary — exactly what rule 3 forbids. Reference by ID instead, and the temptation disappears.
- **Identity map / session vs repositories.** The ORM's unit of work and the DDD unit of work overlap; use one, not both.

Options, in rough order of fidelity to the model:

| Approach | Fit for rich aggregates | Cost |
|---|---|---|
| ORM with entity annotations on domain classes | Medium; leaks into domain | Lowest effort |
| ORM with separate persistence model + mapper | High; domain is pure | One mapper per aggregate; ~10–20% more code |
| Document store (aggregate = document) | High; natural fit | Query flexibility lower; cross-aggregate queries via read models |
| Hand-written SQL per repository | High; full control | Most code; no magic to debug either |
| Event sourcing | Highest for the write side | See below |

The "aggregate as JSON document" approach deserves emphasis: an aggregate is by definition a self-contained unit loaded and saved whole, which is exactly what a document database (or a `jsonb` column in Postgres) does well. It removes the object-relational impedance mismatch entirely for the write side, at the cost of doing ad-hoc queries elsewhere.

## 14.2 Event sourcing

**Event Sourcing** (ES) persists aggregates not as current state but as the sequence of domain events that produced it. To load an aggregate, read its event stream and replay: `Order.rehydrate(events)`. To save, append the new events.

What it gives:

- **A complete audit trail** for free. Not "who last changed this" but "every decision, in order, with the data that informed it." In regulated domains (finance, healthcare) this is the reason to do it.
- **Temporal queries.** "What was the customer's credit limit on March 3rd?" Replay to that point.
- **Retroactive fixes.** A bug in a projection? Fix the projector and replay. (A bug in an *event*? Much harder; see below.)
- **Natural fit with domain events.** The events you raise for integration *are* the persistence.

What it costs:

- **Every read is a replay.** An aggregate with 10,000 events takes time to load. Mitigation: snapshots every N events. Now you maintain snapshot invalidation.
- **Events are forever.** A schema change to an event means either upcasting (translating old events on read, forever) or a migration (rewriting history, which defeats the purpose). Design events as if they will be read in ten years, because they will.
- **Queries need projections.** You cannot `SELECT * FROM orders WHERE status = 'CONFIRMED'`. Every query is a projection you build and maintain. This forces CQRS (below).
- **Operational maturity.** Event stores (EventStoreDB, Axon, Marten, home-grown on Postgres/Kafka) are less understood by most ops teams than a relational database.
- **Eventual consistency between write and read** unless you project synchronously.

Vernon and Evans both say: event sourcing is a *tactical* choice for a *specific aggregate* in a *specific context*, not an architecture. Apply it to the core aggregates where audit and temporal reasoning matter — the ledger, the policy, the order — and use ordinary state persistence everywhere else.

## 14.3 CQRS

**Command Query Responsibility Segregation** (Greg Young, ~2010, from Meyer's command–query separation) splits the model in two:

- The **write model**: aggregates, invariants, commands. Optimised for consistency.
- The **read model(s)**: denormalised, query-shaped projections. Optimised for reads. Possibly a different database (Elasticsearch for search, a materialised view for dashboards, Redis for hot lookups).

Reads never go through aggregates. Writes never go through read models. Events (or change-data-capture) keep the read side updated.

Why it matters for DDD: the write side can be a clean, small, invariant-focused model because it no longer has to also serve the "order list with customer name, product thumbnails, and shipment status" screen. Trying to serve that screen from aggregates is how aggregates become god objects (§9.7).

Degrees of CQRS:

1. **Same database, different code paths.** Commands load aggregates; queries run SQL directly and return DTOs. Cheap, and the right default for most systems.
2. **Same database, separate read tables/views.** Projections updated in the same transaction. Strongly consistent reads.
3. **Separate read store, updated by events.** Eventually consistent. Needed when the read shape or scale demands it.

Most systems should stop at level 1 or 2. Level 3 introduces the "I just saved it, why doesn't it appear in the list?" problem, which the UI must handle (optimistic UI, read-your-writes via version tokens).

## 14.4 Numbers

- **Replay cost:** ~1–5 µs per event for in-memory application; the bottleneck is fetching them. Stream of 1,000 events from Postgres: ~5–20 ms. Snapshot every 100–500 events.
- **Projection lag** at level 3: 10–500 ms typical over Kafka; design the UI for it.
- **Event size:** aim for < 1 KB. Events over 10 KB usually carry state that belongs in a read model.
- **Event schema churn:** a healthy core aggregate changes its event schema a few times a year. Each change needs an upcaster. If you are changing event schemas weekly, the model is not stable enough for ES yet.

## 14.5 How it fails

- **ES everywhere.** Every CRUD table event-sourced. The team spends its life writing projections for reference data.
- **Events that are state diffs.** `OrderUpdated { before, after }`. This is a changelog, not a domain event; it captures none of the *intent*, so the audit trail answers "what changed" but not "why."
- **Rewriting history.** A migration script "fixes" old events. The audit trail is now a lie.
- **CQRS as two full copies of the model.** The read side has its own aggregates and business logic. Now there are two models to keep consistent.

## 14.6 What a senior engineer decides

- Default: state persistence with a separate persistence model or aggregate-as-document; CQRS level 1.
- Event sourcing for aggregates where the business needs audit, temporal queries, or the events are the product — and only after the model has stabilised.
- Level-3 CQRS on measured need (read scale or read shape), never speculatively.

---

# 15 — DDD in Distributed Systems: Microservices, Sagas, and the Outbox

## 15.1 The relationship

Microservices and DDD are frequently conflated. The actual relationship:

- **Bounded contexts are the correct unit for service boundaries.** Sam Newman's *Building Microservices* says so directly; nearly every microservices post-mortem traces back to boundaries drawn by noun or by team rather than by context.
- **A bounded context can contain several services** (e.g. a context with a command service, a projection service, and a scheduler) but **a service must never span contexts** — that is the distributed god object.
- **Aggregates are the unit of data ownership.** Each aggregate type is owned by exactly one service. No other service writes it; no other service reads its tables.
- **Context maps become API and event contracts.** Customer–Supplier becomes an API with a consumer-driven contract test. OHS/Published Language becomes a versioned schema in a registry. ACL becomes an adapter service or module.

## 15.2 The consistency problem, now distributed

Inside a monolith, "one aggregate per transaction" was a discipline. Across services it is a physical constraint: there is no shared transaction. Every business process that spans aggregates in different services is a **saga** — a sequence of local transactions with compensating actions for failure.

Two shapes:

**Choreography.** Each service reacts to events and emits its own. Ordering emits `OrderConfirmed`; Inventory reserves and emits `StockReserved`; Payment captures and emits `PaymentCaptured`; Fulfilment ships. On failure, `StockReservationFailed` → Ordering cancels. No central coordinator. Simple for 3–4 steps; unreadable at 8+ because the process exists only as the sum of handlers.

**Orchestration.** A saga coordinator (often itself an aggregate: `OrderFulfilmentSaga`) holds the process state, sends commands, awaits replies, and runs compensations. The process is explicit and can be visualised, timed out, and retried. More moving parts; much easier to operate. Temporal, Camunda, and Axon Saga are the productised versions.

Compensation is a *business* concept, not a technical rollback. "Un-reserve stock" is easy. "Un-send email" is impossible; the compensation is "send an apology email." "Un-capture payment" is a refund, which has its own fees and delays. Model compensations in the ubiquitous language with the domain experts; they know what "undo" means in their world, and it is rarely symmetric.

## 15.3 The outbox pattern

The problem: an aggregate change and its event must be atomic. Write the aggregate, then publish to Kafka — if the process dies between them, the event is lost (or, if you publish first, a phantom event describes a change that was rolled back).

The **transactional outbox**: in the same database transaction as the aggregate save, insert the events into an `outbox` table. A separate relay (polling, or change-data-capture via Debezium) reads the outbox and publishes to the broker, marking rows as sent. Delivery becomes at-least-once; consumers must be idempotent (dedupe by event ID).

```
BEGIN;
  UPDATE orders SET status='CONFIRMED', version=version+1 WHERE id=$1 AND version=$2;
  INSERT INTO outbox (id, aggregate_id, type, payload, created_at) VALUES (...);
COMMIT;
-- relay: SELECT ... FROM outbox WHERE sent_at IS NULL ORDER BY created_at LIMIT 100 → publish → UPDATE sent_at
```

Cost: one extra insert per write, one relay process. Benefit: the single most common source of "the systems disagree" bugs is eliminated. This is table stakes, not an optimisation. See the exactly-once note in this repo for what the broker side guarantees.

## 15.4 Idempotency and the aggregate

At-least-once delivery means every handler may see the same event twice, and every command may be retried. The aggregate is the natural place to enforce idempotency:

- **Commands carry an idempotency key**; the aggregate records processed keys (or the outcome is naturally idempotent: `confirm()` on an already-confirmed order is a no-op or a specific, benign error).
- **Event handlers record processed event IDs** in the same transaction as their side effect.
- **State-machine transitions are idempotent by construction**: `ship()` from `SHIPPED` does nothing.

## 15.5 Data on the outside vs data on the inside

Pat Helland's 2005 paper of that name is the theoretical grounding for all of this. Data *inside* a service (the aggregate) is current, mutable, and transactionally protected. Data *outside* — in a message, in another service's cache — is a **snapshot from the past**, immutable, and identified by version or time. Treating outside data as if it were inside (assuming it is current, joining across it as if in one database) is the root cause of most distributed-data bugs. Aggregates and domain events are the DDD vocabulary for exactly this distinction: the aggregate is inside; the event is outside.

## 15.6 Querying across contexts

"Show the order with the customer's current name and the product's current image." Three contexts. Options:

1. **API composition.** A BFF calls three services and joins in memory. Simple; latency is the sum; fails if any is down.
2. **Read model with replication.** Ordering keeps a local copy of the customer name and product image, updated by events from the other contexts. Fast, resilient, eventually consistent, and the copy is *deliberately* a snapshot (Helland).
3. **GraphQL federation.** Each context owns its part of the graph; the gateway composes. Elegant for clients; the same latency and failure properties as API composition under the hood. See the GraphQL note.

Option 2 is under-used because "duplicating data" feels wrong. It is not duplication; it is a *cache with a defined update path*, and it is how large systems actually stay available.

## 15.7 How it fails

- **Entity services.** `CustomerService`, `OrderService`, `ProductService` — one per noun. Every business operation calls all of them. It is the shared-database monolith with network latency added. This is the number-one microservices failure, and it is a failure to do strategic DDD.
- **Distributed transactions "to keep it simple."** Two-phase commit across services. It works until one participant is slow, and then everything is slow. Sagas exist because 2PC does not scale across ownership boundaries.
- **Shared database across services.** The "microservices" are a monolith with extra deploy steps. The context boundary is fiction.
- **Events without an outbox.** Lost events; a weekly reconciliation job that "fixes" the discrepancies; nobody knows why they happen.
- **No idempotency.** Duplicate deliveries double-charge a customer.

## 15.8 What a senior engineer decides

- Service boundaries = bounded contexts, found by modelling (§5, §13), not by nouns or by org chart alone.
- Outbox + idempotent consumers on every event-emitting service. Non-negotiable.
- Orchestrated sagas for any process with more than ~4 steps or any step with a non-trivial compensation.
- Local read models over synchronous cross-service joins, for anything on the hot path.

---

# 16 — DDD and Legacy: Bubbles, Anti-Corruption Layers, and the Strangler

## 16.1 The problem

Almost nobody gets to apply DDD to a green field. The real situation is a 10-year-old system that is a big ball of mud, that makes the money, that cannot be paused, and that has a business begging for features the current structure cannot deliver. A rewrite has been proposed twice and failed once.

## 16.2 Evans' guidance: "Getting Started with DDD When Surrounded by Legacy Systems"

Evans' 2013 paper describes four strategies in increasing ambition:

**1. Bubble Context.** Carve out a small, clean model for *one new feature* inside the legacy system, protected by an ACL that translates to and from the legacy data. The bubble has its own vocabulary and its own tactical patterns; the legacy sees only the translated writes. Small, low-risk, and proves the approach. The bubble typically dies when the feature is done, unless it grows into the next pattern.

**2. Autonomous Bubble.** The bubble gets its own database. It synchronises with the legacy via an ACL that runs asynchronously (events, nightly batch, CDC). Now the bubble can be developed and deployed independently. This is the first real bounded context.

**3. Exposing Legacy Assets as Services.** Wrap a piece of the legacy in an Open Host Service with a Published Language, so new contexts can consume it cleanly. The legacy is not fixed, but its ugliness is contained behind a contract.

**4. Expanding the Bubble / Strangling.** The clean context takes over more capability, feature by feature, until the legacy is a shell. Fowler's **Strangler Fig** pattern: route traffic through a facade; move one capability at a time to the new system; when nothing routes to the old one, delete it.

## 16.3 Practical sequence

1. **Draw the map as it is.** Not as you wish it were. Identify the big ball of mud, the accidental contexts (the reporting DB, the vendor system, the stored procedures that run the business), and the actual data flows. This alone is usually revelatory.
2. **Find the core.** The strangling should start with the part of the domain where a better model has the highest business value and the legacy causes the most pain. Not the easiest part.
3. **Build the ACL first.** Before any domain code, the translator between legacy data and the new model. Its size will tell you how bad the legacy model is.
4. **One capability, end-to-end, in production.** A bubble that is not in production is a prototype. Ship the smallest thing that exercises the boundary.
5. **Reverse the data ownership.** The hardest step: the new context becomes the *source of truth* and the legacy becomes a consumer, synced via events. Until this happens, the bubble is subordinate.
6. **Repeat.** Each capability moved reduces the legacy's surface and increases the new model's coverage. Delete legacy code as it is strangled; do not leave it "in case."

## 16.4 Numbers

- **Time to first bubble in production:** 6–12 weeks if the ACL is scoped tightly. Longer means the bubble is too big.
- **Strangling a large monolith:** 2–5 years for a serious enterprise system. Anyone quoting 12 months for a full migration has not drawn the map.
- **ACL as a fraction of bubble code:** often 30–50% early on, falling as the bubble owns more data. If it does not fall, the data ownership never reversed.

## 16.5 How it fails

- **The rewrite.** Stop the world, rebuild from the same undocumented model, discover the exceptions in UAT, ship late with fewer features. The strangler exists because rewrites fail.
- **Two sources of truth forever.** The bubble and the legacy both write the same data; sync goes both ways; conflicts are resolved by a nightly job. This is step 5 never happening.
- **The bubble that conforms.** To save time, the bubble uses the legacy's types. It is not a bubble; it is a new module of the old ball.
- **Strangling by layer.** "First we'll move the UI, then the API, then the DB." The model is never fixed; only its wrapping changes.

## 16.6 What a senior engineer decides

- No rewrite without a context map and a strangling plan.
- First bubble on the core, in production, within a quarter.
- Every bubble has a date by which it owns its data.

---

# 17 — A Worked Example: Modelling Order Fulfilment End to End

A compressed walk through the whole method on a realistic slice: a retailer that takes orders online, reserves stock across several warehouses, charges the customer, ships via multiple carriers, and handles returns.

## 17.1 EventStorming output (abridged)

After a day on the wall, the timeline (events in orange; hotspots marked ⚠):

```
Customer added item to basket → Customer checked out → Order placed
→ Payment authorised (⚠ what if auth succeeds but capture fails later?)
→ Stock reserved per line (⚠ split across warehouses? partial reservation?)
→ Order confirmed
→ Picking list generated → Items picked → Parcel packed → Shipment dispatched
→ Payment captured (⚠ before or after dispatch? Legal says after, finance says before)
→ Parcel delivered
→ Return requested (⚠ within 30 days of *delivery*, not *order*)
→ Return received → Refund issued (⚠ partial refund with promo: pro-rata how?)
```

The hotspots are the model. Each became a session with the right expert:

- Payment: authorise at order, capture at dispatch (legal wins; finance gets a "pending capture" report).
- Stock: an order may be fulfilled from several warehouses; a *Reservation* is per line per warehouse; an order is "confirmed" when all lines are reserved.
- Returns: the window is from delivery; a `Return` is a separate aggregate referencing the original order.
- Pro-rata refunds: the promotion discount is allocated across lines at order time (`Money.allocate`), so each line has its own discounted price and refunds are per line.

## 17.2 Contexts and map

| Context | Type | Vocabulary highlights |
|---|---|---|
| **Ordering** | Core | Order, OrderLine, Promotion allocation, Confirmation |
| **Inventory** | Supporting | StockItem, Reservation, Warehouse, Bin |
| **Payments** | Supporting (ACL to PSP) | Authorisation, Capture, Refund |
| **Fulfilment** | Supporting | PickList, Parcel, Shipment; Conformist to carrier APIs |
| **Returns** | Supporting | Return, ReturnLine, Inspection |
| **Catalogue** | Supporting; OHS/PL | Product, Variant, Price (sell price) |
| **Notifications** | Generic | (buy) |

Ordering is core because the *promotion and confirmation rules* are what the business competes on. It is upstream of Fulfilment, Payments, and Returns via events; downstream of Catalogue via its published language; and talks to Inventory as Customer–Supplier.

## 17.3 Aggregates in Ordering

- **Order** (root): `OrderId`, `CustomerId`, `List<OrderLine>` (values: `Sku`, `Quantity`, `Money unitPrice`, `Money discount`), `OrderStatus`, `ShippingAddress` (value, snapshot), `version`. Invariants: lines non-empty while placed; total = Σ lines; status transitions per the state machine; discount allocation sums to the promotion amount.
- **No `Customer` aggregate here.** Ordering needs only `CustomerId` and a snapshot of the shipping address. Customer identity lives in a separate (generic) context.
- **No `Reservation` here.** Reservations are Inventory's aggregates. Ordering tracks `lineReservationStatus` from `StockReserved`/`StockReservationFailed` events.

State machine, explicit:

```
PLACED ──(all lines reserved ∧ payment authorised)──▶ CONFIRMED ──(dispatched)──▶ SHIPPED ──(delivered)──▶ DELIVERED
  │                                                        │
  └──(reservation failed ∨ auth declined ∨ customer cancel)──▶ CANCELLED
```

## 17.4 The confirmation saga (orchestrated)

`OrderPlaced` starts an `OrderConfirmationSaga` (itself an aggregate, in Ordering):

1. Send `AuthorisePayment(orderId, total)` → await `PaymentAuthorised` | `PaymentDeclined` (timeout 30 s).
2. Send `ReserveStock(orderId, lines)` → await `StockReserved` per line | `StockReservationFailed` (timeout 60 s).
3. On all success: `order.confirm()`. On any failure: compensate — `ReleaseReservations`, `VoidAuthorisation` — then `order.cancel(reason)`.

The saga's own state (`awaitingPayment`, `awaitingReservations: Set<LineId>`) is persisted; each step is idempotent; each command carries the saga ID as its idempotency key. Timeouts are business decisions (the domain expert said "if we can't reserve stock within a minute, something is wrong").

## 17.5 What the code looks like at the edges

- **Inbound:** `POST /orders` → `PlaceOrderHandler` → `Order.place(...)` → `orders.save()` (with outbox) → `201` with `OrderId`. The REST design follows the companion note.
- **Outbound event:** internal `OrderConfirmed` → translated to published `ordering.OrderConfirmed.v1 { orderId, customerId, lines[{sku, qty, warehouseId}], total, confirmedAt }` in the schema registry.
- **Read model:** `order_summary` table (order ID, status, customer display name, item count, total, last updated) maintained by a projector from Ordering's and Customer's events. The "my orders" page reads this, never the aggregate.
- **ACL:** `PspPaymentGateway` translates the payment provider's 40 response codes into three domain outcomes (`Authorised`, `Declined(reason)`, `Retryable`).

## 17.6 What changed because of the modelling

- The initial design had one `Order` with an embedded `payment` and `shipments` list. The wall showed that payments and shipments have their own life cycles, owners, and failure modes; they became separate contexts. The `Order` aggregate shrank from ~25 fields to 8.
- "Status" went from a 14-value enum with 3 booleans to a 5-state machine plus per-line reservation status.
- Returns went from `order.isReturned` to a first-class aggregate, because the business asked "what about returning two of three items, then a third later?"
- The capture-at-dispatch decision was made by legal, not by whoever wrote the payment code, and it is documented in the glossary with the reason.

---

# 18 — Anti-Patterns: How DDD Fails in Practice

A consolidated field guide. Each of these appears in real systems that "do DDD."

| Anti-pattern | What it looks like | Why it happens | Fix |
|---|---|---|---|
| **Anemic domain model** | Entities with getters/setters; logic in services | ORM tutorials; "services are testable" | Move rules onto entities; ban setters |
| **DDD-lite / tactical-only** | Entities, Repositories, Value Objects — inside one giant model | Tactical patterns are easier to learn than strategic | Do the context map; split the model |
| **God aggregate** | `Customer` with everything the customer has ever touched | Modelling by noun; ORM eager loading | Rule 2: small aggregates; reference by ID |
| **Entity services** | One microservice per noun | Boundaries drawn without modelling | Boundaries by capability/language |
| **Shared "common" domain library** | `common-domain` jar with `Customer`, `Product`, `Order` used everywhere | Reuse instinct | Shared Kernel only for tiny, owned set; otherwise duplicate |
| **Repository as DAO** | Generic `Repository<T>` with `findAll`; returns ORM entities | Framework defaults | Per-aggregate, domain-meaningful methods |
| **Domain service as service layer** | `OrderDomainService` with all the logic | Old habits, new name | Push logic onto aggregates; services only for cross-aggregate ops |
| **CRUD events** | `OrderCreated`, `OrderUpdated`, `OrderDeleted` | Modelling the database, not the domain | Events per business fact |
| **Sync events** | Handlers run in the caller's transaction and can fail it | "Simplest thing" | After-commit dispatch via outbox; handlers own their transactions |
| **Event sourcing everywhere** | Reference data event-sourced; 200 projections | Enthusiasm | ES for core aggregates with audit needs only |
| **Framework in the domain** | `@Entity`, `@JsonProperty` on aggregates | Convenience | Separate persistence model or aggregate-as-document; architecture tests |
| **Language drift** | Business says "acceptance", code says "confirmation" | Renames seen as waste | Rename as first-class work; glossary in repo |
| **Modelling by engineers** | Clean model of the wrong domain | No expert access | EventStorming with the people who do the work |
| **Boundary by org chart only** | Contexts = current teams, including accidental ones | Conway without the inverse manoeuvre | Model first; then align teams |
| **Premature microservices** | Contexts extracted before understood; boundaries wrong; distributed transactions | Fashion | Modular monolith; extract on evidence |
| **The big rewrite** | Stop, rebuild, discover the exceptions in UAT | Frustration with legacy | Bubble + strangler |
| **DDD for CRUD** | Aggregates and sagas for a settings page | Applying it everywhere | Classify the subdomain; keep generic/simple parts simple |

---

# 19 — War Stories

Composites drawn from real systems, with details changed. Each teaches one thing.

## 19.1 The customer with 312 columns

A B2B platform's `customers` table had 312 columns and 14 services reading it. Marketing's `tier`, billing's `credit_status`, support's `sla_level`, sales' `account_owner`, compliance's `kyc_state` — all on one row. A change to `tier` semantics (adding a value) broke three services that switched on it, took six weeks to coordinate, and caused a billing incident when one service defaulted an unknown tier to "premium."

The context-mapping exercise found five contexts sharing a `CustomerId`. Each got its own table with its own 10–20 columns. The 312-column table was strangled over 14 months. The "add a tier" change afterwards touched one context and shipped in a day.

**Lesson:** the word "customer" was five words. The database had enforced the fiction that it was one.

## 19.2 The order that locked the warehouse

An e-commerce checkout did `SELECT ... FOR UPDATE` on every product row in the order, then the warehouse row, then inserted the order. Two hundred concurrent checkouts on a flash-sale SKU serialised on that row. p99 went from 180 ms to 9 s; the retry storm from the mobile app made it worse; the site was effectively down for 40 minutes during the highest-revenue hour of the year.

The redesign made `StockItem` (per SKU per warehouse) its own aggregate with an atomic `reserve` (a conditional `UPDATE ... WHERE available >= ?`), decoupled from the order via `OrderPlaced` → reservation saga. Contention dropped to a single row per SKU, and that row's operation went from a transaction-length lock to a sub-millisecond atomic update.

**Lesson:** the aggregate boundary is the lock boundary. The original "transaction" was an accidental aggregate spanning the whole catalogue.

## 19.3 The event that was a changelog

A fintech emitted `AccountUpdated { before: {...}, after: {...} }` for every change. Downstream fraud, reporting, and notifications each reverse-engineered "what happened" by diffing. When the account model added a field, every consumer's diff logic had to be reviewed. When a bug set `status` incorrectly and was fixed by a second update, consumers saw two `AccountUpdated`s and could not tell a correction from a genuine change. The audit trail could show *what* but never *why*, which a regulator eventually asked.

The fix was a year of introducing intent-revealing events (`AccountFrozenForSuspectedFraud`, `AccountLimitRaisedByReview`) alongside the changelog, then migrating consumers.

**Lesson:** an event carries intent or it is not a domain event. "Updated" is the absence of a model.

## 19.4 The shared kernel that ate the company

Two closely related contexts agreed to share `Money`, `CustomerId`, and `Address` in a `shared-kernel` library. Good. Over three years, without an owner or a change process, it grew to 140 classes including `Order`, `Invoice`, and `Product`. Twenty-two services depended on it. A minor version bump required 22 PRs, and a breaking change was effectively impossible. The company had re-created a monolith at the dependency level while running "microservices."

**Lesson:** a Shared Kernel with no owner and no cap is a slow-motion merge of every context.

## 19.5 The rewrite that reproduced the mud

A logistics company's 12-year-old dispatch system was rewritten "with DDD." The engineering team, without warehouse or dispatch staff in the room, produced clean aggregates for `Route`, `Stop`, `Driver`. In UAT the dispatchers explained that a "route" changes meaning at 2 p.m. daily when the "afternoon re-plan" runs, that drivers can be "on a route" while "not dispatched," and that 30% of stops are "exceptions" with a dozen sub-types. None of this was in the model. The rewrite shipped 18 months late with a `RouteExceptionHandler` that was 6,000 lines.

The successful second attempt began with a two-day EventStorming with dispatchers, which surfaced that "re-plan" was the core domain and "route" was a projection of it.

**Lesson:** modelling without domain experts produces an elegant model of the engineers' assumptions.

## 19.6 The bubble that reversed ownership

An insurer with a 25-year-old policy administration system needed a new quoting engine. Rather than rewrite, the team built a `Quoting` bounded context as a bubble: its own model (`Quote`, `RiskProfile`, `PricingDecision`), its own database, and an ACL that read applicant data from the legacy via a nightly extract and wrote accepted quotes back as legacy "proposals."

In month 4 it went live for one product line. In month 9 the ACL was made bidirectional and event-driven. In month 14 Quoting became the source of truth for pricing decisions and the legacy system's pricing module was disabled. By year 3, four more contexts had been strangled out the same way, and the legacy system handled only policy servicing.

**Lesson:** the strangler works when the first bubble is on the core, ships early, and has a plan to own its data.

---

# 20 — The Staff Engineer's Decision Checklist

Use in design reviews. Any "no" is a conversation, not necessarily a block.

**Strategic**

- [ ] Is there a domain vision statement? Does it name the core domain? Does the business agree?
- [ ] Is each subdomain classified (core / supporting / generic), and does investment match?
- [ ] Is there a context map in the repo? Does every edge have a pattern and an upstream/downstream arrow?
- [ ] Can you state, for each pair of adjacent contexts, one word that means different things on each side? If not, are they really two contexts?
- [ ] Does each context have exactly one owning team? Does each team know which contexts it owns?
- [ ] Is every external or legacy integration behind an ACL?
- [ ] Is every Shared Kernel enumerated, owned, and small?

**Language**

- [ ] Is there a glossary per context, in the repo, PR-reviewed?
- [ ] Do class, method, event, and test names use the glossary's terms?
- [ ] Are there terms in the code the business would not recognise? Why?
- [ ] When the business changed a term in the last year, did the code change?

**Tactical**

- [ ] For each aggregate: what invariants does it protect? If none span entities, is it a root plus values?
- [ ] Are aggregates referenced by ID only?
- [ ] One aggregate per transaction? List the exceptions and their removal plan.
- [ ] Does every aggregate have an optimistic-lock version?
- [ ] Are all domain concepts typed (no `String customerId`, no `BigDecimal` for money)?
- [ ] Are values immutable and self-validating?
- [ ] Are there setters on entities? Why?
- [ ] Is every business-significant state change a named event in the past tense?
- [ ] Are internal domain events separate from published integration events?

**Architecture**

- [ ] Does the domain module have zero framework dependencies? Is that enforced in CI?
- [ ] Is the code packaged by context, then by layer?
- [ ] Are read queries kept out of aggregates (CQRS level ≥ 1)?
- [ ] Events dispatched after commit via an outbox? Consumers idempotent?
- [ ] Sagas explicit (orchestrated) for multi-step processes? Compensations modelled with domain experts?
- [ ] Is event sourcing, if used, limited to aggregates with a stated audit/temporal need?
- [ ] Is this a modular monolith or services, and is the choice justified by evidence?

**Process**

- [ ] Was the model built with people who do the work, not proxies?
- [ ] When was the last modelling session with domain experts?
- [ ] Is there a plan to refactor toward deeper insight, or is the first model assumed final?
- [ ] For legacy: is there a map of what exists, a first bubble on the core, and a date for data-ownership reversal?

---

# 21 — Mental Models Worth Keeping

- **The model is the code is the language.** If any two diverge, the third is wrong too.
- **A word with two meanings is two contexts.** The database will lie to you about this.
- **Aggregate = consistency = lock = shard = ownership.** One boundary, five consequences. Draw it for the invariants and the rest follows.
- **Inside is now; outside is the past.** (Helland.) An aggregate is the present; an event is history; a copy in another context is a snapshot. Never confuse them.
- **Eventual consistency is a business question.** "How long can these be out of sync?" has an answer, and it is almost never "zero."
- **Compensation is not rollback.** Undo in the real world is a new business action with its own rules.
- **Core is where the model is the product.** Spend modelling effort there; buy or simplify everywhere else.
- **Strategic before tactical.** Tactical DDD on an unsplit model is a tidier ball of mud.
- **The first model is wrong.** The breakthrough comes on the third iteration; budget for it.
- **Conway is a law, not a suggestion.** Design the teams and the contexts together.
- **Duplicated data with a defined update path is a cache, not a sin.** Shared data with no owner is the sin.
- **DDD is expensive.** That is why you only do it where the domain is the hard part.

---

# 22 — Reading Notes: The Path to Expertise

Read in this order. For each: what it is for, what to take from it, what to skip.

## 22.1 Foundations (read these fully)

**Eric Evans — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003). "The Blue Book."**
The source. Dense, discursive, and still the best treatment of *why*. Read it in this order, not front to back:
- **Part I (ch. 1–3): Putting the Domain Model to Work.** Knowledge crunching, ubiquitous language, model-driven design. The whole philosophy in 60 pages. Read first, re-read yearly.
- **Part IV (ch. 14–16): Strategic Design.** Bounded Context, Context Map and all the relationship patterns, Distillation (core domain), Large-Scale Structure. Read *second*, before the tactical patterns. This is the part most people skip and the part that matters most.
- **Part II (ch. 4–7): The Building Blocks.** Layered architecture, Entities, Value Objects, Services, Modules, Aggregates, Factories, Repositories. Read third. The Aggregate section is short; supplement with Vernon's essays.
- **Part III (ch. 8–13): Refactoring Toward Deeper Insight.** Breakthroughs, making implicit concepts explicit, supple design (intention-revealing interfaces, side-effect-free functions, assertions, closure of operations), analysis patterns, design patterns applied to the domain. Underrated; this is what separates a decent model from a great one. Chapter 8's cargo-shipping breakthrough narrative is worth reading twice.
Take from it: the vocabulary, the *reasons*, the emphasis that this is a creative collaborative discipline, not a pattern checklist.

**Eric Evans — *Domain-Driven Design Reference* (2015, free PDF).** A 60-page condensed pattern summary, updated with Domain Events and the strategic patterns' refined definitions. Keep it open during design reviews.

**Vaughn Vernon — *Implementing Domain-Driven Design* (2013). "The Red Book."**
The Blue Book's practical companion, with a running example (a SaaS collaboration tool) and full code. Strongest chapters:
- **Ch. 2 (Domains, Subdomains, Bounded Contexts) and Ch. 3 (Context Maps).** The clearest explanation of strategic design in print, with a worked map.
- **Ch. 10 (Aggregates).** Incorporates the "Effective Aggregate Design" essays. The four rules. Read with §9 of this note.
- **Ch. 8 (Domain Events)** and **Ch. 13 (Integrating Bounded Contexts).** Event publishing, the outbox idea before it was called that, RESTful and messaging integration.
- **Appendix A (Aggregates and Event Sourcing: A+ES).** The best short introduction to ES on aggregates.
Skip or skim: the ch. 14 application-layer detail and some framework-specific code has aged.

**Vaughn Vernon — *Domain-Driven Design Distilled* (2016).** 150 pages. Strategic design first, then tactical, then EventStorming. If you can only make a team read one book, this is it. Read it yourself in an afternoon to see the whole shape before the Blue Book's detail.

**Vaughn Vernon — "Effective Aggregate Design" (three-part essay, 2011, free online).** The four rules of aggregate design with a worked Scrum-tool example showing a god aggregate being split. Essential; ~30 pages.

## 22.2 Modern treatments

**Vlad Khononov — *Learning Domain-Driven Design* (2021).** The best current single-volume introduction. Clear on subdomain classification (with a useful "core vs generic" decision tree), context mapping, and — unusually — on choosing the *architecture per subdomain* (transaction script for generic, active record for supporting, domain model / ES for core). Chapters 8–10 on architectural patterns, CQRS, and "DDD in the real world" are excellent on *not* over-applying DDD. Read after Vernon Distilled.

**Scott Millett & Nick Tune — *Patterns, Principles, and Practices of Domain-Driven Design* (2015).** Long, C#-based, but the strategic chapters (Part I) and the "applying the principles" chapters are strong, and it is honest about when DDD is not worth it. Good source of alternative explanations if Evans' don't land.

**Scott Wlaschin — *Domain Modeling Made Functional* (2018).** DDD in F#. Even if you never write F#, read this for the *type-driven* view: make illegal states unrepresentable, model workflows as pipelines of typed transformations, values everywhere. It will change how you write value objects and state machines in any language. Pairs with §8.

**Alberto Brandolini — *Introducing EventStorming* (ongoing, Leanpub).** The workshop technique, from its inventor. Chapters 1–5 cover the big-picture format; the rest is facilitation detail. Also read his blog and talks ("50,000 Orange Stickies Later"). Pairs with §13.

**Harry Percival & Bob Gregory — *Architecture Patterns with Python* (2020, free online).** DDD tactical patterns, repository, unit of work, events, CQRS, in plain Python with tests, and a refreshingly pragmatic tone. Good for anyone in a dynamically-typed language who thinks DDD is a Java thing.

**Nick Tune & Jean-Georges Perrin — *Architecture Modernization* (2024).** Strategic DDD at organisation scale: domain discovery, Wardley mapping, bounded context canvas, team topologies alignment, migration planning. The principal-engineer view of §5–§7 and §16.

## 22.3 Essential shorter pieces

- **Martin Fowler — "Anemic Domain Model" (2003).** Why it is an anti-pattern. Two pages; the argument in §2.1.
- **Martin Fowler — "Bounded Context", "Ubiquitous Language", "Domain Event", "CQRS", "Event Sourcing", "Strangler Fig Application", "Value Object" (bliki).** Short, canonical definitions; useful to send to people.
- **Alistair Cockburn — "Hexagonal Architecture" (2005).** The original ports-and-adapters essay. §12.
- **Pat Helland — "Life Beyond Distributed Transactions: An Apostate's Opinion" (2007).** Why entities (aggregates) must be the unit of transaction and why cross-entity work must be message-based and idempotent. The distributed-systems justification for §9 and §15. Also "Data on the Outside versus Data on the Inside" (2005), §15.5.
- **Melvin Conway — "How Do Committees Invent?" (1968).** Conway's law from the source. Four pages.
- **Eric Evans — "Getting Started with DDD When Surrounded by Legacy Systems" (2013).** Bubble context, autonomous bubble, exposing legacy as services. §16.
- **Greg Young — "CQRS Documents" (2010) and the talk "CQRS and Event Sourcing" (2014).** Where CQRS came from and what it is *not*.
- **Udi Dahan — "Don't Delete — Just Don't", "Race Conditions Don't Exist", "Clarified CQRS".** Provocative short posts that reframe deletion, concurrency, and read/write splitting in domain terms.
- **Mathias Verraes — blog (verraes.net).** "DDD and Messaging Architectures", "Eventsourcing Patterns" (especially "Decide-Evolve", "Forgettable Payloads"), "Unrepresentable States", "Emergent Contexts through Refinement". Some of the sharpest tactical thinking online.
- **Chris Richardson — microservices.io patterns: "Saga", "Transactional Outbox", "Database per Service", "API Composition", "CQRS".** Concise pattern write-ups for §15.

## 22.4 Adjacent books that make DDD work

- **Matthew Skelton & Manuel Pais — *Team Topologies* (2019).** Stream-aligned teams, cognitive load, the Inverse Conway Manoeuvre. How to align teams to contexts. §5.3.
- **Sam Newman — *Building Microservices* (2nd ed., 2021)** and ***Monolith to Microservices* (2019).** Service boundaries from bounded contexts; decomposition and data-migration patterns.
- **Cyrille Martraire — *Living Documentation* (2019).** Keeping the glossary, context map, and architecture decisions alive from the code.
- **Martin Fowler — *Patterns of Enterprise Application Architecture* (2002).** Domain Model vs Transaction Script vs Table Module; Repository; Unit of Work; Identity Map; Money. The pre-DDD vocabulary DDD builds on.
- **Gregor Hohpe & Bobby Woolf — *Enterprise Integration Patterns* (2003).** The messaging vocabulary (message, channel, router, translator, idempotent receiver) that domain events ride on.
- **Simon Wardley — *Wardley Maps* (free online).** For the core/supporting/generic classification and the buy-vs-build decision.

## 22.5 Community and ongoing

- **DDD Europe** (dddeurope.com) and **Explore DDD** conference talks, on YouTube. Evans', Vernon's, Brandolini's, Khononov's, and Verraes' talks are the fastest way to get current thinking.
- **Virtual DDD** (virtualddd.com) meetups and the **DDD-CQRS-ES Discord**.
- **Context Mapping / DDD Crew GitHub** (github.com/ddd-crew): Bounded Context Canvas, Context Mapping cheat sheet, Aggregate Design Canvas, EventStorming glossary. Printable, practical.

## 22.6 A suggested sequence

| Stage | Read | Do |
|---|---|---|
| Week 1 | Vernon *Distilled*; Fowler "Anemic Domain Model"; Evans *Reference* skim | Write the glossary for a system you know |
| Weeks 2–4 | Evans Blue Book Parts I and IV; Vernon IDDD ch. 2–3 | Draw the context map for that system, as it actually is |
| Weeks 5–8 | Evans Part II; Vernon "Effective Aggregate Design"; Wlaschin | Refactor one aggregate: values, invariants, events, version |
| Weeks 9–12 | Brandolini; Evans Part III; Khononov | Run an EventStorming on a real problem with real experts |
| Months 4–6 | Helland papers; Richardson patterns; Percival & Gregory; Vernon IDDD ch. 8, 10, 13, App. A | Implement outbox + saga for one cross-context process |
| Months 7–12 | Evans legacy paper; Tune & Perrin; Team Topologies; Newman | Plan and ship a bubble context against a legacy system |
| Ongoing | DDD Europe talks; Verraes; Dahan | Re-read Blue Book Part I yearly |

---

# 23 — A Lab Path

Exercises that build the judgement, roughly in order.

1. **Glossary archaeology.** Pick a codebase you know. List every noun in the class names. For each, write what the business calls it and whether the business would recognise the class. Count the ones they would not. That number is the translation tax.
2. **Find the contexts in a monolith.** Take the schema of a real system. For each table, name the team that owns it and the business capability it serves. Colour by capability. Find the tables with more than one colour (the `customers`, `products`, `orders` tables). Those are the merged contexts. Draw the context map that *should* exist.
3. **Split a god aggregate.** Take an entity with a large object graph. List its invariants. Draw the smallest boundary that protects each one. Count how many aggregates fall out. Estimate the change in lock contention.
4. **Type the primitives.** In a service, replace every `String id`, `BigDecimal amount`, `String email` with a value type. Count the bugs the compiler finds. (There will be some.)
5. **Event catalogue.** For one aggregate, list every business-significant thing that can happen to it, in the past tense, in the domain expert's words. Compare with the events (or `updated_at` columns) the code actually emits.
6. **Outbox from scratch.** Implement the transactional outbox on a toy service with Postgres and Kafka. Kill the process at every point between the aggregate write and the publish; confirm no event is lost and no phantom is published. Then make the consumer idempotent and replay.
7. **Saga with compensation.** Build a three-step orchestrated saga (reserve, charge, ship) with timeouts and compensations. Inject failure at each step. Watch what "undo" means for each.
8. **Event-source one aggregate.** Rebuild the aggregate from §9.5 on an event stream. Add a snapshot. Change an event's schema and write the upcaster. Then decide, honestly, whether the aggregate needed ES.
9. **Run an EventStorming.** With real people about a real process. Facilitate it. The hardest and most valuable exercise on this list.
10. **Write an ACL.** Against a real third-party API with a bad model (there is always one). Measure how much of your domain code the ACL kept clean.
11. **Bubble context.** In a legacy system, deliver one feature via a bubble with its own model and an ACL. Ship it. Then write the plan to reverse data ownership.
12. **Design review.** Take §20's checklist to a real design review. Note which questions produce silence. Those are the team's gaps.

---

*This note is a companion to the software-engineering and distributed-systems notes in this repo. DDD is the discipline that decides* what *the boundaries are; the distributed-systems notes explain what it costs to cross them; the API notes explain what the boundary looks like from outside.*
