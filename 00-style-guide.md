# Style Guide

This document defines the conventions every chapter in this handbook follows. When synthesizing raw drafts, apply these rules. When writing raw drafts, use this as a target.

---

## Chapter Header Format

Every chapter begins with three short sections before the main content:

```
**Prerequisites:** [List concepts assumed from earlier chapters, or "None" for Part I, Ch1]

**New vocabulary introduced:** [Terms defined in this chapter that later chapters will use]

**Key takeaways:**
- [3–5 bullet points — the things a reader should retain after one read]
```

---

## Decision / Principle Template

Every discrete decision or principle follows this structure:

```
### [Decision or Principle Name]

**What it is:** One sentence.

**Why it exists:** The problem this decision or principle addresses.

**Options:** The realistic alternatives — not strawmen.

**Trade-offs:**
[Honest pros and cons per option. Tables are encouraged for side-by-side comparison.
Take positions where evidence is strong. Do not hedge everything with "it depends."]

**When to choose each:** Concrete signals that indicate which option fits.

**Common failure modes:** How this goes wrong in practice, with observable symptoms.

**Example:** A brief concrete illustration, preferably from a real system.
```

---

## Recommendation Labels

Every major recommendation carries one of three labels:

**[Consensus]** — The industry has largely converged on this. Strong evidence, broad agreement among experienced practitioners. Stating disagreement requires a high bar.

**[Strong Recommendation]** — The evidence favors this strongly, but reasonable engineers have made the other call successfully under specific constraints.

**[Legitimate Trade-off]** — Experienced engineers genuinely disagree because different constraints dominate. Present both sides honestly.

---

## "Why Smart Engineers Disagree" Sections

For topics where expert disagreement is real and not just inexperience, include an optional section:

```
### Why Smart Engineers Disagree on [Topic]

[2–4 paragraphs explaining the structural reasons for disagreement —
what constraints or values lead competent engineers to different conclusions]
```

---

## Citing Real Systems

Prefer real systems over hypothetical examples. When a real system illustrates a principle, reference it explicitly:

- Git — content-addressable storage, immutable history
- PostgreSQL — MVCC, WAL, query planning
- SQLite — deliberate simplicity, single-file design
- Linux / Unix — file descriptor abstraction, everything-is-a-file
- Redis — single-threaded execution model, data structure server
- Kubernetes — declarative reconciliation, desired-state model
- Kafka — log-structured append-only storage, consumer group model

---

## Cross-References

Link between chapters using relative markdown paths:

```
See [Coupling and Cohesion](../part01-systems-thinking/ch03-coupling-and-cohesion.md) for the formal definitions used here.
```

---

## What Not to Write

- Do not restate what the section heading already says.
- Do not open with "In this chapter, we will cover…"
- Do not close with "In the next chapter, we will…"
- Do not hedge concrete recommendations into meaninglessness.
- Do not use "simple", "easy", "obvious", or "just" — these words are wrong when the reader is struggling and condescending when they are not.
