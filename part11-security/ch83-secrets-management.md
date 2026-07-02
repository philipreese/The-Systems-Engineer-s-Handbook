# Ch 83 — Secrets Management

**Prerequisites:** [Release Automation](../part07-git-and-delivery/ch61-release-automation.md) (OIDC-based trusted publishing, named but deferred), [Runbooks and Operational Documentation](../part08-documentation/ch68-runbooks-and-operational-documentation.md) (credential handling, deferred), [Logging: What to Log and at What Level](../part09-observability/ch69-logging-what-to-log-and-at-what-level.md) (logging as attack surface, deferred), [Defense in Depth](ch80-defense-in-depth.md), [Authentication vs. Authorization](ch82-authentication-vs-authorization.md)

**New vocabulary introduced:** secret, credential scoping, trusted publishing

**Key takeaways:**
- A **secret** is any credential, key, or token whose disclosure grants an unintended party capability it shouldn't have. The defining property is not confidentiality — it's authority. A leaked secret rarely looks like an exploit; it looks like a legitimately authenticated request, which is exactly what makes it dangerous.
- A secret committed to version control is categorically worse than an ordinary bug: history is close to permanent, so deleting the offending line doesn't undo the exposure — every clone, fork, and mirror already made before the fix potentially retains it. Rotating the credential is the only real recovery.
- [Strong Recommendation] Rotation and **credential scoping** are the two central mitigations, and they address different halves of the same problem: rotation limits how long a leaked credential stays useful; scoping — narrowest-privilege permissions, shortest practical lifetime — limits how much damage it can do before it's rotated.
- [Strong Recommendation] **Trusted publishing** — exchanging a short-lived, pipeline-scoped identity via OIDC for a temporary publish session — is strictly better than rotating a long-lived token, because there's no static credential sitting anywhere to leak in the first place. Eliminating the secret dominates managing it better.
- [Strong Recommendation] A dedicated secret manager is the correct default for anything beyond a single-developer project — Principle 8 applied to credentials specifically: rotation and audit become mechanical properties of the infrastructure instead of a discipline every engineer has to remember and never skip.

---

Authentication and authorization (Ch 82) ultimately rest on credentials. A correctly implemented protocol and a carefully scoped permission policy both become irrelevant the moment the credential backing them is exposed — an attacker holding a valid secret doesn't need to defeat either control, because the system has no way to distinguish them from the legitimate holder. A secret's defining property isn't that its value is confidential; it's that possessing the value grants capability. The security problem is not disclosure in the abstract — it's unauthorized capability, and that reframing is what makes a database password and a signing key comparable risks even though only one of them looks cryptographic.

This is also why a secret checked into version control is a fundamentally different kind of mistake than an ordinary bug. A logic error is fixed by the next commit. A committed credential is not: version history is close to permanent, every clone and fork made before the fix potentially carries a working copy forward indefinitely, and deleting the line from the latest commit changes nothing about what already leaked. The only real recovery is rotating the credential itself — treating the leak as though the secret is already compromised, because for practical purposes, it is.

### Decision: Exchange Ephemeral Identity via OIDC, or Persist Long-Lived Static Credentials

**What it is:** Whether an automated system — a CI pipeline publishing a package, a service deploying to cloud infrastructure — authenticates by presenting a long-lived static token stored in its configuration, or by exchanging a short-lived, cryptographically verifiable identity assertion for a temporary session scoped to exactly that operation. This resolves Ch 61's deferred forward reference in full: OIDC-based **trusted publishing**, already named there as npm and PyPI's direction of travel, is the concrete instance of this decision.

**Why it exists:** A static publishing token stored in CI configuration is a persistent, high-privilege target: anything that can read the pipeline's configuration or environment — a compromised dependency executing during a build, a misconfigured log statement, an insider — can extract a credential valid until someone notices and rotates it. Trusted publishing removes the target rather than defending it better: the pipeline presents a short-lived, workload-scoped identity (a signed token asserting facts like repository and workflow name) to the registry, which verifies it against the identity provider's public keys and issues a session valid only for that specific run. No persisted secret exists anywhere for anything to steal.

