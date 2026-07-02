# Ch 71 — Alerting: Signal vs. Noise

**Prerequisites:** [What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md) (optimization targets), Principle 6, [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) (MTTR), [Process Overhead: The Value Threshold](../part06-engineering-process/ch49-process-overhead-the-value-threshold.md) (proxy-metric process failures), [Runbooks and Operational Documentation](../part08-documentation/ch68-runbooks-and-operational-documentation.md), [Logging: What to Log and at What Level](ch69-logging-what-to-log-and-at-what-level.md) (actionability test), [Metrics vs. Logs vs. Traces](ch70-metrics-vs-logs-vs-traces.md)

**New vocabulary introduced:** alert fatigue, symptom-based alerting

**Key takeaways:**
- An alert is a decision to spend a scarce, non-scalable resource — a human's attention, often outside working hours — not a notification. Ch 69's actionability test applies here at a higher stakes level: page only when the condition has a clear next action a human, not software, must take.
- Alert fatigue is this chapter's central failure mode, and the direct monitoring-domain instance of Ch 49's proxy-metric failure: a paging system optimizing for "an alert was sent" instead of "a human usefully responded" trains responders to dismiss reflexively, and the one alert that matters gets the same dismissal as the hundred before it that didn't.
- [Strong Recommendation] Default to symptom-based alerting — paging on conditions a user would actually notice (elevated error rate, elevated latency) — over cause-based alerting on every internal condition that could plausibly produce one (a CPU threshold, one unhealthy replica). Most internal causes self-correct through the same redundancy and autoscaling the system was built with; paging on all of them manufactures exactly the noise the fatigue argument warns about.
- Not every detected anomaly deserves the same channel. [Strong Recommendation] Route by calculated urgency into distinct tiers — page-worthy, ticket-worthy, log-worthy — and by ownership, to the smallest group actually capable of resolving the condition without a handoff.
- This chapter decides which signals from Ch 69 and Ch 70 clear the bar to interrupt a human; it does not cover what a responder does once paged (Ch 68) or the budget math that ultimately justifies the bar (Ch 73).

---

Ch 68 already defines what a responder reads once paged. This chapter defines what decides to page them in the first place. Applying Principle 6 to real-time operations: interrupting an engineer, especially outside working hours, is a real and expensive operation, and it is justified only when the cost of not knowing immediately exceeds the cost of the interruption itself. Most conditions a monitoring system detects fail that test — they belong on a dashboard or in a ticket, not in someone's pocket at 3 a.m.

### Decision: Gate Every Page Behind the Actionability Test

**What it is:** A page should fire only when the condition has a clear, human-executable next action — ideally one already documented in a Ch 68 runbook — not merely because a metric crossed a threshold or a log line looked interesting.

**Why it exists:** Left unconstrained, monitoring configuration drifts toward alerting on anything an engineer found notable while setting it up. That shifts the system's real optimization target from "a human usefully responded to prevent or mitigate an outage" to the proxy "an anomaly was detected and transmitted" — the identical failure Ch 49 already named for process generally, applied here to paging instead of ticketing. A page with no available response doesn't inform anyone of anything actionable; it only spends attention.

**Options:**
- **Comprehensive anomaly paging** — page on any metric deviation, error spike, or state change outside a statistical baseline.
- **Action-gated paging** — restrict pages to conditions with an expected responder, a known investigation path, and ideally a runbook to follow.

**Trade-offs:** Comprehensive paging guarantees the team is aware of every unusual system state as it happens, but production systems produce constant transient anomalies that self-correct on their own — retries, autoscaling, cache warm-ups — and bombarding responders with all of them breaks focus and sleep for no corresponding benefit. Action-gated paging means a page firing at 3 a.m. carries near-certainty that a real boundary was crossed and manual intervention is required, at the cost of a real risk: an early, non-fatal signal of a genuinely novel failure mode might not yet map to any documented response, and stays off the pager until it does.

**When to choose each:** [Strong Recommendation] Action-gated paging for every production service, live API, and automated infrastructure component. If a condition can wait for an asynchronous ticket or a morning dashboard review, sending a page for it is a configuration failure, not a safety margin. Comprehensive anomaly paging is defensible only inside a bounded window — a chaos-engineering exercise, a high-risk migration's first 48 hours — where the cost of extra noise is temporarily worth the extra coverage.

