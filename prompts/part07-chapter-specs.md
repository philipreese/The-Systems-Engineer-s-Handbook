# Part VII — Chapter Specifications

Pre-filled special instructions for each Part VII chapter. When generating a draft, copy the
block for the chapter you're writing into the `[SPECIAL INSTRUCTIONS]` field of the
chapter-generation prompt.

The instructions do three things:
1. State the chapter's specific role in the handbook's argument
2. Name the key sources, systems, or concepts to anchor the content
3. Explicitly mark what NOT to cover — because those topics belong to another chapter

Prerequisites for Part VII: Part I (Ch 09's reversibility-and-blast-radius quadrant applies
directly to force push and history rewriting; Ch 07's fail-fast principle is reused, not
redefined, in Ch 58). Part VI (Ch 42's issue model — branch names and commit messages
often reference an issue number, referenced not repeated; Ch 43 and Ch 47 both explicitly
deferred "the mechanics of branches, merging, and Git workflows" and "branching, merge
mechanics, commit conventions, and CI policy" to this Part; Ch 48's technical debt framing
recurs as "dependency debt" in Ch 63; Ch 49's process-overhead audit — does this check earn
its cost — applies to every gate this Part adds to a pipeline, and chapters should assume
that lens rather than re-litigate it).

Several forward references from earlier parts must be honored here:
- Part I, Ch 05 and Ch 09, and Part II, Ch 16, all deferred "branching strategies" to Ch 50
  and, in Ch 16's case, "release strategy" to Ch 56.
- Part VI, Ch 43 deferred "the mechanics of branches, merging, and Git workflows" here.
- Part VI, Ch 47 deferred "branching, merge mechanics, commit conventions, and CI policy"
  here, and specifically named pre-merge CI checks as Ch 57's job.
- Part VI, Ch 44 deferred "release mechanics, tagging, and deployment" to Ch 56, Ch 61, and
  Ch 62 by name.

Boundary with adjacent Parts: Part VI owns the *why* of engineering process — issues, the
issue-to-PR mapping, review standards, specs. This Part owns the mechanical *how* of version
control and delivery built on top of that process. Part IX owns observability; Ch 62
(environment promotion) references the metrics and error budgets that gate a rollout without
defining them — that belongs to Ch 73. Part XI owns security; Ch 56, Ch 61, and Ch 63 touch
signing, publishing credentials, and dependency versions without owning the threat model —
secrets handling belongs to Ch 83 and supply-chain risk belongs to Ch 84. Chapters in this
Part that brush against those topics should flag the boundary with a one-line pointer, not
expand into it.

This Part splits naturally into two halves: Git mechanics (Ch 50–56) and the CI/CD delivery
pipeline (Ch 57–63). Ch 50 opens the first half and Ch 63 closes the second.

---

## Ch 50 — Branching Strategies

```
This chapter opens Part VII by establishing the fundamental topology choice every other
Git-mechanics chapter in this Part builds on: how branches are structured to move code from
first commit to production. It resolves the forward reference planted in Part I (Ch 05,
Ch 09) and Part II (Ch 16) — do not re-derive why branching strategy matters to those
chapters' arguments; this chapter owns the mechanics.

Cover the two dominant models honestly: trunk-based development (frequent, small merges
directly to a single long-lived main branch, short-lived feature branches measured in hours
or a few days) versus GitFlow-style long-lived branch hierarchies (develop, release, hotfix,
and feature branches with a more elaborate merge topology, originally formalized for
scheduled, versioned releases). Take a position, backed by the DORA/Accelerate research
(Forsgren, Humble, Kim): trunk-based development with short-lived branches correlates with
elite deployment frequency and lead time, and should be the default for continuously
deployed software. GitFlow's heavier branch hierarchy is a legitimate choice, not a legacy
mistake, for software with a genuinely different release cadence — desktop applications,
embedded firmware, or libraries supporting multiple concurrently maintained major versions —
where "always releasable from main" isn't the actual delivery model.

Cover the core trade-off precisely: short-lived branches minimize merge conflicts and
integration risk (the same argument [Ch 43](Part VI) already made about PR size applies at
the branch-lifetime level — reference it, don't re-derive it) but require supporting
infrastructure (feature flags, from Ch 43) to land incomplete work safely. Long-lived
branches allow work to proceed in more isolation but defer integration risk, echoing the
big-bang-integration failure mode from Ch 44 — a branch that lives for months is a
horizontal-phase plan expressed in Git.

Anchor in real systems: Google and Meta's internal monorepo, trunk-based-development
practice at extreme scale; Vincent Driessen's original 2010 "A successful Git branching
model" post as GitFlow's canonical source, alongside his own later acknowledgment that it
doesn't fit continuous delivery; GitHub Flow (Scott Chacon) as the simpler, short-lived-
branch alternative; the DORA State of DevOps research correlating branch lifetime with
delivery performance.

Do NOT cover: commit message format (Ch 51); whether merged history is squashed or preserved
(Ch 52); branch naming conventions and deletion lifecycle (Ch 53); force push judgment
(Ch 54); merge vs. rebase mechanics for integrating a branch (Ch 55); tagging and release
markers (Ch 56). This chapter is about branch topology only.
```

