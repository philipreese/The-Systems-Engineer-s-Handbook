# Ch 54 — Force Push: When It's Acceptable

**Prerequisites:** [Decision Frameworks for Trade-offs](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md) (reversibility and blast radius), [Branching Strategies](ch50-branching-strategies.md), [Commit Message Conventions](ch51-commit-message-conventions.md)

**New vocabulary introduced:** None — this chapter is a direct application of Ch 09's reversibility-and-blast-radius framework to one specific Git operation.

**Key takeaways:**
- Force push is two operations sharing one name: rewriting a branch only you depend on (low blast radius, often the healthy, expected thing to do before review) and rewriting a branch others have already pulled or built on (high blast radius, effectively irreversible for anyone whose unpushed work gets silently discarded).
- A branch's "private" status is a property of who is actually building on it, not who created it. The moment a second person pulls from a branch or depends on its state, it has become shared, and force-pushing to it carries the same risk as force-pushing to `main`.
- [Strong Recommendation] `--force-with-lease` should be the default over plain `--force`, even for legitimate private-branch use — it aborts instead of overwriting blindly if the remote has moved since your last fetch, which is nearly always the outcome you actually want.
- [Strong Recommendation] Force-push protection on shared branches should be enforced mechanically through branch protection settings (Principle 8), not through memory or team etiquette. A workflow that depends on everyone remembering not to force-push `main` is already broken by design.
- The question that matters is never "can this branch technically accept a force push" — it's "does anyone besides me have unpushed work or dependent state built on top of what I'm about to overwrite."

---

This chapter is a narrow, specific judgment call: a direct application of Ch 09's reversibility-and-blast-radius framework to one Git operation, not a re-derivation of that framework. It does not cover merge-versus-rebase as a general integration strategy (Ch 55 — force-pushing is simply the mechanical consequence of rebasing your own already-pushed branch, referenced here, not re-argued) or branch lifecycle and deletion (Ch 53).

### Private Branches vs. Shared Branches

**What it is:** The distinction between force-pushing a branch that only you have fetched or built on, versus force-pushing a branch that other people, or automated systems, have already pulled or based work on.

**Why it exists:** A force push does more than add history — it discards whatever currently exists at the remote ref and replaces it. On a branch nobody else depends on, that's a private cleanup operation. On a branch others depend on, it silently invalidates every local copy that assumed the old history was still there — Git's distributed model gives collaborators no automatic notification that their base is gone.

**Options:**
1. **Force push to a private branch** — rewriting your own unpublished or single-owner branch, typically after an interactive rebase to remove WIP commits before review.
2. **Force push to a shared branch** — rewriting a branch that others have already fetched, are building on, or that CI or another automated system depends on.

**Trade-offs:** Force-pushing a private branch is low blast radius and high reversibility: worst case, you recover from your own `reflog`. Force-pushing a shared branch is high blast radius and effectively irreversible for anyone else affected: a collaborator's unpushed local commits, based on the history you discarded, become orphaned from the remote with no warning and no straightforward recovery path for them — only you know the rewrite happened; they find out when their next push or pull fails or silently diverges.

[Strong Recommendation] Force-pushing a private branch to clean up commit structure before review is healthy practice, not a risk to manage — it directly improves the commit discipline Ch 51 argues for. Force-pushing a shared branch is, in essentially every legitimate workflow, not something anyone should be doing manually; recover from a shared-branch mistake by reverting forward, not by rewriting history other people already depend on.

**When to choose each:** Force push freely on a branch you're certain has exactly one consumer — you. Treat a branch as shared the instant a second person pulls from it or an automated system builds on it, regardless of whether that was the original intent; ownership is determined by who actually depends on the branch now, not by who created it or what the plan was when it was created.

