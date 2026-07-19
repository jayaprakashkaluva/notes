# Overload Control & Resilience Engineering

> Part 6 of the realtime distributed systems series. Index: [realtime-high-scale-distributed-systems.md](realtime-high-scale-distributed-systems.md)
> **Scope:** The full overload-control stack with the actual formulas and rules used at Google and Amazon — load modeling, criticality, client-side throttling, retry budgets, backoff/jitter algorithms, load shedding, graceful degradation — then the failure dynamics: cascading failures and the metastable-failure framework that unifies them.
> **Primary sources:** [Google SRE Book Ch. 21 "Handling Overload"](https://sre.google/sre-book/handling-overload/) and [Ch. 22 "Addressing Cascading Failures"](https://sre.google/sre-book/addressing-cascading-failures/) · [AWS Builders' Library, "Timeouts, retries, and backoff with jitter"](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) · [AWS, "Exponential Backoff and Jitter"](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) · [metastable failures research](https://sites.psu.edu/timothyz/metastable-failures/).

---

## 1. Model Load Correctly First

Google abandoned "queries per second" as a capacity/quota unit because "different queries can have vastly different resource requirements"; load is modeled in **resource units — normalized CPU cost** (CPU also proxies memory pressure in GC'd services, since collection burns CPU) ([SRE Ch. 21](https://sre.google/sre-book/handling-overload/)). Per-task overload is detected with the **executor load average** — count of active (running or ready) threads, exponentially smoothed; rejection begins when active threads exceed available processors ([same chapter](https://sre.google/sre-book/handling-overload/)).

Why this matters at review time: every downstream mechanism (quotas, shedding thresholds, autoscaling signals) inherits the load model's errors. A QPS-based limiter is defeated by a query-mix shift with zero traffic growth.

## 2. Criticality: Making Triage Machine-Readable

Google standardizes four request criticality values ([SRE Ch. 21](https://sre.google/sre-book/handling-overload/)):

| Level | Semantics |
|---|---|
| `CRITICAL_PLUS` | Failure causes severe user-visible impact |
| `CRITICAL` | Production default; user-visible but less severe |
| `SHEDDABLE_PLUS` | Partial unavailability expected (batch/async) |
| `SHEDDABLE` | Partial-to-full unavailability routine |

Criticality **propagates automatically through the RPC stack** — downstream calls inherit the caller's level unless deliberately overridden — and is set at the outermost layer. Quota costs, shedding order, and retry policy all key off it. The generalizable practice: overload policy must be *data on the request*, decided where business context exists (the edge), enforced where resources run out (every hop).

## 3. Client-Side Adaptive Throttling

Rejecting a request at an overloaded server still costs that server work; at deep overload, rejection traffic alone can saturate it. Google therefore moves rejection *into the client*: each client tracks, over a two-minute window, `requests` (attempts) and `accepts` (backend-accepted). While the backend is healthy `requests ≈ K × accepts` is far off; as rejections mount, clients begin rejecting new requests **locally**, with probability rising as `requests` exceeds `K × accepts` ([SRE Ch. 21](https://sre.google/sre-book/handling-overload/) — the chapter's formula is `P(reject) = max(0, (requests − K × accepts) / (requests + 1))`). Google typically runs **K = 2** — allowing clients to keep probing with real traffic (a self-correcting signal) while capping wasted work; lowering K (e.g. 1.1) trades more aggressive throttling for slower recovery detection.

This is the pattern AWS frames as *putting the smaller service in control*: the party with scarce capacity must dictate flow, by client self-throttling or by converting push to pull/polling ([Builders' Library](https://aws.amazon.com/builders-library/avoiding-overload-in-distributed-systems-by-putting-the-smaller-service-in-control/)).

## 4. Retry Discipline

Retries are capacity spent by clients to buy availability; undisciplined, they are a load *multiplier* precisely when capacity is scarcest. Google's production rules ([SRE Ch. 21](https://sre.google/sre-book/handling-overload/)):

1. **Per-request budget:** at most 3 attempts; then fail upward.
2. **Per-client budget:** retry only while retries < 10% of the client's total attempts — a client seeing mass failure stops retrying at all (the failures are then almost certainly overload, where retries only harm).
3. **Attempt counter on the wire:** each request carries its attempt number (capped at 2); backends histogram it, and when the histogram shows retry amplification, they return **"overloaded; don't retry"** rather than a generic error — collapsing what would otherwise become combinatorial retry explosion across a deep call stack.
4. **Retry at one layer only.** Per-layer retries multiply: [SRE Ch. 22](https://sre.google/sre-book/addressing-cascading-failures/) prescribes retrying at a single level of the stack.

## 5. Timeouts, Backoff & Jitter — the Algorithms

**Timeouts:** every remote call carries a deadline derived from *observed latency distributions*, not intuition; too high exhausts caller resources during dependency brownout, too low fails healthy-but-slow calls spuriously ([AWS: timeouts/retries/jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)). Google pairs this with **deadline/criticality propagation** so a request whose overall deadline is spent isn't retried deep in the stack ([SRE Ch. 21](https://sre.google/sre-book/handling-overload/), [Ch. 22](https://sre.google/sre-book/addressing-cascading-failures/)).

**Backoff with jitter** — from AWS's simulation study of competing clients under optimistic concurrency ([Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)):

```
capped exponential :  sleep = min(cap, base · 2^attempt)          # still synchronized!
Full Jitter        :  sleep = random(0, min(cap, base · 2^attempt))
Equal Jitter       :  temp  = min(cap, base · 2^attempt)
                      sleep = temp/2 + random(0, temp/2)
Decorrelated Jitter:  sleep = min(cap, random(base, 3 · prev_sleep))
```

Findings: exponential backoff *without* jitter leaves clients clustered at predictable instants — recurring contention spikes; **Full Jitter** minimized total work with near-best completion time; Decorrelated Jitter was competitive with slightly more calls; **"Equal Jitter is the loser."** The article's conclusion: jittered backoff should be the standard for remote clients — and per the Builders' Library, jitter belongs on *all* timers, periodic jobs, and delayed work (cron storms are self-inflicted retry storms) ([AWS](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)).

## 6. Server-Side: Shedding, Degradation, and the Cascade Model

### 6.1 The wasted-work cycle

An overloaded server that answers everyone answers no one: responses that arrive after the client's deadline are **pure wasted work**, the client retries, load rises further ([Builders' Library load-shedding analysis](https://lumigo.io/blog/amazon-builders-library-in-focus-2-using-load-shedding-to-avoid-overload/)). Two escapes:

- **Load shedding** — reject excess *cheaply at admission* (before expensive parsing/dispatch), lowest criticality first, so accepted requests keep meeting their deadlines; deliberately sacrificing some requests raises delivered availability ([same source](https://lumigo.io/blog/amazon-builders-library-in-focus-2-using-load-shedding-to-avoid-overload/); [SRE Ch. 21](https://sre.google/sre-book/handling-overload/)).
- **Graceful degradation** — cut the *cost* of responses instead of their count: answer from an in-memory cache subset instead of full disk, switch to a cheaper ranking algorithm ([SRE Ch. 22](https://sre.google/sre-book/addressing-cascading-failures/)). This is harvest traded for yield ([file 01](01-foundations-time-and-ordering.md) framing).

[SRE Ch. 21](https://sre.google/sre-book/handling-overload/)'s meta-rule: **test the overload regime itself** — load-test each component to breaking point so its saturation behavior is a known design property, not an incident discovery.

### 6.2 Cascading failures ([SRE Ch. 22](https://sre.google/sre-book/addressing-cascading-failures/))

Definition: "a failure that grows over time as a result of positive feedback" — one replica fails/slows → its load lands on the others → they saturate → domino. The chapter's cataloged feedback fuels, all seen above: unbudgeted retries; cold caches after restart (a service provisioned assuming warm-cache hit rates cannot carry miss-storm load — cache-as-hard-dependency); health checkers ejecting *overloaded* (not broken) instances, shrinking capacity as load rises ([file 04 §5](04-membership-failure-detection-gossip.md)); queues growing without bound.

Remediation guidance from the chapter, in triage order: **add capacity and reduce load first, debug later** (the system is in a state where analysis time is load); enter degraded modes; drop traffic classes; and structurally — bounded queues, fail-fast on saturation, per-dependency resource limits. Circuit breakers and bulkheads are the pattern-catalog names for those last two: trip to fail-fast when a dependency's error rate proves it down (probing to re-close), and partition thread/connection pools per dependency so one brownout cannot drain shared capacity.

## 7. Metastable Failures: The Unifying Theory

The framework (HotOS 2021; "Metastable Failures in the Wild," OSDI 2022): "a bad feedback cycle causes the system to get persistently stuck in an overloaded state. **Even after the initial cause of the overload is fixed, the system is unable to recover** due to some sustaining effect that perpetuates the overload" ([overview + papers](https://sites.psu.edu/timothyz/metastable-failures/)).

The vocabulary, which is the value:

- **Vulnerable state** — healthy, but a feedback loop is armed (e.g., running above the load where retry amplification is self-sustaining; cache hit rate carrying capacity).
- **Trigger** — any transient: a deploy, a blip, a surge. *Fixing the trigger does not end the outage* — that is the definition of the class.
- **Sustaining effect** — the loop that keeps the overloaded equilibrium stable: retry storms, cold-cache miss amplification, queue backlogs (all of §6.2's fuels, now named as a system property).

Documented at AWS, Azure, and Google Cloud with major revenue impact; simplified reproductions at the [Metastability repo](https://github.com/lexiangh/Metastability); detection/prevention/recovery remain open research problems ([overview](https://sites.psu.edu/timothyz/metastable-failures/)).

Operational consequences:

1. **Recovery must break the loop, not restore the trigger-free state**: shed drastically below normal capacity, disable retries, warm caches before readmitting traffic — each maps to a §4–§6 mechanism run in reverse.
2. **Track distance-to-vulnerability, not just utilization** — the vulnerable/stable boundary (e.g., the load above which a full retry storm is self-sustaining) is a capacity number worth computing and alerting on.
3. Every mechanism in §3–§6 is, in this vocabulary, a **sustaining-effect suppressor**: retry budgets cap the retry loop's gain; client throttling and shedding cut load before the loop closes; jitter decorrelates the synchronized spikes that arm it. That is the correct mental model for *why* this machinery exists — not politeness, but keeping the system's only stable equilibrium the healthy one.
