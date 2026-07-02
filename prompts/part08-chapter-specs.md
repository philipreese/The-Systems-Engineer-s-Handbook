# Part VIII — Chapter Specifications

Pre-filled special instructions for each Part VIII chapter. When generating a draft, copy the
block for the chapter you're writing into the `[SPECIAL INSTRUCTIONS]` field of the
chapter-generation prompt.

The instructions do three things:
1. State the chapter's specific role in the handbook's argument
2. Name the key sources, systems, or concepts to anchor the content
3. Explicitly mark what NOT to cover — because those topics belong to another chapter

Prerequisites for Part VIII: Part I (Ch 02's complexity argument applies directly — documentation
is a cost paid to reduce complexity for a future reader, not a free good, and Principle 6's "process
only when it adds more than it costs" applies to writing documentation exactly as it applies to any
other process); Part IV (Ch 27's file and module structure, since a README lives at exactly that
level; Ch 30 is this Part's single heaviest prerequisite — it established the WHY-vs-WHAT test for
inline comments, ruled that repository-level design rationale belongs in an ADR rather than a
comment, and explicitly deferred API documentation as its own publishing discipline to here).

Several forward references from earlier parts must be honored here:
- Part IV, Ch 30 deferred "API documentation in depth as its own discipline" to this Part (Ch 67
  specifically) and established the inline-comment/docstring boundary this Part's taxonomy chapter
  must resolve directly, not re-argue.
- Part VI, Ch 45 deferred "the broader taxonomy of README vs. spec vs. ADR" to Ch 65.
- Part VI, Ch 46 deferred "documentation taxonomy and where specs live long-term" to Ch 65, and
  named Ch 67 by number as the home for API reference documentation.
- Part VI, Ch 49 deferred "documentation maintenance burden" to this Part — the direct forward
  reference Ch 66 resolves.

Boundary with adjacent Parts: Part VI owns process artifacts — issues, specs, ADRs, code review —
as process: why they exist, what belongs in one, when writing one is worth the effort. This Part
owns the same artifacts as documentation: where each one sits in a taxonomy of audience and
lifespan, and how it's kept honest once written. Chapters here should not re-argue why a spec or an
ADR exists (Ch 45–46 already made that case) or re-derive the API design decisions Part III already
covers (resource modeling, error contracts, auth boundaries, internal-vs-external classification);
this Part documents those decisions, it doesn't make them. Part VII owns commit messages and tags
as Git mechanics, referenced here only where relevant, not re-covered. Part IX owns observability;
Ch 68 touches alerting and error budgets without owning either — flag the boundary with a one-line
pointer to Ch 71 and Ch 73, don't expand into their territory. Part XI owns security; any chapter
that brushes against credentials living in documentation should flag the boundary to Ch 83, not
treat secrets handling as its own topic.

This Part is small and linear by design: Ch 64 decides whether something should be written down
externally at all, Ch 65 decides which of four artifact types it becomes, Ch 66 keeps whichever
artifact was chosen honest over time, and Ch 67 and Ch 68 apply all three of those judgments to the
two audiences — external API consumers and on-call responders — whose documentation needs are
distinct enough to deserve their own chapters rather than falling out of the general case.

---

## Ch 64 — What to Document vs. What to Leave to the Code

