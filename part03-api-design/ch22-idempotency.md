# Chapter 22 — Idempotency

**Prerequisites:** [Part II, Ch 17 — Synchronous vs. Asynchronous Communication](../part02-software-architecture/ch17-sync-vs-async-communication.md), [Part III, Ch 21 — Error Handling Contracts](ch21-error-handling-contracts.md). Specifically: at-least-once delivery, the impossibility of real exactly-once delivery across a network boundary, and structured error responses.

**New vocabulary introduced:** idempotency key

**Key takeaways:**
- Ch 17 established that at-least-once delivery is the realistic default once a call crosses a process or network boundary, and that true exactly-once delivery isn't achievable there. Idempotency is the mechanism that makes that fact survivable: it doesn't prevent a request from arriving twice, it makes arriving twice harmless.
- Some operations are naturally idempotent by construction (`PUT`, `DELETE`) — repeating them produces the same end state. Others (`POST`) are not, and need designed idempotency: a client-supplied key the server uses to recognize and collapse a retry into the original result.
- The only correct enforcement mechanism is a database-level uniqueness constraint, not an application-level check. A `SELECT`-then-`INSERT` check has a race window two concurrent retries can both slip through; an atomic `UNIQUE` constraint can't be raced.
- A server can't remember every key forever. The retention window is a real trade-off — too short and a legitimately delayed retry gets treated as brand new; too long and transport metadata permanently bloats the domain tables it's attached to.

---

## Natural vs. Designed Idempotency

**What it is:** The distinction between operations that are safe to retry purely because of what their HTTP verb means (natural idempotency), and operations that need explicit engineering to become safe to retry (designed idempotency).

**Why it exists:** A client can never be certain whether a request it sent actually reached the server, only that it didn't get a response back. If the network drops the acknowledgment, the client has to retry — and if the API wasn't designed with that in mind, the retry itself is what corrupts the state.

**Options:**
1. **Natural idempotency** (`PUT`, `DELETE`, `GET`) — the operation targets a specific existing identity and replaces or removes its state; repeating it produces the identical end state
2. **Designed idempotency** (`POST`) — the operation creates a new entity or triggers a side effect with no prior identity to target; repeating it by default produces a new effect each time, unless explicitly protected

**Trade-offs:**
- *Natural idempotency:* zero backend tracking or storage cost, and clients can retry as aggressively as they want with no coordination — but it requires the client to already know the target identity, often costing an extra round trip to obtain or generate an ID before the mutation can even happen.
- *Designed idempotency:* lets a client safely execute a complex, multi-step creation in a single `POST` — but it obligates the provider to build and maintain a correct, concurrent, server-side deduplication mechanism.

**When to choose each:**
- *Natural idempotency:* the default mental model for reads, replacements, and deletions — standard HTTP clients and service meshes already retry `GET`/`PUT`/`DELETE` automatically because the protocol guarantees they're safe.
- *Designed idempotency:* required for any `POST` that initiates a state transition, charges money, or triggers an asynchronous side effect.

**Common failure modes:**
- **The unsafe default retry:** a mobile client library retries every timeout by default. Checkout is implemented as a plain, non-idempotent `POST /checkout`. The user loses signal mid-request, the library retries automatically, and the customer is charged twice for one cart — not because anything malfunctioned, but because nothing told the retry it wasn't safe.
- Assuming `POST` is idempotent by accident (no key, no protection) and getting silent duplicate creation instead of a clean failure.

**Example:** `DELETE /users/456` is naturally idempotent: the first call removes the user and returns success; a retry might return `404` instead of `200`, but the actual system state — the user is gone — is identical either way. The retry is genuinely harmless, with no extra design required.

---

## The Idempotency Key Pattern

**What it is:** The client generates a unique identifier for a logical operation (typically a UUID), sends it with the request, and the server uses it to recognize a retry and return the original result instead of re-executing the side effect.

**Why it exists:** It gives the server a way to distinguish "the user genuinely clicked checkout twice" from "the network retried the same checkout once" — a distinction the server has no way to make on its own, since both arrive as identical-looking requests.

**Options:**
1. **Application-level check** — the application runs a `SELECT` to see whether the key already exists before executing the logic and inserting the key
2. **Database-level uniqueness constraint** — the application attempts an `INSERT` into an idempotency-key table with a `UNIQUE` constraint and lets the database reject the duplicate atomically