---

## Ch 51 — Commit Message Conventions

```
This chapter treats the commit as the atomic, permanent unit of a codebase's history — the
thing `git blame` and `git bisect` actually operate on — and argues that its message is not
free-form prose but a structured artifact with a real, mechanical downstream consumer.

Cover the content discipline first: Chris Beams' widely cited seven rules ("How to Write a
Git Commit Message") — separate subject from body, limit the subject line, use the
imperative mood ("Fix," not "Fixed" or "Fixes"), explain why in the body, not just what the
diff already shows. Connect this explicitly to Part IV, Ch 30's comment-rot argument: a
commit message that restates the diff is redundant with `git show`; a commit message that
explains the constraint or reasoning behind the change survives even when the code around
it is later refactored.

Cover the structured-format option: Conventional Commits (`feat:`, `fix:`, `chore:`, etc.)
as a machine-parseable convention, and take a position on why it matters mechanically, not
just stylistically — it's the direct prerequisite for the automated version-bumping and
changelog generation covered in Ch 61. A team that doesn't follow a structured convention
cannot safely automate release notes; commit discipline here is infrastructure for a later
chapter, not merely hygiene.

Cover the distinction between a commit message and an ADR ([Ch 45](Part VI)) explicitly: a
commit message explains this specific, bounded change; an ADR explains a durable
architectural decision that outlives any single commit. Don't let the two blur.

Anchor in real systems: the Linux kernel's commit discipline (Signed-off-by, structured
subject prefixes by subsystem) as the high-discipline, high-scale canonical example;
Conventional Commits as the structured, tooling-oriented convention; semantic-release and
similar tools (forward reference to Ch 61) as the concrete payoff for following either
discipline consistently.

Do NOT cover: branching topology (Ch 50); whether commits are squashed at merge time
(Ch 52 — commit discipline matters regardless of the squash decision, though squashing
changes how much of an individual commit's message survives); linking a PR to its
motivating issue (already covered in Part VI, Ch 43 — reference it).
```

---

## Ch 52 — Squash vs. Preserve History

```
This chapter is about what a merged pull request's history looks like afterward: one clean,
synthetic commit per PR (squash), or the full sequence of commits made during development,
preserved as-is or cleaned up via rebase before merging.

Cover the trade-off precisely, and connect it directly to Ch 51: squash-and-merge is the
pragmatic default for teams whose in-review commit history is genuinely noisy — "fix typo,"
"address review feedback," "oops" — because it produces one commit with one clean message
regardless of how messy the process behind it was. Preserving full history is superior only
when the commits being preserved were actually curated to each be independently meaningful,
which requires the Ch 51 discipline to already be in practice; preserving a noisy history
is strictly worse than squashing it. This is a genuine [Legitimate Trade-off], but it is
gated on an empirical fact about the team's actual commit hygiene, not a matter of taste.

Cover the bisectability cost precisely: `git bisect` is only as useful as the granularity of
history it has to search. Squashing collapses a PR's internal commits into one, which means
`git bisect` can identify which PR introduced a regression but not which specific step
within it did — a real cost for large PRs, and another reason (alongside Ch 43's argument)
to keep PRs themselves small.

Anchor in real systems: GitHub's "Squash and merge" button as the mainstream, deliberately
low-friction default; the Git project's and the Linux kernel's own practice of preserving a
curated, rebased sequence of atomic, individually-reviewed patches rather than squashing —
the canonical example of history preservation done well, precisely because their commit
discipline (Ch 51) supports it.

Do NOT cover: commit message format itself (Ch 51); the rebase operation used to clean up
history before merging (Ch 55 — this chapter is about the merge-time policy decision, not
the mechanics of rebase); force-pushing a branch to apply that cleanup (Ch 54).
```