```
This chapter opens Part VIII by extending Ch 30's WHY-vs-WHAT test one level up: that chapter
asked whether a specific comment earns its maintenance cost; this chapter asks the prior question
of whether any external documentation is warranted at all, before any decision about which artifact
type (Ch 65) even arises. Documentation is not a free good — Ch 02's complexity argument and
Principle 6 both apply directly: every page written is a page that has to be read, trusted, and
eventually corrected, and that cost is paid whether or not the page turns out to be worth it.

Cover the core test: does this information explain something the code genuinely cannot express on
its own — cross-cutting intent, a constraint invisible from any single file, a trade-off made and
rejected alternatives — or could the same information be recovered by better naming (Ch 28), better
structure (Ch 27), or a well-placed inline comment (Ch 30)? Prefer the code-level fix first; reserve
external documentation for information that has nowhere else to live, most often because it spans
multiple files, services, or points in time that no single comment location could cover.

Cover the specific categories of information that genuinely can't live in code: onboarding context
a newcomer needs before they've read anything yet (a README's job, teed up for Ch 65); permanent
records of a decision and its rejected alternatives (an ADR's job, Ch 45 — referenced, not
re-derived); and operational knowledge that only matters during an incident (Ch 68's job). Name
these categories without resolving the taxonomy between them yet — that's Ch 65's job.

Cover the proxy-metric failure mode directly, connecting to Principle 9: a mandated docstring- or
comment-coverage percentage produces the same result Ch 41 already documented for test coverage —
technically present, substantively empty documentation that satisfies the metric without conveying
anything a reader didn't already know from the function's name and signature. A coverage target
measures that something was written, never whether it was worth writing.

Anchor in real systems: the "docs as code" movement (Write the Docs community) as the general
industry position that documentation should be held to the same scrutiny as the code it describes,
not exempted from it; Google's internal engineering practices documentation (published externally
in abridged form) as an example of an organization drawing an explicit line between what belongs in
code comments versus a separate document; the common real-world failure of auto-generated,
one-sentence docstring stubs produced purely to satisfy a linter's coverage requirement as the
concrete expression of the proxy-metric failure mode above.

Do NOT cover: which of the four artifact types (README, spec, ADR, inline comment) a piece of
information belongs in once it's been judged worth writing down externally (Ch 65 — this chapter
answers only whether, not where); how to keep a chosen piece of documentation from going stale after
it's written (Ch 66); the content and judgment calls specific to inline comments (Part IV, Ch 30) or
to specs and ADRs as artifacts (Part VI, Ch 45–46) — all three are referenced here as destinations,
not redefined.
```

---

## Ch 65 — README vs. Spec vs. ADR vs. Inline Comment

```
This chapter resolves the taxonomy forward-referenced by three earlier chapters: Ch 30 deferred the
inline-comment boundary here, Ch 45 deferred "the broader taxonomy of README vs. spec vs. ADR," and
Ch 46 deferred the same question and specifically named this chapter for it. Four documentation
artifacts already exist across the handbook by this point; this chapter is where they're placed
side by side and distinguished on two axes: who is the intended reader, and is the artifact
supposed to track current reality or record a moment in time.

Cover each artifact's answer to those two questions precisely. A README orients someone about to
start working with the code — what this is, how to build and run it, where to go next — and is
expected to always be roughly current; a README that's stale is worse than no README, because it
costs a newcomer their first, most trust-forming hour. A spec (Ch 46) proposes work not yet done and
is explicitly a snapshot — its staleness the moment implementation diverges from it is expected, not
a failure, provided everyone treats it as an archived proposal rather than a live reference once
implementation begins. An ADR (Ch 45) records a decision already made and why alternatives were
rejected; like a spec, it does not describe current system state and is allowed to become
historically accurate but no longer descriptive of the present without being wrong — that's the
opposite failure mode from a stale README, and conflating the two categories is the single most
common taxonomy mistake this chapter should correct directly. An inline comment (Ch 30) is local,
tied to the specific lines it annotates, and moves or dies with the code around it.

Cover the actual selection test as the chapter's central deliverable: identify what question is
being asked, by whom, and whether the artifact needs to track current reality (README, API docs —
Ch 67) or is allowed to be a fixed record of a past moment (spec, ADR). This distinction is the
direct setup for Ch 66, which only applies to the first category — a stale spec isn't a documentation
honesty failure the way a stale README is, because the spec was never supposed to stay current in
the first place.

Anchor in real systems: GitHub's README-as-front-door convention, prominent enough that GitHub
renders it automatically on a repository's landing page, as the de facto standard for the
onboarding-artifact category; Kubernetes' Enhancement Proposal (KEP) process as a large-scale, public
expression of this handbook's "spec" category, explicitly separate from its reference documentation;
Nygard-style ADR directories (already anchored in Ch 45) contrasted directly against a project's
README to make the audience and lifespan distinction concrete rather than abstract.

Do NOT cover: whether a specific comment is worth writing at all, or the comment-rot argument (Ch
30 already owns that judgment); why specs and ADRs exist and what belongs inside one (Part VI, Ch
45–46 already made that case; this chapter only places them in the taxonomy); how to keep a README
or other living artifact from drifting once it's the chosen type (Ch 66 — this chapter picks the
artifact, that chapter keeps it honest); API reference documentation as its own case (Ch 67 — a
specialized, external-audience instance of the "must track current reality" category that earns its
own chapter because the tooling and audience differ enough from an internal README).
```

---

