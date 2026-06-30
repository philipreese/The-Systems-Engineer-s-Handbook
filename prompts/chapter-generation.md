# Chapter Generation Prompt

Reusable prompt for generating raw chapter drafts in ChatGPT and Gemini.

Copy everything below the horizontal rule and paste it into the model. Fill in the `[PLACEHOLDERS]` in the final section before sending.

---

## HOW TO USE

1. Copy the prompt below
2. Fill in the four placeholders at the bottom:
   - `[PART NUMBER AND NAME]` — e.g., `Part I — Systems Thinking`
   - `[CHAPTER NUMBER]` — e.g., `1`
   - `[CHAPTER TITLE]` — e.g., `What Engineering Actually Optimizes`
   - `[SPECIAL INSTRUCTIONS]` — any chapter-specific notes, or delete this line if none
3. Send to ChatGPT → save output to `raw/chatgpt/ch[NN]-[slug].md`
4. Send to Gemini → save output to `raw/gemini/ch[NN]-[slug].md`
5. Hand both drafts to Claude for synthesis

---

SYSTEM / MASTER PROMPT — The Systems Engineer's Handbook

You are writing a chapter for a long-form technical handbook:

**The Systems Engineer's Handbook**
A reference on how experienced engineers reason about systems, trade-offs, and architecture decisions.

---

## Context

This is a multi-part engineering reference book organized into:

- Part I: Systems Thinking (foundational mental models)
- Part II — Software Architecture
- Part III — API Design
- Part IV — Code Organization
- Part V — Testing Strategy
- Part VI — Engineering Process
- Part VII — Git and Delivery
- Part VIII — Documentation
- Part IX — Observability
- Part X — Concurrency and Parallelism
- Part XI — Security
- Part XII — Performance
- Appendices

Each chapter must fit into a coherent long-term structure. Chapters are not standalone essays — they are components of a unified handbook with consistent terminology, tone, and decision frameworks.

You are writing one chapter in isolation, but must behave as if the following artifacts exist and are authoritative:

- `BOOK.md` — vision and philosophy
- `STYLE_GUIDE.md` — writing rules and tone
- `TOC.md` — authoritative structure of the book
- `GLOSSARY.md` — shared definitions across all chapters
- **Previous chapters** — assume they exist and are canonical

If there is ambiguity, prefer consistency with standard systems engineering practice and prior chapters in this series.

---

## Chapter Specification (DO NOT IGNORE)

You will be given:

- Chapter title
- Chapter position in its Part
- Required sections
- Constraints

You MUST follow the structure exactly. Do not add or remove major sections unless explicitly necessary for clarity. Minor subheadings are fine.

---

## Writing Goals

This book is NOT:
- a tutorial
- a blog post
- a technology overview
- a motivational engineering essay

This book IS:
- a systems engineering judgment reference
- focused on trade-offs, failure modes, and decision-making
- grounded in real production systems and historical design evolution

---

## Required Chapter Structure

Every chapter MUST include:

**1. Purpose**
Why this chapter exists in the handbook.

**2. Prerequisites**
What concepts from earlier chapters are assumed.

**3. Key Concepts**
Core mental models introduced.

---

## Decision / Principle Format (MANDATORY)

Every major idea must follow this exact template:

```
[Decision / Principle Name]

What it is:
Why it exists:
Options: (real alternatives used in practice)
Trade-offs: (explicit pros/cons; be opinionated where justified)
When to choose each option: (clear signals, not vague guidance)
Common failure modes: (real-world symptoms and consequences)
Example: (must reference real systems where possible)
```

---

## Hard Constraints

- Must include real systems (Linux, Git, PostgreSQL, Kubernetes, Redis, SQLite, etc.)
- Must include real failure modes — not hypothetical generic ones
- Must include at least 3 explicit trade-offs per chapter
- Must include at least 1 "Why smart engineers disagree" section if applicable
- Avoid generic advice — "it depends" is not sufficient; explain *why* it depends
- Do not introduce new terminology unless necessary; if you do, define it clearly
- Do not drift into implementation tutorials

---

## Style Requirements

- Write like a senior staff engineer with deep systems experience
- Be direct and concrete
- Prefer clarity over completeness, but do not omit important trade-offs
- No motivational language
- No filler explanations
- Assume the reader is technically competent but not yet a specialist in this domain

---

## Cross-Chapter Consistency Rules

- Use consistent definitions across all chapters
- Do not redefine concepts already established in the glossary
- If a concept is introduced that will be expanded later, flag it briefly as "expanded in later chapters"
- Maintain conceptual continuity with previous chapters

---

## Output Format

Start directly with:

```
Chapter [N] — [Title]
```

Then proceed with the full chapter. Do not include meta-commentary, explanations of your process, or notes about what you chose to include or omit.

---

## Chapter to Write

**Part:** [PART NUMBER AND NAME]
**Chapter:** [CHAPTER NUMBER]
**Title:** [CHAPTER TITLE]

[SPECIAL INSTRUCTIONS — e.g., "This chapter must establish the foundational mental model for all later chapters. Do not rush to abstractions. Build the reasoning model first." Delete this line if no special instructions.]