---

## Ch 53 — Branch Naming and Lifecycle

```
This chapter covers the mechanical conventions for naming a branch and, more importantly,
the discipline of treating branches as disposable rather than as things anyone needs to
remember.

Cover naming conventions briefly and pragmatically: type prefixes (`feature/`, `fix/`,
`chore/`) and issue-number embedding (referencing [Ch 42](Part VI)'s issue as the unit of
intent, without repeating that chapter's argument) as the common, low-value-add-but-
consistent pattern. Don't overweight this — the naming convention itself is close to
bikeshedding; the lifecycle discipline is where the real argument is.

Cover the lifecycle argument as the chapter's core: branches should be short-lived (echoing
Ch 50's trunk-based recommendation) and deleted immediately on merge, not kept "just in
case." Draw the direct, explicit parallel to Ch 42's write-only tracker: a repository with
hundreds of stale, forgotten branches is the exact same failure mode as an unpruned issue
backlog — a collection that was cheap to add to and has become a graveyard nobody trusts or
navigates confidently. Take a position: automatic branch deletion on merge is the correct
default, mechanically enforced (Principle 8) rather than left to manual cleanup discipline.

Anchor in real systems: GitHub's and GitLab's "automatically delete head branches" merge
setting as the mechanical enforcement; stale-branch bots and scheduled cleanup jobs as the
tooling response for repositories that haven't adopted the setting; the observable failure
mode of a repository's branch list becoming unusable for anyone trying to find active work.

Do NOT cover: the branching strategy or topology itself (Ch 50); issue tracking and what
makes a branch's underlying unit of work well-defined (Part VI, Ch 42 — reference it).
```

---

## Ch 54 — Force Push: When It's Acceptable

```
This is a narrow, specific judgment-call chapter, and should be treated as a direct
application of Ch 09's reversibility-and-blast-radius framework (Part I) to one specific
Git operation — reference that framework, don't re-derive it.

Cover the core rule precisely: force-pushing to a private, personal branch that no one else
has based work on is low blast radius and often expected — cleaning up commits via
interactive rebase before requesting review is a normal, healthy practice. Force-pushing to
a shared branch (main, a release branch, or any branch others have already pulled or built
on) is high blast radius and effectively irreversible for anyone who loses unpushed local
work as a result — a specific, well-documented failure mode: a collaborator's un-pushed
commits are silently discarded with no warning and no easy recovery for them.

Cover the mechanical safety net: `--force-with-lease` versus plain `--force` — lease refuses
the push if the remote branch has moved since your last fetch, which is nearly always what
you actually want, and should be the default recommendation over unqualified `--force` even
for legitimate use on a private branch. Cover branch protection rules (GitHub's "Do not
allow force pushes" setting on protected branches) as the mechanical enforcement (Principle
8) of the shared-branch rule — the safest version of this policy is one a human never has
to remember to follow.

Anchor in real systems: GitHub/GitLab branch protection settings disallowing force push on
protected branches as the concrete mechanical guardrail; the well-documented, recurring
incident pattern of a force push to a shared branch discarding a teammate's work as the
canonical cautionary failure mode.

Do NOT cover: merge vs. rebase as a general integration strategy (Ch 55 — force-pushing is a
mechanical consequence of rebasing your own branch, referenced here, not re-argued); branch
lifecycle and deletion (Ch 53).
```

---

## Ch 55 — Merge vs. Rebase