## Ch 66 — Keeping Documentation Honest

```
This chapter resolves Ch 49's explicit forward reference: "documentation maintenance burden belongs
to Part VIII." It's Ch 30's comment-rot argument applied at the scale of a whole documentation
corpus rather than one inline comment — but the failure is worse here, not merely bigger: a stale
comment sits next to the code it's wrong about, giving an attentive reader some chance of noticing
the mismatch; a stale README or wiki page can be several clicks removed from the code it describes,
with nothing structurally forcing a reader — or a person changing the underlying behavior — to
notice the two have diverged at all. This chapter applies only to documentation that's supposed to
track current reality (README, API docs); Ch 65 already established that a stale spec or ADR is a
different, expected condition, not this chapter's failure mode.

Cover the core mechanical fix, an instance of Principle 8: store documentation as close to the code
as possible, ideally in the same repository and the same pull request review flow ("docs as code"),
so a behavioral change and its documentation update have a structural chance of landing together.
This is an improvement, not a guarantee — nothing forces a developer to update the doc — but it
removes the worst version of the failure, where documentation lives in a separate system nobody's
code review ever touches.

Cover the split between what can be mechanically verified and what remains a human judgment call.
Executable documentation — code samples inside docstrings that are actually compiled and run in CI,
link checkers, doc linters — converts a subset of "is this still accurate" into a build failure
instead of a silent drift. Everything else — is this explanation still the right one, does this
guide still reflect the recommended approach — stays a human call, and connects directly to Ch 49's
audit discipline: documentation should be periodically reviewed, and deleting a page that's wrong or
no longer relevant is a legitimate, healthy action, not a loss to be avoided.

Cover the write-only-tracker failure mode by name, directly reusing Ch 42's term for the same
pattern in a new medium: documentation accumulates because writing is cheap and pruning feels risky
or thankless, until the corpus as a whole is untrusted and stops being consulted — which defeats
every page in it, including the ones that are still accurate, because nobody can tell which is
which without checking.

Anchor in real systems: the "docs as code" practice as implemented by GitLab's and GitHub's own
documentation, both maintained as reviewed markdown in-repository rather than a separate wiki;
Rust's `rustdoc` doctest feature, which compiles and executes code examples embedded in
documentation comments as part of the normal test run — a concrete, widely-used instance of
mechanically enforced documentation accuracy; the common, widely recognized failure pattern of an
abandoned internal wiki (a "Confluence graveyard") as the cautionary case for documentation with no
mechanical connection to the code it describes.

Do NOT cover: which artifact type a piece of information belongs in (Ch 65 already made that
decision; this chapter keeps the chosen artifact honest afterward); the ADR graveyard or frozen-spec
failure modes (Ch 45 and Ch 46 already cover staleness for those two artifact types specifically,
and their staleness is expected by design, not this chapter's subject); process overhead as a
general audit framework (Ch 49 already owns that framework — this chapter is its direct application
to documentation, not a restatement of it).
```

---

## Ch 67 — API Documentation: What Consumers Need

```
This chapter resolves the forward references from Ch 30 and Ch 46, both of which named this chapter
for API reference documentation specifically. It earns separate treatment from the rest of this
Part because its reader is structurally different: every other chapter here assumes a reader who
already has the source in front of them; this chapter's reader, per Part III's internal-vs-external
distinction (Ch 25), often does not have the source and may not be inside the same organization at
all.

Cover the content a consumer actually needs, connecting directly to Part III rather than re-deriving
API design: what each operation does, request and response shapes, the error contract (Ch 21 —
referenced, not redefined), authentication requirements (Ch 24 — referenced, not redefined), and
working, copyable examples, since a consumer trying to integrate wants a runnable example far more
than prose describing the same fields a second time.

Cover generated versus hand-written documentation as the chapter's central trade-off and take a
position, applying Principle 8 once more: generate the reference layer — parameters, types, status
codes — mechanically wherever tooling allows (OpenAPI/Swagger definitions, docstring-derived
references like Rustdoc or Sphinx), because a generated reference cannot drift from the interface it
describes the way hand-written prose can. Reserve hand-written material for what generation can't
produce: a getting-started narrative, realistic end-to-end integration patterns, the "why would I
reach for this" context. This narrative layer is exactly the kind of external documentation Ch 66
already flagged as prone to drift — its accuracy depends on a human noticing it's wrong, not a
compiler.

Cover Hyrum's Law's direct, specific implication for documentation (Part III, Ch 25): an incomplete
or ambiguous documented contract doesn't prevent consumers from depending on undocumented behavior —
it only makes that dependency invisible to the API owner until a change breaks it. Precise,
complete documentation of the actual contract is a defense against Hyrum's Law risk, not merely a
courtesy to the reader.

Anchor in real systems: the OpenAPI/Swagger specification as the dominant standard for generating
REST API reference documentation directly from a machine-readable contract; Stripe's API
documentation as a widely cited industry benchmark for example-heavy, interactive reference material;
Rustdoc, Javadoc, and Sphinx as docstring-to-reference tooling that keeps documentation mechanically
tied to source across different language ecosystems; GitHub's own generated REST and GraphQL API
references as a large-scale real-world instance of the generated-reference pattern.

Do NOT cover: resource modeling, error contract design, or authentication scheme design (Part III,
Ch 20, Ch 21, Ch 24 already own designing these; this chapter documents the result, it doesn't
design it); the internal-vs-external classification decision itself (Ch 25 already covers when and
why an API becomes an external contract; this chapter assumes that decision is made); versioning
semantics and the deprecation lifecycle (Ch 16 already owns what a version promises and the sunset
pattern for retiring one — this chapter covers ordinary reference content, not deprecation
communication).
```

