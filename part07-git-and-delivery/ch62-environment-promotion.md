# Ch 62 — Environment Promotion

**Prerequisites:** [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) (MTTR), [Versioning and Backward Compatibility](../part02-software-architecture/ch16-versioning-backward-compatibility.md), [Tagging and Release Markers](ch56-tagging-and-release-markers.md), [Release Automation](ch61-release-automation.md)

**New vocabulary introduced:** build-once-promote-many

**Key takeaways:**
- This chapter assumes Ch 61's job — producing a versioned, tagged artifact — is already done. It covers only how that artifact moves through environments afterward: development, staging, production.
- [Strong Recommendation] Build a single immutable artifact exactly once and promote that identical artifact through every environment. Rebuilding at each stage is a real correctness risk, not merely inefficiency — a rebuild can silently resolve a different dependency version or produce different compiler output than whatever was actually validated in the prior environment, so "it passed staging" stops reliably implying anything about what's now running in production.
- The Twelve-Factor App's build/release/run separation is the canonical formalization: build produces the artifact, release combines it with environment-specific configuration, run executes it. The artifact has to stay identical across environments precisely so that only configuration is allowed to vary — anything else reintroduces the same drift risk rebuilding does.
- Progressive rollout strategies — canary, blue-green — are promotion patterns applied within production itself, reducing blast radius by increasing exposure gradually instead of replacing the entire fleet at once. The metrics and error budgets that decide whether a rollout proceeds or rolls back belong to Part IX, Ch 73; this chapter covers the mechanism of promotion, not the signals that gate it.
- A canary running at a small traffic percentage can hide a real regression from aggregate, fleet-wide alerting precisely because it's a small fraction of total traffic. A rollout mechanism without telemetry that specifically watches the canary segment isn't providing the safety it appears to.

---