**Common failure modes:** *The reflexive dismissal routine.* A team pages on-call whenever a worker node's temporary disk reaches 85% capacity, because a cleanup cron job always clears it within fifteen minutes — the condition never requires human action. Over three months the rotation absorbs 600 such pages. When a genuine failure later fills the same disk to 100% in under two minutes from a runaway log loop, the responder reflexively acknowledges and silences it without reading the payload, assuming it's the usual self-correcting 85% alert. The system stays down for four hours — not because the alert was inaccurate, but because 600 non-actionable pages had already taught the responder that this alert never needed a response.

**Example:** A payment gateway returning errors to customers should page immediately — the failure is user-visible, urgent, and has a defined mitigation. A failed nightly report generation job, with no user impact and no time-sensitive fix, belongs in a ticket for the next working day, not a page.

---

### Decision: Alert on Symptoms Users Would Notice, Not Every Internal Cause That Could Produce One

**What it is:** The choice between triggering pages from user-visible degradation (a symptom) versus from internal component-level conditions that might, but frequently don't, ever reach a user (a cause).

**Why it exists:** Cause-based alerting is the single largest driver of alert fatigue, because modern systems are built with resilience layers — load balancers, retries, circuit breakers, autoscalers — specifically so that component-level anomalies don't reach users. An alert wired to the internal condition ignores the fact that the system was deliberately engineered to absorb it, and pages a human for an event the architecture already handled.

```
Cause-based (high noise):
Host CPU > 90%  ──> Page ──> Self-corrects via autoscaling ──> Alert fatigue

Symptom-based (high signal):
Client success rate < 95%  ──> Page ──> Requires human intervention ──> MTTR reduced
```

**Trade-offs:** Cause-based alerting gives earlier visibility into component-level physics — an overheating replica, a leaking memory pool — hours before it would exhaust a resource buffer, at the cost of a high false-positive rate: a routine garbage-collection pause or a scheduled index rebuild can saturate a host's CPU for minutes while an edge cache continues serving every client request without any degradation. Symptom-based alerting has a near-zero false-positive rate — when it fires, the system actually is failing a user or a consumer — at the cost of not firing until degradation has already begun, converting what a cause-based signal might have caught proactively into an active incident.

**When to choose each:** [Strong Recommendation] Symptom-based alerting as the default gatekeeper for every page-worthy, wake-someone-up condition. Reserve cause-based signals for secondary, non-paging pipelines: feeding an automated remediation system (a Kubernetes autoscaling policy), populating a dashboard for root-cause analysis once a symptom page has already fired, or routing to an asynchronous ticket queue.

**Common failure modes:** *The batch ingestion CPU storm.* A nightly data-sync job intentionally saturates 100% of a host's CPU cores to finish quickly, with zero impact on the user-facing servers it runs alongside. A generic cause-based rule (CPU > 85% for five minutes → page) wakes the on-call engineer every night at 12:05 a.m. for a month, for a condition that never once affected a real user. Two senior engineers leave the rotation, and eventually the team, over the accumulated sleep debt from a single improperly scoped alert.

**Example:** Google's Site Reliability Engineering practice codifies this as the "Four Golden Signals" — latency, traffic, errors, and saturation — with pages wired strictly to a drop in client-facing success rate or a breach of a latency percentile budget. Host-level metrics like raw memory or disk I/O remain available on a dashboard as contextual evidence for the responder once they're already investigating, not as independent triggers in their own right.

---

### Decision: Tier Alerts by Time-to-Impact and Route by Ownership

**What it is:** Whether every detected condition funnels into one shared channel, or is partitioned into distinct tiers — page, ticket, or passive signal — by urgency, and routed to whichever team can actually resolve it.

**Why it exists:** Treating every anomaly identically eliminates prioritization: a routine capacity-planning note and a customer-facing outage compete for the same attention in the same channel, and responders lose the ability to tell which one requires acting on right now. Fast notification also has little value if it reaches a team without the ownership or permission to resolve the condition — time is then lost to a handoff instead of a diagnosis.

