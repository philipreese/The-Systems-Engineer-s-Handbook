# Ch 50 — Branching Strategies

**Prerequisites:** [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) (fail-fast), [Decision Frameworks for Trade-offs](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md) (reversibility and blast radius), [Issue as Tracking Unit vs. PR as Review Unit](../part06-engineering-process/ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md) (small-batch integration, feature flags), [Milestone and Phase Planning](../part06-engineering-process/ch44-milestone-and-phase-planning.md) (big-bang integration), [Process Overhead: The Value Threshold](../part06-engineering-process/ch49-process-overhead-the-value-threshold.md)

**New vocabulary introduced:** branch topology, trunk-based development, GitFlow

**Key takeaways:**
- Branching strategy is a decision about *when* integration risk gets paid, not whether it exists. Trunk-based development pays it continuously, in small amounts; GitFlow-style hierarchies defer it to a stabilization event, where it arrives concentrated and expensive.
- [Strong Recommendation] Trunk-based development — short-lived branches merged into a single perpetually-releasable main within hours or days — is the correct default for continuously deployed software, and DORA's research backs it with elite-performance correlation data, not intuition alone.
- [Legitimate Trade-off] GitFlow's long-lived branch hierarchy (`develop`, `release/*`, `hotfix/*`) is not a legacy mistake. It is the right topology when release cadence is genuinely decoupled from `main` — embedded firmware, packaged desktop software, libraries backporting security fixes across several live major versions.
- Long-lived branches are a horizontal-phase plan expressed in Git: the same big-bang-integration failure mode from Ch 44, at branch-lifetime granularity.
- Branch topology is a downstream consequence of release cadence, not a stylistic preference. Every other Git-mechanics chapter in this Part assumes the topology chosen here.

---

Branching strategy is the topology decision every other Git-mechanics chapter in this Part builds on: how many long-lived branches exist, how long a branch is allowed to diverge before it integrates, and how work flows between them. It is not about commit formatting ([Ch 51](ch51-commit-message-conventions.md)), squash-versus-preserve merge policy ([Ch 52](ch52-squash-vs-preserve-history.md)), branch naming or deletion ([Ch 53](ch53-branch-naming-and-lifecycle.md)), force-push judgment ([Ch 54](ch54-force-push-when-its-acceptable.md)), merge-versus-rebase mechanics ([Ch 55](ch55-merge-vs-rebase.md)), or tagging ([Ch 56](ch56-tagging-and-release-markers.md)). All six assume a topology already exists; this chapter is where it gets chosen.

### Branch Topology: Trunk-Based Development vs. GitFlow

**What it is:** The structural choice between a single shared branch that stays perpetually releasable (trunk-based development, including its lightweight variant, GitHub Flow) and a multi-tiered hierarchy of long-lived branches — typically `main`, `develop`, `release/*`, and `hotfix/*` — that isolates work until a scheduled stabilization phase (GitFlow, formalized by Vincent Driessen in his 2010 post "A successful Git branching model").

**Why it exists:** Every active branch is a fork of the codebase's reality. The topology decision is really a decision about how many forks are allowed to exist at once and how long they're allowed to diverge before reconciling — which is the same trade-off Ch 43 already made at the level of a single PR, applied here at the level of the whole repository's structure.

**Options:**
1. **Trunk-based development (TBD).** All work integrates into one long-lived `main` via branches measured in hours or a few days. GitHub Flow is TBD's minimal, PR-centric expression: branch from `main`, open a pull request, merge back into `main`, deploy from `main`.
2. **GitFlow.** `develop` acts as the integration branch; `feature/*` branches merge into `develop`; a `release/*` branch stabilizes a batch of features before merging into `main`; `hotfix/*` branches patch `main` directly for emergencies and must be merged back down into `develop`.

**Trade-offs:**

| Dimension | Trunk-Based Development | GitFlow |
|---|---|---|
| Integration risk | Paid continuously, in small increments | Deferred to release-branch stabilization, paid in a concentrated batch |
| Blast-radius containment | Shifted to runtime guardrails (feature flags) — incomplete code can sit on `main` behind a flag | Contained by branch isolation itself — incomplete code never reaches `main` |
| CI discipline required | High — a broken `main` blocks every engineer immediately | Lower on `main` itself, but `develop` and `release/*` need their own gates |
| Structural overhead | Low — one branch, one merge target | High — multiple long-lived branches, explicit promotion between them |
| Fit for scheduled, versioned releases | Poor — TBD assumes "always releasable" is the actual delivery model | Native — the hierarchy exists to serve exactly this cadence |

[Strong Recommendation] Trunk-based development is the default for continuously deployed software. DORA's *Accelerate* research (Forsgren, Humble, Kim) correlates short branch lifetimes with elite deployment frequency and lead time, driven by less merge-risk accumulation, not by any inherent virtue of having fewer branches. [Legitimate Trade-off] GitFlow is the right choice — not a compromise — when "always releasable from `main`" isn't the actual delivery model: embedded firmware and certification cycles, packaged desktop software (IDEs, office suites) tied to install-based distribution, or libraries that must backport security fixes across several concurrently maintained major versions. Driessen himself updated his original 2010 post in 2020 to state that GitFlow adds unneeded complexity for web applications and continuous delivery, and that such teams should default to something simpler like GitHub Flow — the model's own author drew this same boundary.