```
This chapter covers the two mechanisms for integrating one branch's changes into another,
and the different shape of history each produces: a merge commit that preserves the actual
diverging-and-converging shape of development, versus a rebase that replays commits onto a
new base to produce a linear history as if they'd been written sequentially in the first
place.

Cover the "golden rule of rebasing" precisely: never rebase a branch that other people have
already pulled or built work on top of — doing so rewrites commit history that others
depend on and forces every downstream branch into a painful, error-prone reconciliation.
Rebasing your own private, unpublished branch to clean it up before review is both safe and
recommended; rebasing anything already shared is not, and this is the same reversibility
judgment Ch 54 already applies to force push, because a shared rebase requires exactly the
force push Ch 54 warns against.

Cover the genuine, lower-stakes disagreement honestly: whether a team's default integration
of a finished branch into main should produce a merge commit (preserving the branch
structure explicitly) or a linear history (via rebase or fast-forward). Take a position:
this is a [Legitimate Trade-off] that matters far less than either side's advocates tend to
claim, as long as the team is consistent — and note that squash-merge ([Ch 52](this Part))
sidesteps the debate entirely for most PR-based workflows by collapsing the question to "one
commit, no merge-commit noise either way."

Anchor in real systems: Atlassian's Git tutorial articulation of the golden rule of
rebasing; Linus Torvalds' public, on-record defense of merge commits and criticism of
rebase-only, enforced-linear-history workflows as the canonical statement of the pro-merge
position; teams and projects that enforce strict linear history via rebase as the
counter-example.

Do NOT cover: squash-and-merge as a specific GitHub mechanism and when it's preferable
(Ch 52 — reference its conclusion); the force-push mechanics rebasing shared work requires
(Ch 54); branch topology (Ch 50).
```

---

## Ch 56 — Tagging and Release Markers

```
This chapter resolves the forward reference from Part II, Ch 16 ("release strategy for the
code that implements these versions") and from Part VI, Ch 44 ("release mechanics, tagging,
and deployment"). It covers the mechanical Git artifact that marks a specific commit as a
release — not what a version number semantically promises, which Ch 16 already covers and
should be referenced, not re-derived.

Cover annotated versus lightweight tags precisely, and take a position: annotated tags
(carrying a tagger, date, message, and optionally a GPG signature) are the correct default
for anything called a release; a lightweight tag is just a named pointer with none of that
metadata and is appropriate only for throwaway, purely local bookmarks. Cover Semantic
Versioning (Tom Preston-Werner) as the versioning scheme a release tag typically encodes
(`v1.4.2`), explicitly deferring to Ch 16 for what the major/minor/patch distinction actually
promises to consumers — this chapter is about the tag as an artifact, not the version
number's meaning.

Cover signed tags briefly as the integrity mechanism worth naming — a GPG-signed tag lets a
consumer verify a release actually came from an authorized maintainer — and flag, without
expanding on it, that the full supply-chain threat model this defends against belongs to
Part XI, Ch 84.

Anchor in real systems: Git's own distinction between lightweight and annotated tags;
Semantic Versioning's specification and adoption as the near-universal versioning
convention tags encode; GPG-signed release tags as practiced by major open-source projects
(the Git project itself signs its own release tags) as the integrity-conscious example.

Do NOT cover: what a version number promises about compatibility (Part II, Ch 16 —
reference it directly); how a tag gets created as part of an automated release pipeline
(Ch 61); how a tagged artifact then moves through environments (Ch 62).
```

---

## Ch 57 — What Belongs in CI (and What Doesn't)

