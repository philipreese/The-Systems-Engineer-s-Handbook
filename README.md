# The Systems Engineer's Handbook

A comprehensive engineering reference covering the decisions senior engineers face, why those decisions exist, and the trade-offs between realistic alternatives. This is not a tutorial or a project-specific document — it is a career-long reference organized so you can look up any decision and understand the full landscape before choosing.

---

## How This Book Is Built

Each chapter is produced through a three-model editorial process:

1. **ChatGPT** writes a draft → committed to `raw/chatgpt/`
2. **Gemini** writes a draft → committed to `raw/gemini/`
3. **Claude** synthesizes the best of both → committed to the final chapter file in the appropriate part folder

Raw drafts are preserved so the editorial reasoning can be traced and revisited.

---

## Structure

| File | Purpose |
|------|---------|
| `00-style-guide.md` | Chapter template, labeling conventions, writing rules |
| `01-glossary.md` | Authoritative term definitions (grows with each chapter) |
| `02-table-of-contents.md` | Full outline with chapter status |
| `03-design-principles.md` | Core axioms the handbook does not contradict |

| Folder | Contents |
|--------|---------|
| `raw/chatgpt/` | Raw ChatGPT drafts, one file per chapter |
| `raw/gemini/` | Raw Gemini drafts, one file per chapter |
| `part1-systems-thinking/` | Final synthesized chapters |
| `part2-software-architecture/` | Final synthesized chapters |
| `part3-api-design/` | … |
| `part4-code-organization/` | … |
| `part5-testing-strategy/` | … |
| `part6-engineering-process/` | … |
| `part7-git-and-delivery/` | … |
| `part8-documentation/` | … |
| `part9-observability/` | … |
| `part10-concurrency/` | … |
| `part11-security/` | … |
| `part12-performance/` | … |
| `appendices/` | Decision frameworks, smells catalog, patterns, glossary |

---

## Chapter Status Key

| Status | Meaning |
|--------|---------|
| `[Stub]` | Chapter listed, no drafts yet |
| `[Draft]` | Raw drafts exist, synthesis pending |
| `[Complete]` | Final synthesized chapter committed |

See `02-table-of-contents.md` for the current status of every chapter.
