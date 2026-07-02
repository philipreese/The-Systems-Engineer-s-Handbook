# Ch 57 — What Belongs in CI (and What Doesn't)

**Prerequisites:** [The Testing Pyramid](../part05-testing-strategy/ch34-the-testing-pyramid.md) (feedback loop latency), [What Belongs at Each Layer](../part05-testing-strategy/ch35-what-belongs-at-each-layer.md), [Code Review](../part06-engineering-process/ch47-code-review.md) (Principle 8)

**New vocabulary introduced:** None — this chapter applies feedback loop latency (Ch 34) and mechanical enforcement (Principle 8, Ch 47) to pipeline scope rather than coining new concepts.

**Key takeaways:**
- CI's scope question was left open by Ch 47's mandate to automate machine-verifiable checks out of human review; this chapter answers it: CI is everything cheap, fast, and deterministically enforceable per change — not everything that could conceivably be tested.
- [Consensus] Martin Fowler's original definition of Continuous Integration (2006) is about feedback speed, not test completeness. A pipeline that takes hours, or is routinely red for reasons unrelated to the change under test, has already defeated its own purpose regardless of how thorough it's become — the same feedback loop latency concept Ch 34 already established, applied to the pipeline itself.
- [Strong Recommendation] Anything mechanically verifiable — lint, formatting, type-checking, the unit-test layer of the testing pyramid (Ch 34–35) — belongs in the blocking pipeline and out of human review entirely, per Principle 8. A reviewer's attention spent on brace placement is attention not spent on the actual risk in a change.
- What doesn't belong in the blocking path is whatever is genuinely expensive relative to the value of running it on every single commit — full end-to-end suites, exhaustive security scans, multi-service simulations. These aren't worthless; they don't earn a place gating every change, and belong on a slower, scheduled cadence instead.
- A pipeline that grows past the point where developers actually wait for it has stopped functioning as continuous integration, no matter how much it checks. The failure mode isn't insufficient coverage — it's a system nobody trusts enough to use the way it was designed to be used.

---