```
This chapter opens the second half of the Part — the CI/CD delivery pipeline — by
answering a scope question first: what should an automated pipeline actually check, versus
what belongs elsewhere (a local pre-commit hook, human code review, a manual QA pass). This
directly resolves the forward reference from Part VI, Ch 47, which explicitly deferred
"pre-merge CI checks and what belongs in them" here.

Cover the core argument: CI should run everything that is cheap, fast, and mechanically
verifiable — lint, formatting, type-checking, the unit-test layer of the testing pyramid
(reference Part V, Ch 34–35 directly, don't re-derive the pyramid) — because Part VI, Ch 47
already established that anything a machine can check must be automated out of human review
entirely (Principle 8). What doesn't belong in the blocking, on-every-commit pipeline is
whatever is genuinely expensive relative to the value of running it on every single change —
full end-to-end suites, exhaustive security scans — which belong on a slower cadence
(nightly, pre-release) rather than gating every commit.

Cover Martin Fowler's original definition of Continuous Integration (2006) as the canonical
statement of CI's actual promise: fast, trustworthy feedback on every integration. Connect
this explicitly to Part V, Ch 34's *feedback loop latency* concept — a CI pipeline that
takes hours, or is routinely red for reasons unrelated to the change being tested, has
already defeated its own purpose regardless of how much it checks.

Anchor in real systems: Fowler's "Continuous Integration" article as the primary source;
the near-universal industry practice of gating merges on a green CI run as the mechanical
enforcement of code health (Ch 47); nightly or scheduled pipelines as the standard home for
expensive verification that shouldn't block every PR.

Do NOT cover: the testing pyramid and what belongs at each test layer (Part V, Ch 34, Ch 35
already own this — reference their conclusions); the code review standard itself (Part VI,
Ch 47); the order and failure behavior of the checks once you've decided what's in the
pipeline (Ch 58 — that's ordering and failure semantics, this chapter is scope).
```

---

## Ch 58 — Fail-Fast vs. Fail-Safe Pipeline Design

```
Given the checks Ch 57 puts in the pipeline, this chapter covers the order they run in and
what happens when one fails: stop immediately (fail-fast) or run every check to completion
and report everything at once. This is a direct, specific application of the fail-fast
principle already defined in Part I, Ch 07 — reference that definition, do not redefine
fail-fast from scratch; this chapter is about applying it to pipeline architecture
specifically.

Cover the trade-off honestly: a fail-fast pipeline stops on the first red check, saving
compute time and giving the fastest possible signal that something is wrong — but it can
hide a second, unrelated failure behind the first one, costing a developer an extra
round-trip once they fix issue one and discover issue two on the next run. A fail-safe (run-
to-completion) pipeline costs more compute and wall-clock time but gives the developer the
complete picture in a single pass. Take a position: within a single logical stage, fail-fast
is usually right — no reason to keep running unit tests once the build itself doesn't
compile — but across independent, parallel check categories (lint, unit tests, integration
tests), running to completion and aggregating all failures into one report is usually the
better default, since a developer fixing three failures one round-trip at a time is a worse
experience than fixing three failures reported together once.

Cover ordering as the second, related lever regardless of fail-fast/fail-safe choice: cheap
checks (lint, type-check) should gate expensive ones (integration, E2E) so a trivial lint
failure aborts a pipeline before it burns twenty minutes on a suite that was never going to
matter once the lint failure is fixed anyway.

Anchor in real systems: GitHub Actions' literal `fail-fast: true/false` matrix-job setting
as the mechanical, named expression of this exact choice (flag that matrix mechanics
specifically are Ch 60's job, not this chapter's); Part I, Ch 07's fail-fast design
principle as the concept being applied here at the pipeline-architecture level.

Do NOT cover: what checks belong in the pipeline at all (Ch 57); caching strategy (Ch 59);
the mechanics of matrix builds themselves, where `fail-fast` is a literal per-matrix
configuration option (Ch 60 — this chapter covers the general principle, Ch 60 covers the
matrix-specific mechanics).
```

---

## Ch 59 — Caching Strategy in CI

```
This chapter covers how to make a CI pipeline fast without making its results untrustworthy
— the central risk being a cache that serves stale, wrong artifacts and produces a "green"
build that would not actually build clean from scratch.

Cover the primary lever: dependency and build-artifact caching (package manager caches,
Docker/BuildKit layer caching) as the highest-leverage way to cut pipeline time, since
dependency resolution and compilation are typically the most expensive, most repeatable
parts of a build. Cover cache key design as the chapter's central technical argument: a
cache should be keyed on a hash of the lockfile (package-lock.json, Cargo.lock, poetry.lock,
etc.) so it invalidates precisely when dependencies actually change — not on a branch name,
a date, or anything broader, which either serves stale results across unrelated commits or
fails to reuse the cache when it safely could.

Cover the failure mode by name: cache poisoning / stale-cache masking, where an overly broad
or incorrectly scoped cache key serves an artifact from a different dependency state,
producing a pipeline that passes for the wrong reason. Take a position as the mitigation:
maintain a scheduled, cache-bypassing "clean build" job (nightly or pre-release) specifically
to catch drift a warm cache would otherwise hide indefinitely.

Anchor in real systems: GitHub Actions' and GitLab CI's cache-key-from-lockfile-hash pattern
as the standard mechanical implementation; Docker BuildKit's layer caching as the
containerized-build expression of the same principle; documented incidents of CI passing on
a stale cache while a clean build would have failed as the cautionary case for the scheduled
clean-build practice.

Do NOT cover: what checks run in the pipeline (Ch 57); matrix build mechanics (Ch 60);
toolchain and dependency version pinning itself (Ch 63 — caching is about build speed given
a fixed dependency state; Ch 63 is about how that dependency state is chosen and updated
over time).
```

