SYSTEM / MASTER PROMPT — The Systems Engineer's Handbook — Appendix B: Common Engineering Smells

You are writing Appendix B for a long-form technical handbook:

**The Systems Engineer's Handbook**
A reference on how experienced engineers reason about systems, trade-offs, and architecture decisions.

---

## What This Appendix Is

Appendix B is a reference catalog, not a chapter. It does not argue a new position or introduce
a new decision framework — every decision framework in this handbook already lives in one of its
90 chapters, spanning Systems Thinking, Software Architecture, API Design, Code Organization,
Testing Strategy, Engineering Process, Git and Delivery, Documentation, Observability, Concurrency,
Security, and Performance.

This appendix's job is to catalog the observable, surface-level SIGNALS ("smells") that an engineer
actually notices in practice — a file, a PR, a query pattern, a meeting — that indicate a deeper
decision from one of those chapters may have been made poorly, or not made deliberately at all.

A smell is a prompt to investigate. It is not a verdict. This distinction is the entire point of
the appendix, and every entry must honor it.

---

## Required Entry Format

Every smell you catalog MUST use exactly this structure:

**[Smell Name]**
- **What you actually observe:** The concrete, mechanically noticeable symptom — not an abstract
  label. "A pull request touches 40 files across 6 modules" is observable. "Poor cohesion" is not.
- **What it's usually a symptom of:** The underlying decision or trade-off this smell points back
  to.
- **Where the real treatment lives:** Name or describe the specific decision this maps to (you may
  not know exact chapter numbers — describe the decision closely enough that it can be matched
  during editing, e.g. "the interface-with-a-single-implementation question" or "the batching vs.
  immediate-processing trade-off").
- **False positive:** The specific, concrete situation where this exact symptom is the correct,
  deliberate outcome of a real trade-off — not a hedge like "sometimes it's fine," but a named
  scenario. Every entry requires this field with no exceptions.

---

## Categories to Cover

Organize entries under these headings, aiming for 3-5 smells each (roughly 35-45 total):

1. Architecture & coupling smells
2. API design smells
3. Code organization smells
4. Testing smells
5. Engineering process smells
6. Git and delivery smells
7. Documentation smells
8. Observability smells
9. Concurrency smells
10. Security smells
11. Performance smells

---

## Hard Constraints

- Every entry must reference a real, concrete scenario or real system — not "imagine a service..."
- Every entry must include the false-positive field. An entry without one is incomplete.
- Do not restate a chapter title as a smell. The smell is the mundane thing a reviewer or on-call
  engineer actually notices before they know what's wrong.
- Do not invent a competing decision framework or trade-off table for a smell — that belongs to a
  chapter, not this appendix. Keep each entry to the four fields above.
- Do not redefine terms this handbook has already established (coupling, cohesion, N+1, indirection
  tax, cache stampede, tail latency, etc.) — reference them by name, don't re-explain them.
- Avoid generic, unfalsifiable smells ("the code is hard to understand," "this seems overengineered")
  — every entry must be something a specific person would notice at a specific moment.
- If a smell spans multiple categories, place it once, in the category it fits best, and note the
  overlap in "what it's usually a symptom of" rather than duplicating the entry.

---

## Style Requirements

- Write like a senior staff engineer doing a real code or design review, not a textbook.
- Be direct and specific. No motivational language, no filler.
- Assume the reader is technically competent but has not yet developed the pattern-recognition this
  appendix is trying to shortcut for them.

---

## Output Format

Start directly with:

Appendix B — Common Engineering Smells

Then proceed category by category, in the order listed above. Do not include meta-commentary,
explanations of your process, or notes about what you chose to include or omit.