This chapter assumes a released, tagged artifact already exists (Ch 61's job) and covers how it moves through environments with a guarantee that what was actually tested is what ships — not a rebuilt approximation of it. It does not cover how that artifact or its version is generated (Ch 61), the specific metrics or SLOs used to gate a canary or promotion decision (Part IX, Ch 73 — referenced, not defined here), or incident response and rollback as a general reliability practice (Ch 07 already covers MTTR — referenced briefly, not re-derived).

### Build Once, Promote Many

**What it is:** Building a single immutable artifact — a container image, a compiled binary, a packaged library — exactly once, then deploying that identical artifact, unchanged, through development, staging, and production, rather than recompiling from source at each stage.

**Why it exists:** Testing establishes confidence in a specific artifact, not in the source code that happened to produce it. Rebuilding at each environment produces a different artifact every time — one that can silently resolve a slightly different transitive dependency version, or come out of a compiler with different output, than whatever was actually exercised in the previous environment. That's a real correctness risk, not only wasted compute: it means "this passed staging" no longer reliably implies anything about what's now running in production, because production may be running something that was never actually validated at all.

**Options:**
1. **Build once, promote many** — compile or package the artifact once, at the earliest point in the pipeline, and move that exact artifact through every subsequent environment.
2. **Per-environment rebuilding** — recompile from source independently at each environment's deployment stage.

**Trade-offs:** Build-once promotion guarantees that what was validated is what's running, since nothing about the artifact can change between environments — the only remaining question is configuration, covered next. Per-environment rebuilding allows environment-specific compilation choices, but at the cost of exactly the guarantee that matters: a rebuild can pick up an unpinned dependency update between one environment's build and the next, producing a production artifact that never went through staging at all despite every dashboard showing a green staging run.

[Strong Recommendation] Build once, promote many, as the default for essentially all production systems. Per-environment rebuilding is defensible only where the build itself is genuinely the object under test — not where the deployed artifact is what matters.

**When to choose each:** Build-once promotion for any system where "staging passed" needs to mean something about production. Per-environment rebuilding only in narrow cases — legacy platforms physically unable to accept runtime configuration, for instance — and even then, treated as a gap to close rather than a stable architecture.

**Common failure modes:** The stale-manifest dependency injection — a team rebuilds independently per environment, relying on a dependency manifest that isn't pinned by exact hash (Ch 63). A change clears every staging gate on a Thursday; overnight, a third-party library ships a broken patch to an unpinned transitive dependency; Friday's production build pulls that update during its independent rebuild, and the deployment crashes on a regression that never existed anywhere staging actually tested.

**Example:** A Kubernetes deployment referencing a mutable tag like `image: service:latest` or `image: service:staging` has no way to guarantee which actual bytes get pulled at any given moment. Referencing an immutable content digest instead — `image: service@sha256:d58e38d...` — makes the promoted artifact a cryptographic contract: the container runtime refuses to start anything whose content doesn't match that exact hash, which is what actually makes "the same artifact in every environment" a guarantee rather than an intention.

### Build, Release, Run: Configuration Varies, the Artifact Doesn't

**What it is:** The Twelve-Factor App's formalization of software delivery into three distinct phases: build (source becomes an immutable artifact), release (that artifact is combined with environment-specific configuration to produce a versioned release), and run (the release executes). Environment-specific values — database endpoints, API keys, feature flags — are supplied at the release or run phase, never compiled into the artifact itself.

**Why it exists:** Environments differ operationally — different databases, different endpoints, different secrets — but not functionally; the code that runs in staging is meant to be the same code that runs in production. If configuration is compiled into the binary, the binary is no longer environment-agnostic, and you're back to needing a different artifact per environment, which reintroduces the exact drift risk build-once-promote-many exists to eliminate.

**Options:**
1. **Embedded configuration** — environment-specific values compiled or bundled directly into the artifact (a `config.staging.json`, a `config.production.json` shipped alongside the code).
2. **Runtime configuration** — the artifact is environment-agnostic; all environment-specific values are injected via environment variables or an external configuration source at release or run time.

**Trade-offs:** Embedded configuration feels convenient locally — a developer edits a file to match a target environment — but it requires a distinct artifact per environment, and it puts environment-specific secrets inside version-controlled files, where they can leak into the wrong place through an ordinary copy-paste mistake. Runtime configuration keeps a single artifact genuinely universal across every environment and keeps secrets out of source control entirely, at the cost of needing an actual mechanism — environment variables, a config service — to inject values at deploy time.

[Strong Recommendation] Runtime, config-via-environment as the default for anything that isn't a genuinely fixed-purpose system (firmware being the standard exception). This is the direct mechanical consequence of build-once-promote-many: if the artifact must stay identical across environments, configuration is the only thing left that's allowed to vary.

**When to choose each:** Runtime configuration for any distributed or cloud-deployed application. Embedded configuration only for fixed-purpose systems that don't move between environments at all.

**Common failure modes:** The staging stored-token production leak — a team keeps separate `config.staging.json` and `config.production.json` files checked into the repository. During an emergency fix, a developer pastes a production database connection string into the staging config file to test something quickly, and the change merges. The next staging run executes its test suite directly against the live production database, using test data operations against real customer records, because the "staging" configuration was never actually isolated from production credentials in the first place.

**Example:** A container image deploys identically to every environment while its database connection string, feature flags, and API keys are supplied entirely through environment variables set at deploy time — the same digest running in staging and production, differing only in what the orchestrator injects into it at startup.

### Progressive Promotion Within Production

**What it is:** Deployment strategies that introduce a newly promoted artifact to production gradually rather than replacing the entire fleet at once — canary releases (a small percentage of traffic shifts to the new version, increasing over time) and blue-green deployments (a full parallel environment is stood up, validated, and traffic is switched to it all at once).

**Why it exists:** Even an artifact that cleared every staging gate can still fail only under real production conditions — production-scale load, production data shapes, production traffic patterns that staging never fully replicates. Applying Ch 09's blast-radius framework here: shipping straight to 100% of production traffic means a production-only defect affects everyone immediately, while a gradual rollout bounds how many users are exposed before a problem is caught.

**Options:**
1. **Immediate full deployment** — the new artifact replaces the old one across all of production at once.
2. **Canary rollout** — traffic shifts to the new version incrementally (1%, then 5%, then 25%, then 100%), based on live metrics at each step.
3. **Blue-green deployment** — a full second production environment runs the new version in parallel; once it's validated, all traffic switches to it at once, and the old environment stays available for immediate rollback.

**Trade-offs:** Immediate deployment is operationally straightforward but maximizes blast radius — any production-only defect affects every user at once. Canary rollouts minimize initial exposure and run within existing infrastructure without duplicating it, but require real traffic-splitting infrastructure and extend the deployment window to hours instead of a single flip. Blue-green deployment gives near-instant rollback by routing traffic back to a still-running old environment, at the cost of running double the infrastructure and a real risk around data written by the new version during the window before a rollback, which an instant traffic flip doesn't undo.

**When to choose each:** Canary rollouts for horizontally scaled, stateless services where independent nodes running different versions briefly can coexist safely. Blue-green for systems where mismatched concurrent versions would corrupt shared state or break compatibility — a shared transactional database, for instance. Immediate full deployment only for genuinely low-risk internal systems where the cost of a full-fleet failure is negligible.

**Common failure modes:** The long-running orphaned canary — a rollout promotes a new version to a small canary segment (say, 2% of traffic) without pairing it to telemetry that watches that segment specifically. The canary has a slow memory leak that crashes its instances periodically, but because it's only 2% of total traffic, aggregate, fleet-wide error metrics never cross an alerting threshold. The broken canary runs for days, quietly dropping a small slice of production requests, invisible to a monitoring setup built to watch the whole fleet rather than the specific slice actually running the new version.

**Example:** A progressive delivery controller manages a canary rollout through explicit, paused traffic-weight steps — 5% of traffic for ten minutes, then 20% for an hour, then 50%, then 100% — evaluating live error rates and service-level metrics at each pause before proceeding automatically. What those metrics actually are and how they gate the decision to proceed or roll back is Part IX, Ch 73's job; this chapter is only about the mechanism that shifts traffic in stages.

---

### Why Smart Engineers Disagree

The disagreement is about where the authoritative record of what's actually deployed to an environment should live: inside an active pipeline that pushes changes out, or inside a Git repository that a running system continuously pulls from.

One position treats promotion as something an automation runner actively executes: once an artifact clears staging validation, the pipeline directly updates the target environment — pushing a new image reference straight to a production cluster. This keeps the mental model direct and the pipeline log itself is the record of what happened and when, with no extra indirection between "validation passed" and "the new version is live." The opposing position rejects giving an external pipeline runner direct write access to production at all, treating that access itself as a risk. Instead, promotion is a pull request against a repository of environment manifests — changing a version reference in a YAML file — and a separate agent running inside the target environment continuously watches that repository and pulls the new state itself once the change merges.

Both are addressing a real concern. The push model is more direct and faster, with fewer moving parts between a validated artifact and it actually running. The pull model trades that directness for a stronger security boundary — no external CI system ever holds write credentials to production — and a Git history that's the complete, auditable record of every environment's state, not only of the actions a pipeline happened to take. Which one is right tracks an organization's actual constraints: a regulated environment where auditability and credential separation are non-negotiable has real reasons to prefer the pull model; a small team optimizing purely for velocity, with less at stake in a compromised pipeline credential, can reasonably choose the more direct push model instead — provided, either way, that what's actually being promoted is still the same immutable, content-addressed artifact this chapter opened with.
