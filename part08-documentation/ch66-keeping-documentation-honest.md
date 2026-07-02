# Ch 66 — Keeping Documentation Honest

**Prerequisites:** [Comments: What to Comment, What Not To](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md) (comment rot), [Issue Tracking: What Makes a Good Issue](../part06-engineering-process/ch42-issue-tracking-what-makes-a-good-issue.md) (write-only tracker), [Process Overhead: The Value Threshold](../part06-engineering-process/ch49-process-overhead-the-value-threshold.md) (audit discipline), [README vs. Spec vs. ADR vs. Inline Comment](ch65-readme-vs-spec-vs-adr-vs-inline-comment.md) (living document vs. point-in-time record)

**New vocabulary introduced:** executable documentation

**Key takeaways:**
- This chapter applies only to living documents (Ch 65) — a README, a guide, anything expected to always describe the system as it is today. A stale spec or ADR is a different, expected condition covered by Ch 65, not a failure this chapter addresses.
- [Principle 8] Storing documentation in the same repository and pull request review as the code it describes doesn't guarantee it stays accurate, but it's the only structural change that gives a behavior change and its documentation update a chance to land in the same review.
- Split the verification problem: anything objectively checkable — a code example that compiles, a link that resolves — should be automated into a build failure. Everything else — is this still the right explanation, does this still reflect current practice — stays Ch 49's periodic audit responsibility, not something left to be noticed by accident.
- A documentation corpus is exactly as vulnerable to becoming a write-only tracker (Ch 42) as an issue backlog: writing feels productive, pruning feels risky and gets no credit, and an untrusted corpus poisons even its accurate pages, because nothing distinguishes them from the stale ones without checking.
- Deleting a page that's wrong or no longer relevant is a legitimate, healthy maintenance action — not a loss to be avoided.

---

[Ch 30](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md) established comment rot: a stale comment is worse than no comment because it actively misleads a reader with no signal that it's wrong. This chapter applies the same failure at the scale of an entire documentation corpus, and the failure is worse here, not merely bigger. A stale comment sits next to the code it misrepresents, giving an attentive reader some chance of catching the mismatch. A stale README or wiki page can be several clicks removed from the code it describes, with nothing structurally forcing a reader — or the engineer changing the underlying behavior — to notice the two have diverged at all. Everything below applies only to documentation that is supposed to track current reality; Ch 65 already established that a stale spec or ADR is expected, not a failure this chapter is answering.

### Decision: Keep Documentation Where the Code's Own Review Can See It

**What it is:** Storing documentation as close as possible to the implementation it describes — ideally the same repository, reviewed through the same pull request flow ("docs as code") — rather than in a separately maintained system.

**Why it exists:** This is Principle 8 applied to prose: mechanical enforcement beats human discipline. The greatest cause of drift is organizational separation — when documentation lives in a different system from the code, an engineer can merge a behavioral change without the documentation platform ever entering their workflow. Co-location doesn't force an update; it gives one a structural chance to happen in the same review, which a wiki never offers.

**Options:**
1. **Same repository** — markdown files and docstrings reviewed alongside the code changes that motivate them.
2. **Separate documentation repository** — useful when documentation genuinely spans many independent codebases.
3. **Separate wiki or knowledge base** — no structural relationship to any repository's review flow.

**Trade-offs:** Same-repository documentation shares version history with the code and surfaces naturally in the pull requests that touch it, at the cost of a larger repository and a barrier for contributors who aren't comfortable with the source tree. A separate documentation repository serves multiple codebases from one place but makes divergence easier and requires explicit coordination to keep in sync. A separate wiki is the easiest to edit and the most accessible to non-engineers, but it has no structural connection to the code review that changes the system it describes.

[Strong Recommendation] Same-repository, docs-as-code for anything that describes a specific codebase's current behavior. Reserve separate systems for content that is genuinely independent of any one repository's execution path — compliance policy, non-technical process documents — not for onboarding guides or architecture overviews that happen to feel easier to edit elsewhere.

**Common failure modes:** *The ghost PR.* A team migrates a system's folder layout to a new infrastructure provider and carefully updates code and configuration in one pull request. Nothing checks whether the README's setup instructions still match, so nobody touches them. The PR merges; the next engineer to clone the repository spends their first morning running setup steps for an architecture that no longer exists.

**Example:** GitHub and GitLab both maintain the bulk of their own engineering and product documentation as markdown inside their source repositories, reviewed through the same pull request process as their code, rather than as a separately maintained publishing platform.

---

### Decision: Automate What Can Be Verified, Audit What Can't

**What it is:** Splitting documentation correctness into what a build can check mechanically and what remains a human judgment call, and applying the right mechanism to each.

**Why it exists:** Some documentation content is objectively testable — a code example either compiles or it doesn't, a link either resolves or it doesn't. Other content — is this explanation still the best one, does this guide still reflect the recommended approach — has no compiler that can check it. Treating both as the same problem either wastes automation on judgment calls it can't make, or leaves objectively broken content to rot because nobody happened to notice.