**Trade-offs:**
- *Application-level check:* simple to write, no schema changes needed — but it's structurally broken: two concurrent retries can both run the `SELECT`, both see no existing key, and both proceed to execute the side effect before either has written its key. The check and the use aren't atomic.
- *Database-level constraint:* the database's own concurrency control makes the race impossible — but it requires writing the idempotency key and the resulting response in the same transaction as the side effect, adding write volume and coupling the API response layer to persistence.

**When to choose each:**
- *Database-level constraint:* the only acceptable choice for financial transactions, state-machine mutations, or any designed-idempotency endpoint.
- *Application-level check:* never, in any system with real concurrency. It isn't a lighter-weight option — it's the failure mode this section exists to rule out.

**Common failure modes:**
- **The lost response:** the `UNIQUE` constraint correctly prevents the double charge, but the server responds to the duplicate with a bare `400` or `409`. The client retried precisely because it never saw the first response — now it has no way to know whether the original operation actually succeeded.

**Example:** Stripe's `Idempotency-Key` header is the textbook implementation. A `POST` carries the header; Stripe inserts the key into a sharded PostgreSQL table with a `UNIQUE` constraint in the same transaction as the charge. If the insert succeeds, the payment runs and the response is cached against the key. If a retry arrives with the same key, the constraint violation short-circuits execution entirely, and Stripe returns the cached original response — the client experiences a seamless success either way. **[Strong Recommendation: enforce idempotency with a database uniqueness constraint, written atomically with the side effect — never as a separate read-then-write check]**

---

## The Retention Window

**What it is:** How long the server keeps an idempotency key and its cached response before purging it — the boundary past which a retry stops being recognized as a retry.

**Why it exists:** Storage isn't free, and a system processing millions of mutations a day can't keep every key and response forever. At some point the server has to be allowed to forget — which means a sufficiently late retry will eventually be treated as a new request, by design.

**Options:**
1. **Infinite retention** — the idempotency key lives permanently on the domain record itself (an `idempotency_key` column on `orders`)
2. **Finite retention window** (commonly 24–48 hours) — keys and cached responses live in a dedicated, fast store (Redis with a TTL, a table with a cleanup job) and are purged after a fixed period

**Trade-offs:**
- *Infinite retention:* mathematically safe against a retry arriving at any point in the future — but it bloats domain tables with transport-layer metadata that has nothing to do with the domain, and it can't reconstruct the original HTTP response (headers, status code), only the resulting entity.
- *Finite window:* keeps storage lean and can cache the exact original response — but it creates a real edge case: a retry that legitimately arrives after the window closes is silently treated as brand new, and the side effect runs again.

**When to choose each:**
- *Finite window:* the industry-standard default, typically 24–48 hours.
- *Infinite retention:* only when the idempotency key is the same value as a core domain identifier the client generated itself (the client mints the order ID as a UUID, merging the two concepts into one).

**Common failure modes:**
- **The late-retry double charge:** a mobile order succeeds server-side, but the connection drops before the client sees the response. The phone loses signal for three days. When it reconnects, a background worker blindly retries the original request — and because the server's 24-hour window has already expired, the key is gone and the order is created a second time.

**Example:** Stripe documents its retention window explicitly: 24 hours. A retry with the same key after that window is the client's responsibility, not a provider bug — the contract defines exactly how long a network retry has to be resolved within.

---

## Why Smart Engineers Disagree: What to Return for a Duplicate Key

The sharpest disagreement in idempotency design isn't about the mechanism — it's about what status code to return when a duplicate is actually caught.

REST purists argue a duplicate `POST` against an already-created resource is, structurally, a conflict, and should return `409 Conflict`. Returning `200 OK` or `201 Created` for a request that didn't actually create anything in that execution, to them, misrepresents what just happened.

Pragmatists argue this defeats the entire purpose of the idempotency key. The client is only retrying because the network failed it — it has no idea the first attempt succeeded. A `409` breaks its retry loop and forces it to write extra error-handling logic just to go figure out, via a separate `GET`, what state things are actually in.

The pragmatic answer wins here, and it follows directly from what idempotency is for: the `Idempotency-Key` header is a contract that says "make sure this happens exactly once, and give me the result" — not "tell me whether this specific HTTP call was the one that did it." Returning the cached success response (`200`/`201` with the original body) fulfills that contract completely. The whole point of the mechanism is to hide the network's unreliability behind a stable, idempotent abstraction; punishing the client with a `409` for a partition it didn't cause reintroduces the exact problem idempotency exists to make irrelevant.