**Options:** A practical model partitions detected conditions into three tiers by time-to-impact — **page-worthy** (immediate human response required), **ticket-worthy** (investigate during normal working hours), and **log-worthy** (no alert; recorded as a queryable signal per Ch 69 and Ch 70, with no action expected). Within the page-worthy tier, route by service or infrastructure ownership, escalating to a broader group only if the initial responder doesn't acknowledge within a defined window.

**Trade-offs:** A single flat channel is trivial to set up and gives every engineer full visibility into every event, but buries the rare urgent signal under routine, high-volume noise, extending MTTR exactly when speed matters. Tiered, owned routing keeps the page-worthy channel quiet enough that a notification there commands full attention, and gets routine conditions resolved on a normal schedule instead of interrupting anyone — at the cost of ongoing discipline to keep the tier and ownership mapping accurate as services and teams change.

**When to choose each:** [Strong Recommendation] Tiered, ownership-based routing for any organization running live production traffic across more than one team. Flat routing is defensible only for a small, early-stage team (roughly under four engineers) whose total daily signal volume is still small enough for everyone to track by hand.

**Common failure modes:** *The shared Slack firehose.* An organization retires its paging tool to cut licensing cost and routes every alert — routine container scaling, minor lock delays, test-suite runs — into one shared channel with desktop notifications on for the whole team. Within a week the channel receives 14,000 messages a day. When the payment gateway's SSL certificate silently expires, the critical warning is pushed off-screen within seconds by unrelated noise, and the outage runs for 14 hours before a customer support escalation surfaces it manually — the same failure the fatigue argument predicts, concentrated into a single unfiltered channel instead of a single overused alert.

**Example:** Incident-routing platforms like PagerDuty implement this tiering directly: page-worthy alerts bypass do-not-disturb settings entirely — phone calls, SMS, multi-level on-call escalation — while lower tiers file directly into a ticket queue for the next working day, with no interruption to anyone already focused on live work.

---

What a responder does once paged, and how the runbook they follow is structured, is Ch 68's content, not this chapter's. The mechanics of collecting, indexing, and storing the logs and metrics an alert evaluates belong to Ch 69 and Ch 70; this chapter only decides which of those already-collected signals justify a page. And the specific math behind how much error or latency a system can tolerate before an alert must fire — the error budget and burn rate an alert is ultimately protecting — is Ch 73's, referenced here as the target, not derived.

### Why Smart Engineers Disagree

The disagreement isn't over whether alert fatigue is real — everyone agrees it is. It's over how much proactive, cause-based warning is worth the fatigue risk it carries.

One position argues that pure symptom-based alerting is structurally reactive: waiting for a user-visible error or latency spike means the system has already failed someone before a human is even notified. This position favors aggressive predictive alerting on saturation trends — a disk projected to hit capacity in four hours, a thread pool trending toward exhaustion — treating early warning as a controlled, proactive intervention rather than an unnecessary interruption.

The opposing position treats predictive, cause-based paging as fragile in practice: automated projections trigger false alarms from transient batch spikes and shifting traffic patterns constantly, and every page that turns out to need no action erodes the same trust in the paging system that the fatigue argument already warns about — directly inflating MTTR the next time a real incident needs fast, trusting response. This position argues that if a condition is predictable hours in advance, the correct fix is automated remediation — an autoscaling policy, a log rotator — not a human page.

The resolution tracks two axes this handbook has already named: reversibility and blast radius (Ch 09). If a saturation condition cannot be mitigated by software alone — a legacy database that requires a manual node rebuild to add storage — and the failure it leads to is irreversible, such as structural data corruption, proactive cause-based paging is justified; the cost of an occasional unnecessary page is small next to the cost of the failure it prevents. If the underlying infrastructure is elastic and self-healing — a stateless pod behind an autoscaler — the same proactive alert is accidental complexity with no failure left for it to prevent. Whichever side of that line a specific alert falls on, it is held to the same bar as any other page: if a responder repeatedly opens a console only to watch a cause-based alert self-correct, it has failed the actionability test and belongs at the platform layer, not on the pager.
