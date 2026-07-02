# Ch 73 — Error Budgets and SLOs

**Prerequisites:** [What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md) (optimization targets), Principle 6, [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) (MTBF, MTTR), [Environment Promotion](../part07-git-and-delivery/ch62-environment-promotion.md) (canary and progressive rollout), [Runbooks and Operational Documentation](../part08-documentation/ch68-runbooks-and-operational-documentation.md), [Alerting: Signal vs. Noise](ch71-alerting-signal-vs-noise.md) (symptom-based alerting, actionability test)

**New vocabulary introduced:** Service Level Indicator (SLI), Service Level Objective (SLO), Service Level Agreement (SLA), error budget, burn rate

**Key takeaways:**
- Three distinct layers are routinely conflated: an SLI is the measured value; an SLO is the internal target for it; an SLA is the external, often contractual commitment — deliberately set looser than the SLO to leave a defensive margin before a missed internal target becomes a missed, and often penalized, external one.
- [Strong Recommendation] An error budget is the explicit, spendable inverse of an SLO (1 − SLO) — a 99.9% target grants roughly 43 minutes of allowed failure a month — and it should be spent deliberately on risky deploys, migrations, and experiments, not hoarded. A service running at 100% against a 99.9% objective isn't over-delivering; it's under-spending a resource its own target already allows.
- [Strong Recommendation] Gate pages on burn rate measured across multiple time windows, not a single static error-rate threshold. A static threshold either wakes someone for a brief, self-correcting spike that costs a negligible fraction of the budget, or stays silent while a slow, low-grade leak drains the entire month's budget over days without ever crossing the line.
- The budget this chapter defines is the metric that gates Ch 62's canary and progressive-rollout promotion decisions, and the target Ch 68's runbooks exist to protect by recovering quickly — both referenced here as the destination this chapter's math points at, neither re-derived.
- This chapter closes Part IX by naming what logging, the signal taxonomy, alerting, and tracing all ultimately serve: none of it is collected for its own sake — it exists to measure a system against an explicit, numeric reliability target.

---

Every signal Part IX has covered so far — what to log, which signal answers which question, what's worth paging a human for, how to trace a request across a boundary — assumes a target already exists to measure against. This chapter supplies that target, closing three forward references at once: Ch 07 promised error budgets, SLOs, and alerting strategy to this Part by number; Ch 62 promised the specific metric that gates a canary promotion; Ch 68 promised what a runbook ultimately protects. It also returns directly to Ch 01: once reliability has an explicit numeric target, engineering effort spent past that target is a trade-off to justify, not an unquestioned virtue.

### Decision: Separate the SLI, the SLO, and the SLA Into Distinct Layers

**What it is:** Three related but distinct concepts get routinely conflated into one: the SLI is what's actually measured, the SLO is the internal target for that measurement, and the SLA is the external, often contractual commitment made to customers.

**Why it exists:** Anchoring internal operational response directly to an external SLA destroys the defensive margin Ch 07 depends on for early mitigation. An SLA is set by legal and business functions specifically to protect the organization, with a wide safety buffer built in on purpose. If a team pages only when the SLA itself is breached, by the time that page fires, the system has already crossed into a contractual — often financial — violation, with none of the runway Ch 07's MTTR argument assumes an alert should provide.

**Options:**
- **Unified, SLA-targeted instrumentation** — page only when the externally published commitment is actually breached.
- **Multi-tiered taxonomy** — an SLI (the raw measurement), a tighter internal SLO (e.g., 99.9%), and a looser external SLA (e.g., 99.0%), kept as three separately tracked numbers.

**Trade-offs:** A single unified target requires minimal configuration and gives every audience — engineering, executives, customers — one number to reason about, but it eliminates all defensive margin: any unpredicted failure crosses directly into contractual violation before a runbook can even be opened. Separating the three layers costs ongoing data-engineering discipline — maintaining and documenting separate rolling calculations across every service — but buys an explicit early-warning window: a service burning through its 99.9% SLO gets a team paged hours or days before that erosion threatens the looser 99.0% SLA line.

**When to choose each:** [Strong Recommendation] A multi-tiered taxonomy for any production system with real commercial or customer-facing availability expectations. A unified target is defensible only for an internal, non-commercial tool where a missed target carries no real financial or reputational consequence.

