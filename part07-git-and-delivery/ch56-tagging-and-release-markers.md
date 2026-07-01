# Ch 56 — Tagging and Release Markers

**Prerequisites:** [Versioning and Backward Compatibility](../part02-software-architecture/ch16-versioning-backward-compatibility.md), [Milestone and Phase Planning](../part06-engineering-process/ch44-milestone-and-phase-planning.md), [Branching Strategies](ch50-branching-strategies.md), [Merge vs. Rebase](ch55-merge-vs-rebase.md)

**New vocabulary introduced:** annotated tag, lightweight tag

**Key takeaways:**
- A Git tag is the mechanical artifact that marks a specific commit as a release point. It says nothing about what a version number promises to consumers — that's Ch 16's argument. This chapter is about the tag as artifact, not the meaning of the string it carries.
- [Strong Recommendation] Annotated tags — carrying a tagger, timestamp, message, and optionally a GPG signature — are the correct default for anything called a release. A lightweight tag is a bare pointer with none of that metadata, appropriate only for local, throwaway bookmarks, never a production release marker.
- A signed tag lets a consumer verify a release actually came from an authorized maintainer, at a real but modest key-management cost — worth adopting for anything distributed beyond a single trusted pipeline. The full supply-chain threat model signing defends against belongs to Part XI, Ch 84, not here.
- Semantic Versioning is the scheme a release tag typically encodes (`v1.4.2`), but what major/minor/patch actually promises about compatibility is Ch 16's argument, not this chapter's. Here it's only the string the tag carries.
- Unlike a branch, a tag is meant to stay fixed. Deleting and recreating a published version tag on a different commit defeats the entire reason a tag exists — it silently redefines what a version number pointed to, out from under anyone who already fetched it.

---

