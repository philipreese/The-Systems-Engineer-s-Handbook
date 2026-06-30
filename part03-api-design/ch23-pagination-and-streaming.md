# Chapter 23 — Pagination and Streaming

**Prerequisites:** [Part I, Ch 06 — Cost Models and Mechanical Sympathy](../part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md), [Part II, Ch 16 — Versioning and Backward Compatibility](../part02-software-architecture/ch16-versioning-backward-compatibility.md), [Part III, Ch 20 — Resource Modeling](ch20-resource-modeling.md), [Ch 21 — Error Handling Contracts](ch21-error-handling-contracts.md). Specifically: sequential vs. random I/O cost, and backward-compatible API surfaces.

**New vocabulary introduced:** cursor (keyset pagination), page drift

**Key takeaways:**
- Any API returning a collection eventually has to decide how to hand back more results than fit in one response. That decision is a consistency and performance contract, not a UI convenience — it determines whether results stay correct under concurrent writes and how expensive a deep query gets.
- Offset/limit pagination is the simplest mental model and the one every SQL query naturally produces — but it's structurally unstable under concurrent mutation (page drift) and gets linearly more expensive with depth, because the database still has to scan and discard every row before the offset.
- Cursor-based (keyset) pagination anchors each request to a specific, indexed position instead of a relative offset, giving constant performance regardless of depth and immunity to page drift — at the cost of giving up arbitrary "jump to page N" access and requiring a genuinely unique, deterministic sort key.
- Streaming abandons the idea of discrete pages entirely for datasets that are open-ended, real-time, or simply too large to paginate sanely — trading HTTP's normal request/response statelessness for continuous delivery and the operational complexity that comes with holding a connection open.

---

## Offset/Limit Pagination

**What it is:** Returning a slice of a sorted result set by specifying how many items to skip (`OFFSET`) and how many to return (`LIMIT`).

**Why it exists:** It mirrors how people already think about a list — page 1, page 2 — and maps directly onto SQL, letting a client jump to any arbitrary point in a collection and letting a server compute total page counts trivially.

**Options:**
1. **Explicit offset** — the client supplies the raw integer (`?offset=100&limit=50`)
2. **Page numbers** — the client supplies a page number and size (`?page=3&size=50`); the server computes the offset

**Trade-offs:**
- Both give maximum client flexibility — a "jump to page 25" control is trivial to build — but both structurally guarantee **page drift**: if an item is inserted at the front of the list while a user is on page 1, page 2 now silently contains a duplicate of what used to be the last item on page 1, or skips an item entirely.
- Both also impose the depth penalty: query time grows linearly with the offset, because the database still has to read and discard every row before it, even though none of those rows are returned.

**When to choose each:**
- Acceptable only for small, rarely mutated datasets where write-safety under concurrency genuinely doesn't matter — a list of geographic regions, a bounded admin dashboard.
- Never for a high-velocity feed or a large multi-tenant table.

**Common failure modes:**
- **The deep-offset timeout:** a client requests page 10,000 of an events table — `OFFSET 500000`. PostgreSQL doesn't have a way to skip rows for free; under MVCC it has to evaluate visibility rules for every one of those half-million skipped rows just to confirm they should be discarded, burning CPU on work whose entire output is thrown away. At scale, this is effectively a self-inflicted denial-of-service vector exposed directly to the public internet.

**Example:** `SELECT * FROM events ORDER BY created_at DESC LIMIT 50 OFFSET 10000` looks innocuous and falls off a real performance cliff at depth — the database can't optimize the skip away, it has to do the work the offset implies every single time. **[Consensus: offset pagination is fine for small, static data and actively dangerous for anything large or frequently written]**

---

## Cursor-Based (Keyset) Pagination

**What it is:** Anchoring each page request to a specific, unique, sortable position in the data — "give me items after this one" — rather than a relative position counted from the start.

**Why it exists:** Because the query targets a specific indexed row instead of a relative offset, the database can jump straight to it via the index (typically a B-tree) without scanning anything that precedes it — eliminating both page drift and the depth penalty by construction, not by tuning.

**Options:**
1. **Transparent cursors** — the cursor is the entity's own identifier (`?starting_after=evt_456`)
2. **Opaque cursors** — the server encodes the sort field and unique ID together (commonly base64), so the client can't inspect or construct one itself