**Options:**
- **OIDC trusted publishing** — a short-lived, pipeline-scoped identity assertion exchanged for a single-use publish session; no static secret stored anywhere.
- **Static configuration tokens** — a long-lived, high-privilege token generated once and stored in the CI platform's secret configuration.

**Trade-offs:** Trusted publishing eliminates the entire category of persistent-credential exposure — a compromised pipeline run leaks, at most, a token scoped and time-boxed to that one run — at the cost of a real cryptographic trust relationship that has to be correctly configured and maintained between the identity provider and the consuming registry, and a runtime dependency on both systems' availability. Static tokens require no federation setup at all and work regardless of either side's real-time availability, but the credential's validity is open-ended: once issued, it remains a standing target that has to be tracked, rotated, and audited by hand for as long as it exists, and if it leaks, it's valid until someone notices.

**When to choose each:** [Strong Recommendation] OIDC trusted publishing as the default for any automated publishing or deployment pipeline where a credential leak's blast radius is more than trivial. Static tokens remain the fallback only where the counterparty genuinely can't validate OIDC claims — a legacy vendor platform with no support for federated identity — not a default to reach for out of convenience.

**Common failure modes:** *The static-variable harvest.* A team stores a long-lived publishing token as a CI variable. An attacker compromises a dependency pulled in during the test phase; the malicious code reads the environment, exfiltrates the token, and hours later uses it — fully valid, fully unscoped — to overwrite published packages with a compromised version, propagating the compromise to every downstream consumer.

**Example:** A GitHub Actions workflow publishing to PyPI under trusted publishing never holds a password. It requests a short-lived signed JWT from GitHub's OIDC provider; PyPI validates the token against GitHub's public keys, confirms the repository and workflow match its configured trust relationship, and issues a publish session that expires when the job completes — resolving Ch 61's forward reference directly.

### Decision: Retrieve Secrets Dynamically From a Manager, or Inject Them Statically Into the Environment

**What it is:** Whether an application authenticates to a dedicated secret manager at runtime to obtain short-lived, automatically rotated credentials, or receives credentials as static values injected into its process environment or a configuration file at startup.

**Why it exists:** Static credentials depend entirely on human discipline: someone has to remember to rotate them, track everywhere they're used, and revoke them promptly when something changes. A dedicated secret manager makes those practices mechanical rather than optional — Principle 8 applied directly to credential handling. Environment variables have a structural weakness beyond staleness, too: they're inherited by every child process the application spawns, routinely captured by crash-dump and diagnostic tooling, and readable by anything with local access to the process — a much wider exposure surface than most engineers configuring `DATABASE_URL=...` are thinking about at the time.