---

## Ch 68 — Runbooks and Operational Documentation

```
This chapter closes Part VIII by covering the one documentation genre written for a reader under
time pressure — someone responding to an incident, often at an inconvenient hour — rather than a
reader doing ordinary development work. The correctness bar is different because of who's reading
and when: a runbook that's slightly wrong isn't merely unhelpful the way other stale documentation
is (Ch 66), it actively costs a responder time they don't have during exactly the situation the
document exists to make faster. This chapter connects directly to Ch 07's MTTR framing — a runbook
is a mechanism that exists specifically to reduce time-to-recovery, and its value should be judged
against that goal, not against how thorough it looks.

Cover what belongs in a runbook: specific, executable steps tied to a specific, named failure mode
— what to check, what to run, when to escalate and to whom — not general architecture explanation
(a README's job) and not open-ended troubleshooting philosophy. A runbook is a script for a known
failure class, not a tutorial about the system.

Cover the "tested" qualifier as the chapter's central argument: a runbook that has never actually
been executed carries the same false-confidence risk Ch 41 already established for coverage
metrics — its existence is not evidence that it works. Take a position: runbooks should be
deliberately exercised — a game-day drill, a walkthrough during onboarding, at minimum — rather than
trusted because they were carefully written. An unexercised runbook is a worse version of Ch 66's
honesty problem, because the only reader positioned to notice it's wrong is someone already mid-
incident, the single worst moment for that discovery to happen for the first time.

Cover the boundary with automation directly, applying Principle 8 one final time: any runbook step
that is genuinely mechanical and repeatable should be automated rather than documented as a manual
procedure. A runbook is what's left for judgment calls that can't yet be automated, and a runbook
that keeps growing is often a signal that more of it should have become tooling instead of prose.

Flag, without expanding: what triggers a responder to open a runbook in the first place is an
alerting decision (Part IX, Ch 71 — signal versus noise), and what a runbook ultimately protects is
an error budget or SLO (Part IX, Ch 73) — this chapter covers what's written down and how it stays
trustworthy, not the monitoring and alerting that gets a human to read it. Flag, without expanding,
that a runbook step should reference where to obtain a credential and never embed one directly
(Part XI, Ch 83 owns secrets management).

Anchor in real systems: Google's Site Reliability Engineering book as the canonical published source
for the runbook/playbook discipline, including its "toil" framing — manual, repetitive operational
work that should be automated away, which directly supports the automation-boundary argument above;
PagerDuty's published incident-response and runbook templates as a mainstream industry convention;
game-day and chaos-engineering practice, in the lineage of Netflix's Chaos Monkey, as the concrete
mechanism for actually exercising a runbook rather than trusting it unread.

Do NOT cover: alerting design and signal-to-noise tuning (Part IX, Ch 71); error budgets and SLOs
(Part IX, Ch 73); secrets management (Part XI, Ch 83); broader incident-response process and
postmortem practice beyond the runbook document itself, which this handbook does not currently
allocate a dedicated chapter to and this chapter should not expand to cover.
```
