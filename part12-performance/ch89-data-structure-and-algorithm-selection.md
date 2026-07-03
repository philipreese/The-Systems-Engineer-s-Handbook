# Ch 89 — Data Structure and Algorithm Selection in Practice

**Prerequisites:** [Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md) (the leaky ORM example), [Cost Models and Mechanical Sympathy](../part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md) (Big-O vs. cache locality, the linked-list-vs-array example), [API Surface Design](../part02-software-architecture/ch15-api-surface-design-expose-hide.md) (consumer-driven N+1 risk), [Data Ownership Boundaries](../part02-software-architecture/ch18-data-ownership-boundaries.md) (cross-service N+1 fan-out), [REST vs. RPC vs. Event-Driven](../part03-api-design/ch19-rest-vs-rpc-vs-event-driven.md) (the local-method illusion), [Resource Modeling](../part03-api-design/ch20-resource-modeling.md) (referencing vs. embedding), [Profiling-First](ch86-profiling-first.md)

**New vocabulary introduced:** None — this chapter generalizes vocabulary already introduced elsewhere in the handbook (Big-O, cache locality, N+1) rather than introducing new terms.

**Key takeaways:**
- [Consensus] Asymptotic complexity eliminates fundamentally unsuitable candidates; it does not select the winner among the survivors. Ch 06's linked-list-versus-array example already showed a "worse" algorithm on paper beating a "better" one in practice because of cache locality — this chapter generalizes that result rather than re-deriving it.
- [Strong Recommendation] Choose a data structure by measuring it at realistic data size, access pattern, and concurrency using Ch 86's profiling and benchmarking methodology — not by reasoning from a complexity table alone.
- A hash map trades ordering and cache locality for average O(1) point lookup; a sorted structure or B-tree trades some lookup speed for range queries and locality. PostgreSQL's use of B-trees for indexes — sized to the page, not to minimize comparisons — is the concrete instance of this trade-off once I/O, not comparison count, is the dominant cost.
- The N+1 problem, encountered throughout this handbook in a leaky ORM (Ch 04), a consumer-driven API (Ch 15), a cross-service fan-out (Ch 18), and the local-method illusion (Ch 19), is the same underlying failure every time: an algorithm-selection mistake at the data-access-pattern level, where a sequence of round trips substitutes for one batched or joined access. The fix is always a different access pattern, never a faster implementation of the same round trip.

---

Ch 06 already made the core argument this chapter extends: Big-O notation assumes every memory access costs the same, real hardware does not, and for any dataset that fits in cache, a theoretically worse algorithm with good locality routinely beats a theoretically better one that fights the hardware — worked through with a linked list losing to an array despite its better nominal insertion complexity. That example is not repeated here. What follows generalizes its conclusion into a practical selection discipline, and resolves N+1's recurring appearances across the handbook into the one lesson they were all instances of.

### Decision: Select by Asymptotic Complexity or by Measured Performance

**What it is:** Whether a data structure or algorithm is chosen primarily by comparing Big-O growth rates, or primarily by measuring candidates against realistic data size and access patterns using Ch 86's methodology.

**Why it exists:** Asymptotic complexity describes behavior as input size grows without bound and says nothing about the constant factor or memory-access pattern that dominates at realistic, bounded sizes — which describes most production inputs. A structure that promises better asymptotic behavior can still lose badly in practice if its layout defeats the cache.

**Options:**
- **Asymptotic-first selection** — choose based on Big-O growth rate alone.
- **Measurement-first selection** — narrow candidates by eliminating asymptotically unsuitable ones, then decide among the rest by profiling and benchmarking at realistic scale.

**Trade-offs:** Asymptotic-first selection is fast, requires no working implementation, and correctly rules out anything that will hit a genuine scaling cliff at large input — an O(n²) algorithm remains unsuitable at scale no matter how cache-friendly its constant factor is. It says nothing, however, about which of several asymptotically-acceptable options will actually be faster on real data, and routinely leads to selecting a structurally complex option that loses to something simpler at the sizes that actually occur. Measurement-first selection catches exactly what asymptotic analysis misses — constant factors, memory layout, branch prediction — at the cost of requiring a working candidate to measure in the first place.

**When to choose each:** [Consensus] Use asymptotic reasoning to reject candidates that will not scale to the input the system will actually see — that filter is cheap and catches real, unbounded failure. Make the final choice among the asymptotically viable candidates using Ch 86's profiling and benchmarking discipline at production-representative data volume, not from the complexity table alone.

**Common failure modes:** *The pointer-chasing cache stall.* An engineer selects a balanced binary tree or a chained hash table for an in-memory collection of a few hundred items, reasoning from its O(log n) or average O(1) lookup guarantee. Because its entries are allocated non-contiguously on the heap, every traversal chases pointers across scattered addresses, forcing a cache miss at nearly every step. A flat array with a nominally worse O(n) linear scan executes faster at this size, because the CPU prefetcher pulls the entire contiguous array into cache in one pass — the same mechanism, at smaller scale, as Ch 06's original linked-list example.

**Example:** Production standard libraries routinely implement hybrid sorting and collection algorithms rather than a single theoretically optimal one, and language runtimes such as Go and Rust back small maps and vectors with a flat inline array below a size threshold, deliberately suspending pointer-based structure until data volume grows large enough to offset the constant cost of memory access — because measured behavior at realistic sizes, not the complexity table, is what these implementations were tuned against.

### Decision: Hash Map, Sorted Structure, or B-Tree

**What it is:** Which structure backs a collection that needs to be searched, given the actual mix of point lookups, range queries, insertions, and ordering the workload requires.

