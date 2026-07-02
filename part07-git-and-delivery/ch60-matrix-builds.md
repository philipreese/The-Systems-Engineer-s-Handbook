# Ch 60 — Matrix Builds

**Prerequisites:** [What Belongs in CI (and What Doesn't)](ch57-what-belongs-in-ci-and-what-doesnt.md), [Fail-Fast vs. Fail-Safe Pipeline Design](ch58-fail-fast-vs-fail-safe-pipeline-design.md), [Caching Strategy in CI](ch59-caching-strategy-in-ci.md)

**New vocabulary introduced:** None — this chapter applies the fail-fast/fail-safe trade-off (Ch 58) and cache-key precision (Ch 59) to one specific mechanism rather than introducing new concepts.

**Key takeaways:**
- A matrix build isn't a different pipeline — it's a Cartesian expansion of one pipeline across dimensions like operating system, runtime version, or dependency set. Total job count is the product of every dimension's size, which means it scales multiplicatively, not additively — a cost teams reliably underestimate when adding "just one more" axis.
- [Strong Recommendation] Matrix dimensions should encode a project's actual, published support commitments, not a maximized theoretical net. Testing every patch version of an OS when only the major version changes observable behavior is waste, not thoroughness — pruning a cell that doesn't represent a genuinely distinct risk is a legitimate optimization, not a coverage gap.
- The `fail-fast` matrix setting is the same trade-off Ch 58 already established — compute savings versus complete diagnostic visibility — applied at the granularity of one matrix cell instead of one pipeline stage. This chapter covers the specific, named configuration option; that chapter owns the general argument.
- Context changes the right default: cancel remaining cells on the first failure for a fast, everyday PR pipeline, where speed matters more than knowing every affected platform at once; let every cell run to completion for a release-gating or nightly run, where the entire point is knowing exactly which platforms are affected before a release ships.
- A matrix that tests environments nobody actually deploys to doesn't increase confidence — it manufactures the appearance of it, at a real and multiplying compute cost.

---

This chapter covers running the same verification logic across multiple environment dimensions to catch environment-specific breakage a single-configuration pipeline would never surface. It does not re-derive the general fail-fast-versus-fail-safe principle (Ch 58 — this chapter applies it to one specific configuration option), how caching interacts with a matrix (Ch 59 — each cell needs its own cache key, referenced here, not re-argued), or what checks are worth running at all (Ch 57).

### Matrix Dimension Selection: Support Commitments, Not Theoretical Coverage

**What it is:** A matrix build expands a single logical pipeline into independent execution cells across variability axes — operating system, language or runtime version, dependency set. Total job count is the product of every dimension's cardinality: three operating systems by four runtime versions by two dependency configurations is 3 × 4 × 2 = 24 independent jobs, and adding a third dependency axis with three values doesn't add three jobs, it multiplies the total to 72.

**Why it exists:** Each additional axis represents a genuinely independent source of behavioral variability that a single-environment pipeline can't detect — code that compiles and passes every test on Linux can still fail on Windows due to filesystem case sensitivity or path separator handling that never surfaces anywhere else. But coverage only has value where the axis actually reflects a real difference a user will encounter; testing a dimension that doesn't correspond to anything in production is coverage in name only.

**Options:**
1. **Exhaustive matrix** — every available combination of OS patch version, language version, and dependency variant.
2. **Support-commitment matrix (pruned)** — dimensions and values constrained to exactly what the project publicly commits to supporting, with redundant or non-distinct cells explicitly excluded.

**Trade-offs:** An exhaustive matrix maximizes theoretical coverage, but its cost grows multiplicatively with every dimension added — a config that looked reasonable at three axes becomes unmanageable the moment a fourth is added without anyone deliberately pruning it. A pruned matrix keeps cost bounded and aligned with what's actually shipped, at the real but narrow risk of missing a defect that exists only in a specific untested combination that genuinely wasn't worth testing in the first place.

[Strong Recommendation] Constrain matrix dimensions to a project's actual, published support commitments — major OS families and major runtime versions the project actually claims to support — rather than maximizing for theoretical thoroughness. Reserve genuinely exhaustive matrices for foundational, low-level software — compilers, cryptography libraries, database engines — where an environment-specific defect has consequences disproportionate to the software's own size, because it corrupts everything built on top of it.

**When to choose each:** Support-commitment matrices are the right default for essentially all application and product code. Exhaustive matrices are defensible only for foundational infrastructure with a very large number of downstream consumers, and even then usually belong on a slower, asynchronous cadence (Ch 59's clean-build pattern applies here too) rather than gating every commit.

**Common failure modes:** The combinatorial queue explosion — a matrix already covering every OS variant, several container architectures, and multiple dependency permutations receives a handful of quick, unrelated commits fixing a cosmetic layout issue. Each push schedules the full matrix; the shared build pool fills with jobs validating a change that carries essentially no environmental risk, and the entire organization's pipeline queue stalls for hours to validate something cosmetic.

**Example:** A cross-platform library supporting four minor Python versions across three operating systems doesn't need all twelve combinations — testing the lowest and highest supported runtime version on the primary platform, and only the highest version on secondary platforms, using explicit exclusion rules to drop the redundant cells, catches the real cross-platform risk without paying for combinations that don't represent a distinct one.

### Fail-Fast in Matrix Execution

**What it is:** The literal `fail-fast: true`/`false` setting on a matrix job, controlling whether a failure in one cell cancels every other cell in the same matrix run or lets them all continue to completion independently.

**Why it exists:** This is Ch 58's general fail-fast-versus-fail-safe trade-off, applied at the granularity of one matrix cell instead of one pipeline stage. Canceling on the first cell failure saves compute and clears the queue faster; letting every cell finish costs more compute but reveals whether a failure is isolated to one platform or affects all of them, which matters differently depending on why the matrix is running at all.

**Options:**
1. **`fail-fast: true`** — the first cell to fail cancels every other active or pending cell in the same matrix.
2. **`fail-fast: false`** — every cell runs independently to its own completion regardless of any other cell's result.

**Trade-offs:**

| Dimension | `fail-fast: true` | `fail-fast: false` |
|---|---|---|
| Compute cost | Low — unneeded jobs are canceled immediately | High — every cell runs its full duration |
| Diagnostic completeness | Weak — hides whether the failure is isolated to one platform or shared | Complete — shows the exact environmental scope of the failure at once |
| Queue throughput | Fast — clears cluster capacity quickly | Slower — occupies runners until the slowest cell finishes |

**When to choose each:** `fail-fast: true` for everyday pull request pipelines, where a red cell already means the change needs work and speed of that signal matters more than knowing every platform it happens to also break. `fail-fast: false` for release-gating and nightly runs, where the actual goal is knowing precisely which supported platforms are affected before a release ships — canceling early there trades away that knowledge for a compute saving that isn't the point of running the matrix in the first place.

**Common failure modes:** The interleaved multi-platform ping-pong — a pipeline defaults to `fail-fast: true` everywhere. A change breaks all three platforms in a matrix for unrelated reasons; the Windows cell fails first and cancels macOS and Linux before they can report anything. The developer fixes the Windows issue, pushes again, and only now discovers the macOS failure that was there the entire time; fixes that, pushes a third time, and finally uncovers the Linux-specific issue that had been hidden behind the first two cancellations all along. Three sequential round-trips resolve what a single run-to-completion pass would have surfaced at once.

**Example:** A GitHub Actions matrix spanning three operating systems and three runtime versions sets `fail-fast: true` for its pull request workflow but flips to `false` in its nightly and release-branch workflow — the same matrix definition, deliberately configured differently depending on whether the run's purpose is a fast day-to-day signal or a complete pre-release picture.

---

### Why Smart Engineers Disagree

The disagreement is about how much environmental risk actually justifies a matrix's cost — and it tracks a real difference in what kind of software is being tested, not a difference in engineering judgment.

One position holds that software can't be considered reliable unless it's been tested against every environment permutation it might conceivably encounter — a subtle libc version difference, an instruction-set difference between x86_64 and ARM64, a compiler optimization flag — because any of these can trigger a defect invisible in a standard environment. To this position, a slow, expensive matrix is a reasonable price for that assurance, and shrinking it is cutting a corner. The opposing position treats CI as a resource with real financial and latency constraints, and points out that most application code runs inside a managed runtime that already isolates it from exactly the low-level host variation the first position worries about — testing redundant patch-level OS variants for code that never touches the host directly buys no real signal, only cost.

Both positions are correctly reading their own situation. The first is right about software that operates close to the hardware or is depended on by enormous numbers of downstream consumers — a compiler, a cryptography library, a database engine — where an environment-specific defect has consequences wildly disproportionate to the software's own size. The second is right about the much more common case: application code running in a standardized, managed environment, where the realistic risk surface is far narrower than the theoretical one. The resolution isn't to average the two positions — it's to recognize which kind of software is actually being built, and size the matrix to the risk that specific software actually carries, not to a generic notion of thoroughness that doesn't distinguish between the two.