This chapter opens the second half of Part VII by resolving the scope question Ch 47 left open when it deferred "pre-merge CI checks and what belongs in them" here. It covers what an automated pipeline should validate on every change and what should be deferred elsewhere — not the testing pyramid itself (Ch 34, Ch 35 already own what belongs at each test layer; this chapter references their conclusions rather than re-deriving them), not the code review standard (Ch 47), and not the order checks run in or what happens when one fails (Ch 58 — that's ordering and failure semantics; this chapter is scope only).

Fowler's original definition of Continuous Integration frames the whole question: CI is the practice of integrating code frequently, with automated verification that gives fast, trustworthy feedback on every integration. Fast and trustworthy are both load-bearing. A check that's thorough but slow, or a suite that's fast but flaky, fails the definition as completely as having no automation at all — either one breaks the same feedback loop latency Ch 34 already identified as the thing that determines whether developers actually verify their changes before moving on.

### Pipeline Scope: Blocking Pre-Merge vs. Asynchronous Cadence

**What it is:** The architectural split between a blocking pipeline that runs synchronously on every commit before a merge is allowed, and a non-blocking pipeline that runs on a periodic schedule — nightly, weekly, or pre-release — independent of any single change.

**Why it exists:** Three constraints bound what belongs in the blocking path: cost per run, determinism (the same input has to produce the same output, or the signal can't be trusted), and how quickly a developer can act on the result. Anything that violates one of these — expensive, flaky, or slow enough that developers stop waiting for it — doesn't belong gating every commit, regardless of how much value it provides in aggregate.

**Options:**
1. **Monolithic gate verification** — the entire test and audit portfolio, including deep static analysis, full security scans, and end-to-end flows, runs on every commit before merge.
2. **Partitioned cadence verification** — the blocking path is constrained to fast, deterministic, mechanically verifiable checks; everything expensive or non-deterministic runs asynchronously on a schedule instead.

**Trade-offs:** Monolithic verification gives the strongest guarantee that nothing regressed before it reaches the trunk, at the cost of pipeline latency that scales with the full portfolio's runtime — and high latency reliably distorts behavior: when clearing CI takes hours, developers batch unrelated changes into larger PRs to avoid paying that cost repeatedly, which is exactly the large-batch integration risk Ch 43 already argues against. Partitioned verification keeps the blocking path fast enough to actually support continuous integration, at the cost of a real regression window — a change that only the asynchronous suite would have caught can land on the trunk and sit there until the next scheduled run surfaces it.

**When to choose each:** Partitioned cadence verification is the right default for essentially any continuously deployed system — the blocking path should complete in minutes, not hours. Monolithic verification is defensible only where the entire suite already completes well within that window, or in domains where zero trunk regressions is a genuine, non-negotiable requirement (safety-critical firmware, for instance) and the team has deliberately chosen to trade velocity for that guarantee.

**Common failure modes:** The congested PR chokepoint — a multi-hour end-to-end browser suite gets added directly to the blocking pipeline. A one-line documentation fix now has to wait behind the same multi-hour queue as every substantive change; PRs back up, review velocity collapses, and developers start bypassing CI entirely under emergency overrides to ship anything at all. The failure isn't that the E2E suite lacks value — it's that its cost was paid on every commit instead of on a cadence that matched its actual value.

**Example:** Kubernetes partitions its pipeline exactly this way: pre-merge PR checks run focused unit tests, linters, and light integration mocks that complete quickly, while multi-hour, multi-cloud conformance suites and fuzzing runs are pushed to separate, periodic pipelines that report regressions on their own schedule without blocking daily development.

### Mechanical Verifiability: What CI Enforces So Review Doesn't Have To

**What it is:** The practice of identifying every property of a change that a linter, formatter, type-checker, or compiler can verify programmatically, and enforcing it in the blocking pipeline rather than leaving it to a human reviewer to notice.

**Why it exists:** This is Principle 8 applied directly: anything a machine can check must be automated out of human review entirely. A reviewer's attention is an expensive, finite resource; spending it on formatting, import order, or type mismatches a tool would catch instantly is waste that also crowds out attention that should go to the parts of a change that actually carry risk.

**Options:**
1. **Human-regulated compliance** — style and structural rules live in a wiki or style guide; reviewers are expected to catch violations by reading the diff.
2. **Programmatic pipeline enforcement** — formatters, linters, and type-checkers run in the blocking pipeline and reject a commit that doesn't comply, before a human ever sees it.

**Trade-offs:**

| Dimension | Human-regulated compliance | Programmatic enforcement |
|---|---|---|
| Review signal-to-noise | Low — comment threads fill with formatting debate | High — reviewers see only logic and design |
| Consistency | Depends on which reviewer happens to be paying attention | Identical every time, for every change |
| Onboarding cost | High — new engineers learn conventions from prose | Low — the tool corrects mistakes immediately |

[Strong Recommendation] Programmatic enforcement, essentially without exception. A pull request should be effectively unreviewable by a human until the mechanical checks are already green — there's no argument for spending review time on something a linter settles in seconds.

**When to choose each:** Programmatic enforcement is the default for any shared codebase. Human-regulated compliance is only what's left over for the (shrinking) category of stylistic judgment calls that genuinely can't be reduced to a mechanical rule yet.

**Common failure modes:** The peer-review style war — a team without automated formatting or lint checks receives a substantial architectural refactor. Three reviewers spend the review debating brace placement, naming casing, and import ordering; the PR stalls for weeks over cosmetic disagreement, and an actual design flaw in the change passes into production completely unexamined, because the review attention that should have caught it was spent elsewhere.

**Example:** Projects like Redis and SQLite treat formatting as an objective, binary gate: a linter runs as the very first pipeline step and rejects anything that doesn't match the style guide before any test executes, catching the mechanical issue in seconds and ensuring a human reviewer never has to look at it.

---

### Why Smart Engineers Disagree

The disagreement is about what promise CI is actually supposed to keep: an absolute guarantee that the trunk is always clean, or a fast feedback loop that supports frequent integration.

One position holds that merging any code without running every available validation — full end-to-end flows, deep security analysis, everything the organization owns — is an abandonment of engineering discipline, and that a slow pipeline is an acceptable price for a trunk that's genuinely always deployable. The opposing position, closer to Fowler's original framing, holds that a pipeline taking hours has already destroyed the practice it claims to implement: when clearing CI is slow and painful, developers stop integrating frequently, retreat into longer-lived branches, and defer exactly the integration risk Ch 50 already argues should be paid continuously and in small amounts, not deferred and paid all at once.

Both positions are responding to a real cost — the first to the cost of an undetected regression, the second to the cost of a slow feedback loop that changes how people actually work. The resolution tracks the same distinction the rest of this chapter draws: a team with disciplined small-batch changes and runtime guardrails like feature flags (Ch 43) can afford to keep the blocking path fast and push everything expensive to an asynchronous cadence, accepting a bounded regression window in exchange for a loop fast enough that people actually use it. A team without that discipline, or in a domain that genuinely cannot tolerate any trunk regression, is choosing a different trade-off deliberately — not failing to understand this one.