**When to choose each:** Choose trunk-based development when deployment is continuous and rollback is cheap — most SaaS and cloud-native systems. Choose GitFlow when release cadence is fixed by something outside the team's control — an app store review cycle, a certification process, a customer's IT department that only installs quarterly. The Linux kernel occupies a middle position worth naming: Linus Torvalds' tree integrates changes through maintainer-owned subsystem trees, a controlled, staged form of delayed integration that reflects its distributed-ownership model rather than a batching-for-batching's-sake choice.

**Common failure modes:**
- **The Ghost Hotfix.** A production regression is patched directly on `main` (or a `hotfix/*` branch) and deployed, but the fix is never merged back down into `develop`. The next scheduled release from `develop` silently overwrites the patch, reintroducing the regression.
- **Trunk always red.** Teams adopt TBD without CI reliable enough to support it; a perpetually broken or flaky `main` defeats the entire premise of continuous integration and pushes engineers back toward informal, ad hoc isolation.
- **Feature-flag bankruptcy.** TBD shifts blast-radius containment onto feature flags. Without cleanup discipline, expired flags accumulate into a non-deterministic configuration space where hidden code paths interact and produce failures no test environment reproduces.
- **Integration hell.** GitFlow's `release/*` consolidation becomes the single point where every deferred conflict lands at once, turning what should be a routine merge into a multi-day event.

**Example:** Google and Meta run trunk-based development inside massive internal monorepos — thousands of engineers committing directly to a shared trunk daily, made viable only by heavy automated pre-merge testing and feature flags decoupling deployment from activation. Traditional desktop software vendors shipping versioned releases through `release/*` branches represent the opposite, equally legitimate case: production isn't continuously updated, so there's no ongoing trunk to protect from divergence in the first place.

### Branch Lifetime as the Risk Accumulator

**What it is:** Regardless of topology, how long any individual branch is allowed to exist before merging — the dial that actually determines how much integration risk has accumulated by the time a merge happens.

**Why it exists:** Divergence between a branch and its parent doesn't grow linearly with time; every other change landing on the parent while a branch sits unmerged adds another axis of potential conflict. A branch's age is a direct proxy for how much of that compounding it has already absorbed.

**Options:**
1. **Short-lived branches** (hours to a few days) — the mechanical requirement for trunk-based development, containing one atomic change (Ch 43's unit of review).
2. **Long-lived branches** (weeks to months) — an entire feature initiative developed in isolation before a single integration event.

**Trade-offs:** Short-lived branches keep conflicts small and reversible, at the cost of requiring the work itself to be decomposed into pieces small enough to integrate continuously — real upfront planning discipline, not a free lunch. Long-lived branches let a developer proceed without that decomposition discipline, but the integration cost doesn't disappear; it compounds silently until the merge, at which point it lands all at once, on code that has since drifted underneath it.

[Consensus] Short-lived branches are correct for production systems: DORA's research ties branch lifetimes under a day to elite delivery performance, and a branch that lives for months is functionally a horizontal-phase plan (Ch 44) expressed in Git — the same big-bang-integration failure mode, at branch-lifetime granularity, with the same root cause: risk that was deferred rather than eliminated, discovered at the point in the process with the least slack left to absorb it.

**When to choose each:** Default to short-lived branches for any code heading to production. Long-lived isolation is defensible only for genuinely speculative, throwaway exploration that isn't expected to integrate at all, or for open-source projects accepting asynchronous contributions from untrusted external contributors who can't be given direct trunk access.

**Common failure modes:** The merge-conflict loop — a branch falls weeks behind, the engineer spends days resolving hundreds of lines of conflicts, and by the time they finish, trunk has moved again, restarting the cycle. Each iteration effectively rewrites the feature against a moving target instead of building it once.

**Example:** A branch that lives for three months is not "protected" from integration risk — it has deferred all of it to a single event at the end, guaranteeing that event is expensive. This is the same lesson Ch 43 draws about PR size, measured in elapsed time on a branch rather than lines in a diff.

---

### Why Smart Engineers Disagree

The disagreement here is not about facts; both sides agree that integration cost exists and must be paid somewhere. It's about which cost is more tolerable to force onto the team day-to-day.

Engineers who favor long-lived branches or GitFlow's fuller hierarchy are usually optimizing for individual focus and finished-feature quality: forcing incomplete work into a shared trunk introduces constant churn and interrupts other people's context, so a feature should be built and polished in isolation before it ever touches shared reality. From that vantage point, a large merge at the end is a bounded, acceptable tax for protecting the trunk's stability the rest of the time.

Engineers who favor trunk-based development treat isolation as a statistical illusion rather than a genuine safety property: the longer a branch stays separate, the further it drifts from what the rest of the system actually looks like, and that drift is not avoided by staying isolated longer — it's compounded. Forcing integration daily surfaces architectural mismatches and breaking changes while they're still cheap to fix, rather than letting them harden, undetected, inside a branch nobody else is looking at.

Both positions are rational given their constraints. If deployment is continuous and reversible, the trunk-based argument wins outright — DORA's data isn't ambiguous on this point. If deployment is genuinely scheduled and externally gated, the isolation the GitFlow side wants is not paranoia; it's a correct response to a release cadence that doesn't reward daily integration into a branch nobody can ship from anyway. The actual failure mode is neither position — it's applying GitFlow's stabilization assumptions inside a system that already deploys continuously, or forcing trunk-based development onto a team that genuinely cannot deploy `main` on any given day.