**Common failure modes:** *The SLA-equalized pager melt.* A gateway service publishes a legal SLA of 99.9% uptime, and the team wires its paging rule to the identical 99.9% number. A database connection storm drops success rates to 99.8% for twenty minutes. Because there was no internal margin between the operational alert and the external commitment, the dip is simultaneously the first the team hears of it and a breach of a paid contract — triggering six-figure service credits before the on-call engineer has even opened a dashboard to look for a cause.

**Example:** Major cloud providers publish external SLAs — 99.9% or 99.99% monthly uptime, with rebate tiers attached — that are consistently looser than the internal SLOs their own infrastructure teams operate against. The gap between the two numbers is the organization's own early-warning runway, made explicit as a design choice rather than left to chance.

---

### Decision: Treat the Error Budget as a Resource to Spend, Not a Score to Protect

**What it is:** An error budget is the direct mathematical consequence of an SLO — 1 minus the objective — representing the volume of failure a service is explicitly permitted to spend within a given window, not an indicator of how well the team is doing.

**Why it exists:** Treating every unit of unreliability as an unqualified failure, and chasing availability toward 100% regardless of the stated objective, is Principle 6 violated directly: moving from three nines to four nines costs exponentially more engineering effort for a improvement most users will never perceive. The budget exists precisely to put a ceiling on that instinct — a 99.9% SLO isn't an aspiration, it's an explicit grant of roughly 43 minutes of allowed failure a month, and a team that never spends any of it isn't excelling, it's declining to take risk the objective already authorized.

**Options:**
- **Zero-tolerance hoarding** — treat any budget consumption as a crisis, pushing availability as close to 100% as possible regardless of the stated SLO.
- **Strategic spending** — treat the budget as a resource to deliberately consume on deployments, migrations, and controlled experiments, as long as the rolling window stays within the SLO.

**Trade-offs:** Hoarding produces a pristine historical uptime graph, at the cost of paralyzing delivery — engineers spend real time defensively wrapping already-stable code paths against risk the objective itself says is acceptable to take. Strategic spending maximizes delivery velocity and gives teams a factual basis for accepting deployment risk, but demands real organizational discipline: leadership has to actually stand down when a deploy consumes budget as intended, rather than treating any dip as a postmortem-worthy failure.

**When to choose each:** [Strong Recommendation] Strategic spending for essentially every product service and API. An error budget left completely untouched at the end of its tracking window is evidence of restricted velocity, not success. Zero-tolerance hoarding is defensible only for the narrow set of systems where a single failure carries irreversible, life-critical, or existential consequences — a flight-control loop, a core financial ledger's isolation kernel.

**Common failure modes:** *The stagnant innovation outage.* A team maintains an unspent, 100% availability record on an internal service for a year out of risk aversion, repeatedly deferring a database engine upgrade to avoid any risk of a transient connection drop. When a zero-day vulnerability forces an emergency patch, the team has no incremental migration path left — only a single, untested, "big-bang" cutover attempted under pressure. The untested schema change fails catastrophically, and the service it was protecting is down for three days: the exact outcome the hoarding was meant to prevent, caused directly by the hoarding.

**Example:** As formalized in Google's Site Reliability Engineering practice, the error budget functions as a mechanical gate on Ch 62's canary and progressive-rollout promotions — the promotion mechanism itself belongs entirely to that chapter, but the criterion it checks before continuing a rollout is this chapter's budget. When the rolling window's budget is exhausted, further automated promotion halts, and engineering attention shifts to recovery — which Ch 68's runbooks exist to accelerate, since every minute of unresolved incident is budget still being spent.

---

### Decision: Gate Pages on Burn Rate Across Multiple Windows, Not a Static Threshold

**What it is:** Whether to trigger an alert from a fixed, instantaneous error-rate threshold (error rate > 1% for five minutes), or from the mathematical rate at which the error budget itself is being consumed, evaluated across more than one sliding time window at once.

**Why it exists:** A static threshold fails in both directions against a budget it isn't actually measuring. A brief, total failure spike — a container restart causing a 15-second 100% error burst — crosses a static threshold instantly and pages someone, even though it consumed a negligible fraction of the monthly budget. A slow, persistent leak sitting fractionally under the threshold — a subtle bug returning errors for 0.5% of requests when the alert fires at 1% — never pages at all, and can silently exhaust the entire month's budget days before anyone notices.

