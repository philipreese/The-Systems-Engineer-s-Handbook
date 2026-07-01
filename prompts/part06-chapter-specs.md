# Part VI — Chapter Specifications

Pre-filled special instructions for each Part VI chapter. When generating a draft, copy the
block for the chapter you're writing into the `[SPECIAL INSTRUCTIONS]` field of the
chapter-generation prompt.

The instructions do three things:
1. State the chapter's specific role in the handbook's argument
2. Name the key sources, systems, or concepts to anchor the content
3. Explicitly mark what NOT to cover — because those topics belong to another chapter

Prerequisites for Part VI: Part I (especially Ch 01: explicit optimization targets, Ch 02:
complexity, Ch 09: decision frameworks and the reversibility/blast-radius quadrant). This
Part is where several handbook design principles get their process-level expression:
Principle 6 ("Process only when it adds more than it costs") receives its full treatment in
Ch 49, and Principle 9 ("A proxy metric, once made a mandatory target, stops measuring what
it was designed to measure") bears directly on Ch 47 and Ch 49.

Two forward references from earlier parts must be honored here:
- Part I Ch 09 introduced ADRs as a concept and explicitly deferred "the specific format
  and lifecycle of ADRs" to Ch 45.
- Part IV Ch 30 established that repository-level design rationale — the reasoning behind
  large-scale decisions — belongs in ADRs, not inline comments, and deferred the full
  treatment to Ch 45.

Boundary with adjacent Parts: Part VII owns everything about git mechanics — branching,
commit messages, PR merge strategies, CI. Part VIII owns documentation as a discipline —
what to document, README vs. spec vs. ADR taxonomy, API docs. Part VI chapters that touch
those areas (Ch 43 on PRs, Ch 45 on ADRs, Ch 46 on specs) must stay on the process side of
the line: why the artifact exists and how teams use it, not the mechanics of the tool or
the writing of the document.

---

## Ch 42 — Issue Tracking: What Makes a Good Issue

```
This chapter opens Part VI by establishing the smallest unit of process: the issue. The
rest of the Part builds upward from it — issues map to PRs (Ch 43), group into milestones
(Ch 44), and the whole apparatus must justify its overhead (Ch 49). Get the unit right here
so the aggregation chapters don't have to relitigate it.

Cover what makes an issue actionable: enough context that someone other than the author
could pick it up — the problem being solved (not the solution assumed), acceptance criteria
that make "done" checkable, and for bugs specifically: reproduction steps, expected
behavior, actual behavior, and environment. An issue that says "fix the login page" is a
reminder, not an issue.

Cover issue scope: one issue should represent one unit of deliverable, verifiable work. An
issue that cannot be closed by a bounded amount of work — "improve performance," "clean up
the codebase" — is a theme, not an issue, and will sit open forever. Cover the distinction
between issue types (bug, feature, task/chore) and why the distinction matters: they have
different definitions of done and different information requirements.

Cover the lifecycle honestly: issues are cheap to create and expensive to maintain as a
collection. The failure mode is the write-only tracker — a backlog of hundreds of stale
issues that no one will ever do, which makes the tracker useless as a signal of what the
team actually intends. Take a position: closing an issue as "won't do" is a legitimate and
healthy outcome, and a periodically pruned backlog is more valuable than a complete one.
This is the same asymmetry as Ch 39's test deletion argument — removal is a legitimate
engineering decision.

Anchor in real systems: Joel Spolsky's "Painless Bug Tracking" (2000) as the canonical
statement of what a bug report needs (repro steps, expected, observed); GitHub Issues'
deliberately minimal model vs. Jira's configurable-workflow model as the two poles of
tracker philosophy; issue templates (GitHub's ISSUE_TEMPLATE mechanism) as mechanical
enforcement of issue quality — Principle 8 applied to process; Linear's opinionated
stance that backlogs should decay (auto-archiving stale issues) as a real tool taking a
position on backlog pruning.

Do NOT cover: how issues relate to PRs and review units (Ch 43 — that is the next
chapter's entire job); grouping issues into milestones and planning (Ch 44); writing
specs for larger work (Ch 46 — an issue references a spec, it does not contain one);
whether the tracking process itself is worth its overhead (Ch 49).
```

---

## Ch 43 — Issue as Tracking Unit vs. PR as Review Unit

```
This chapter draws a distinction most teams never make explicit and then suffer for: the
issue is the unit of intent (why this work exists and what outcome closes it), and the PR
is the unit of review (a bounded diff a human can actually evaluate). They are different
units with different natural sizes, and the mapping between them is one-to-many, not
one-to-one.

Cover the two units precisely: an issue describes a problem and its acceptance criteria;
it is closed when the outcome is achieved. A PR describes an implementation step; it is
merged when the code is correct and reviewable. Forcing them to coincide produces one of
two failure modes: monster PRs that close an entire issue in one unreviewable diff, or
fragmented issues sliced to match whatever diff was convenient, destroying the tracker's
ability to represent actual intent.

Cover the empirical case for small PRs: review quality degrades sharply with diff size —
reviewers approve large changes with less scrutiny per line, not more. Cover the practical
techniques for keeping the mapping one-to-many: incremental PRs that each leave the system
working, feature flags to merge incomplete-but-inert work, and stacked PRs for sequences
of dependent changes.

Cover the linkage discipline: every PR should reference the issue that motivated it
(GitHub's closing keywords — "fixes #123" — as the mechanical expression), and an issue
with no linked activity is a signal worth noticing. The failure mode to name: the PR-only
workflow, where intent lives nowhere — the diff explains how but nothing records why, and
six months later the only archaeology available is the commit history.

Anchor in real systems: Google's published code review research ("Modern Code Review: A
Case Study at Google," Sadowski et al., 2018) documenting median change sizes far smaller
than industry norms and fast review turnaround as a deliberate, enforced practice; the
SmartBear/Cisco review study's finding that defect detection collapses beyond roughly
200–400 lines per review; GitHub's issue-linking keywords as the mechanical linkage;
stacked-PR tooling (Graphite, `git rebase` chains) as the ecosystem response to the
one-issue-many-PRs reality.

Do NOT cover: what makes the issue itself good (Ch 42); the practice of reviewing — what
reviewers should look for and what standard to apply (Ch 47); branching strategies and
merge mechanics (Part VII, Ch 50, Ch 55); commit message conventions (Part VII, Ch 51).
```

---

## Ch 44 — Milestone and Phase Planning

```
This chapter is about grouping work into shippable increments. Its central argument: the
milestone is a coordination and risk-reduction tool, not a deadline ritual. A milestone
earns its existence by forcing integration, surfacing hidden dependencies, and creating a
point where reality can correct the plan.

Cover the fundamental orientation choice: vertical slices vs. horizontal phases. A
horizontal phase plan ("first the data layer, then the API, then the UI") defers all
integration risk to the end, where it is most expensive — nothing works until everything
works. A vertical slice ("one complete user journey, thin, end to end") proves the
architecture early and de-risks everything after it. Take a position: the first milestone
of any new system should be a walking skeleton — the thinnest possible end-to-end slice
that exercises every layer — and subsequent milestones should thicken it.

Cover the fixed-time vs. fixed-scope trade-off explicitly: a milestone can fix the date
and flex the scope, or fix the scope and flex the date, but not both. Take a position:
fixed time with variable scope is the default that keeps plans honest, because scope is
negotiable in fine increments and dates are not. Cover risk-first ordering within a
milestone: do the parts most likely to invalidate the plan first, not the parts that are
most pleasant to build.

Cover the planning failure modes by name: the 90%-done syndrome (progress reported as
percentage of tasks complete rather than working functionality delivered); big-bang
integration (horizontal phases converging in a final "integration phase" that consumes
the schedule); and the planning fallacy itself — estimates systematically compress
because they imagine the best case.

Anchor in real systems: the walking skeleton (Alistair Cockburn) and tracer bullet
(Hunt & Thomas, The Pragmatic Programmer) as the canonical names for the vertical-slice
first milestone; Basecamp's Shape Up (fixed six-week appetite, variable scope, circuit
breaker) as a complete published system built on fixed-time/variable-scope; Hofstadter's
Law as the honest statement of the planning fallacy; the documented history of big-bang
integration failures in large waterfall projects as the horizontal-phase cautionary tale.

Do NOT cover: what makes individual issues good (Ch 42); release mechanics, tagging, and
deployment (Part VII, Ch 56, Ch 61, Ch 62 — a milestone is a planning unit, not a release
artifact); spec writing for the work being planned (Ch 46); whether planning process
overhead is justified for a given team size (Ch 49).
```

---

## Ch 45 — Architecture Decision Records (ADRs)

```
This chapter carries two explicit forward references and must honor both: Part I Ch 09
introduced the ADR concept — what one is, and that hard-to-reverse decisions deserve one —
and deferred the format and lifecycle here. Part IV Ch 30 ruled that repository-level
design rationale belongs in ADRs rather than inline comments and likewise deferred here.
This chapter owns the format, the lifecycle, and the judgment of when an ADR is warranted.
Do not re-derive the ADR's justification from scratch — reference Ch 09 and build on it.

Cover the canonical format (Michael Nygard, "Documenting Architecture Decisions," 2011):
Title, Status, Context, Decision, Consequences. Emphasize the two sections that do the
real work: Context captures the constraints that were true at decision time — the section
that lets a future reader judge whether the decision still applies — and Consequences
records what was accepted as a cost, including the alternatives rejected and why. Per
Ch 09, the rejected-alternatives record is what prevents the decision from being
relitigated by someone who encounters the same options without the history.

Cover the lifecycle precisely: proposed → accepted → deprecated or superseded. Take a
position: accepted ADRs are immutable. When a decision changes, a new ADR supersedes the
old one and links back; the old record remains as history. Editing an accepted ADR in
place destroys exactly the thing the ADR exists to preserve — the record of what was
believed and when.

Cover the judgment of when an ADR is warranted, anchored to Ch 09's reversibility and
blast-radius quadrant: hard-to-reverse decisions with wide blast radius always warrant
one; cheap-to-reverse local decisions never do. Cover storage: in the repository, next to
the code the decisions govern, versioned with it — not in a wiki that drifts.

Cover the failure modes: the ADR graveyard (records written to satisfy process and never
read again — check whether ADRs are actually cited in reviews and onboarding); retroactive
ADR theater (documenting a decision after the fact with sanitized context, which records
the outcome but launders the reasoning); and ADR inflation (recording trivial decisions
until the signal drowns — if everything gets an ADR, nothing does).

Anchor in real systems: Nygard's original 2011 blog post as the canonical source and
format; adr-tools and MADR as the tooling/template ecosystem that grew from it;
ThoughtWorks Technology Radar's "Lightweight Architecture Decision Records" adoption as
the industry-mainstreaming moment; the convention of a `docs/adr/` or `docs/decisions/`
directory of numbered records as the standard in-repo layout.

Do NOT cover: the decision-making frameworks themselves — reversibility, blast radius,
Cynefin (Part I, Ch 09 owns those; this chapter is about recording decisions, not making
them); the broader documentation taxonomy of README vs. spec vs. ADR (Part VIII, Ch 65);
inline code comments (Part IV, Ch 30); spec documents for proposed work (Ch 46 — a spec
proposes and explores, an ADR records what was decided).
```

---

## Ch 46 — Spec-First Development

```
This chapter argues for writing the specification before the implementation — not as
bureaucracy, but as the cheapest possible review layer. The core economic claim: a flaw
found in a one-page spec costs minutes to fix; the same flaw found in a merged
implementation costs days. Spec review is defect detection moved to where defects are
cheapest.

Cover what a spec is for: surfacing disagreement before code exists. A spec forces the
author to state the problem, the constraints, the proposed approach, and the alternatives
considered — and forces reviewers to object now rather than at PR time, when objections
are expensive and demoralizing. Cover what belongs in a spec: problem statement,
constraints and non-goals, proposed design at the level of interfaces and data flow (not
implementation detail), alternatives considered, and open questions stated as open
questions. Non-goals deserve emphasis: an explicit "this does not attempt X" prevents
scope creep more effectively than any later review comment.

Cover proportionality explicitly, connecting to Principle 6: the spec's weight must match
the decision's weight. A one-way-door architectural change deserves a full design document
and review cycle; a bounded feature deserves a paragraph in the issue; a bug fix deserves
nothing. A team that requires heavyweight specs for everything gets spec theater; a team
that requires none gets design review at the most expensive possible moment.

Cover the failure modes: spec theater (specs written after the implementation to satisfy
a process checkbox — the review happened, but on fiction); the frozen spec (the document
diverges from what was actually built and becomes actively misleading — state clearly
that a spec is a proposal artifact, not living documentation, and what survives it is the
code, the tests, and the ADR); and analysis paralysis (the spec cycle used to defer
commitment indefinitely — connect to Ch 09's decision-quadrant guidance that reversible
decisions deserve speed, not study).

Anchor in real systems: Amazon's working-backwards PR/FAQ and six-page narrative memos as
the strongest-form industrial practice of writing the artifact before the work; Google
design docs as the published mainstream practice (problem, goals/non-goals, design,
alternatives); the RFC lineage — IETF RFCs, Python PEPs, Rust RFCs — as the open-source
expression of spec-first for changes with wide blast radius, including the documented
cost: RFC processes are slow by design, which is correct for language-level one-way doors
and wrong for reversible changes.

Do NOT cover: recording the decision after it is made (Ch 45 — the spec proposes, the ADR
records; a spec's "alternatives considered" section often becomes the ADR's raw material,
and that handoff can be mentioned but not expanded); the documentation taxonomy and where
specs live long-term (Part VIII, Ch 65); issue writing (Ch 42); API reference
documentation (Part VIII, Ch 67).
```

---

## Ch 47 — Code Review

```
This is the most practiced and least examined process in software engineering. The
chapter's job is to state honestly what review is for — because the answer determines
every policy decision about it — and then take positions on standard, size, latency, and
scope.

Cover the purposes ranked honestly: the research says review's durable value is code
health and knowledge transfer, with defect detection real but weaker than the process's
reputation suggests. Most defects are caught by tests and types; what review uniquely
catches is design problems, unclear code, and missing context — and what it uniquely
builds is shared ownership. A team that believes review exists primarily to catch bugs
will tolerate slow, adversarial review; a team that understands it as a code-health and
knowledge mechanism will optimize for throughput and teaching.

Cover the review standard, and take Google's published position explicitly: approve when
the change definitely improves overall code health, even if it is not perfect. The
reviewer's question is "is this better than what exists?" not "is this how I would have
written it?" Perfection-gating produces stalled PRs and demoralized authors; the
mechanical corollary is that preference-level comments should be labeled as optional
("nit:") and must not block.

Cover what review should and should not spend attention on, connecting to Principle 8:
anything a machine can check — formatting, style, lint rules — must be automated out of
review entirely. Human attention is the scarcest resource in the process; spending it on
brace placement is misallocation. Review attention belongs on correctness, design,
readability, and test adequacy.

Cover size and latency as the two variables that dominate review quality: small changes
get better review (per the Ch 43 evidence — reference it, don't re-derive it), and slow
review is a tax on the whole team's cycle time that compounds through rebases and context
loss. Take a position: review turnaround is a team SLO measured in hours, not days.

Cover the failure modes: the nitpick storm (twenty style comments, zero design comments —
the reviewer performed thoroughness without engaging the change); rubber stamping
(approval-as-formality, often the end state of monster PRs and review-fatigue); and
review-as-gatekeeping (review used to enforce personal preference or status rather than
code health). Connect rubber stamping to Principle 9: if approval count is the metric,
approvals will be produced; whether scrutiny occurred is a different question.

Anchor in real systems: Google's published engineering practices documentation
(the "eng-practices" review guidelines and the code-health standard) as the canonical
public statement; "Modern Code Review: A Case Study at Google" (Sadowski et al., 2018)
for the empirical purposes-and-practices data; the SmartBear/Cisco study for the
size-effectiveness ceiling; automated formatting (gofmt, Black, Prettier) as the
Principle 8 mechanism that removed style from review in entire language communities.

Do NOT cover: PR sizing and the issue-to-PR mapping (Ch 43 — reference its conclusions);
branching and merge mechanics (Part VII, Ch 50, Ch 55); commit message conventions
(Part VII, Ch 51); pre-merge CI checks and what belongs in them (Part VII, Ch 57).
```

---

## Ch 48 — Technical Debt

```
This chapter rescues a useful concept from the mush it has become. "Technical debt" now
means "any code someone dislikes," which makes it useless for decisions. The chapter's job
is to restore the precise meaning and then treat debt as what it is: a financial
instrument that is sometimes the correct thing to issue.

Cover the original metaphor precisely: Ward Cunningham (1992 OOPSLA experience report)
described shipping code that embodies your current, incomplete understanding in order to
learn faster — with the intention of folding the learning back in. The debt is the gap
between the design you shipped and the design your current understanding calls for.
Cunningham's later clarification is essential and should be quoted or closely
paraphrased: the metaphor was never a license to write bad code and call it debt. Sloppy
code is not debt; it is just sloppy code. Debt requires a deliberate decision and a
payoff.

Cover Fowler's technical debt quadrant (deliberate/inadvertent × prudent/reckless) as the
classification that makes the term decision-useful: deliberate-prudent debt ("ship now,
we know the cost, here's the payback plan") is a legitimate instrument;
reckless-deliberate ("no time for design") is not debt but damage; inadvertent-prudent
("now we know how we should have done it") is the unavoidable product of learning and the
case Cunningham actually described.

Cover the economics: interest is the drag on every subsequent change that touches the
debt — slower features, riskier changes, longer onboarding — and it compounds. Name the
false dichotomy directly: quality vs. speed is a short-horizon illusion; internal quality
is what makes sustained speed possible, and the crossover arrives in weeks, not years.
Cover repayment strategies with positions: opportunistic repayment (leave it better than
you found it) as the default; scheduled dedicated capacity for debt that opportunism
cannot reach; and tracking known debt visibly (in the tracker, per Ch 42) so it is a
managed liability rather than tribal knowledge.

Cover the failure modes: debt-as-excuse (the "we'll fix it later" with no recorded later
— if there is no tracked payback intention, it was not a debt decision, it was a quality
decision made silently); the big-rewrite response (declaring bankruptcy and rewriting
from scratch, which discards the accumulated bug fixes and domain knowledge embedded in
the old system); and debt denial (treating all pressure for internal quality as gold-
plating until velocity quietly collapses).

Anchor in real systems: Cunningham's 1992 OOPSLA report and his 2009 video clarification
as the primary sources; Fowler's technical debt quadrant and Design Stamina Hypothesis as
the decision framework and the economics; Joel Spolsky's "Things You Should Never Do"
(the Netscape rewrite) as the canonical big-rewrite cautionary tale.

Do NOT cover: complexity itself and why it is the enemy (Part I, Ch 02 — debt is the
process-level management of a gap, not a synonym for complexity); when abstractions help
vs. obscure (Part IV, Ch 31); refactoring mechanics and code-level cleanup practice;
deciding not to test as a scope decision (Ch 39 of Part V).
```

---

## Ch 49 — Process Overhead: The Value Threshold

```
This is the capstone of Part VI and the full treatment of Design Principle 6: every
process is overhead, and overhead is justified only when the benefit exceeds the cost —
a threshold higher than most teams assume. Every preceding chapter in this Part described
a process and argued for its value; this chapter supplies the standing audit that keeps
those arguments honest. It is deliberately the same move Part V made with Ch 39: after
seven chapters of "do this," one chapter of "and here is when not to."

Cover the origin dynamics of process: nearly every rule is a scar — a response to a
specific past failure. That origin is what makes process feel unremovable (the failure
was real) and what makes it accumulate (failures keep happening, rules keep being added,
and no rule's original failure is ever re-examined). Name the process ratchet as the
central failure mode: process is added in the emotional aftermath of an incident and
removed never. The result, compounded over years, is a team whose process stack reflects
its entire failure history rather than its current risk profile.

Cover proportionality as the operative judgment: process weight must match team size,
blast radius, and reversibility. A ten-person team adopting a thousand-person company's
review, planning, and approval apparatus is buying insurance against risks it does not
have, at a price it cannot afford — cargo-culting big-company process is the
organizational version of speculative generality. The reverse failure is real but rarer:
a hundred-person team running on ten-person informality, coordinating through vibes.

Cover the audit discipline concretely: every standing process should be re-justifiable on
demand — what failure does this prevent, how often would that failure occur without it,
and what does it cost per week in engineer-hours and cycle time? A process that cannot
answer is a candidate for removal, and removal should be run as an experiment, not a
debate: drop it for a quarter and watch whether the failure it guarded against actually
returns. Connect to Principle 9 for process metrics: measure outcomes (cycle time, defect
escape rate), never compliance (tickets filed, meetings held, fields filled) — compliance
metrics will be satisfied by compliance, not by the outcome they proxy.

Cover Chesterton's fence honestly in both directions: do not remove a process before
understanding why it exists — but understanding why it exists includes discovering that
the reason no longer applies, and at that point the fence must actually come down.
Chesterton's fence is an argument for investigation, not preservation.

Anchor in real systems: Netflix's "context, not control" culture memo as the published
argument that process substitutes for judgment and high-trust teams should minimize it;
Basecamp/37signals' published positions as the small-team counterweight to enterprise
process; the well-documented pattern of cargo-cult agile — ceremonies retained,
purposes forgotten — as Principle 9 operating at the process level; GitHub's issue
templates and CODEOWNERS as examples of process encoded mechanically (Principle 8) so
that it costs nothing to follow, which is the cheapest kind of process to keep.

Do NOT cover: the specific value cases for issues, milestones, ADRs, specs, review, and
debt management (Ch 42–48 made those arguments; this chapter audits them, it does not
repeat them); CI pipeline design and automation mechanics (Part VII, Ch 57–61);
documentation maintenance burden (Part VIII, Ch 66).
```