---

## Ch 60 — Matrix Builds

```
This chapter covers running the same pipeline across multiple dimensions — operating
system, language/runtime version, dependency version — to catch environment-specific
breakage that a single-configuration pipeline would miss entirely.

Cover the combinatorial cost explicitly: a matrix of 3 operating systems by 4 language
versions by 2 dependency-version sets is 24 jobs, and that cost grows multiplicatively with
every dimension added, not additively — a fact many teams underestimate when adding "just
one more" axis. Take a position: matrix dimensions should be chosen to match a project's
actual, published support commitments (the language versions and platforms it genuinely
promises to support), not maximized for theoretical thoroughness. Testing ten patch versions
of an OS when the only real behavioral variable is the major version is pure waste; pruning
a matrix cell that doesn't represent a genuinely distinct risk is a legitimate optimization,
not a coverage gap.

Cover the `fail-fast` matrix setting as the concrete mechanical link back to Ch 58: when one
matrix cell fails, should the rest of the matrix be cancelled immediately (saving compute,
at the cost of not knowing whether other cells would also have failed) or allowed to run to
completion. Reference Ch 58's general trade-off rather than re-arguing it; this chapter
covers the specific, named configuration option.

Anchor in real systems: GitHub Actions' and GitLab CI's matrix build syntax, including the
literal `fail-fast` key, as the mechanical implementation; the common real-world matrix shape
of {OS} × {language version} as the default, minimal-but-meaningful configuration for most
cross-platform libraries.

Do NOT cover: the general fail-fast-vs-fail-safe principle (Ch 58 — reference it); caching
strategy, which interacts with matrix builds (each cell may need its own cache key) but is
covered on its own terms in Ch 59; what checks are worth running at all (Ch 57).
```

---

## Ch 61 — Release Automation

```
This chapter covers turning code that has passed CI into a published, versioned release
without manual, error-prone, human-executed steps — and it depends directly on Ch 51's
commit message discipline, which this chapter should treat as a hard prerequisite rather
than an optional nicety.

Cover the mechanical chain precisely: tools like semantic-release, release-please, and
Changesets parse structured commit messages (Conventional Commits, from Ch 51) to
automatically determine the next version number (feat → minor, fix → patch, a documented
breaking-change marker → major) and generate a changelog, then create the tag (Ch 56) and
publish the artifact. Take a position: release automation is a [Strong Recommendation] once
a team actually follows a structured commit convention — but automating a release process on
top of inconsistent, unstructured commit messages just automates unreliable version bumps
faster; the automation is only as trustworthy as the commit discipline feeding it.

Cover publishing credentials at the level of a boundary flag, not a deep treatment: modern
package registries increasingly support OIDC-based trusted publishing (npm, PyPI) as a
replacement for long-lived API tokens stored as CI secrets — name this as the current
direction of travel, and explicitly defer the full secrets-management treatment to Part XI,
Ch 83.

Anchor in real systems: semantic-release (npm ecosystem) and release-please (Google's tool,
used widely across its own open-source projects) as concrete, widely-adopted automation
tools built directly on commit-message parsing; npm's and PyPI's newer OIDC trusted-
publishing flows as the credential-free publishing model gaining adoption.

Do NOT cover: what a version number promises about compatibility (Part II, Ch 16); the
commit message format this automation depends on (Ch 51 — reference it as a prerequisite,
don't re-specify it); how the resulting artifact then moves through environments after
release (Ch 62); secrets management for publishing credentials (Part XI, Ch 83).
```

