# Ch 51 — Commit Message Conventions

**Prerequisites:** [Comments: What to Comment, What Not To](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md), [Issue as Tracking Unit vs. PR as Review Unit](../part06-engineering-process/ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md), [Architecture Decision Records (ADRs)](../part06-engineering-process/ch45-architecture-decision-records.md), [Branching Strategies](ch50-branching-strategies.md)

**New vocabulary introduced:** None — this chapter applies comment-rot (Ch 30) and ADR (Ch 45) concepts already defined, rather than coining new terms.

**Key takeaways:**
- Code gets refactored. Once a commit is merged, its message is effectively permanent — the reasoning survives long after the exact lines it describes have been rewritten or deleted.
- [Strong Recommendation] Structured, human-readable commit messages — Chris Beams' seven rules: separated subject/body, imperative mood, body explains *why* not *what* — are the baseline for any codebase expected to outlive its author or gain a second contributor.
- [Strong Recommendation] Conventional Commits' machine-parseable prefixes (`feat:`, `fix:`, `chore:`) are the direct, hard prerequisite for the automated version-bumping and changelog generation covered in Ch 61 — this is infrastructure a release pipeline depends on, not stylistic preference.
- A commit message documents one bounded change; an ADR documents a decision that outlives any single commit. A commit that reaches for architectural justification should reference the ADR by ID, not restate its reasoning — reasoning trapped only in a commit body disappears the moment the file it was attached to is deleted or rewritten.
- Commit discipline matters independent of whether history is later squashed at merge (Ch 52): a disciplined history is what makes preserving it worthwhile, and squashing a disciplined history still produces one good message, while squashing an undisciplined one only hides the mess instead of fixing it.

---

This chapter treats the commit message as a structured artifact with real downstream consumers — `git blame`, `git bisect`, and automated release tooling — not as free-form developer prose. It covers content discipline and machine-parseable format; it does not cover branch topology (already covered in [Ch 50](ch50-branching-strategies.md)), whether history is squashed or preserved at merge time ([Ch 52](ch52-squash-vs-preserve-history.md) — commit discipline matters regardless of that decision), or linking a change to the issue that motivated it (already covered in [Ch 43](../part06-engineering-process/ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md)).

A commit message survives its own diff. The code it describes will be refactored, moved, or partially undone; the message explaining why that code existed in the first place does not change unless someone deliberately rewrites history. That asymmetry — transient diff, durable explanation — is the entire reason commit discipline is worth enforcing rather than leaving to individual taste.

### Content Discipline: Chris Beams' Seven Rules

**What it is:** A widely adopted set of formatting and content rules ("How to Write a Git Commit Message," Chris Beams) that standardizes the human-readable structure of a commit: separate the subject from the body with a blank line, limit the subject to about 50 characters, capitalize it, don't end it with a period, write it in the imperative mood ("Fix," not "Fixed" or "Fixes"), wrap the body at 72 characters, and use the body to explain *what* and *why* — not *how*, since the diff already shows how.

**Why it exists:** A commit message that restates the diff is redundant with `git show`; a commit message that explains the constraint or incident behind the change survives even after the surrounding code is refactored beyond recognition. This is the same comment-rot argument from Ch 30, applied one level up: an inline comment can drift out of sync with the code next to it, but a merged commit message is fixed at the exact state of the world it described, immune to the rot that affects comments living inside evolving files.

**Options:**
1. **Free-form messages** — whatever the author felt like typing, no required structure.
2. **Beams' structured format** — subject/body separation, imperative mood, rationale in the body.

**Trade-offs:** Free-form messages cost nothing to write and are fine for work nobody will ever need to trace back through. Structured messages cost a small amount of discipline per commit but remain interpretable years later, independent of whatever happens to the code around them.

**When to choose each:** Use the structured format for anything merged into a shared, long-lived branch. Free-form is defensible only for local, unpushed, throwaway work that will never be merged.

**Common failure modes:** The redundant diff summary — a message like "fixed bug" or "added if statement to check for null" that adds no information `git show` doesn't already provide, and says nothing about why the bug existed or what upstream condition produced it. The opposite failure is equally damaging: a commit that omits rationale entirely, so a later engineer, seeing only the diff, "fixes" the same problem back into existence because the constraint that justified the original change was never recorded anywhere.

**Example:** The Linux kernel enforces this discipline at massive scale: subject lines are prefixed by subsystem (`kernel/sched:`, `drivers/net:`), bodies frequently document hardware errata or memory-ordering constraints that justify the exact change, and every commit carries a `Signed-off-by` line establishing provenance. This isn't stylistic — maintainers navigate a history with hundreds of contributors using `git bisect`, and that only works because the messages actually explain what each commit was for.

### Conventional Commits: A Machine-Parseable Format

**What it is:** A specification that prefixes a commit subject with a semantic token — `feat:`, `fix:`, `chore:`, `refactor:`, or one of these with a `!` or `BREAKING CHANGE:` footer to flag an incompatible change — following the grammar `<type>[optional scope]: <description>`.

**Why it exists:** Beams' rules make a commit readable to a human; Conventional Commits makes a commit readable to a machine. A tool can parse a stream of `feat:`/`fix:`/breaking-change tokens and derive a semantically correct version bump and changelog entry without a human deciding it manually — the mechanical chain that Ch 61's release automation is built directly on top of.

**Trade-offs:**

