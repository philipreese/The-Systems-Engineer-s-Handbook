# Ch 55 — Merge vs. Rebase

**Prerequisites:** [Decision Frameworks for Trade-offs](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md) (reversibility and blast radius), [Branching Strategies](ch50-branching-strategies.md), [Squash vs. Preserve History](ch52-squash-vs-preserve-history.md), [Force Push: When It's Acceptable](ch54-force-push-when-its-acceptable.md)

**New vocabulary introduced:** None — the golden rule of rebasing is a widely used industry term this chapter applies, not a handbook-coined concept.

**Key takeaways:**
- Merge and rebase aren't competing correctness mechanisms — they produce different history shapes from the same correct code. The decision that actually matters is the golden rule of rebasing; the choice between a merge commit and a linear history at integration time matters far less.
- [Consensus] Never rebase a branch that other people have already pulled or built work on top of. Rebase rewrites commit identity, not only content — every replayed commit gets a new SHA — so rebasing shared history invalidates every downstream clone's understanding of where that history is, and requires exactly the force push Ch 54 already warns against.
- Rebasing a private, unpublished branch to clean up before review is safe and recommended, precisely because nobody else's history depends on the commits being rewritten.
- [Legitimate Trade-off] Whether a finished branch's integration produces an explicit merge commit or a linearized history matters less than either side's advocates claim, as long as a team is consistent about it — and squash-merge (Ch 52) sidesteps the debate entirely for most PR-based workflows by collapsing the question to one commit either way.
- The deeper disagreement isn't about which integration style is correct — it's about whether Git history should function as an audit log of what actually happened or a reasoning tool optimized for whoever reads it next. Both are legitimate, and which one applies depends on how parallel the actual development really is.

---

This chapter covers the two mechanisms for integrating one branch's changes into another and the history shape each produces. It does not re-litigate branch topology (Ch 50), commit message format (Ch 51), or the specific server-side force-push protections that block a rewritten shared branch (Ch 54) — this chapter references that framework, applying it to one specific operation, rather than re-deriving it.

### The Golden Rule of Rebasing: Private vs. Shared Branches

**What it is:** Rebase takes a sequence of commits and replays them on top of a new base, giving each one a new parent and therefore a new SHA — it rewrites commit identity, not only content. The golden rule, as stated in Atlassian's canonical Git tutorial: never rebase a branch that other people have already pulled or built work on top of.

**Why it exists:** Because a rebased commit is a different object from the one it replaced — same content, different identity — anyone who already has the old commit is now working from history that no longer exists anywhere else. This is the exact reversibility judgment Ch 54 already applies to force push, because publishing a rebase requires exactly the force push that chapter warns against: rebasing shared history and force-pushing it are the same action described from two different angles.

**Options:**
1. **Rebase only private, unpublished branches** — the standard, safe default for cleaning up commit structure before opening a PR.
2. **Rebase a shared branch with explicit, coordinated synchronization** — rare, and only defensible in tightly controlled environments where every affected collaborator is stopped and resynchronized deliberately.
3. **Never rebase anything shared** — the simpler, conservative default for teams that don't need the coordination overhead of option 2.

**Trade-offs:** Rebasing a private branch removes catch-up noise (repeated "merge main into feature" commits) at zero coordination cost — nobody else's history is affected. Rebasing a shared branch removes the same noise but at a cost that scales with every downstream branch or clone: each one now has commits that no longer exist upstream, and reconciling that is manual, error-prone work that falls on people who didn't make the decision to rebase.

**When to choose each:** Rebase freely and often on a branch only you depend on. Once a branch has a second consumer — another developer's clone, or an automated system tracking specific commit hashes — treat it as shared and stop rebasing it, exactly as Ch 54 already argues for force push, because rebasing shared history is the same operation under a different name.

**Common failure modes:** An engineer rebases a branch that collaborators are already using as their own baseline, then force-pushes the result. When those collaborators next pull, Git detects divergent histories; reconciliation routinely produces duplicate commits, silently revives code that was deliberately deleted, and can drop unpushed local work entirely — hours of manual workspace reconstruction, scaling with how many branches were built on top of the one that got rewritten. A subtler cost: rebasing changes the commit's SHA, which invalidates any signed-commit verification or code-review platform state (approvals, comment threads) that was tied to the original hash.

**Example:** Atlassian's Git documentation states the rule directly — never use `git rebase` on a public branch — as a direct reflection of Ch 09's reversibility-and-blast-radius framework: rebasing a private branch has a small blast radius and is fully recoverable from `reflog`; rebasing a shared branch has a team-wide blast radius and is, for anyone who already pulled the old history, effectively irreversible.

### Merge Commit vs. Linear History at Integration Time

**What it is:** Once a branch is finished and ready to integrate, whether that integration produces an explicit merge commit (`git merge --no-ff`, preserving the fact that two lines of development diverged and converged) or a linearized history (rebase, then fast-forward the target branch, `--ff-only`, leaving no trace that a branch existed at all).

**Why it exists:** This decides whether the target branch's log records real development topology — where work split off and rejoined — or reads as a flat, purely sequential timeline. Both are internally consistent choices about what the permanent record is optimized for.

**Trade-offs:**

| Dimension | Explicit merge commit (`--no-ff`) | Linear history (rebase + `--ff-only`) |
|---|---|---|
| Topological accuracy | High — records exactly when and where parallel work occurred | Low — presents a sequential history that wasn't actually sequential |
| Feature-level revert | Direct — one merge commit revert undoes an entire feature | Harder — requires identifying and reverting an ordered chain of individual patches |
| Log scannability | Lower — a high-velocity team produces a web of crossing lines | Higher — the log reads as one straightforward timeline |
| `git bisect` behavior | Can step across a merge as one block | Steps through every individual commit |

[Legitimate Trade-off] Neither side is wrong, and the choice matters less than either side's strongest advocates claim — as long as a team applies it consistently. Squash-and-merge (Ch 52) sidesteps this entire question for most PR-based workflows, since it produces one commit either way and there's no merge-commit noise or linearization decision left to make.

**When to choose each:** Choose explicit merge commits for systems where topology and feature-level revertibility matter — large, multi-team codebases, or long-lived release lines where reverting an entire feature by one commit is genuinely valuable. Choose linear history for smaller, high-discipline teams that value a scannable, uncluttered log more than an audit trail of exactly when parallel work happened. Most teams merging through pull requests should sidestep the question entirely and squash (Ch 52).

**Common failure modes:** The interleaved regression maze — a team enforces strict linear rebase-and-fast-forward integration without squashing. Several developers' branches land on the trunk over a few days, fully interleaved with no structural markers separating one feature's commits from another's. A regression surfaces; because there's no merge commit framing any single feature, reverting it means identifying and reverting a dozen disconnected patches scattered across other people's unrelated work, rather than one clean revert.

**Example:** Linus Torvalds has publicly and repeatedly defended merge commits in Linux kernel development, criticizing enforced-linear, rebase-only workflows that strip them out. His argument: a clean linear history is a cosmetic illusion that erases real information about when an integration happened and what state it was validated against — the kernel's history isn't linear because kernel development genuinely isn't linear, and pretending otherwise doesn't make it simpler, only less honest.

---

### Why Smart Engineers Disagree

The disagreement isn't really about correctness — both produce working code — it's about what a repository's history is for.

One view treats history as a record of what actually happened: if two developers worked concurrently for weeks on separate modules, the log should show two parallel tracks converging at a clear point, because flattening that into a straight line implies a false sequential relationship — that one developer's work depended on or followed the other's, when in fact neither knew what the other was doing until they converged. The opposing view treats the log as a document written for whoever reads it next, not a diary of how it was produced: a maintainer debugging a regression doesn't benefit from seeing the exact concurrency of how the code was written, and a graph cluttered with crossing merge lines adds cognitive load without adding anything actionable.

Both views are correct for different systems. Heavy, genuinely parallel development — a kernel, a large multi-team monorepo — has real information encoded in its topology, and preserving it is worth the graph complexity. Smaller, more sequential product development usually has little of that information to lose, and a linear history is the more useful artifact. Neither view is about to be settled by more argument, because they're optimizing for different things: an audit log of what happened, versus a reasoning tool for what to do next.
