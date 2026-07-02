# Ch 63 — Toolchain and Dependency Management

**Prerequisites:** [Internal vs. External API Design](../part03-api-design/ch25-internal-vs-external-api-design.md) (Hyrum's Law), [Code Review](../part06-engineering-process/ch47-code-review.md), [Technical Debt](../part06-engineering-process/ch48-technical-debt.md), [Caching Strategy in CI](ch59-caching-strategy-in-ci.md)

**New vocabulary introduced:** dependency debt

**Key takeaways:**
- A build is only reproducible if it can be recreated with the exact same compiler, runtime, and dependency graph. Lockfiles exist to pin that graph precisely, and Ch 59's cache-key design depends directly on the lockfile being committed and accurate — this isn't housekeeping, it's part of the build system itself.
- [Strong Recommendation] Pin exact versions through a lockfile for reproducibility, but pair that with small, frequent, automated update proposals (Dependabot, Renovate) reviewed like any other pull request (Ch 47's standard applies directly) — rather than either of the two failure extremes: freezing indefinitely, or floating on latest with no pin at all.
- Dependency debt: the accumulated gap between a project's pinned versions and current upstream reality. A direct instance of Ch 48's technical debt framing — every postponed update increases what eventually has to be absorbed at once, paid down continuously in small increments or all at once in a large, high-risk migration.
- Floating on unpinned version ranges sacrifices reproducibility and carries the same Hyrum's Law risk as any other undocumented-behavior dependency: enough consumers rely on an upstream package's incidental behavior that any change to it, even one that's technically SemVer-compliant, breaks someone who never asked for that guarantee in the first place.
- Whether a dependency update should merge automatically isn't answered by "did CI turn green" alone — it tracks the same blast-radius reasoning used throughout this handbook: a low-blast-radius, isolated dependency can auto-merge on green CI; a dependency sitting in a core, hard-to-reverse part of the system needs human review regardless of how green the pipeline looks.

---

This chapter is the capstone of Part VII: how a team manages the versions of the compiler, runtime, and third-party packages its build depends on over the project's entire lifetime, not only at a single point in time. It covers version management mechanics, not the security threat model of a compromised or malicious dependency (Part XI, Ch 84 — that's a different question from whether a version is current), and it applies Ch 48's technical debt framing to dependencies specifically rather than re-deriving technical debt from scratch.

### Lockfiles: The Foundation of Reproducibility

**What it is:** A lockfile — `package-lock.json`, `Cargo.lock`, `Gemfile.lock`, `poetry.lock` — records the exact, fully resolved version of every direct and transitive dependency actually used to produce a build, as distinct from the manifest file's version ranges (`^4.17.0`), which only describe what's acceptable. The same discipline extends to the toolchain itself: pinning the exact compiler and runtime version, not only the packages built with it.

**Why it exists:** Without a lockfile, the same source code can resolve a different dependency tree on different days, as the packages it depends on continue to publish new versions upstream — two developers running the same install command a week apart can end up with different binaries from identical source. A lockfile makes the build reproducible by removing that ambiguity entirely, and it's the direct prerequisite for Ch 59's cache-key design: a cache keyed on a hash of the lockfile only invalidates correctly if the lockfile is the actual, precise record of what changed.

**Options:**
1. **Range-based floating resolution** — the manifest specifies acceptable ranges; the exact tree is resolved fresh on every install.
2. **Lockfile pinning** — a committed, machine-generated file locks the exact version and checksum of every package in the tree, resolved once and reused thereafter.

**Trade-offs:** Floating resolution automatically absorbs upstream security patches without anyone opening a pull request, but it destroys build determinism — an unreviewed, unremarked bump in a transitive dependency can silently change application behavior with no corresponding change anywhere in the project's own history. Lockfile pinning guarantees the same resolved tree everywhere, at the cost of a machine-generated file that adds churn to the repository and can conflict during a merge (Ch 55).

[Strong Recommendation] Lockfile pinning for any top-level application, service, or deployable — reproducibility is worth the churn. Range-based manifests remain appropriate for a library's own published version constraints, since a library's downstream consumers need the flexibility a strict lockfile would deny them; the library's own development lockfile still pins exactly what its own tests actually ran against.

**Common failure modes:** The phantom mirror regression — a service relies on floating, range-based dependency resolution. A feature passes staging cleanly on a Tuesday. Overnight, an upstream utility package ships a subtle performance regression in a routine most of the codebase touches. An unrelated hotfix triggers a fresh build the next day, which resolves the new upstream version as part of routine dependency resolution; the deploy goes out, latency climbs, and the team spends hours investigating the unrelated hotfix's diff, because nothing in the project's own history shows that the actual change was an upstream dependency nobody reviewed.

**Example:** Rust's Cargo checks for `Cargo.lock` on every build and resolves the compiler tree from its exact recorded checksums if present — if an upstream author later deletes or alters a published version, the local build either reproduces identically from the lock or fails fast with an explicit checksum mismatch, never silently substituting something different. Many Rust projects extend the same pinning discipline to the toolchain itself, committing a `rust-toolchain.toml` alongside `Cargo.lock` so the compiler version is reproducible along with the dependency graph it compiles.

### Currency vs. Stability: Pin, and Update Continuously

**What it is:** The ongoing tension between keeping dependencies current with upstream and keeping a system's behavior stable and reproducible — and the position that these aren't actually opposed, provided updates happen continuously in small increments rather than being deferred.

**Why it exists:** Dependencies evolve whether a project tracks that evolution or not. Freezing a dependency graph indefinitely doesn't stop that evolution — it only defers absorbing it, and the deferred change doesn't shrink while it waits. Every postponed update increases what eventually has to be absorbed all at once: this is dependency debt, a direct instance of Ch 48's technical debt framing, where the gap between what's pinned and what upstream has moved on to accrues the same way any other deliberately deferred cost does, except here nobody explicitly chose to defer it — it simply accumulated by default.

**Options:**
1. **Long-term freezing** — pin a baseline and don't touch it except under forced circumstances (an unavoidable security advisory, a hard architectural migration deadline).
2. **Continuous micro-upgrades** — automated tooling (Dependabot, Renovate) continuously proposes small, single-dependency version bumps as ordinary pull requests, reviewed under the same standard as any other change (Ch 47).

**Trade-offs:**

| Dimension | Long-term freezing | Continuous micro-upgrades |
|---|---|---|
| Short-term maintenance cost | Near zero — nothing to review | Steady — a regular stream of small review work |
| Upgrade risk when it finally happens | Severe — years of change land at once | Low — each increment is independently reviewable |
| Alignment with upstream ecosystem | Degrades continuously | Stays current |

[Strong Recommendation] Continuous micro-upgrades as the default operating mode: pin exact versions through the lockfile for reproducibility, and let automated tooling propose small, individually reviewable updates on a regular cadence, rather than treating "not touching dependencies" as the safe choice — it isn't; it's a choice to accumulate debt instead of paying it continuously.

**When to choose each:** Continuous updates for essentially all actively developed software. Long-term freezing only for systems genuinely locked by external constraints that make any version change expensive regardless of engineering discipline — a certified medical device, deeply embedded firmware tied to fixed hardware — and even there, the freeze should be a deliberate, documented decision, not a default nobody revisited.

**Common failure modes:** The big upgrade bankruptcy — a team freezes its language runtime and core framework version for several years to focus entirely on feature velocity. A critical, unpatched vulnerability is eventually discovered in the frozen framework version, which upstream stopped supporting long ago; the only path to a patched version requires jumping several major versions at once. Thousands of breaking API changes, behavioral shifts, and type errors hit the codebase simultaneously; the build doesn't compile, the test suite doesn't pass, and feature development stops entirely for weeks while the team pays down years of deferred dependency debt under the worst possible conditions — during an active security incident, with no ability to defer any further.

**Example:** Dependabot or Renovate, running continuously against a repository, opens a pull request the moment an upstream dependency publishes a new version — a single, small, precisely described change like "bump `library-abc` from 1.2.1 to 1.2.2" — reviewed and merged through the exact same process as any other code change, converting what used to be a deferred, dreaded migration into a routine, low-risk habit.

---

### Why Smart Engineers Disagree

The disagreement isn't about whether dependency updates should be automated — both sides already agree they should be proposed automatically. It's about whether a green CI run is sufficient justification to merge one without a human ever looking at it.

One position holds that if a team has invested in a genuinely strong testing pyramid (Ch 34) — solid unit coverage, type-checking, linting — that infrastructure should be trusted: requiring a human to manually approve every one of a dozen routine patch-level bumps a week is process overhead that, in practice, degrades into rubber-stamping anyway, since nobody meaningfully scrutinizes the fifteenth nearly-identical dependency PR of the month. The opposing position points out that automated test suites are necessarily finite — they can't exercise every real production condition, and a dependency's Hyrum's-Law-driven behavioral shift can pass every test while still breaking something that only manifests under real traffic. To this position, a human glancing at a changelog before merging is a cheap, meaningful check that automation alone can't replace.

The resolution tracks blast radius, not a blanket rule either way. A dependency that's genuinely isolated — a logging utility, a documentation formatter, something with a narrow, well-contained failure surface — can reasonably auto-merge once its own integration checks pass, because the cost of being wrong is small and easily reversed. A dependency sitting inside a core domain module, a database driver, or anything touching cryptography carries a blast radius that a green test suite alone doesn't bound, and deserves the same human review discipline as any other change to that part of the system — not because automation is untrustworthy in general, but because this specific location in the system is exactly where Ch 09's reversibility-and-blast-radius argument says extra deliberation is warranted.

Part VII began with the choice of how branches are structured to move a single change from first commit to production (Ch 50). It ends here, with the same underlying question posed at the longest time scale a codebase has to answer it: how to keep a system evolving continuously without ever losing the ability to reproduce, trust, and reason about the exact state it's actually running.