**Options:**
1. **Executable documentation** — code samples embedded in docs that are actually compiled and run in CI, link checkers, schema validators against real API definitions (Ch 67). Converts a specific class of drift into a build failure instead of a silent one.
2. **Periodic human audit** — Ch 49's audit discipline, applied directly: documentation is reviewed on a cadence, and a page that's wrong or no longer relevant is deleted or rewritten, not left in place out of inertia.
3. **No verification** — drift accumulates until someone notices by accident, usually during an incident or onboarding.

**Trade-offs:** Executable documentation gives immediate, consistent feedback, but only for content that's objectively checkable — it can confirm a code sample still compiles and has no way to know whether the surrounding prose still describes the recommended approach. Human audit catches conceptual drift no tool can see, at the cost of depending entirely on someone actually doing it on schedule. No verification is free until the first reader trusts something wrong.

[Consensus] Automate everything that's objectively checkable; accept that explanatory accuracy is permanently a human responsibility, and give it the same audit rhythm Ch 49 already established for other artifacts rather than leaving it to whoever happens to notice.

**Common failure modes:** A documentation page passes every automated check — its one code sample compiles, its links resolve — while the surrounding explanation describes a deployment workflow the team abandoned six months ago. Green CI creates false confidence about a page that mechanical verification was never capable of evaluating in the first place.

**Example:** Rust's `rustdoc` doctest feature compiles and executes runnable code examples embedded in documentation comments as part of the normal `cargo test` run. When a function's signature changes and an example is never updated to match, the build fails immediately instead of quietly shipping a documentation page that no longer compiles.

---

### Decision: Prune the Corpus Before It Becomes a Write-Only Tracker

**What it is:** Applying [Ch 42](../part06-engineering-process/ch42-issue-tracking-what-makes-a-good-issue.md)'s write-only tracker failure mode — a collection that grows because insertion is cheap and pruning is thankless, until nobody trusts it enough to consult it — directly to a documentation corpus.

**Why it exists:** Writing documentation feels productive; deleting it feels risky and earns no recognition. Left unchecked, a corpus grows while its average accuracy declines, until engineers stop consulting it — at which point even the pages that are still accurate lose their value, because nothing distinguishes them from the stale ones without independently checking.

**Options:**
1. **Active curation** — update pages that remain valuable, and delete or archive pages that don't, on a real cadence.
2. **Append-only accumulation** — write new documentation continuously and rarely remove anything.

**Trade-offs:** Active curation costs regular pruning effort but keeps the corpus small enough that a reader can trust what's there. Append-only accumulation costs nothing up front, but trust declines steadily as contradictory and obsolete pages accumulate alongside current ones, until search results are dominated by material nobody can distinguish from what's actually correct.

[Strong Recommendation] Treat deletion as a normal, healthy maintenance action for a living document, exactly as Ch 49 already treats it for other process artifacts — not a loss to be justified, but the default response to a page that's wrong or no longer relevant.

**Common failure modes:** An internal wiki accumulates years of onboarding guides, architecture diagrams, and deployment instructions — a "Confluence graveyard." Some pages are still accurate, but engineers can no longer tell which without checking the code directly, so they stop consulting the wiki altogether and default to asking a colleague instead — which defeats every page in the corpus, including the correct ones.

**Example:** Large internal wikis reach this state routinely: incremental additions over years produce a corpus where every search returns a mix of current and obsolete pages with no reliable way to tell them apart, and engineers begin treating the wiki as a place to start guessing rather than a source they trust outright.

---

### Why Smart Engineers Disagree

The friction is about what to do with a living document once part of it is known to be wrong: delete it immediately, or preserve it with a warning.

One position — call it scorched-earth — holds that a page containing any known-stale detail should be deleted outright. Wrong documentation is worse than no documentation, because missing documentation forces a reader back to the source, while wrong documentation actively misleads someone who has no reason to doubt it. A stale banner doesn't fix this — most readers trust the content and skip the warning.

The opposing position — call it preservationist — argues that deleting a page because part of it aged out throws away real context: the structural shape of an old design, the constraints that no longer apply but explain why something looks the way it does. Better to mark it clearly stale, or move it to an archive, than lose that information outright.

Both positions are correct — about different categories, and the disagreement dissolves once that's made explicit. Scorched-earth is right for the subject of this chapter: a living document that's wrong is actively misleading a reader who has no signal to distrust it, and fixing or deleting it is the only response that removes the danger. Preservationist reasoning is also right — but it's the argument Ch 65 already made for specs and ADRs, artifacts that were never supposed to describe current reality and stay correct precisely by not changing. Applying preservationist reasoning to a stale README, or scorched-earth reasoning to a two-year-old ADR, is the same taxonomy conflation Ch 65 warned against, one level down: the question isn't which philosophy is right, it's which category of document you're actually holding.
