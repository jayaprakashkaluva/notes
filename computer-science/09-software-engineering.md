# Software Engineering: Programs That Survive Time, Teams, and Users

Assumes [Programming Fundamentals](02-programming-fundamentals.md). Programming is making a computer do something once; **software engineering is programming integrated over time** (Google's definition) — code that must be changed by people who didn't write it, years after the decisions were made, without breaking users who depend on it. The dominant cost of real software is not writing it but *changing* it; nearly everything in this document is a technique for keeping change cheap and safe.

---

## Table of Contents

1. [The Enemy: Complexity](#1--the-enemy-complexity)
2. [Version Control: Git](#2--version-control-git)
3. [Testing](#3--testing)
4. [Design in the Small: Code Quality](#4--design-in-the-small-code-quality)
5. [Design in the Large: Architecture](#5--design-in-the-large-architecture)
6. [APIs and Compatibility](#6--apis-and-compatibility)
7. [The Delivery Pipeline: CI/CD](#7--the-delivery-pipeline-cicd)
8. [Running Software: Observability and Reliability](#8--running-software-observability-and-reliability)
9. [Working in Teams](#9--working-in-teams)
10. [Estimation, Process, and the Classic Failure Modes](#10--estimation-process-and-the-classic-failure-modes)
11. [Using AI Tools Well](#11--using-ai-tools-well)
12. [The Road to Expertise](#12--the-road-to-expertise)

---

# 1 — The Enemy: Complexity

Software is the most complex artifact humans build — millions of parts (lines), each potentially interacting with any other, no physics constraining the mess. The field's foundational essay (Brooks, *No Silver Bullet*) split the difficulty into **essential complexity** (the problem itself is intricate — tax law is tax law) and **accidental complexity** (mess we added — tangled dependencies, unclear names, clever hacks). Engineering can only remove the accidental kind, and every practice below is one of three moves against it:

1. **Modularity + information hiding**: divide the system so each part can be understood — and changed — *alone*. A module is defined by its **interface** (small, stable, what others see) hiding its **implementation** (large, changeable). Ousterhout's formulation: the best modules are **deep** — simple interface, substantial implementation behind it (`open/read/write/close` hiding all of [OS doc Part 7](operating-systems.md) is the canonical deep module). Shallow modules (interface as complex as what they hide) add surface without hiding anything.
2. **Coupling down, cohesion up**: coupling = how much modules must know about each other (change ripples across coupled modules); cohesion = how much a module's contents belong together. Every architecture debate is these two words wearing costumes.
3. **Feedback loops, shortened**: tests, CI, reviews, monitoring, iterative process — all exist to catch mistakes when they're cheap (seconds after typing) instead of expensive (production, or a decision three months ossified). Cost of a defect grows roughly an order of magnitude per stage it survives.

# 2 — Version Control: Git

The non-negotiable foundation — a database of every version of the code, by everyone, forever, with time travel and parallel universes.

## 2.1 The model (learn this, not command incantations)

A **commit** is a snapshot of the whole tree + metadata (author, message, *parent commit(s)*), named by its content hash. History is therefore a **DAG** of snapshots ([doc 04 §6](04-algorithms.md)); a **branch** is merely a movable pointer to a commit; `HEAD` is "where you are." Everything mystifying about Git dissolves against this model: merging = creating a commit with two parents (walking both histories to a common ancestor); rebase = replaying your commits onto a new base (new hashes — history *rewritten*, hence the rule: never rebase what others have pulled); a **merge conflict** = both branches edited the same lines and the textual merge needs a human ruling (Git knows nothing of semantics — compiling and testing after resolution is on you).

Daily muscle memory: `status`/`diff`/`log`; the two-step `add` (stage — *choose* what enters the commit) then `commit`; `push`/`pull` to sync with the shared remote (GitHub/GitLab); branch-per-change workflow: branch → commits → push → **pull request** → review (§9) → merge to `main`. Rescue kit, pre-learned calmly: `git bisect` (binary-search history for the commit that broke things — [doc 04 §2](04-algorithms.md) applied to time), `reflog` (where HEAD has been — the undo for "I destroyed everything"), `stash`, `revert` (new commit undoing an old one — safe on shared history) vs `reset` (move the pointer — local only).

## 2.2 Commits as communication

Small, single-purpose commits with messages explaining **why** (the diff already shows *what*) turn history into the project's best documentation: `git blame` + a good message answers "why is this line here?" years later. "Fix stuff" answers nothing forever.

# 3 — Testing

Automated tests are executable claims about behavior — the machinery that makes **change safe**: with a good suite, you refactor and upgrade fearlessly (run the suite, believe the green); without one, every change is a prayer, so change slows, so the code rots.

- **The pyramid**: many fast **unit tests** (one function/module, milliseconds — the base), fewer **integration tests** (modules together: real database, real HTTP — slower, catch the seams), few **end-to-end tests** (whole system as a user — slow, brittle, invaluable in small numbers). Inverted pyramids (all E2E) yield hour-long flaky suites nobody trusts.
- **What makes a good unit test**: tests *behavior via the public interface*, not implementation details (tests coupled to internals break on every refactor — the opposite of their purpose); one logical assertion; readable as a specification (`test_expired_token_is_rejected`); fast and deterministic. Structure: arrange, act, assert.
- **Testability is a design pressure — the secret benefit**: code that's hard to test (buried I/O, global state, giant functions) is *badly designed by the §1 criteria*, and the test suite is the first to tell you. **Dependency injection** (pass collaborators in rather than reaching out to globals) plus **test doubles** (fakes/stubs/mocks standing in for the database or network) is the standard pattern — and doc 02's "pure core, imperative shell" is the testability jackpot: pure functions need no mocks at all. (Mock sparingly, at real boundaries; mock-everything suites test the mocks.)
- **TDD** (write the failing test first, make it pass, refactor): a discipline worth trying seriously — its red-green-refactor loop keeps design honest; its dogmatic form isn't mandatory. **Regression tests** are non-negotiable: every fixed bug gets a test pinning it dead (bugs cluster and resurrect).
- Beyond examples: **property-based testing** (state an invariant — "decode(encode(x)) == x" — and the framework generates hundreds of adversarial inputs; Hypothesis/QuickCheck) and **fuzzing** (random inputs hunting crashes — [doc 10](10-security-and-cryptography.md)'s workhorse). **Coverage** is a *gap-finder*, not a target — 100%-covered code can still be wrong (Goodhart's law: chase the metric, get the metric).

# 4 — Design in the Small: Code Quality

Extending doc 02 §4/§13 from functions to codebases — the principles that survived the pattern wars:

- **Make it work, make it right, make it fast — in that order**; and **YAGNI** ("you aren't gonna need it"): build for today's known requirements, design so tomorrow's can be *added*. Speculative generality is accidental complexity on spec. Balanced by Ousterhout's "design it twice": sketch two interfaces before committing to one — cheap now, expensive never.
- **Refactoring** — improving structure without changing behavior — is not a special event but a *continuous* motion (the boy-scout rule: leave code slightly better than found), made safe by §3's tests, made honest by small steps. Named refactorings (extract function, rename, inline) are in your IDE; use them over hand-editing.
- **Technical debt** — the metaphor that earns its keep: shortcuts borrow speed now, charge interest (every future change costs more) until repaid (refactoring) or the codebase goes bankrupt (rewrite — almost always the wrong call: years of encoded edge-case knowledge discarded, the second-system builds slower than the first decays; Netscape's rewrite famously handed the market to IE). Manage debt like money: some is leverage, track it visibly, pay down high-interest items (hot, frequently-changed files) first — `git log` frequency × ugliness is your prioritized list.
- **Code smells** worth an instinct: duplication (but see doc 02 — wrong abstraction beats it), long parameter lists, feature envy (module A constantly poking B's data — the function lives in the wrong module: a *coupling* symptom), boolean flags changing a function's whole meaning, comments explaining *what* (rewrite the code) instead of *why* (keep those).
- **Naming and conventions at scale**: a codebase should read as if written by one careful person — hence formatters (Black, gofmt, Prettier: automate the argument away) and linters (mechanical smell detection) run in CI (§7), not negotiated in review.

# 5 — Design in the Large: Architecture

Architecture = the decisions expensive to reverse: what are the modules, what owns which data, what talks to what, over which contracts. §1's forces at building scale:

- **Layering** — the curriculum's own structure, applied: UI → domain logic → data access, dependencies pointing one way. The load-bearing rule is the **dependency direction**: business logic must not depend on delivery mechanisms (web framework, database driver) — depend on *interfaces*, wire implementations at the edges ("hexagonal"/"clean" architecture is this one rule with branding). Payoff: the core is testable (§3) and the expensive-to-reverse choices (framework, DB) become… less so.
- **Monolith vs microservices** — the defining modern trade-off. Microservices (many small deployables communicating over the network — [doc 05 §10](05-computer-networks.md)) buy independent deployment/scaling/team-ownership, paid for in the full distributed-systems tax: network failure modes everywhere, no cross-service transactions ([../distributed-systems/](../distributed-systems/)), operational sprawl, and the worst outcome available — the *distributed monolith* (services so coupled they deploy together anyway: all costs, no benefits). The professional default: **modular monolith first** (enforce module boundaries *inside* one deployable); extract services only when a *measured* need (team contention, divergent scaling) forces it — boundaries are vastly cheaper to redraw inside a process. Conway's law governs either way: architecture mirrors org structure, so design both together.
- **State is the hard part** (doc 02 §6, at system scale): stateless services scale by copying ([doc 05 §6](05-computer-networks.md)); the state concentrates in databases ([doc 06](06-databases.md)) and queues (async decoupling: producer and consumer needn't be up simultaneously — but you inherit eventual consistency and duplicate-delivery handling, [doc 05 §10](05-computer-networks.md)'s idempotency again).
- **Document the decisions**: ADRs (architecture decision records — one page: context, options, choice, consequences) are the difference between "why is it like this?" having an answer or a shrug in three years.

# 6 — APIs and Compatibility

An API (function signature, REST endpoint, library interface, file format) is a **promise to strangers**. Once published, people depend on it — including on its bugs (Hyrum's law: *every observable behavior* will be depended on by someone). Hence:

- **Semantic versioning** (MAJOR.MINOR.PATCH): patch = fixes, minor = additions (backward-compatible), major = **breaking changes**. The contract that makes dependency ecosystems (§7's package managers) function.
- **Compatible evolution**: add optional fields, never repurpose or remove published ones; new endpoints/versions for new shapes; **deprecate** (mark, warn, document the migration, wait) before removing. Schema'd formats (protobuf/Avro — [doc 05 §10](05-computer-networks.md)) mechanize these rules.
- Design APIs from the *caller's* seat (write the calling code first); make misuse hard (types encoding validity — doc 02 §9; good defaults; fail loudly). Every public API is a deep-module bet (§1): you are choosing forever-interface over changeable-implementation.

# 7 — The Delivery Pipeline: CI/CD

Automation of the path from commit to production — §1's feedback-loop principle industrialized:

- **Continuous integration**: every push, a server builds and runs the full check suite (tests, linters, formatters, type checkers, security scanners) — merge is blocked on green. Kills "works on my machine" (the CI environment is the referee) and integration hell (merging months of divergence). Corollary discipline: integrate small and often; a broken main branch is everyone's emergency.
- **Continuous delivery/deployment**: green main is *deployable* (delivery) or *deploys itself* (deployment). Small frequent releases are **safer** than big rare ones — less diff per release to suspect, and a practiced, boring release process (DORA research: elite teams deploy more often *and* break less — speed and stability correlate, because both come from small batches + automation).
- Release safety valves: **feature flags** (ship code dark, enable gradually — decouples deploy from release), **canary/rolling deploys** (1% of traffic first, watch §8's dashboards, proceed or auto-rollback), blue-green (instant switchback).
- **Dependencies** — most of your application is code you didn't write, resolved by a package manager (npm/pip/cargo/maven) from your manifest + **lockfile** (exact pinned versions — commit it; builds must be reproducible). The costs: transitive sprawl (one install, hundreds of packages), supply-chain attacks (a compromised popular package ships malware to everyone — left-pad's removal broke half the internet *by absence*; typosquats and hijacked maintainers do it *by presence*, [doc 10](10-security-and-cryptography.md)), and upgrade debt (automate with Dependabot-style bots + §3's suite as the safety net).

# 8 — Running Software: Observability and Reliability

Shipped is not done; software earns its keep in production, and production is where [OS](operating-systems.md), [networks](05-computer-networks.md), and [databases](06-databases.md) knowledge convenes:

- **The three signals**: **logs** (structured — JSON with request IDs, not printf prose — so they're queryable; a request's ID traces it across services), **metrics** (cheap numbers over time: request rate, error rate, latency — as **percentiles**, never averages: p50 hides the p99 an unlucky user always hits; [../distributed-systems/07-tail-latency-and-ha-architecture.md](../distributed-systems/07-tail-latency-and-ha-architecture.md)), **traces** (one request's tree of calls across services, timed — where the 3 seconds actually went).
- **Alerting discipline**: page on *user-visible symptoms* (error rate, latency SLO burn), not on every internal twitch; a noisy pager trains responders to ignore it (alert fatigue is how real outages get missed). **SLOs** + error budgets (Google SRE's frame): define acceptable unreliability (99.9% = 43 min/month); spend the budget on velocity; freeze features when it's exhausted. 100% is the wrong target — the marginal nine costs more than users notice.
- **Incidents**: runbooks written *before* 3am; mitigate first (rollback — §7 made it cheap), diagnose after; then the **blameless postmortem** — the field's hardest-won cultural artifact: name the systemic causes (the missing guardrail, the confusing interface), never the human (punish honesty once and the next incident's timeline arrives pre-laundered). Action items that actually close the loop.
- **Design for failure** (the [doc 05 §10](05-computer-networks.md) catechism, service-side): timeouts, retries+backoff+jitter, idempotency, circuit breakers, graceful degradation, and **backpressure** (shed load early when overwhelmed) — [../distributed-systems/06-overload-control-and-resilience.md](../distributed-systems/06-overload-control-and-resilience.md) for the full treatment.

# 9 — Working in Teams

Software is a team sport played in text; the coordination practices are engineering, not overhead:

- **Code review** (the PR): the highest-value quality practice we have — catches bugs, but its *larger* products are shared knowledge (no bus-factor-one modules) and converged standards. Review well: small PRs (a 2,000-line PR gets "LGTM"; a 200-line one gets read), review the *design* not just the lines, comment on the code not the person, distinguish blocking issues from nits (say which), approve generously once concerns are met. Author's side: self-review first, explain the *why* in the description, don't take findings personally — the review is the system working.
- **Documentation**, by durability: README (what/why/how-to-run — the front door), ADRs (§5), comments (*why*, in-place), API docs, runbooks (§8). Write for the reader with zero context — who is you, later. Docs rot; near the code they rot slower.
- **Communication defaults**: overcommunicate status (surprises are the sin, not slips); write decisions down (chat evaporates); ask questions after honest effort, *with* the effort shown ("tried X, expected Y, got Z" — coincidentally the shape of a good bug report). Psychological safety — "I don't know," "I broke it," "why do we do this?" being safe to say — is the strongest measured predictor of team performance (and the soil §8's blameless culture grows in).

# 10 — Estimation, Process, and the Classic Failure Modes

- **Why estimates fail**: software tasks are novel by definition (repeats get automated — a compile, a deploy — so what remains is always the unestimated part); unknowns are discovered mid-flight; and **Hofstadter's law** ("it always takes longer, even accounting for Hofstadter's law") is empirically undefeated. Pros: decompose until pieces are small (error grows superlinearly with size), give ranges, track actuals, and treat wild divergence as *information about hidden complexity*, not a performance failing. **Brooks's law**: adding people to a late project makes it later (onboarding + communication overhead, which grows O(n²) in team size — the deepest reason for §5's small-teams-own-modules architecture).
- **Process**: waterfall (specify everything, then build — fails because requirements are *discovered by using software*, not known upfront) vs **agile** — whose actual content is one idea: **short iterations of working software shown to real users, plans revised on feedback** (§1's feedback principle at project scale). The ceremonies (standups, sprints, points) are optional costumes; teams drowning in ceremony while never shipping have kept the costume and lost the idea. Ship thin vertical slices (a working end-to-end sliver, then thicken) — not horizontal layers that integrate never.
- The classic failure modes, named for recognition: the **second-system effect** (the rewrite accumulates every deferred wish and collapses — §4's rewrite warning), **gold-plating** (polishing unasked-for corners), **analysis paralysis** (the cure is a spike: timebox a throwaway experiment, learn, decide), and **the death march** (sustained overtime producing *negative* marginal code — defect injection outpaces progress; sustainable pace is a productivity strategy, not a kindness).

# 11 — Using AI Tools Well

The newest force multiplier, and a genuine shift in the craft. The engineering frame that survives model generations:

- AI assistants are strongest exactly where this document's guardrails are strong: generated code is *plausible* — §3's tests, §7's CI, and §9's review are what turn plausible into trusted. "AI wrote it" changes authorship, not accountability; you ship it, you own it (including its licenses and its security posture — [doc 10](10-security-and-cryptography.md)).
- Leverage tracks specification skill: precise prompts with context, constraints, and examples are the same skill as writing good issues and API docs (§6, §9) — garbage in, plausible-looking garbage out.
- The failure mode to guard: **comprehension debt** — merging code nobody on the team understands is §4's technical debt at compound interest; review AI output at least as hard as a stranger's PR, and use the assistant to *explain and explore* (unfamiliar codebases, error messages, alternatives) as much as to produce.
- What it doesn't change: the essential complexity (§1) — deciding *what to build*, whose trade-off to take, which promise to make strangers (§6) — remains the engineer's job; the tools compress the accidental part. (Building *with* LLM APIs is its own discipline: [../ai/llm-applications/](../ai/llm-applications/).)

# 12 — The Road to Expertise

## 12.1 Books

1. **A Philosophy of Software Design** (Ousterhout) — §1 and §4's spine; short; reread yearly.
2. **The Pragmatic Programmer** (Hunt & Thomas, 20th-anniv. ed.) — the craft's field manual.
3. **Software Engineering at Google** (free online) — time-scale thinking, testing, and tooling at the extreme; skim what doesn't apply.
4. **Refactoring** (Fowler) — the catalog and the mindset; **Working Effectively with Legacy Code** (Feathers) — the survival manual for code without tests (i.e., most code).
5. **The Mythical Man-Month** (Brooks) — 1975 and undated: Brooks's law, second-system effect, no silver bullet.
6. **Site Reliability Engineering** (Google, free online) — §8 in full; **Accelerate** (Forsgren et al.) — the DORA evidence behind §7.
7. **The Phoenix Project** (novel!) — flow, feedback, and ops culture; the painless way to internalize §7–8.

## 12.2 Doing

1. **Git fluency drill** (one afternoon, throwaway repo): branch, merge, *cause* a conflict and resolve it, rebase, bisect a planted bug, reflog-rescue a hard reset. Never fear Git again.
2. **Adopt the loop on a personal project**: repo + branch-per-change + tests + GitHub Actions CI (lint, type-check, test on push) + tags for releases. This — not any book — is where the practices become reflexes.
3. **Write tests for untested code** (yours from six months ago qualifies): feel how design resists or welcomes testing; refactor with Feathers's techniques until it welcomes.
4. **Contribute to open source**: find a `good-first-issue`, read the contributing guide, submit a real PR, absorb the review. The full teams-workflow (§2, §3, §9) with strangers, for free.
5. **Run one project end-to-end** for a real user (a friend's club's site counts): requirements conversation → slices → deploy → monitoring → the 2am bug → the postmortem. §10's lessons only land experientially.
6. Read one great codebase's structure (SQLite, Redis, Flask): map its modules to §1's principles — where are the deep modules? which direction do dependencies point?

## 12.3 Ideas to retain forever

1. **Software engineering is programming integrated over time** — optimize for the change after next.
2. **Complexity is the enemy; modules are the weapon**: deep interfaces, hidden implementations, dependencies pointing at abstractions.
3. **Feedback loops shortened is the meta-practice** — tests, CI, reviews, iterations, monitoring are one idea at five timescales.
4. **Git is a DAG of snapshots**; commits are messages to the future; bisect is binary search over history.
5. **Tests buy fearless change** — test behavior not implementation; testability is design feedback; every bug gets a headstone test.
6. **YAGNI + refactor continuously + manage debt like money** — and almost never rewrite.
7. **Published interfaces are promises to strangers** (Hyrum's law) — version semantically, evolve compatibly, deprecate patiently.
8. **Small frequent releases beat big rare ones** — speed and stability are allies (DORA), not trade-offs; decouple deploy from release with flags.
9. **Production truth**: percentiles not averages, symptoms not twitches, blameless postmortems, design for failure — 100% reliability is the wrong target.
10. **The human system is part of the architecture** — Conway's law, Brooks's law, psychological safety, small PRs; and AI tooling amplifies exactly the engineers who hold the rest of this list.

---

*Next: [Security & Cryptography](10-security-and-cryptography.md) — the adversarial lens over everything built so far.*
