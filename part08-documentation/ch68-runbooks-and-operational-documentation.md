# Ch 68 — Runbooks and Operational Documentation

**Prerequisites:** [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) (MTTR), [Coverage: What It Measures and What It Doesn't](../part05-testing-strategy/ch41-coverage-what-it-measures-and-what-it-doesnt.md) (execution-verification gap), [Keeping Documentation Honest](ch66-keeping-documentation-honest.md)

**New vocabulary introduced:** paper runbook

**Key takeaways:**
- A runbook is a script for one specific, named failure mode — the failure condition, verification steps, recovery commands, and escalation criteria — not general architecture explanation (a README's job) and not an open-ended troubleshooting essay.
- A paper runbook — one written but never exercised — carries the same false-confidence risk Ch 41 already established for coverage metrics: its existence is not evidence it works. [Strong Recommendation] Every runbook should be exercised deliberately — a game-day drill or an onboarding walkthrough, at minimum — rather than trusted because it was carefully written.
- Any step that is genuinely mechanical and deterministic should become automation instead of prose (Principle 8). A runbook that keeps growing is often a signal that more of it should have become tooling.
- What triggers a responder to open a runbook (an alerting decision, Ch 71) and what a runbook ultimately protects (an SLO or error budget, Ch 73) are both out of scope here; so is embedding a credential directly instead of pointing to where to obtain one (Ch 83).
- A runbook's value is judged by exactly one thing: whether it reduces MTTR (Ch 07). Length and apparent thoroughness are not the metric, and can actively work against it.

---

Every other reader in this Part is doing ordinary development work with time to think. A runbook's reader isn't — they're responding to a known operational failure, often at an inconvenient hour, often with incomplete information. That difference changes the correctness bar. Ordinary stale documentation wastes a reader's time (Ch 66); a wrong runbook step wastes exactly the time a responder doesn't have, during the specific situation the document exists to make faster. This chapter covers what belongs in that document and how it earns the trust a responder places in it under pressure — not the alerting that gets someone to open it, and not the broader incident-response or postmortem process around it, which this handbook doesn't allocate a dedicated chapter to.

### Decision: Write a Runbook Around One Named Failure Mode

**What it is:** A runbook answers a single question — "the system is exhibiting this specific failure, what do I do?" — rather than serving as general operational or architectural documentation.

**Why it exists:** General troubleshooting guidance is hard to apply under pressure; a procedure tied to a specific, recognizable symptom isn't. Mixing the two — background theory alongside recovery steps — forces a responder to parse context they don't need before reaching the command they do, and every extra paragraph is time added directly to MTTR.

**Options:** A runbook contains the failure condition, initial verification steps, diagnostic commands with expected output, the recovery procedure, and escalation criteria with contacts. It does not contain system architecture, onboarding context, or design rationale — those belong to the README (Ch 65) and other artifacts this Part already covers.

**Trade-offs:** A failure-specific runbook is fast to follow and easy to locate during the exact failure it names, at the cost of one more document to maintain per failure mode. A general operational guide covers more ground in one place, but forces a responder to interpret which part applies while the incident is active — exactly the cognitive cost a runbook exists to remove.

**Common failure modes:** A runbook opens with several pages of architectural background before reaching the actual mitigation command. During an active incident, a responder burns critical minutes skimming theory to find the one shell command that matters, while the failure continues.

**Example:** PagerDuty's published incident-response templates organize runbooks around recognizable incident types, with the highest-visibility section reserved for immediate actions — exact commands and console links — rather than leading with system description.

---

### Decision: Every Step Should Be Something a Responder Executes, Not Interprets

**What it is:** Each instruction names a concrete action — a command to run, a dashboard to check, an expected output — rather than general advice a responder has to translate into action themselves.

**Why it exists:** A responder mid-incident shouldn't have to convert "investigate the issue" into an actual first move. Every step that requires interpretation is a step that can be interpreted differently by different responders, or badly by a tired one.

**Options:** Useful steps specify the exact command to run, the exact dashboard or log to check, and the exact threshold that triggers escalation. Vague guidance — "investigate," "check the system," "look for errors" — is not a step; it's a restatement of the fact that something is wrong.

**Trade-offs:** Specific, executable steps produce faster, more consistent responses, at the cost of needing an update whenever the underlying tooling or procedure changes — the same maintenance obligation every artifact in this Part carries, applied here to something with a much higher cost of being wrong. Vague guidance requires less maintenance and is more likely to still be technically applicable years later, but at the direct cost of the response time and consistency a runbook exists to provide.

**Common failure modes:** A step ends with "continue investigating" instead of a next command. A responder who has never touched the affected system has no way to translate that into action, and the runbook has effectively ended without resolving anything.

**Example:** A useful step names the exact endpoint to query, the exact status to expect back, and the exact condition — "if this persists past five minutes" — that means escalate, rather than describing the failure and trusting the responder to work out what to check.

---

### Decision: Treat an Unexercised Runbook as Unverified — a Paper Runbook

**What it is:** A runbook that has been written but never actually executed against a real or simulated failure — a **paper runbook** — whose correctness is an assumption, not a demonstrated fact.

**Why it exists:** Ch 41 already established that a coverage metric measures execution, not verification — code can run inside a test without anything checking that it did the right thing. A runbook exhibits the identical gap: the document existing is not evidence it works. Worse, the only reader positioned to discover a missing step or an obsolete command is someone already mid-incident, which is the single worst moment for that discovery to happen for the first time — a more expensive version of Ch 66's honesty problem, because the reader here has no slack to absorb the surprise.

**Options:**
1. **Exercise it** — an onboarding walkthrough, a tabletop exercise, a scheduled game day, or a controlled failure simulation.
2. **Review it without running it** — read the document periodically and update anything that looks wrong.
3. **Leave it unexercised** — trust that careful writing was sufficient.

**Trade-offs:** Exercising a runbook costs real engineering time, but it's the only option that actually validates the commands still work, the responder still has the permissions the steps assume, and the escalation contacts still exist. Reviewing without running catches obviously wrong prose but can't catch a command whose flag was renamed by an unrelated tooling change — the document can look correct and still fail the moment it's executed for real. Leaving it unexercised is free until the first real incident, which then pays the entire accumulated cost at once, in production, under time pressure.

[Strong Recommendation] Exercise every runbook that protects a system whose failure matters, on a cadence that matches how critical the path is. A runbook nobody has run is a hypothesis about how to recover, not a verified procedure.

**Common failure modes:** *The stale command trap.* A runbook instructs a responder to run a utility with a flag that was removed six months earlier during an unrelated rewrite. At 2 a.m., the command returns an error instead of clearing the failure, and the responder now has to debug the runbook itself before they can even start debugging the actual incident.

**Example:** Game-day exercises, in the lineage of Netflix's Chaos Monkey, deliberately trigger a failure under controlled conditions and hand the responder the runbook exactly as written. If the documented steps don't recover the system, that's logged as a failure of the runbook, not just the drill — and the runbook is fixed before it's trusted again.

---

### Decision: Automate the Mechanical, Keep the Judgment

**What it is:** Any runbook step that is fully deterministic — the same trigger always calls for the same action — is a candidate to become automation rather than remain prose a human executes by hand.

**Why it exists:** This is Principle 8 one last time: mechanical enforcement over human discipline. Google's Site Reliability Engineering practice names this directly as toil — repetitive, mechanical operational work with no enduring value, which should be eliminated through automation rather than repeatedly performed and documented. A runbook that only grows, never shrinks, is usually a sign that steps which should have become tooling are still prose instead.

**Options:** Manual execution is appropriate for judgment and diagnosis — deciding whether a symptom warrants escalation, or which of several plausible causes applies. Automation is appropriate for deterministic recovery, repetitive validation, and environment preparation. A hybrid keeps the mechanical portion automated and leaves the human step to decide when to trigger it and confirm the result.

**Trade-offs:** Automation drives recovery time down and removes operator error, at the cost of needing to be built and then maintained itself — it doesn't remove the maintenance obligation, it relocates it from documentation to code. Manual procedures are cheaper to write initially but accumulate real operational toil every time the same mechanical sequence is run by hand.

**Common failure modes:** A team experiences the same recoverable failure repeatedly as their fleet grows — five nodes becomes two hundred — and keeps responding with the same manual runbook instead of automating it. The alert frequency scales with fleet size; on-call time spent executing an unchanged mechanical sequence eventually consumes the engineering capacity that would have fixed the underlying cause.

**Example:** Many teams replace a multi-step manual rollback procedure with a single verified automation command once the sequence has proven itself stable. The runbook that remains is shorter and more valuable: it now covers only the judgment call of *when* to roll back, not the mechanics of how.

---

Three boundaries are worth naming without expanding into them. What triggers a responder to open a runbook in the first place is an alerting decision — the signal-versus-noise tuning covered in Part IX, Ch 71 — not this chapter's subject. What a runbook ultimately protects is an SLO or error budget, covered in Part IX, Ch 73. And any step requiring a credential should point to where to obtain it — Part XI, Ch 83 owns secrets management — never embed one directly in the document itself.

### Why Smart Engineers Disagree

The disagreement isn't about whether MTTR matters — everyone agrees a runbook exists to reduce it. It's about where the long-term investment should go: more operational guidance, or more operational automation.

One position favors comprehensive runbooks that cover nearly every foreseeable variation of a failure, arguing that a well-prepared responder with explicit guidance performs better than one working from a thinner document and their own judgment.

The other treats every repeated manual step as unfinished engineering work rather than permanent documentation, deliberately automating aggressively so that runbooks shrink over time as tooling absorbs the deterministic portions of recovery, leaving prose only for what genuinely still requires a human.

Both share the same objective and the same maintenance constraint — every manual step is a future point of drift (Ch 66) and every hour spent executing it by hand is toil that could have gone toward the fix that makes the runbook unnecessary. The practical answer tracks how often and how predictably a failure recurs: a rare, well-understood failure may never justify the engineering cost of automating it, and the guidance position wins by default; a frequent, well-understood one accumulates enough repeated manual cost that automation pays for itself quickly, and the runbook is what's left over once it does.