This chapter resolves the forward references from Ch 16 and Ch 44: it covers the Git-level artifact that anchors a release, not what a version number promises (Ch 16 — reference it directly, this chapter doesn't re-derive compatibility semantics) or how that tag gets created inside an automated pipeline (Ch 61) or moves through environments afterward (Ch 62).

### Annotated vs. Lightweight Tags

**What it is:** A lightweight tag is a bare, named pointer to a commit SHA — mechanically, a branch that never moves. An annotated tag is a full object in Git's database: it carries the tagger's identity, a timestamp, a message, a reference to the target commit, and optionally a GPG signature.

**Why it exists:** Git provides both because not every tag needs to be a formal artifact — a local bookmark for your own experimentation doesn't need a tagger identity or a message. But a tag meant to represent a release does need those things, because "who created this, and when, and why" is exactly the information anyone auditing a release later needs, and a lightweight tag simply doesn't carry it.

**Options:**
1. **Lightweight tag** — a direct pointer, created instantly, with no metadata.
2. **Annotated tag** — a full object with tagger, timestamp, and message.
3. **Annotated, signed tag** — an annotated tag with a GPG signature added, letting a consumer verify the tag was actually created by an authorized maintainer's key rather than merely claiming to be.

**Trade-offs:** A lightweight tag costs nothing to create and carries nothing beyond a commit reference — no record of who made it, when, or why, and no way to distinguish an intentional release marker from an accidental local bookmark that got pushed by mistake. An annotated tag costs a small amount of extra ceremony (a message, at minimum) in exchange for exactly the accountability a lightweight tag lacks. Signing adds a real GPG key-management cost on top of that, in exchange for a consumer being able to verify provenance rather than trust a plain-text tagger field that anyone with local write access could fake.

[Strong Recommendation] Annotated tags are the default for anything called a release; reserve lightweight tags for genuinely local, throwaway bookmarks that will never be pushed to a shared repository. Add signing for anything consumed outside a single, fully trusted internal pipeline — public releases, open-source distribution, anything where "this tag really came from the maintainer it claims to" matters to someone downstream.

**When to choose each:** Lightweight tags for personal, ephemeral markers during local development. Annotated tags for every production release, public API version, or library version. Signed annotated tags specifically once the artifact leaves a fully trusted, single-pipeline environment — public distribution, regulated environments, or anywhere the tagger field being trustworthy actually matters to a downstream consumer.

**Common failure modes:** A lightweight tag gets pushed and mistaken for an official release, with no metadata available to determine whether it was intentional; later, the same tag name gets silently deleted and recreated on a different commit to sidestep a pipeline gate, and any system that already cached the old reference now deploys an inconsistent, non-deterministic state with no warning that the pointer moved. On the signing side: a team creates annotated tags but never actually verifies the signature downstream, which means the cryptographic guarantee exists on paper but protects nothing in practice, since an unverified signature is no different from no signature at all.

**Example:** The Git project itself uses annotated, signed tags for its own releases — each carries a message describing the release and a GPG signature, letting anyone who downloads the source verify with `git tag -v` that the release actually came from an authorized maintainer before trusting a single line of it.

### Semantic Versioning as the Tag's Encoding

**What it is:** Semantic Versioning (Tom Preston-Werner) is the near-universal scheme a release tag's name typically encodes — `v1.4.2` as major, minor, patch. This chapter covers only that the tag carries this string; what incrementing major versus minor versus patch actually promises about compatibility is Ch 16's argument.

**Why it exists:** Automated tooling — package managers, dependency resolvers, release pipelines (Ch 61) — needs a machine-parseable convention to reason about a release's relationship to prior ones. A tag name that encodes SemVer gives tooling that structure directly, without needing to inspect the release's actual contents.

**Trade-offs:** SemVer-encoded tags are immediately machine-interpretable, at the cost of forcing a real compatibility judgment (Ch 16's job) every time a tag is created. Calendar- or arbitrary-named tags (`release-apollo`, `2026-07-01`) are readable to humans tracking a release by date or codename, but tell a machine nothing about whether one release is compatible with, or even newer than, another — a hotfix tagged with a later timestamp than a since-reverted release can sort ahead of it alphanumerically despite representing an older logical state.

**When to choose each:** SemVer for anything with programmatic consumers — libraries, public APIs, anything resolved by a package manager. Calendar or arbitrary naming only for single-deployable, internally consumed systems with no dependency graph to reason about.

**Common failure modes:** A tag's version number is bumped as a formality without reflecting what actually changed — a breaking change tagged as a patch release, or vice versa — decoupling the tag's meaning from Ch 16's compatibility promise and making the tag cosmetic rather than a contract automated tooling can trust.

**Example:** A tag `v2.3.1-rc.1` is parsed by release tooling into major (2), minor (3), patch (1), and pre-release (`rc.1`) fields — fields whose actual compatibility meaning Ch 16 already defines, and which this chapter only treats as a string the tag object carries.

---

### Why Smart Engineers Disagree

The disagreement is about where the authoritative record of "what got released" should actually live: inside Git's own tag graph, or inside the artifact registry — the container registry, package index, or binary store — that the deployed artifact actually comes from.

One position holds that once code passes through CI, the source commit isn't the operational asset anymore — the compiled container image or published package is what actually runs in production, identified by its own content-addressable digest. From this view, a Git tag is a redundant, decorative pointer that can drift out of sync with what was actually published if a build fails partway through, and the registry's digest is the only thing that should be trusted as the release record. The opposing position treats the registry as a volatile, replaceable adapter — subject to outages, vendor lock-in, or index corruption — that carries no information about what commits, reviews, or issues produced the artifact it's serving. From this view, an isolated digest is close to meaningless without a trail back to the source state it came from, and the Git tag is what actually anchors a shipped artifact to the human history that produced it.

Both are right about a real risk. A registry digest is what's actually running, and treating a Git tag as sufficient on its own ignores that a tag can exist without a corresponding successful publish. A Git tag is what carries traceability back to source, and treating a registry digest as sufficient on its own discards exactly the audit trail a tag provides. The practical resolution most systems converge on is to have both, deliberately linked: the release pipeline (Ch 61) creates the tag as part of the same process that produces and publishes the artifact, so neither one exists without the other, and neither is asked to serve as sole source of truth for something it wasn't designed to fully capture alone.
