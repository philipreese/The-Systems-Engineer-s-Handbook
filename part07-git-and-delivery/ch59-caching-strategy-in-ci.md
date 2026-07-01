# Ch 59 — Caching Strategy in CI

**Prerequisites:** [What Belongs in CI (and What Doesn't)](ch57-what-belongs-in-ci-and-what-doesnt.md), [Fail-Fast vs. Fail-Safe Pipeline Design](ch58-fail-fast-vs-fail-safe-pipeline-design.md)

**New vocabulary introduced:** stale-cache masking

**Key takeaways:**
- Caching makes CI fast by reusing work; done wrong, it makes CI *lying* fast instead of actually fast. A cache is only correct if its key encodes exactly the inputs that determine the build's output — nothing broader, nothing narrower.
- [Strong Recommendation] Key dependency and build-artifact caches on a hash of the lockfile — never on a branch name, a date, or anything broader, which either serves stale results across unrelated commits or fails to invalidate when it should.
- Stale-cache masking: an overly broad or incorrectly scoped cache key serves an artifact from a different dependency state than the one actually being tested, producing a pipeline that passes for the wrong reason — a build that would fail from scratch shows green because the cache silently substituted something that no longer matches reality.
- [Strong Recommendation] Maintain a scheduled, cache-bypassing clean build — nightly or pre-release — specifically to catch the drift a warm cache would otherwise hide indefinitely. No amount of careful key design eliminates every source of drift; an unpinned system package or an upstream registry change can still slip past a cache that's keyed correctly on everything it was designed to track.
- Caching is mandatory for CI to stay fast, but it is not part of CI's correctness model. A CI system that fails a clean build is merely broken; one that only passes because of its cache is worse — it's indistinguishable from correctness until the day it fails somewhere that isn't warmed by the same cache.

---

This chapter covers how a CI pipeline reuses work across runs without letting that reuse quietly substitute for correctness. It does not cover what checks belong in the pipeline (Ch 57), matrix build mechanics (Ch 60), or how a project's dependency versions are chosen and updated over time (Ch 63 — this chapter is about build speed given a fixed dependency state; that chapter is about how the state itself changes).

### Cache Key Design: The Correctness Constraint

**What it is:** The rule that a cache key must encode exactly the inputs that determine a build's output — most commonly, a hash of the project's lockfile for dependency and package caches, or the ordering of instructions in a Dockerfile for container layer caches. Anything the key omits that actually affects the output risks serving a stale result; anything it includes that doesn't affect the output only reduces the cache's hit rate without improving correctness.

**Why it exists:** Dependency resolution and package installation are among the most expensive, most repeatable parts of a build — expensive enough that caching them is the single highest-leverage lever for cutting pipeline time, and repeatable enough that a cache is safe as long as it invalidates the instant the underlying dependency state actually changes. A cache keyed on anything broader than that — a branch name, a date — has no way to tell the difference between "dependencies are unchanged" and "dependencies changed, but the cache doesn't know."

**Options:**
1. **Broad identity keys** — a branch name or a coarse timestamp (`deps-main`, `deps-2026-07-01`).
2. **Lockfile-hash keys** — a cryptographic hash of the lockfile (`package-lock.json`, `Cargo.lock`) as the primary cache key.
3. **Container layer ordering** — for containerized builds, structuring the Dockerfile so the dependency-manifest copy and install step come before the volatile application source is copied in, isolating the expensive, stable layer from the cheap, frequently-changing one.

**Trade-offs:** Broad keys are trivial to set up but have no mechanical connection to what actually changed — a dependency upgrade on a branch using a broad key can silently reuse the previous, now-incorrect cache, because nothing about the key changed even though the dependency state did. Lockfile-hash keys invalidate exactly when the dependency graph changes and are shared safely across any branch with an identical lockfile, at the cost of requiring the lockfile itself to be fully committed and accurate. Container layer ordering achieves the same precision for containerized builds without a separate key mechanism — the layer's own content hash is the key — but only if the Dockerfile is actually structured to put volatile content last; get the ordering backward and every layer downstream of the volatile one invalidates on every commit, defeating the cache entirely.

[Strong Recommendation] Lockfile-hash keys for dependency caches, full stop — broad keys should not appear in a production-critical pipeline. For containerized builds, structure the Dockerfile so dependency manifests are copied and installed before application source, so the expensive layer only invalidates when dependencies actually change.

**When to choose each:** Lockfile-hash keying is the standard for essentially any package-manager cache. Container layer ordering is the direct equivalent for anything built as an image — the ordering itself does the job a separate key would do in a non-containerized build.

**Common failure modes:** The overly broad fallback regression — a cache key falls back to a prefix that doesn't include the lockfile hash (for example, `${{ runner.os }}-node-` with no lockfile suffix). A developer removes a dependency and regenerates the lockfile; the pipeline's fallback still matches the broad prefix, so it pulls the old warm cache containing the now-deleted package, tests pass because the stale dependency is still physically present, and the change merges — only for a downstream clean-environment deployment to fail immediately because the dependency was never actually removed from what's shipped. The container equivalent: a Dockerfile copies the entire working directory (`COPY . .`) before installing dependencies. A one-line change to an unrelated front-end file busts the cache for that early layer, and every layer after it — including the expensive dependency install — rebuilds from scratch, turning a ten-second change into a fifteen-minute pipeline run.

**Example:** A correctly configured dependency cache keys on `hashFiles('**/Cargo.lock')`, with a broader `restore-keys` fallback used only for a partial warm start, never as the primary correctness boundary — any change to `Cargo.lock` forces a fresh resolution regardless of what the fallback would have matched. A correctly ordered Dockerfile copies only `package.json` and `package-lock.json` first, runs the install, and copies the rest of the source afterward — so a source-only change reuses the dependency layer untouched, and only a lockfile change invalidates it.

### Stale-Cache Masking and the Scheduled Clean-Build Backstop

**What it is:** Stale-cache masking is the failure mode where a cached artifact is valid under the cache key's own rules but no longer matches what a build from scratch would actually produce — the pipeline passes because the cache served something that looks right, not because the code is actually correct. The mitigation is a scheduled job that bypasses caching entirely, run on a cadence (nightly, or pre-release) independent of any single commit.

**Why it exists:** A cache key can be designed perfectly and still miss sources of drift it was never meant to track — an unpinned system library the CI image happened to already have installed, an upstream package registry serving a slightly different resolved version than it did last week. These aren't cache key bugs; they're the cache correctly doing its job while the world around it quietly changed in a way the key was never built to detect. A clean, cache-bypassed build is the only way to verify that a warm pipeline's "green" actually still means what it's supposed to mean.

**Options:**
1. **Permanent cache reliance** — every pipeline run, on every branch, uses the cache; nothing is ever built from scratch except by manual intervention.
2. **Scheduled clean-build auditing** — pull request pipelines cache aggressively for speed, while a separate, cache-bypassing job runs on a fixed schedule against the trunk to verify the build still succeeds from nothing.

**Trade-offs:** Permanent reliance minimizes compute cost and pipeline complexity, but leaves drift completely undetected until something forces a cache clear — often during an incident, which is the worst possible moment to discover an unrelated build regression that's been hiding for weeks. Scheduled clean-build auditing costs the extra compute of a periodic full rebuild, but catches exactly that kind of drift on its own schedule, before it becomes an emergency.

[Strong Recommendation] Scheduled, cache-bypassing clean builds — nightly for most systems, or gating any pre-release milestone — as a standing practice, not an occasional debugging step. This is the correctness backstop that no amount of cache key precision alone can substitute for.

**Common failure modes:** The hidden toolchain crash — a service's CI image happens to have a system library pre-installed from a prior build phase, and a change starts relying on it without anyone noticing, because the warm cache keeps serving an image that already has it. Weeks later, the underlying CI runner image is refreshed to a clean baseline that no longer includes that library; the next deployment that hits a genuine cache miss fails immediately, and the team has no idea why, because the regression was introduced weeks earlier and only surfaces now that the warm cache masking it is finally gone.

**Example:** A nightly, cron-triggered workflow deliberately omits every caching directive its PR-facing counterpart uses, forcing a full dependency resolution and build from a genuinely clean environment. When this job fails and the day's PR pipelines are all green, that gap is the signal: something the warm cache was quietly papering over has drifted, and it's being caught on a predictable schedule instead of during a production incident.

---

### Why Smart Engineers Disagree

The disagreement is about whether a pipeline's caching layer is purely a speed optimization or a genuine correctness risk that has to be actively managed.

One position treats pipeline runtime as the dominant variable: developers who wait too long for feedback context-switch, and anything that can safely be cached should be, because a well-designed key already prevents the failure modes that matter. The opposing position treats a warm cache as inherently non-deterministic — it can carry forward state that a clean build never would have produced — and holds that the only real guarantee of correctness is a build that started from nothing, so no amount of caching should be trusted without a way to independently verify it.

Both are responding to something real. Cache key design done well genuinely does eliminate most of the risk the second position worries about, and refusing to cache at all trades away real, meaningful speed for a guarantee that's mostly already available more cheaply. But "mostly" is the operative word — a cache can be correctly keyed on everything it was designed to track and still miss the small set of things it wasn't, and that gap is exactly what a scheduled clean build closes. The two positions aren't actually incompatible: aggressive caching on the fast, everyday path and a periodic clean-room audit on a separate, slower cadence gets the speed the first position wants and the guarantee the second position wants, without asking either one to give up what it's actually optimizing for.
