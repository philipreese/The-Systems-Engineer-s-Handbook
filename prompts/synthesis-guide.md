# Synthesis Guide — Editorial Instructions for Claude

This document tells you (Claude) exactly what to do when you receive a pair of raw chapter
drafts from ChatGPT and Gemini. Read this before touching any draft files.

---

## Your Role

You are the editor, not an author. Your job is to produce a final chapter that is better
than either raw draft by taking the best of each — the strongest framing, the most concrete
examples, the sharpest failure modes — while cutting anything that belongs in a different
chapter. You are not merging the two drafts. You are making editorial judgments.

---

## Before You Start

Read these files before synthesizing any chapter:

1. `00-style-guide.md` — the chapter template, recommendation labels, and writing rules
2. `01-glossary.md` — terms already defined; do not redefine them
3. `02-table-of-contents.md` — confirm which chapter number this is and its position
4. `03-design-principles.md` — core axioms this handbook does not contradict. The principles emerge from the content, not the other way around.
5. `prompts/part03-chapter-specs.md` (or the relevant part's spec file) — the "Do NOT cover"
   list is the most important thing you will read. It tells you what to cut from both drafts.

---

## Finding the Raw Drafts

Raw drafts are committed to:
- `raw/chatgpt/` — ChatGPT's draft
- `raw/gemini/` — Gemini's draft

File names may not be consistent (e.g., `Ch01.md`, `01.md`, `ch01-title.md`). Check both
directories for the most recently added files if the chapter number is not obvious.

---

## How to Evaluate Each Draft

Read both drafts completely before writing anything. As you read, note:

**What to take:**
- The framing or central argument that is cleaner or more accurate
- Examples that reference real systems specifically (not hypothetical "imagine a service...")
- Failure modes that are concrete and observable (not "things may go wrong")
- Sections that cover territory uniquely — if only one model covered something well, take it
- The better "Why Smart Engineers Disagree" section if both have one

**What to cut:**
- Anything on the "Do NOT cover" list from the chapter spec
- Generic advice that hedges into uselessness ("it depends on your use case")
- Hypothetical examples when a real system example exists or could be substituted
- Motivational or filler language ("this is crucial for building robust systems")
- Meta-commentary ("in this section we will explore...")
- Repeated content — if both models covered the same point, pick the better version, do not
  include both

**What to fix:**
- Apply the chapter template format consistently (see `00-style-guide.md`)
- Add Consensus / Strong Recommendation / Legitimate Trade-off labels to major recommendations
- Ensure the chapter opens with Prerequisites, New vocabulary introduced, and Key takeaways
- Ensure cross-references point to the correct chapter numbers and file paths

---

## Synthesis Rules

1. **Do not concatenate.** A chapter that is the ChatGPT draft followed by the Gemini draft
   is not a synthesis. Cut ruthlessly. The final chapter should be shorter than either raw
   draft if both models were verbose.

2. **Take the best framing, not the most content.** If ChatGPT's central argument is
   stronger, use it as the spine even if you're taking most of the examples from Gemini.

3. **Real systems beat hypotheticals.** If one model used a real system (PostgreSQL, Git,
   Redis, Linux, Kubernetes) and the other used a hypothetical, use the real one. If neither
   did, substitute a real example.

4. **The "Do NOT cover" list is a hard constraint.** If a section in a raw draft covers
   territory explicitly marked as belonging to another chapter, cut it entirely. A one-line
   flag ("this is expanded in Ch XX") is sufficient.

5. **Failure modes must be concrete.** "The system may become slow" is not a failure mode.
   "PostgreSQL query plans optimized around outdated schema assumptions cause full table scans
   that appear suddenly after a data distribution shift" is a failure mode.

6. **Take positions.** If both models hedged, and the industry has actually converged on an
   answer, say so and label it [Consensus] or [Strong Recommendation]. The handbook is
   opinionated where evidence is strong.

---

## Output: What to Produce

### 1. The final chapter file

Location: `part[N]-[part-slug]/ch[NN]-[chapter-slug].md`

Examples:
- `part1-systems-thinking/ch02-complexity-is-the-enemy.md`
- `part2-software-architecture/ch10-monolith-vs-service-decomposition.md`

Use lowercase kebab-case for the filename. Match the slug to the chapter title.

### 2. Update the Table of Contents

In `02-table-of-contents.md`, change the chapter's status from `[Stub]` or `[Draft]` to
`[Complete]`.

### 3. Update the Part README

In the relevant `partN-[slug]/README.md`, change the chapter's status to `[Complete]`.

### 4. Update the Glossary

In `01-glossary.md`, add an entry for every new term introduced in this chapter that will
be used by later chapters. Format:

```
**Term**: Definition in one to three sentences.
First introduced in: [Part I, Ch NN](path/to/chapter.md).
```

Only add terms that are genuinely new and cross-chapter. Do not add terms that are
self-explanatory or only used within this chapter.

### 5. Update the Design Principles

Only add new design principles that were genuinely discovered during the creation of the chapters. 
Ensure you follow the guidelines at the top of `03-design-principles.md`.

---

## Commit Message Format

The commit message should document the editorial decisions so the history is traceable:

```
Add synthesized Ch [NN]: [Chapter Title]

Synthesized from ChatGPT and Gemini raw drafts. [One sentence on ChatGPT's
main contribution.] [One sentence on Gemini's main contribution.]

Cut from both drafts: [list what was removed and which chapter it belongs to].

Also updates ToC, Part [N] README ([Chapter Title] → [Complete]), and populates
the glossary with [N] terms introduced in this chapter.
```

---

## Checklist Before Committing

- [ ] Final chapter file exists at the correct path
- [ ] Chapter opens with Prerequisites, New vocabulary, Key takeaways
- [ ] Every major decision/principle follows the template from `00-style-guide.md`
- [ ] All Consensus / Strong Recommendation / Legitimate Trade-off labels are applied
- [ ] Nothing on the chapter spec's "Do NOT cover" list survived into the final chapter
- [ ] Cross-references use correct chapter numbers and relative file paths
- [ ] `02-table-of-contents.md` shows `[Complete]` for this chapter
- [ ] Part README shows `[Complete]` for this chapter
- [ ] New glossary terms added to `01-glossary.md`
- [ ] Commit message documents what came from each model and what was cut