**Why it exists:** No structure is optimal for every operation simultaneously. A hash map optimizes point lookup by giving up ordering and locality; a sorted array or B-tree optimizes range access and locality by giving up cheap insertion.

**Options:**
- **Hash map** — keys distributed across buckets by a hash function.
- **Sorted array** — keys held contiguously in sorted order.
- **B-tree** — a wide, balanced tree whose node size is tuned to a physical page or block.

**Trade-offs:** A hash map delivers average O(1) point lookup with minimal comparison, but offers no ordering guarantee, makes range queries require a full scan, and scatters entries across memory with poor cache locality since related keys hash to unrelated buckets. A sorted array gives excellent cache locality and cheap binary search and range iteration, at the cost of expensive insertion and deletion since maintaining sort order requires shifting elements. A B-tree keeps most of a sorted array's range-query and locality advantages while making insertion and deletion efficient, at the cost of a more complex implementation and more per-node comparison overhead than a simple binary search.

**When to choose each:** Hash maps where exact-key lookup dominates and ordering is irrelevant — a session cache keyed by token, a set membership check. Sorted arrays for largely static datasets with frequent range queries and iteration. B-trees wherever ordered storage must scale onto disk or any block-oriented medium — this is not a close call once data no longer fits in memory.

**Common failure modes:** *The index I/O explosion.* A custom on-disk storage engine uses a binary search tree as its primary index. Because a binary tree's branching factor is exactly two, searching millions of entries requires dozens of sequential node hops, and because each node lives at an effectively random disk location, every hop costs a full page fetch — exhausting available I/O capacity on a single query, long before CPU comparison cost becomes relevant.

**Example:** PostgreSQL uses B-tree indexes rather than binary search trees precisely because index performance is dominated by page reads, not comparisons, once data is too large to fit in memory. A B-tree's branching factor is explicitly tuned to the database's page size (commonly 8 KB), so a single disk or memory-page read retrieves dozens of keys at once — minimizing expensive page-boundary crossings is the correct optimization target once I/O, not comparison count, is what actually costs time.

### The General Lesson Behind N+1

N+1 has already appeared in this handbook four separate times, each time as a symptom local to whatever chapter was discussing it: a leaky ORM generating one query per row instead of one query for all of them (Ch 04); a consumer-driven API surface that lets a client's query shape drift toward one request per related record (Ch 15); a cross-service fan-out where a page needs data from three services and pays three round trips instead of one (Ch 18); and the local-method illusion, where a generated client stub that looks like a function call gets placed inside a loop and quietly multiplies into a network call per iteration (Ch 19). Ch 20's referencing-versus-embedding trade-off named the same cost from the data-modeling side: referencing keeps a payload small but forces the client into the same round-trip-per-record pattern to assemble a full picture.

None of those chapters were describing different problems. Each was describing the same algorithm-selection failure at the data-access layer: a sequence of independent round trips substituting for one operation that batches or joins the same work. The fix in every case was never a faster version of the per-item operation — it was replacing the access pattern itself, with a batched `WHERE id IN (...)` query, a join, or a single bulked request, collapsing what had been N round trips into one regardless of how fast any single one of them ran.

**Common failure modes:** An engineer profiles a slow endpoint, finds a per-row database query executing in a loop, and optimizes that query's execution plan — shaving milliseconds off an operation that is being run hundreds of times per request. The query got faster; the endpoint did not, because the bottleneck was never the query's execution time, it was the number of round trips. The fix that actually matters is structural: one batched query instead of many fast ones.

**Example:** Resolving an N+1 pattern, whether it crosses a database boundary or a microservice boundary, follows the same move every time: replace the per-item loop with a single consolidated request — a SQL join, a batched `IN` filter, or one bulk RPC call — compressing what had been a linear number of network or query round trips into a single one.

### Why Smart Engineers Disagree

There is little real disagreement that data structure selection should ultimately be validated by measurement — that much is close to consensus. The genuine fault line is over how far to push mechanical sympathy into ordinary application code versus reserving it for foundational infrastructure.

One position, common among engineers building storage engines, kernels, and other core infrastructure, holds that any layout introducing pointer indirection or heap fragmentation is a defect to be designed out from the start. They accept more complex code, manual memory management, and real costs on small inserts in exchange for keeping hot execution paths perfectly cache-local — to this group, wrapping data in a pointer-linked tree or a polymorphic structure is an abstraction tax that should be justified explicitly, not assumed affordable.

The opposing position, common in application and business-logic code, treats that same discipline as a poor trade when applied above the infrastructure layer: business requirements change, and a flexible object graph or associative structure that supports cheap structural change is worth its cache-miss overhead, provided the access pattern itself is sound — free of N+1 fan-out, in particular. To this group, forcing every collection into a flat memory arena in service code trades a real, ongoing maintenance cost for a performance gain the workload was never shown to need.

The resolution tracks where the collection sits, not a global rule. Inside a storage engine, a kernel, or any component whose whole job is to be fast at this one thing, cache-local layout is close to mandatory — the same argument Ch 06 already made for mechanical sympathy generally. In application and business logic, a flexible structure is the right default, right up until profiling (Ch 86) identifies it as an actual, measured bottleneck — at which point this chapter's discipline, not intuition, decides what replaces it.

---

Choosing the right structure or access pattern is a different fix from caching a slow one — Ch 88 covers the latter, and the two should not be treated as interchangeable responses to the same symptom. A structural fix removes the cost; a cache only defers or amortizes it.
