# Ch 84 — Dependency Supply Chain Risk

**Prerequisites:** [Tagging and Release Markers](../part07-git-and-delivery/ch56-tagging-and-release-markers.md) (signed tags, deferred supply-chain threat model), [Toolchain and Dependency Management](../part07-git-and-delivery/ch63-toolchain-and-dependency-management.md) (dependency debt, deferred security threat model), [Defense in Depth](ch80-defense-in-depth.md), [Secrets Management](ch83-secrets-management.md)

**New vocabulary introduced:** lockfile, provenance attestation, SBOM, dependency confusion

**Key takeaways:**
- A dependency is code the team didn't write and typically didn't fully review, pulled from a registry whose trust model is usually "anyone can publish a package," and updated as part of routine maintenance (Ch 63's subject). That update cadence question — is this version current — is entirely separate from this chapter's question: is this version trustworthy. A dependency can be exactly current and still be malicious.
- [Strong Recommendation] A **lockfile** pinning the exact, hashed version of every transitive dependency is the mechanical floor — Principle 8 applied to the build plane. It converts "trust that nothing changed since last time" into a fact the build system verifies automatically, rather than a discipline anyone has to remember.
- **Provenance attestation** extends Ch 56's tag-signing argument one level further: beyond confirming that a version number matches, to confirming that the artifact was actually built by the expected pipeline from the expected source — closing the gap a stolen publishing token or a compromised local workstation leaves wide open even when the version number looks correct.
- A **Software Bill of Materials (SBOM)** answers "what is actually inside this build" as a queryable fact rather than a manual archaeology project — the difference between minutes and days when a critical vulnerability is disclosed in some transitive dependency nobody on the team directly chose.
- The 2018 event-stream compromise is the canonical case: a maintainer handoff to a bad-faith volunteer, a malicious transitive dependency published under a legitimate, currently-updated package name, targeting one specific downstream application. Nothing about the version number indicated compromise — the trust relationship itself was what got exploited.

---

Part XI opened with trust boundaries (Ch 79) and closes here, at the boundary furthest from anything the team actually controls: the code it depends on but never wrote. A dependency is pulled from a registry whose baseline trust model, across npm, PyPI, Maven Central, and most of their peers, amounts to "anyone can create an account and publish a package." It executes with the same privileges as application code the moment it's imported — often before that, during install-time scripts — unless something has gone out of its way to constrain it. And it is not a static artifact: it is updated automatically or semi-automatically as routine maintenance, which is exactly Ch 63's subject and exactly not this chapter's. Ch 63 asks whether a dependency's version is current. This chapter asks a different question entirely: whether the current version can be trusted, which a version number alone cannot answer.

Three attack patterns recur across ecosystems because they exploit the same trust gap in different ways. **Maintainer compromise** — an attacker takes over a legitimate maintainer's account or is voluntarily handed publishing rights — is the most dangerous of the three precisely because it preserves the package's expected identity: downstream systems auto-update or install the new version exactly as they would any routine release, and nothing about the package name or version number signals anything changed. **Typosquatting** publishes a malicious package under a name that closely resembles a legitimate one (`reqeusts` instead of `requests`), relying on a developer's typing mistake rather than any weakness in the trust model itself. **Dependency confusion** exploits namespace ambiguity between a private and a public registry: an organization references an internally named package, an attacker publishes a same-named package to the public registry with a higher version number, and a build system configured to check the public registry — or to prefer the higher version — resolves to the attacker's package instead of the internal one.

### Decision: Pin Dependencies With a Cryptographically Verified Lockfile, or Resolve Versions Dynamically

**What it is:** Whether a build validates every dependency it pulls — direct and transitive — against a static, hashed manifest recorded in a **lockfile**, or resolves versions dynamically at build time against whatever a semantic version range currently matches on the registry.

**Why it exists:** This is Principle 8 applied to the build plane. A public registry is a mutable, continuously changing graph; without a lockfile, the exact code a build pulls in can differ from one run to the next with no change to the project's own source at all. If an upstream dependency is compromised and a malicious version is published within a range the project's manifest already permits, dynamic resolution injects that exploit into the next build automatically, with no human decision point anywhere in the path.

**Options:**
- **Cryptographic lockfile enforcement** — an immutable manifest recording the exact version, source, and content hash of every dependency; CI runs in a mode (`npm ci`, `cargo --frozen`) that fails the build outright on any hash mismatch or unlisted dependency.
- **Dynamic version resolution** — the build queries the registry at install time and resolves the dependency tree fresh against the manifest's semantic version ranges.

**Trade-offs:** A lockfile guarantees bit-for-bit reproducibility across every environment and neutralizes the moment-of-build injection risk dynamic resolution carries, at the cost of routine merge friction on lockfile changes and the need for active tooling (Dependabot, Renovate) to keep paying down the currency debt Ch 63 already named, since pinning doesn't update itself. Dynamic resolution eliminates that lockfile overhead and lets a build inherit upstream patches without any developer action, but it makes the build's exact contents non-deterministic — the same manifest can resolve to different, unreviewed code depending on exactly when it runs.

**When to choose each:** [Strong Recommendation] A pinned, hash-verified lockfile is the non-negotiable baseline for any production build, container image, or CI pipeline. Dynamic resolution is defensible only for throwaway local scripts or exploratory spikes that will never run in a trusted environment or reach source control.

**Common failure modes:** *The ghost patch.* A production Dockerfile runs an install command with no lockfile. An attacker compromises a minor utility deep in a widely used package's transitive tree and publishes a patch version with a silent backdoor. An unrelated hotfix the next morning triggers the pipeline to pull dependencies fresh, the backdoored patch resolves cleanly under the existing version range, and the compromised container deploys to production having never passed through code review at all — the vulnerability entered through the build system, not a pull request.

**Example:** Go enforces this at the toolchain level rather than leaving it to convention: `go.sum` records an append-only log of cryptographic checksums for every module version, and the global checksum database (`sum.golang.org`) detects if a maintainer ever rewrites the contents of an already-published version tag. A hash mismatch fails the build immediately, rather than silently accepting whatever code is currently sitting behind a version string.

### Decision: Require Artifact Provenance Attestation, or Trust Registry Identity Alone

**What it is:** Whether a dependency must carry a verifiable, cryptographic record of exactly which source commit and which build pipeline produced it — **provenance attestation** — or whether a correct package name, version, and publishing account are treated as sufficient. This extends Ch 56's tag-signing argument one level further, from "who possessed the signing key at release time" to "was this specific artifact actually built the way it claims to have been."

**Why it exists:** A correct version number proves nothing about origin. A stolen publishing token lets an attacker push a malicious artifact under a legitimate version number; a compromised developer workstation lets an attacker compile a malicious binary locally and sign it with valid credentials without ever touching the project's actual source repository or CI pipeline. Provenance shifts the trust question from "does this account's token match" to "was this exact binary produced by the pipeline it claims, from the source commit it claims" — a structurally stronger claim that a leaked token alone can't satisfy.

**Options:**
- **Provenance attestation** — build gates accept only dependencies carrying a verifiable attestation (Sigstore-signed, SLSA-attested) proving the artifact was produced by a specific, hardened CI pipeline from a specific commit.
- **Registry identity alone** — a package is trusted if its name, version, and publishing account resolve cleanly, with no verification of how or where the artifact was actually built.

**Trade-offs:** Provenance attestation closes the gap a stolen credential alone cannot: even with a valid publishing token, an attacker can't produce a passing attestation without also compromising the specific, hardened pipeline the attestation is bound to. The cost is real ecosystem immaturity — most existing open-source packages don't yet publish attestations, forcing a choice between meaningfully restricting which dependencies are usable or spending real effort maintaining an internally vetted mirror. Registry identity alone imposes no such restriction and works with the entire existing ecosystem unmodified, but it inherits the full weight of every individual maintainer's personal security hygiene — one compromised laptop anywhere upstream is a direct path into every downstream consumer.

**When to choose each:** [Strong Recommendation] Provenance attestation for critical infrastructure, financial or cryptographic cores, and any environment where executing an unverified binary is a genuinely unrecoverable risk. Registry identity alone remains the practical default elsewhere in the ecosystem today, ideally paired with the lockfile and SBOM controls this chapter covers, given how much of the open-source supply chain doesn't yet support attestation at all.

**Common failure modes:** A maintainer's personal machine is compromised by infostealer malware that extracts a long-lived publishing token from local shell history; because the registry trusts the token alone, the attacker compiles and publishes a backdoored version directly from their own machine, and it reaches every downstream consumer with a valid signature and a legitimate version number, because nothing checked whether it was actually built by the project's own pipeline.

**Example:** npm provenance and PyPI trusted publishing (Ch 83) tie a published artifact to a verifiable CI run rather than a bare, long-lived account credential. Sigstore's Cosign lets a Kubernetes admission controller reject any container image lacking a valid attestation that it was built inside a specific, trusted pipeline — blocking a manually pushed or externally built image regardless of whether its name and tag look correct.

### Decision: Maintain a Software Bill of Materials, or Rely on Point-in-Time Scanning

**What it is:** Whether every release generates and persists a machine-readable inventory — an **SBOM** — of every direct and transitive dependency actually built into it, or whether dependency risk is assessed only when a scanner is run against the current state of a repository.

**Why it exists:** When a serious vulnerability is disclosed in some far-downstream transitive dependency nobody on the team directly chose, the operative question is immediate: which of our production systems actually contain it. Without a persisted inventory, answering that means re-cloning and re-scanning every repository from scratch — a process that takes days precisely when speed matters most. An SBOM converts that question from an active investigation into a query against an asset already on hand.

**Options:**
- **SBOM generation and enforcement** — every release pipeline emits a standardized inventory (CycloneDX, SPDX) of every dependency and toolchain component, indexed centrally and queryable on demand.
- **Ad hoc scanning** — a point-in-time scanner runs against source repositories on a schedule or per pull request, with no persisted record of what's actually deployed.

**Trade-offs:** An SBOM gives near-immediate exposure mapping during an active disclosure — a direct query identifies every affected build instead of a manual audit — at the cost of real tooling investment: generating, storing, and keeping accurate metadata current across every service and image adds a genuine maintenance burden. Ad hoc scanning is far cheaper to adopt, often a single SaaS integration with no pipeline changes required, but it reports what's in a repository's current branch, not necessarily what's actually running in a container that hasn't been rebuilt or redeployed in months — exactly the gap that matters most during an incident.

**When to choose each:** [Strong Recommendation] An enforced SBOM as soon as a dependency graph is nontrivial, more than one team shares a codebase, or an external customer requires verifiable transparency — which describes most production systems past an early stage. Ad hoc scanning alone remains reasonable for a small, single-repository system one team can still hold a complete mental model of.

**Common failure modes:** A severe vulnerability is disclosed in a compression library buried deep in several unrelated services' transitive trees. With no centralized SBOM, the security team spends days manually re-auditing repositories one at a time while the exposure remains unaddressed — the exact window an attacker needs to find and exploit it first. Also common: an SBOM generated once at release and never reconciled against what's actually still deployed, so it documents a build that no longer matches reality.

**Example:** A CI pipeline generates a CycloneDX SBOM immediately after a container build, signs it, and pushes it alongside the image to the registry; a centralized scanner continuously checks that inventory against active CVE feeds, so the moment a dependency is flagged, the exact affected image coordinates are already known — no source repository needs to be touched to find them.

### Why Smart Engineers Disagree on Vendoring Versus Ecosystem Integration

The disagreement here is not about whether supply-chain risk is real. It's about how far to go in decoupling a system from the public ecosystem it depends on.

One position argues for vendoring the critical path: clone, audit, and check third-party source directly into an internal tree, compiling from code the organization has actually reviewed rather than a registry it merely trusts. This decouples the system's availability and security posture entirely from the public internet — a compromised upstream account or a deleted package simply can't reach a dependency that was never fetched live in the first place — at the cost of a real, ongoing velocity tax: vendored code doesn't inherit upstream fixes automatically, and keeping an internal fork current is exactly the kind of manual, easily-deferred discipline that produces the dependency debt Ch 63 already warned accumulates when nobody explicitly owns it.

The opposing position treats heavy vendoring as trading one risk for a worse one: an organization that vendors broadly ends up maintaining an ad hoc, home-grown operating system layer of stale forks that miss security patches for the same reason any deferred maintenance does — because updating them isn't anyone's job by default. This position integrates directly with the public ecosystem and surrounds it with the mechanical guardrails this chapter already covers — lockfiles, provenance verification, an audited internal proxy cache — arguing that automated verification at the boundary scales better than manually re-auditing vendored code forever.

Both are right about the risk they're naming, and the resolution in practice is rarely uniform across an entire dependency graph: vendor and audit by hand only the small, ultra-critical core — a cryptographic engine, a consensus algorithm, a ledger's core logic — where a compromise is unrecoverable, and rely on lockfiles, provenance checks, and an internal proxy cache for the much larger volume of ordinary library dependencies where automated verification is the more sustainable control.

### Case Study: The 2018 event-stream Compromise

```
[Attacker] ── gains maintainer trust via social engineering ──► [event-stream, v3.3.6]
                                                                       │
                                                     injects new dependency: flatmap-stream
                                                                       │
                                                                       ▼
                                                    [published to npm — resolves cleanly]
                                                                       │
                                             pulled in by the Copay Bitcoin Wallet's build
                                                                       │
                                                                       ▼
                                          [payload activates only inside Copay's build] ──► exfiltrates private keys
```

`event-stream` was a widely used Node.js utility with millions of weekly downloads. Its original maintainer, without commercial backing and burned out, was approached by a volunteer offering to take over maintenance — and, acting in good faith, handed over full publishing rights. The new maintainer was not acting in good faith. Rather than inject a detectable backdoor into `event-stream` itself, which automated scanning of the package's own code might have caught, they published a routine-looking update that added a new transitive dependency, `flatmap-stream`, containing an encrypted payload engineered to do nothing at all in the overwhelming majority of environments it landed in.

The payload activated only when it detected one specific downstream consumer: the Copay Bitcoin Wallet. When Copay's build pipeline ran its routine dependency installation, the package resolved cleanly — current version, published by an account with legitimate publishing rights, no vulnerability signature any scanner had indexed, because none existed yet. Inside Copay's build specifically, the payload decrypted, hooked into the wallet's cryptographic key-handling code, and exfiltrated private keys and seed phrases to an external server. The compromise wasn't discovered through any automated control; an unrelated engineer noticed an unexplained deprecation warning from a dependency nobody recognized during a routine local build, weeks after the malicious version had shipped.

Nothing about `event-stream`'s version number or publishing account indicated anything had changed — the update was current, signed by an account the registry had every reason to trust, and passed every check that treats "version is current" as the relevant question. The trust relationship itself, inherited through a maintainer handoff nobody outside the project had visibility into, was what the attacker actually exploited. A pinned lockfile would have surfaced the unexpected new transitive dependency as a reviewable diff instead of a silent addition. An SBOM would have made "do we depend on `flatmap-stream`" an instant query the moment the compromise became public, instead of a scramble to determine exposure after the fact. Neither control would have prevented the maintainer handoff itself — that risk has no purely mechanical fix — but either one would have caught its consequence well before it reached a production wallet holding real funds.