---

## Ch 62 — Environment Promotion

```
This chapter assumes a released artifact already exists (Ch 61's job) and covers how it
moves through environments — development, staging, production — with the guarantee that
what was actually tested is what ships, not a rebuilt approximation of it.

Cover the build-once-promote-many principle as the chapter's central argument: build a
single immutable artifact (a container image, a compiled binary, a packaged release) once,
and promote that exact artifact through each environment, rather than rebuilding from source
at each stage. Take a position on why rebuilding at each stage is a real risk, not just
inefficiency: a rebuild can silently resolve a slightly different dependency version or
compiler output than what was actually tested in the prior environment, so "it passed
staging" no longer reliably implies anything about what's now running in production.

Cover the Twelve-Factor App's build/release/run separation as the relevant, widely-cited
formalization of this idea — build produces an artifact, release combines it with
environment-specific config, run executes it — and connect config-via-environment
explicitly to why the artifact itself must stay identical across environments while only
its configuration varies.

Cover progressive rollout strategies (canary releases, blue-green deployment) as promotion
patterns within production itself, and flag explicitly, without expanding on it, that the
metrics and error budgets used to decide whether a canary should proceed or roll back belong
to Part IX, Ch 73 — this chapter covers the mechanism of promotion, not the signals that
gate it.

Anchor in real systems: the Twelve-Factor App methodology (Heroku) as the canonical
build/release/run formalization; container image promotion via a registry (the same image
digest deployed to each environment in turn) as the standard modern build-once-promote-many
implementation; documented incidents caused by per-environment rebuilds producing subtly
different artifacts as the cautionary case.

Do NOT cover: how the artifact and its version are generated (Ch 61); the specific metrics
or SLOs used to gate a canary or promotion decision (Part IX, Ch 73 — reference, don't
define); incident response and rollback as a reliability practice in general (Part I, Ch 07
already covers MTTR — reference it briefly, don't re-derive it).
```

---

## Ch 63 — Toolchain and Dependency Management

```
This is the capstone of Part VII: how a team manages the versions of the compiler, runtime,
and third-party packages its build depends on over the project's entire lifetime, not just
at a single point in time.

Cover lockfiles as the foundational mechanism: `package-lock.json`, `Cargo.lock`,
`Gemfile.lock`, `poetry.lock`, and equivalents pin the exact, fully-resolved dependency
versions a build uses, making builds reproducible — and connect this directly back to
Ch 59's cache-key design, which depends on exactly this lockfile existing and being precise.

Cover the currency-versus-stability trade-off explicitly, and take a position: pin exact
versions in a lockfile for reproducibility, but pair that with automated, small, frequent
update proposals (Dependabot, Renovate) reviewed like any other pull request (Part VI,
Ch 47's standard applies directly — reference it) rather than either of the two failure
extremes. Freezing dependencies indefinitely accumulates what is functionally *dependency
debt* — a direct, named callback to Ch 48's technical debt framing: an increasingly large,
increasingly risky gap between the pinned versions and current reality, paid down all at
once in a painful "big upgrade" instead of continuously. Floating on latest with no pinning
sacrifices reproducibility and risks an unreviewed minor bump silently changing behavior —
the same Hyrum's Law risk (Part III) where enough consumers depend on an undocumented
behavior that any change to it breaks someone.

Cover automated update tooling as small, frequent, reviewable PRs as the mechanical
resolution of the trade-off — treating each dependency bump with the same review discipline
as any other change, rather than either auto-merging blindly or manually tracking updates
ad hoc.

Anchor in real systems: lockfiles across major package ecosystems (npm, Cargo, Bundler,
Poetry) as the near-universal reproducibility mechanism; Dependabot and Renovate as the
dominant automated-update tools, both built around opening small, individually reviewable
PRs rather than bulk, unreviewed upgrades.

Do NOT cover: the security threat model of a compromised or malicious dependency (Part XI,
Ch 84 — this chapter is about version management mechanics, not the supply-chain security
model); technical debt as a general concept (Part VI, Ch 48 — reference the "dependency
debt" framing, don't re-derive technical debt from scratch); caching mechanics that depend
on the lockfile (Ch 59 — reference it).
```