**Options:**
- **Dedicated secret manager** (HashiCorp Vault, a cloud provider's native key management service) — the application authenticates with its own workload identity and receives short-lived, dynamically generated, automatically rotated credentials.
- **Static injection** (environment variables, configuration files) — credentials are fixed values placed into the process's environment or filesystem at boot.

**Trade-offs:** A secret manager centralizes auditing, rotates credentials automatically without requiring a coordinated redeploy, and keeps live credentials out of the filesystem and process environment entirely — at the cost of real code-level coupling: applications need SDK logic to authenticate, fetch, and renew credentials, and the secret manager's own availability becomes a dependency every credential fetch or renewal now relies on. Static injection takes minutes to wire up and completely decouples application code from the security architecture behind it, but every credential is a fixed value with only as much protection as whatever else can read the process's environment or the filesystem — no audit trail showing who or what accessed it, and rotation means a coordinated restart across everything using the old value.

**When to choose each:** [Strong Recommendation] A dedicated secret manager for anything beyond a single-developer local project — multi-service, multi-environment, or multi-engineer systems are exactly where manual rotation and tracking stop being reliable. Static injection remains reasonable for local development and genuinely small deployments where the operational cost of a secret manager isn't yet justified by what it protects.

**Common failure modes:** Static production credentials sitting in a configuration file with no audit trail showing when or by whom they were last accessed; environment variables copied by hand between systems, so nobody can say with confidence how many places still hold a credential that was supposed to be retired; assuming a secret manager's presence alone means secrets are handled correctly, when an application still reads a stale cached copy well past its intended lease.

**Example:** HashiCorp Vault's Agent Injector pattern mounts an in-memory volume into an application container; a sidecar authenticates using the pod's own workload identity, requests a database credential with a bounded lease (say, sixty minutes), and writes it to that memory-backed file. Vault drops the corresponding database user when the lease expires, so a credential captured from that file is worthless within the hour regardless of what an attacker does with it.

### Decision: Scope Credentials to Minimum Necessary Authority and Lifetime, or Grant Broad, Long-Lived Access

**What it is:** Whether a credential is issued with exactly the permissions and validity window its task requires, or a single broadly privileged, long-lived credential is reused across multiple purposes for convenience.

**Why it exists:** Some credentials will eventually be compromised regardless of how carefully they're otherwise handled — rotation exists because that assumption is realistic, not because rotation alone is sufficient. Scoping is the complementary control: it determines how much damage that eventual compromise can do. A credential capable only of reading one database is a fundamentally smaller asset to an attacker than one capable of administering an entire cloud account, independent of whether either one ever leaks.

**Options:**
- **Narrowly scoped credentials** — issued per application, per environment, per purpose, with the shortest lifetime the task tolerates.
- **Broad, shared, long-lived credentials** — one credential reused across multiple services, environments, or purposes because provisioning individual ones feels like overhead.

**Trade-offs:** Narrow scoping shrinks blast radius, makes audit logs meaningful (a specific credential's activity maps to a specific purpose instead of an undifferentiated shared identity), and limits how far a compromise can move laterally — at the cost of more credentials to provision, track, and eventually retire. Broad, shared credentials are less operational overhead up front, but every service holding one inherits the full blast radius of whatever that credential can do, and revoking access from one consumer without disrupting the others sharing it is rarely possible.

**When to choose each:** [Strong Recommendation] Narrow scoping by default, everywhere practical — a deployment pipeline that only publishes packages should never also hold infrastructure-administration credentials, regardless of how convenient one shared credential would be to set up once. Broader scope is defensible only where the operational cost of finer-grained credentials genuinely exceeds the risk of the shared one, which is a narrower case than most systems that default to sharing actually are.

**Common failure modes:** One credential shared across services because provisioning individual ones felt unnecessary at the time, until incident response can't determine which service actually made the malicious request; a development credential reused in production because rotating to environment-specific ones was deferred and never revisited; administrative permissions granted "to be safe" well beyond what the task in front of the engineer at the time actually required.

**Example:** A deployment pipeline that publishes packages needs publish permission on one registry, scoped to one project, for the duration of one job — not standing infrastructure-administration credentials that happen to also be capable of publishing.

### Secrets in Documentation and Logs

This resolves the remaining two deferred forward references directly and narrowly, without re-deriving either chapter's broader subject.

Ch 68 already established that a runbook step should point to where a credential is obtained, never embed the value itself. *The runbook hardcoding leak* is the failure mode that rule prevents: an engineer debugging a production incident runs a diagnostic command that dumps the container's environment variables, then pastes the output into a shared incident channel to document what they found — inadvertently publishing a production database password into a persistent, broadly readable text store that nobody thinks to treat as a secret-handling incident, because nobody framed pasting diagnostic output as a credential-handling decision at all.

Ch 69 already established that logging is part of a system's attack surface; this chapter supplies the specific mechanic Ch 69 deferred. A credential passed into a log statement — directly or nested inside a logged object, such as an HTTP request or response — is serialized into a log store that is copied, indexed, retained, and read by people who should never hold a production credential. The fix is structural rather than a discipline to remember: wrapping credential values in a type that overrides default string and serialization behavior (returning a fixed masked value instead of the real one) makes an accidental log statement emit a masked placeholder automatically, regardless of whether the engineer writing that particular log line was thinking about secrets at all — Principle 8 again, applied at the type-system level this time.

### Why Smart Engineers Disagree on Coupling Availability to a Secret Store

The disagreement here isn't about whether dynamic secret retrieval is more secure — both sides agree it is. It's about whether a system's core availability should depend on a secrets store being reachable at every moment a credential is needed.

One position treats continuous, real-time retrieval as non-negotiable: every credential fetch, every connection-pool refresh, authenticates fresh against the central store, and if the store says a credential is revoked, every consumer should fail fast rather than keep operating on a cached value — caching a credential anywhere outside volatile memory is treated as a lapse in zero-trust discipline. This position accepts that the secrets store becomes a hard, system-wide dependency, on the reasoning that operating on a credential state that might already be revoked is a worse failure than an outage.

The opposing position treats that same coupling as the actual architectural mistake: a core transactional service's availability SLO should never depend on the real-time uptime of an internal management API, and a secrets-store network partition shouldn't be able to cascade into an outage of everything that needs a credential at that moment. This position reads credentials once at boot or on a bounded refresh interval and runs independently of the store's continuous availability afterward, accepting a wider (but bounded) window before a revocation takes effect everywhere.

Neither side is wrong about the cost it's naming — this is an explicit trade between blast-radius containment and infrastructure availability, not a technical question with one correct answer. The resolution in practice matches control tightness to what's actually being protected: continuous dynamic retrieval for high-criticality paths — signing keys, financial transaction credentials — where a stale revoked credential is the worse outcome, and cached, bounded-refresh credentials for stateless application tiers where uptime is the dominant commercial concern and the exposure window a bounded cache introduces is one the system's risk profile can actually tolerate.

### Case Study: The 2021 Codecov Breach

```
[Attacker]
   │  harvests a leaked GCP credential from Git history
   ▼
[Codecov's internal infrastructure]
   │  modifies the public Bash Uploader script
   ▼
[Customer CI/CD pipelines worldwide]
   │  malicious script reads the plaintext process environment
   ▼
[Attacker's collection server] → mass harvest of customer secrets
```

Codecov's Bash Uploader script was embedded directly into thousands of customer CI pipelines. The breach's origin was an ordinary instance of the version-control exposure this chapter opened with: a static, high-privilege Google Cloud service-account credential had been committed to a Codecov repository, then later removed from the active file — which, as already established, does nothing about what the commit history still retains. An attacker recovered the credential from that history and used it to gain write access to the storage bucket serving the Bash Uploader script.

From there, the attacker modified the script itself to silently harvest and exfiltrate the CI environment of every customer pipeline that ran it — and because those pipelines followed the static-injection pattern this chapter's second decision covers, every credential a customer had placed into its build environment (database passwords, API keys, repository tokens, all as plaintext environment variables) was sitting in exactly the place the malicious script was positioned to read. The exploit ran undetected for over two months, harvesting credentials from thousands of enterprises.

Two independent controls from this chapter, either one alone, would have contained it well short of that scale. Pre-commit scanning that caught the credential before it entered Git history would have prevented the initial compromise entirely. And for every downstream customer, credentials obtained through OIDC-based trusted publishing or a dedicated secret manager rather than injected as static environment variables would have handed the malicious script nothing but expired session identifiers — the same structural argument this chapter has made throughout: eliminating a static secret is worth more than managing one better, because there is nothing left in the process environment for even a fully successful exfiltration to steal.