| Dimension | Prose Commit Logs | Conventional Commits |
|---|---|---|
| Automation potential | None — release notes and version bumps require manual curation | Full — drives automated versioning tools natively |
| Expressive freedom | High — no forced categorization | Constrained — every change must fit a predefined type |
| Enforcement | Manual, via review | Programmatic, via a commit-lint hook or CI gate |

[Strong Recommendation] Adopt Conventional Commits once a team has, or wants, an automated release pipeline — at that point it's infrastructure the pipeline depends on, not a style preference. A team with no automated release process gains little from the constraint and can reasonably stay with prose, provided Beams' content rules still apply.

**Common failure modes:** The mislabeled breaking change. An engineer drops a column from a shared schema and commits it as `refactor: clean up user schema fields` instead of `feat!: remove deprecated user schema fields` or attaching a `BREAKING CHANGE:` footer. An automated release tool parses `refactor:` as a no-op for versioning purposes, ships a patch-level bump, and a change that should have required a major version and a migration window instead rolls out silently and breaks every consumer still reading the dropped column. The failure isn't the tool — it's a commit type chosen carelessly, which the tool has no way to catch.

**Example:** In a pipeline built on Conventional Commits, `fix(auth): catch token expiration exception` triggers a patch bump (`v2.4.1` → `v2.4.2`); `feat(billing): support Adyen gateway integration` triggers a minor bump (`v2.4.1` → `v2.5.0`); a commit carrying a breaking-change marker triggers a major bump (`v2.4.1` → `v3.0.0`). Tools like semantic-release scan history since the last tag, compute the version, and publish a changelog — entirely mechanically, entirely dependent on the type prefixes being accurate.

### Commit Messages vs. Architecture Decision Records

**What it is:** The boundary between what a commit message documents (a single, bounded change, tied to a diff) and what an ADR documents (a durable architectural decision, expected to outlive any individual commit or file it originally touched).

**Why it exists:** Some decisions matter beyond the lifetime of the code that first implemented them. A commit message explaining a major architectural pivot is only discoverable for as long as `git blame` can trace a line back to it — the moment the file is deleted or substantially rewritten, that explanation is effectively gone. An ADR, stored as its own artifact, survives independent of any specific file's history.

**Options:**
1. **Commit-only** — architectural reasoning lives entirely inside commit bodies.
2. **ADR-only** — architectural reasoning lives entirely in the ADR registry, commits stay purely tactical.
3. **Commit references ADR** — the commit stays short and tactical, and points to the ADR by ID when the change implements or touches a recorded decision.

**Trade-offs:** Commit-only reasoning is convenient to write in the moment but evaporates under refactors, file deletions, and squash merges. ADR-only reasoning is durable but, without a pointer from the commit, a reader following `git blame` to a specific line has no way to discover it exists. Referencing the ADR from the commit costs one extra line and gets both properties: a short, targeted commit message plus a durable trail to the full reasoning.

**When to choose each:** Use commit-only for genuinely local, bounded changes with no architectural weight. Reference an ADR from the commit whenever the change implements, revises, or depends on a decision that already has (or should have) one — small systems with no formal ADR practice can reasonably skip this, but any team already keeping ADRs (Ch 45) should default to cross-referencing rather than duplicating.

**Common failure modes:** The evaporated architecture context — a team migrates a service from synchronous transactions to an event-driven model and the only explanation of *why* lives in a 300-line commit body on the initial scaffolding commit. Two years later that scaffolding is deleted and replaced, the explanation disappears with it, and new engineers reverse-engineer speculative theories about a decision that was actually well-reasoned at the time, only undiscoverable.

**Example:** An ADR at `docs/adr/0034-multi-region-replication.md` records the trade-offs behind adopting multi-region replication. The commit that implements it stays tactical: `fix(db): handle replica synchronization timeouts (ref: ADR-0034)`. If the database driver is replaced next year, that specific fix commit fades into ordinary history, but the durable justification for replication remains fully discoverable in the ADR registry, independent of which commit or file first implemented it.

---

### Why Smart Engineers Disagree

The real disagreement is about where engineering intent should actually live: the local, terminal-native Git log, or the code-hosting platform's pull request interface.

Engineers who center the PR argue that commit-level discipline is a holdover from email-driven, pre-platform open-source workflows. The PR description is where review actually happens — it supports rich text, threaded comments, and links to the issue that motivated the change (Ch 43) — and if the branch is going to be squash-merged into one commit anyway (Ch 52), polishing every intermediate commit message is effort spent on text that won't survive to `main`.

Engineers who center the local Git log point out that a PR description is metadata owned by an external vendor's database, not the repository itself — if the organization migrates hosting providers, or the platform has an outage, that record is gone or unreachable, while the Git history travels with every clone. They also note that `git bisect` checks out commits in a detached-HEAD state with no platform context attached at all: an on-call engineer bisecting a regression at 3 a.m. is reading commit messages, not pull request threads, and a commit history that outsourced its reasoning to the platform gives that engineer nothing.

Both positions are correct about the tool they're optimizing for. A team that reliably squash-merges and treats the PR description as the permanent record can get away with lighter intermediate-commit discipline, provided the final squashed message meets Beams' bar. A team that preserves full history (Ch 52) or relies on `git bisect` across long-lived history has no substitute for disciplined commits, because by the time anyone needs that history, the platform context bisect actually runs against is the Git log and nothing else.
