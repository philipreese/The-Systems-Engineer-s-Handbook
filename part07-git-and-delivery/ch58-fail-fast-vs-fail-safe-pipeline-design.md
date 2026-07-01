# Ch 58 — Fail-Fast vs. Fail-Safe Pipeline Design

**Prerequisites:** [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) (fail-fast), [The Testing Pyramid](../part05-testing-strategy/ch34-the-testing-pyramid.md) (feedback loop latency), [What Belongs in CI (and What Doesn't)](ch57-what-belongs-in-ci-and-what-doesnt.md)

**New vocabulary introduced:** None — this chapter applies Ch 07's fail-fast principle to pipeline architecture rather than introducing a new concept.

**Key takeaways:**
- Whether to fail fast is already settled by Ch 07; this chapter is about where inside a pipeline's architecture that principle applies. The answer isn't the same at every level.
- [Strong Recommendation] Within a single logical stage, fail-fast is correct: once a build doesn't compile, running the rest of that stage's checks against uncompilable code produces noise, not additional signal.
- [Strong Recommendation] Across independent, parallel check categories — lint, unit tests, integration tests — running to completion and aggregating every failure into one report is the better default. It costs more compute, but it saves developers from paying a full CI round-trip for each failure discovered one at a time.
- Ordering is a second, independent lever from the fail-fast/fail-safe choice: cheap, fast checks (lint, type-check) should gate expensive ones (integration, end-to-end) regardless of how failures are aggregated, so a trivial failure aborts the pipeline before it burns twenty minutes on a suite that was never going to matter once that failure gets fixed anyway.
- GitHub Actions' literal `fail-fast: true/false` matrix-job setting is the concrete, named expression of this exact trade-off at the orchestration level — though the mechanics of matrix builds themselves belong to Ch 60, not this chapter.

---

Given the checks Ch 57 puts in the pipeline, this chapter covers the order they run in and what happens when one of them fails. It's a direct, specific application of the fail-fast principle already defined in Ch 07 — reused here, not redefined — applied to pipeline architecture rather than to a single running program. It does not cover what checks belong in the pipeline at all (Ch 57), caching strategy (Ch 59), or the specific configuration mechanics of matrix builds, where `fail-fast` is a literal per-matrix setting (Ch 60 — this chapter covers the general principle; that chapter covers the matrix-specific configuration).

### Fail-Fast Within a Stage, Fail-Safe Across Stages

**What it is:** Pipeline failure behavior operates at two different levels that are commonly conflated: intra-stage behavior (what happens when a check inside one logical unit — a compilation step, a test run — hits a fatal condition) and inter-stage behavior (how independent, parallel check categories are coordinated with each other). The correct answer is different at each level.

**Why it exists:** Continuing to execute a stage after a condition that already invalidates its output is wasted computation — if a build doesn't compile, running the linter against generated artifacts or attempting downstream steps produces failures that are artifacts of the first failure, not independent information. But stopping the entire pipeline the moment any one of several independent, parallel categories fails has a different cost: it hides every other category's result behind the first one reported, so a developer who fixes the lint failure only discovers the type-check failure on the next run, and the test failure on the run after that.

**Options:**
1. **Fail-fast within a stage** — a stage stops the instant it hits a fatal condition.
2. **Global fail-fast across the pipeline** — the entire pipeline aborts on the first failure anywhere, including in unrelated parallel categories.
3. **Run-to-completion across independent categories** — every independent category runs regardless of another category's failure, and results are aggregated into one report.

**Trade-offs:** Fail-fast within a stage wastes no compute on checks that are already meaningless given an earlier fatal failure. Global fail-fast across the whole pipeline saves the most compute overall, but at the cost of a full CI round-trip per independent failure — a developer fixes one thing, waits for the next run, discovers the next unrelated thing, and repeats. Run-to-completion across independent categories costs more compute (every category runs even when one has already failed) but delivers every failure in a single pass, which is a better outcome for the person who has to act on the result, not only a cheaper one for the machine running it.

[Strong Recommendation] Fail-fast inside a single stage; run-to-completion across independent, parallel stages. There's no case for continuing unit tests once the build itself failed to compile, and there's rarely a case for hiding a unit-test failure behind an unrelated lint failure that has nothing to do with it.

**When to choose each:** Apply fail-fast wherever a later step in the same stage genuinely depends on an earlier one succeeding — compilation gating the tests that run against its output, dependency resolution gating everything downstream of it. Apply run-to-completion wherever categories are genuinely independent of each other — lint, type-check, and unit tests don't depend on each other's outcomes, so there's no reason one's failure should suppress the others' results.

**Common failure modes:** The blind round-trip loop — a pipeline is configured for global fail-fast across every stage. A large change fails lint on the first run; the developer fixes it and pushes, only to have the next run fail on an unrelated type mismatch four minutes later; they fix that and push again, only to hit a unit test failure on the third run. Three sequential fixes that had nothing to do with each other cost three full pipeline cycles and an hour of context-switching, when a single aggregated report would have surfaced all three at once.

**Example:** GitHub Actions exposes this exact trade-off as a literal, named setting on matrix jobs:

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
  fail-fast: true
```

Setting `fail-fast: true` cancels the Ubuntu and macOS jobs the instant the Windows job fails; setting it to `false` lets all three run to completion regardless of any one failing. The specific mechanics of configuring a matrix this way are Ch 60's job — this chapter is only about the underlying principle the setting encodes.

### Execution Ordering: Cheap Checks Gate Expensive Ones

**What it is:** The sequence pipeline stages run in, independent of whether the pipeline is fail-fast or run-to-completion at the aggregation level — specifically, whether cheap, fast checks are guaranteed to run before expensive, slow ones.

**Why it exists:** A fatal, cheap-to-detect failure (a lint error, a type mismatch) makes a pull request unmergeable regardless of what an expensive downstream suite would have found. Running that expensive suite anyway — before the cheap check has even reported — burns real compute and wall-clock time validating a change that was already dead on arrival.

**Options:**
1. **Flat parallel execution** — every check, cheap or expensive, starts the moment a commit lands.
2. **Cost-prioritized tiered execution** — cheap, fast, deterministic checks (lint, type-check) run first and must pass before slower, more expensive checks (integration suites, end-to-end tests) start at all.

**Trade-offs:** Flat execution minimizes total wall-clock time when everything passes, since nothing waits on anything else — but when a cheap check fails, every expensive check that started in parallel with it still burns its full runtime for no benefit, since the change was already going to be rejected. Tiered execution adds wall-clock time in the success case (each tier waits on the one before it) but eliminates that wasted spend entirely: an expensive suite never starts against a change that a cheap check has already disqualified.

**When to choose each:** Tiered execution is the right default for any pipeline where the expensive tiers cost meaningfully more than the cheap ones — which is nearly every real system. Flat parallel execution is only defensible where the entire portfolio is already fast enough that there's nothing expensive to protect against running unnecessarily.

**Common failure modes:** The burned twenty-minute void — a pipeline runs everything in parallel to minimize wall-clock time. A trivial lint error fails in four seconds, but a twenty-minute, cloud-billed end-to-end suite that started at the same moment keeps running its full duration anyway, because nothing gated it on the lint result. The team pays for twenty minutes of compute validating a change that was already known to be rejected after the first four seconds.

**Example:** A tiered Kubernetes-style pipeline runs `golangci-lint` and `go fmt` in well under thirty seconds as Tier 1; only if that passes does Tier 2 run a fast unit-test profile in a few minutes; only if that passes does Tier 3 spin up a full integration environment and run conformance suites that take twenty minutes. A change with a formatting error never reaches Tier 3 at all — it's rejected in Tier 1, before any expensive resource is ever allocated.

---

### Why Smart Engineers Disagree

The disagreement is about which cost a pipeline should be optimized to minimize: compute spend, or the developer's iteration time.

One position treats compute cycles as a real, measurable financial cost: letting parallel test runners burn minutes or hours of CPU time after a build has already failed is straightforward waste, and a pipeline that halts the instant anything breaks is the fiscally responsible default. The opposing position treats a developer's context-switching cost as the more expensive variable in the loop: when a pipeline halts on the very first failure, a developer pays the cost of re-establishing context on the same change once per independent failure, and burning some extra compute to hand them a complete report in one pass is worth it if it saves several rounds of that switching cost.

Both are describing a real cost, and the resolution isn't to pick one side — it's to recognize that the two costs live at different levels of the same pipeline. Compute waste from continuing a stage after it's already fatally broken is close to pure loss, with no compensating benefit; that's where fail-fast belongs. Compute spent running independent, parallel categories to completion buys something real — a complete report instead of a sequential trickle of surprises — which is why that's where run-to-completion belongs instead. The disagreement resolves once the question stops being "fail-fast or fail-safe" as a single global choice and becomes "which of these am I applying, and at which level."