**Trade-offs:**
- *Cursor pagination generally:* perfectly consistent latency at any depth, and items are never skipped or duplicated under concurrent writes — but arbitrary page access is gone entirely (there's no way to ask for "page 10" without already having the cursor for it), and the sort has to be backed by a genuinely unique, deterministic, indexed ordering.
- *Transparent vs. opaque:* a transparent cursor is simpler when the client already understands the identifier format; an opaque cursor keeps the freedom to change the underlying indexing strategy later without breaking every client URL that depends on its shape.

**When to choose each:**
- *Cursor pagination:* the default for any high-throughput, frequently mutated API, infinite-scroll feed, or synchronization endpoint.
- *Opaque cursors:* the default for public APIs, specifically to preserve the right to change the index later.
- *Transparent cursors:* fine when sorting strictly by a chronological primary key the client already understands.

**Common failure modes:**
- **The non-unique sort:** a cursor is implemented sorted purely by `created_at`. Two records land in the same millisecond. With no tie-breaker, ordering between them becomes nondeterministic, and one of the two is silently skipped across a page boundary — invisible unless someone notices a record that's simply never returned.

**Example:** Stripe's `starting_after`/`ending_before` parameters, tied to an object's own unique ID, are the industry-standard case of transparent cursor pagination. GitHub's REST API does the opaque version, driving clients forward entirely through RFC 5988 `Link` headers (`Link: <https://api.github.com/...&cursor=xyz>; rel="next"`) without ever exposing the underlying index mechanics or requiring the client to construct the next URL itself.

---

## Streaming as a Non-Pagination Model

**What it is:** Abandoning discrete, bounded pages entirely in favor of a single open connection over which the server continuously pushes records.

**Why it exists:** Some result sets are genuinely open-ended, real-time, or simply too large for the overhead of issuing thousands of sequential paginated requests and tracking cursors between them to make sense.

**Options:**
1. **Newline-delimited JSON (NDJSON)** — the server flushes one JSON object per line over a standard HTTP response
2. **Server-Sent Events (SSE)** — a unidirectional event stream over one HTTP connection, natively supported by browsers
3. **gRPC server streaming** — a single RPC call yielding a continuous stream of protobuf messages over multiplexed HTTP/2

**Trade-offs:**
- Streaming radically cuts network overhead and gives the fastest possible time-to-first-byte for a large dataset — but it gives up HTTP's normal statelessness: standard load balancers can drop long-lived connections, a mid-stream failure has no obvious resume point, and memory pressure shifts onto the client, which now has to keep up with what the server is pushing or fall behind.

**When to choose each:**
- *gRPC streaming:* internal microservice bulk transfers where HTTP/2 is already controlled end-to-end.
- *NDJSON / SSE:* public-facing firehose APIs and large asynchronous exports.

**Common failure modes:**
- **Unbuffered memory exhaustion:** a service streams a five-million-row table via NDJSON, but the ORM underneath loads the entire result set into memory before writing the first byte. The process OOMs — defeating the exact physical constraint streaming was supposed to work around.
- Treating a stream like pagination and expecting to rewind it, or losing stream position on disconnect with no resumable token to recover from.

**Example:** The Twitter/X "firehose" is the canonical streaming collection — there genuinely is no page when reading the global stream of all tweets. The API just opens a persistent connection and pushes JSON objects as they happen, leaving the client responsible for consumption speed and reconnection state.

---

## Why Smart Engineers Disagree: UI Flexibility vs. System Safety

The most common version of this fight runs straight across the frontend/backend boundary.

Frontend engineers defend offset pagination because enterprise users expect a real data grid — "Page 1 of 50," with the ability to jump straight to page 25. Cursor pagination breaks that UI pattern outright, forcing a fallback to next/previous buttons or infinite scroll, which is a worse experience for someone trying to locate a specific historical record.

Backend and systems engineers see offset pagination at scale as a standing operational vulnerability. Exposing a raw `OFFSET` to the public internet is, functionally, a denial-of-service vector — a request for offset 1,000,000 forces the database to burn real CPU scanning and discarding dead rows, putting cluster stability at risk for the sake of a UI affordance.

The resolution favors the database, because the constraint here isn't a matter of taste — it's hardware physics (Ch 06). Offset pagination doesn't scale with collection size; cursor pagination does, by construction. The pragmatic answer is to mandate cursor pagination at the API boundary to guarantee stable performance regardless of depth, and let the frontend adapt its design language — next/previous, infinite scroll — to the physical constraints of the system underneath it, not the other way around. Offset pagination assumes a mostly static dataset; cursor pagination assumes a changing but orderable one; streaming assumes no stable snapshot exists at all. Picking the wrong one doesn't just cost performance — it silently redefines what "the result set" even means.