**Common failure modes:** The silent collaborator trample — two developers share a feature branch; one rebases and force-pushes to clean up their own earlier commits, not realizing the other has already pulled and built dependent work on top of the old history. The push succeeds, the old commits vanish from the remote, and the second developer's next pull or push breaks in a way that isn't self-explanatory — their unpushed local work is now orphaned, recoverable only through a manual `reflog` search, if it's recoverable at all.

**Example:** GitHub's and GitLab's "Do not allow force pushes" branch protection setting is the mechanical enforcement of this rule (Principle 8) — it rejects a `git push --force` targeting `main` or a `release/*` branch at the server, regardless of who runs the command or whether they meant to. This is the correct implementation: the policy doesn't depend on anyone remembering it.

### `--force` vs. `--force-with-lease`

**What it is:** Two variants of force push with different safety properties. Plain `--force` overwrites the remote ref unconditionally. `--force-with-lease` first checks that the remote ref still matches what your local tracking branch last saw, and aborts the push if it doesn't.

**Why it exists:** A force push is dangerous largely because it's blind — it overwrites whatever is at the remote ref without checking whether that ref has changed since you last looked. `--force-with-lease` adds exactly one guard: if someone else pushed to the branch since your last fetch, the push is rejected instead of silently destroying their commit.

**Trade-offs:**

| Dimension | `--force` | `--force-with-lease` |
|---|---|---|
| Safety | Blind — overwrites regardless of concurrent upstream changes | Checked — aborts if the remote has moved since your last fetch |
| Failure mode on rejection | None — it doesn't reject, it overwrites | Requires a fetch and manual reconciliation before retrying |
| Appropriate default | Scripted recovery in a single-threaded, fully controlled environment | Any interactive, human-run force push |

[Strong Recommendation] `--force-with-lease` should be the default for any manual force push, including on branches you believe are private — "I'm the only one on this branch" is an assumption, and the lease check is what happens when that assumption turns out to be wrong. Reserve plain `--force` for automated, single-threaded pipeline contexts where the remote state is already guaranteed by an orchestration lock, not for anything run from a human's terminal.

**Common failure modes:** A developer rebases a feature branch they believe is private, but a teammate or a bot pushed a small fix to it minutes earlier. A plain `--force` succeeds and silently deletes that intervening commit with no error and no trace beyond the other party's `reflog`. The same sequence with `--force-with-lease` is rejected outright — "stale info," push refused — forcing a fetch, a look at what changed, and a deliberate decision about how to reconcile it, instead of an accidental deletion nobody notices until later.

**Example:** Running `git push --force-with-lease` against a branch a teammate recently pushed to fails immediately with `! [rejected] ... (stale info)`, instead of the silent overwrite plain `--force` would have produced. The fix is to fetch, review what changed, incorporate it, and retry — friction that exists precisely to catch the case a blind force push can't.

---

### Why Smart Engineers Disagree

The disagreement is about where responsibility for history integrity should sit — with individual developer discipline, or with the system itself.

One position holds that force push, including on branches shared during review, should simply never be trusted: once a commit reaches a central server it's part of the permanent record, and any correction — even fixing a typo flagged in review — should be a new commit, not a rewrite. This avoids every failure mode force-pushing can cause, at the cost of a noisier history and, per Ch 52, a history that's harder to justify preserving in the first place. The opposing position treats force push, guarded by `--force-with-lease`, as an ordinary and necessary tool for curating a branch's history before it becomes permanent — refusing to ever rewrite pre-merge history means every noisy, exploratory commit made during development survives forever, which is its own cost.

Both positions are right about different branches. A branch with exactly one consumer can be rewritten freely without the risk the first position is guarding against — there's no collaborator to trample. The same branch, the moment a second developer or an automated system starts depending on it, has crossed into the territory the first position is actually worried about, and at that point the two positions agree completely: don't force-push it. The dividing line isn't a philosophy about history's sanctity — it's a factual question about who else currently depends on the branch, which is exactly what determines blast radius under Ch 09's framework in the first place.