**Options:** A **burn rate** is the ratio of the current error rate to the rate that would exactly exhaust the budget over its full window. Evaluating it across two windows at once catches both failure shapes a single static threshold misses:

```
Burn rate = observed error rate / allowed error rate

Critical page  — burn rate > 14.4 sustained over a 1-hour AND 5-minute window
                 (consumes ~2% of a 30-day budget in 1 hour — fast, severe)

Warning ticket — burn rate > 1.08 sustained over a 3-day window
                 (consumes ~10% of a 30-day budget in 3 days — slow, still real)
```

**Trade-offs:** A static threshold takes little effort to write and needs no historical calculation, but floods on-call with false positives from transient blips while missing exactly the slow leaks that matter most over a full window. Multi-window burn-rate alerting requires real query complexity — nested calculations most engineers won't write correctly from scratch — but eliminates pages for self-correcting noise while guaranteeing detection of a leak long before it fully drains the budget.

**When to choose each:** [Strong Recommendation] Multi-window burn-rate alerting for any production system where alert fatigue (Ch 71) must be actively controlled. A static threshold remains adequate only for a non-production environment where a false page costs nothing and a missed slow leak has no real consequence. How the resulting page reaches a human — routing, escalation, on-call rotation — is Ch 71's decision entirely; this chapter only supplies the number that decides whether a page should fire at all.

**Common failure modes:** *The silent 30-day budget death.* A billing service with a 99.9% SLO develops a bug that returns errors for 0.8% of payment submissions — comfortably under the team's static 2% alert threshold. The dashboard stays green. Twelve days pass with the bug live in production, quietly consuming roughly 80% of the service's monthly error budget, before a customer-facing team notices the pattern manually and escalates it — nearly two weeks after a burn-rate alert on the same data would have caught it in hours.

**Example:** Google's SRE Workbook documents this exact multi-window, multi-burn-rate approach as the mechanism that connects a budget's math to Ch 71's alerting decision: a fast, high-severity window catches a sudden spike before it does serious damage, and a slower window catches a persistent leak a single-window alert would never surface.

---

The mechanics of how traffic actually shifts during a canary or progressive rollout belong entirely to Ch 62; this chapter supplies only the burn-rate signal that gates whether promotion continues. The structure and content of the runbook a responder opens once paged belong entirely to Ch 68; this chapter only establishes that recovering faster directly protects the budget being spent. And how a page generated by this chapter's math actually reaches a human — routing, escalation, on-call rotation — is Ch 71's decision, referenced here, not re-derived.

### Why Smart Engineers Disagree

The disagreement isn't about whether an exhausted budget matters — it's about whether exhausting it should trigger an automatic, mechanical consequence or a human, contextual one.

One position argues that an error budget policy is meaningless unless it's enforced by an unyielding, automated gate: if a service's rolling budget is exhausted, the deployment pipeline should freeze all new feature promotion automatically, admitting only security patches and reliability fixes until the budget recovers. This position holds that product pressure will always argue for one more exception if a human is left to decide case by case, and that only a hard, programmatic barrier reliably protects stability from that pressure.

The opposing position treats an automatic freeze as its own failure mode: a missed SLO is a signal that needs human interpretation, not an automatic penalty. If a budget was burned by an upstream provider's outage entirely outside the team's control, freezing that team's own feature delivery provides no safety benefit and actively damages the business for a failure it didn't cause. This position argues for a required postmortem review instead, where engineering and product leadership jointly decide whether an override is warranted given the actual cause and blast radius.

The resolution tracks the same two axes this handbook has used for this exact tension before (Ch 09, Ch 71): reversibility and system maturity. In a highly decoupled, microservice architecture where one unstable service's automated freeze can't cascade into stalling unrelated, healthy teams, an automated block is cheap to apply and prevents an unstable dependency from spreading further risk. In a tightly coupled monolith or an early-stage system still chasing product-market fit, the same automated freeze is disproportionate — it can stall the whole organization over a budget miss with no real blast radius. A practical middle path locks only the automated, low-trust path — Ch 62's automatic progressive rollout — while leaving an explicit, audit-logged override available to a human who can weigh the actual cause. Either way, the trigger point is the same: this chapter's number decides when the question gets asked at all.
