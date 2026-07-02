# Ch 61 — Release Automation

**Prerequisites:** [Versioning and Backward Compatibility](../part02-software-architecture/ch16-versioning-backward-compatibility.md), [Commit Message Conventions](ch51-commit-message-conventions.md) (hard prerequisite, not optional), [Tagging and Release Markers](ch56-tagging-and-release-markers.md), [What Belongs in CI (and What Doesn't)](ch57-what-belongs-in-ci-and-what-doesnt.md)

**New vocabulary introduced:** None — this chapter mechanizes concepts already established (Conventional Commits from Ch 51, tags from Ch 56, version semantics from Ch 16) rather than introducing new ones.

**Key takeaways:**
- Release automation turns a validated, merged change into a published, versioned release without manual editing of the version number, the changelog, or the tag. Every one of those decisions is only as reliable as the structured commit history (Ch 51) feeding it — this is a hard prerequisite, not an optional nicety.
- [Strong Recommendation] Release automation is the correct default once a team's commit discipline is genuinely, consistently in practice. Automating a release process on top of inconsistent, unstructured commit messages doesn't make the process safer — it computes unreliable version bumps faster and with less human oversight than a manual process would have caught.
- Two real automation shapes exist, not one: instant, synchronous publishing the moment code merges to trunk (semantic-release), and deferred staging through a persistent "release PR" that accumulates changes until a human merges it to trigger publication (release-please, Changesets). The second keeps a repository's own manifest files in sync with what's actually published and adds a final human checkpoint, at the cost of the manual gate continuous delivery otherwise removes.
- Publishing credentials are worth naming, not deeply treating, here: OIDC-based trusted publishing (npm, PyPI) is the current direction of travel away from long-lived API tokens stored as CI secrets. The full secrets-management treatment belongs to Part XI, Ch 83.
- This chapter is about how a version number gets produced mechanically, not what it promises about compatibility — that's Ch 16's argument, treated here as a fixed input, not re-derived.

---

This chapter covers turning code that has already passed CI into a published, versioned release without manual, error-prone, human-executed steps. It does not re-specify the commit message grammar this automation depends on (Ch 51 — a hard prerequisite, referenced here, not re-argued), what a version number promises about compatibility (Ch 16), how the published artifact then moves through environments afterward (Ch 62), or the full treatment of secrets management for publishing credentials (Part XI, Ch 83).

### From Commit History to Published Release

**What it is:** A pipeline that parses structured commit history since the last release, determines whether a release is warranted, computes the next version number, generates a changelog, creates the release tag (Ch 56), and publishes the artifact — with none of those steps requiring a human to manually choose or type anything, provided the commit history is structured enough to parse.

**Why it exists:** Manual version selection is repetitive, and repetitive human judgment calls are exactly where inconsistency creeps in — a developer picks a minor bump where a major was warranted, or forgets a changelog entry entirely. If the team already follows Conventional Commits (`feat:` for new functionality, `fix:` for a bug correction, a breaking-change marker for an incompatible change), the information needed to make every one of those decisions correctly already exists in the commit history; automation only executes what the history already implies.

**Options:**
1. **Manual release process** — a human selects the version number, writes the changelog, creates the tag, and publishes by hand.
2. **Instant, synchronous automation** (the semantic-release pattern) — the moment code merges to trunk, the pipeline parses new commits, computes the version, generates the changelog, tags, and publishes, all within the same automated run.
3. **Deferred staging automation** (the release-please or Changesets pattern) — the pipeline maintains a persistent, continuously updated "release pull request" containing the computed version bump and changelog; publication happens only once a human merges that PR.

**Trade-offs:** Manual releases give explicit human oversight at every step but don't scale and are exactly as consistent as the humans performing them, which is to say, not very. Instant automation minimizes the time between a validated change and its release, removing the human bottleneck entirely — but because the version bump happens inside an isolated CI run, a repository's own tracked files (a `package.json` version field, for instance) can drift out of sync with what's actually published unless an extra back-write step is added. Deferred staging keeps those files perfectly synchronized, since the version bump is committed to the repository as part of the release PR itself, and gives a final human checkpoint to review the generated changelog before it goes out — at the direct cost of reintroducing the manual gate that instant automation was designed to remove.

[Strong Recommendation] Automate, in either shape, once commit discipline is actually enforced — ideally mechanically, via a commit-lint check (Ch 51) rather than left to memory. Automating on top of commit messages that aren't reliably structured doesn't add safety; it adds speed to a process that was already computing the wrong answer.

**When to choose each:** Instant, synchronous automation fits services and internal packages with no user-facing manifest file that needs to visibly match the published version, and where minimizing lead time matters most. Deferred staging fits public SDKs, libraries, and multi-package monorepos, where the checked-in version files need to match what ships and where a human reviewing the generated changelog's wording before it reaches consumers is worth the small delay.

**Common failure modes:** The unstructured garbage escalation — a team runs automated version synthesis but doesn't actually enforce Conventional Commits. An engineer merges a change that deletes part of a public API but commits it with an unparseable message like "working on updates." The automation finds no breaking-change token, defaults to a safe-looking patch bump, and publishes it. Every downstream consumer that auto-updates on patch releases pulls the change and immediately hits the removed API, with no warning that anything incompatible had shipped — because the one piece of information the whole pipeline depended on was never actually recorded.

**Example:** Google's release-please runs across a large share of its open-source projects: a commit like `feat(api): support streaming parameters` doesn't publish anything immediately — it updates a standing pull request titled something like `chore: release v1.5.0`, regenerating the changelog and bumping the version files inside that PR. The only manual step left is merging a pull request whose contents were already computed correctly; nobody is deciding the version number from scratch.

A brief note on publishing credentials: a release pipeline has to authenticate to whatever registry it publishes to, and the direction of travel is away from long-lived API tokens stored as CI secrets and toward OIDC-based trusted publishing — both npm and PyPI now support exchanging a short-lived, pipeline-scoped identity for a publish session with no persisted secret at all. This chapter names that direction rather than treating it in depth; the full secrets-management argument — rotation, scoping, what happens when a credential leaks — belongs to Part XI, Ch 83.

---

### Why Smart Engineers Disagree

The disagreement isn't about whether releases should be automated — both sides agree the mechanical steps should be. It's about whether the moment of publication itself should ever happen without a human in the loop.

One position holds that a release should happen automatically the instant code passes every check on trunk: if a change clears linting, type-checking, the full test suite, and carries a properly structured commit history, it's ready, and inserting a human decision point after all of that adds delay without adding real information. The opposing position holds that automated checks can't reason about everything a release might need to account for — coordinated marketing timing, a legal review requirement, a subtle interaction the version-bump logic has no way to see — and that the actual trigger for publishing to the public should stay behind a human review step even if every mechanical calculation leading up to it is automated.

Both are right about different kinds of software. An internal service or a SaaS product with no external version-file contract and cheap rollback has little to lose from instant, synchronous publication — the semantic-release model. A public library or SDK, consumed by parties who can't easily un-pull a bad release, benefits from the deferred-staging model's final checkpoint, where a human reviews already-computed output before it becomes irreversible for someone else. The choice tracks the actual cost of being wrong at the moment of publication, not a general belief about whether automation can be trusted.
