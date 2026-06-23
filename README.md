# System Design — Complete Study Guide

Self-contained teaching notes. Each concept is explained from first principles with the *why*, the mechanism, the trade-off, and an example. Reading this is meant to replace reading the books.

---

# BOOK 1 — Designing Data-Intensive Applications (Kleppmann)

The book's whole thesis: modern apps are *data-intensive* (not compute-intensive). The hard problems are storing, moving, and keeping data correct as you scale and as things fail. There is no perfect database — you compose specialized tools, and every choice is a trade-off.

## 1.1 The three goals

**Reliability — keep working when things go wrong.**
- A *fault* is one component misbehaving; a *failure* is the whole system stopping. Goal: prevent faults from becoming failures.
- Three fault classes:
  - *Hardware* — disks die (~2–5% annual failure), power fails. Mitigate with redundancy (RAID, multiple machines).
  - *Software* — a bug triggered by a specific input cascades across all nodes (correlated failure, worse than hardware). Mitigate with testing, process isolation, monitoring.
  - *Human* — operators cause most outages. Mitigate with good abstractions, sandboxes/staging, fast rollback, telemetry.
- Practical rule: deliberately trigger faults (chaos engineering) so failover code actually gets exercised.

**Scalability — cope with growth in load.**
- First *describe load* with **load parameters**: requests/sec, read:write ratio, cache hit rate, fan-out (e.g., a tweet by a celebrity fans out to millions of timelines).
- Worked example (Twitter): posting a tweet is cheap, but a *home timeline read* is expensive because it must merge tweets from everyone you follow.
  - Approach 1: compute timeline on read (query at read time) — cheap writes, expensive reads.
  - Approach 2: fan-out on write — when you tweet, push into all followers' precomputed timelines. Expensive writes, cheap reads. Twitter uses a hybrid: fan-out for normal users, read-time merge for celebrities (high fan-out would be too costly).
- Then *describe performance* with **percentiles, never averages**. p50 = median, p95/p99 = tail. The slowest 1% (p99) often hits your most valuable customers (most data, most requests). One slow backend call can dominate a request that fans out to many services ("tail latency amplification").

**Maintainability — keep it operable and changeable.**
- *Operability* — make life easy for ops (good monitoring, predictable behavior).
- *Simplicity* — fight *accidental complexity* (complexity not inherent to the problem) using good abstractions.
- *Evolvability* — make change easy; requirements always change.

## 1.2 Data models

Pick the model by how data is *related* and *accessed*.

- **Relational** — data in tables/rows; great when relationships are many-to-many; joins done by the DB. The dominant model for 30+ years because it handles arbitrary new query patterns.
- **Document** (MongoDB, JSON) — store a tree (e.g., a résumé with nested jobs). Wins for *one-to-many* tree data and *locality* (one read gets the whole document). Loses on *many-to-many* and joins (often must do them in app code).
- **Graph** (Neo4j) — when *many-to-many* dominates and relationships matter as much as data (social networks, fraud rings, road networks). Property graph + Cypher, or triple-stores + SPARQL.
- **Impedance mismatch** — the awkward translation between in-memory objects and relational tables; ORMs reduce but don't eliminate it.
- Rule of thumb: document = data comes in self-contained chunks with few cross-references; relational/graph = lots of cross-references.

## 1.3 Storage engines — how a DB actually persists bytes

Two families. Understanding them tells you a DB's performance profile.

**Log-structured (LSM-tree)** — Cassandra, RocksDB, LevelDB, Bigtable.
- Writes go to an in-memory sorted structure (**memtable**). When full, flush to disk as an immutable sorted file (**SSTable**).
- Reads check memtable, then newest→oldest SSTables.
- Background **compaction** merges SSTables, drops overwritten/deleted keys.
- **Bloom filter** (probabilistic set) tells you "key definitely not here" so you skip reading an SSTable from disk.
- Profile: *excellent write throughput* (sequential appends, no random in-place writes), reads can touch multiple files.

**B-tree** — every major RDBMS (Postgres, MySQL/InnoDB, SQL Server).
- Fixed-size pages (e.g., 4KB) in a balanced tree; updates happen *in place*.
- A **write-ahead log (WAL)** records every change before applying it, so a crash mid-write can be recovered.
- Profile: *predictable, low read latency*; strong transactional story; write amplification from in-place page updates.

**The trade-off (RUM):** LSM = better writes, more read/compaction work; B-tree = better reads, more write amplification. Choose by your read:write ratio.

## 1.4 Encoding & schema evolution

Data outlives code, so old and new formats coexist. Two compatibility directions you must preserve:
- **Backward compatibility** — *new* code can read *old* data (usually easy).
- **Forward compatibility** — *old* code can read *new* data (harder; old code must ignore unknown fields).

Formats:
- *Text* (JSON, XML, CSV) — human-readable, verbose, ambiguous types (numbers, no binary).
- *Binary schema* (Protobuf, Thrift) — fields identified by **tag numbers**, not names. Rules: you may add new fields only as *optional* (or with defaults); you may *never* reuse or change a tag number.
- *Avro* — no tag numbers; uses a writer schema + reader schema resolved at read time. Great for large datasets / data pipelines where schemas evolve.

## 1.5 Replication — same data on multiple nodes

Why: tolerate failures, serve reads locally (lower latency), scale read throughput.

**Single-leader (master-slave):** all writes go to the leader; leader streams a *replication log* to followers; reads can hit any replica.
- *Synchronous* replication = leader waits for follower ack (no data loss, but blocks if follower is slow). *Asynchronous* = leader doesn't wait (fast, but a leader crash loses un-replicated writes). Most systems use *semi-sync*: one sync follower, rest async.
- **Failover** (promote a follower) is dangerous: risk of lost async writes, **split brain** (two leaders), and wrong timeout tuning.

**Multi-leader:** multiple nodes accept writes (multi-datacenter, offline-capable apps). Cost: **write conflicts** — same record edited in two places. Resolve via:
- *Last-write-wins (LWW)* — simple, silently drops data.
- *Version vectors* — track causality, detect concurrent writes.
- *Application merge* — app decides (e.g., merge two shopping carts).

**Leaderless (Dynamo-style):** Cassandra, Riak. Client (or coordinator) writes to many nodes; reads from many.
- **Quorum**: with `n` replicas, require `w` write acks and `r` read responses; if `w + r > n`, read and write sets overlap so a read sees the latest write.
- Staleness repair: *read repair* (fix stale replicas during a read) and *anti-entropy* (background sync).

**Replication lag causes anomalies** — these are guarantees you may need to add:
- *Read-your-own-writes* — after you post, you must see your post (route your reads to leader briefly).
- *Monotonic reads* — never see newer-then-older data on refresh (pin a user to one replica).
- *Consistent prefix reads* — see writes in causal order (don't see an answer before its question).

## 1.6 Partitioning (sharding) — split data across nodes

Needed when data/throughput exceeds one machine.

- **By key range** — keys A–F on node 1, G–M on node 2... Range scans are easy, but uneven access creates **hotspots** (e.g., all of today's timestamped data on one node).
- **By hash of key** — spreads load evenly, but kills efficient range queries.
- **Hot key fix** — append a random suffix to spread a single hot key across partitions (then reads must query all suffixes).

**Secondary indexes** under partitioning:
- *Local (document-partitioned)* — each partition indexes its own docs; writes are simple, but a query by secondary key must hit *all* partitions (scatter/gather).
- *Global (term-partitioned)* — index itself is partitioned by the indexed term; reads are efficient, writes touch multiple partitions.

**Rebalancing** (when adding nodes): never use `hash mod N` (changing N reshuffles almost everything). Instead use a *fixed large number of partitions* assigned to nodes, or *consistent hashing*.

**Request routing** — how a client finds the right partition: a routing tier, a coordination service (ZooKeeper), or gossip among nodes.

## 1.7 Transactions — grouping operations so they're all-or-nothing

**ACID** (note: each vendor interprets loosely):
- *Atomicity* — all writes commit or none do (abort on error).
- *Consistency* — app-level invariants hold (this one is really the app's job).
- *Isolation* — concurrent transactions don't step on each other.
- *Durability* — committed data survives crashes (WAL, replication).

**Race conditions** isolation must prevent: dirty reads/writes, read skew, lost updates, write skew, phantoms.

**Isolation levels (weak → strong):**
- *Read Committed* — no dirty reads, no dirty writes. The common default.
- *Snapshot Isolation (MVCC)* — each transaction reads a consistent snapshot as of its start; readers never block writers. Implemented by keeping multiple row versions. Great default for read-heavy systems.
- *Serializable* — strongest: result is *as if* transactions ran one at a time. Implementations:
  - *Actual serial execution* — single-threaded (Redis, VoltDB); works if transactions are short and fit in memory.
  - *Two-phase locking (2PL)* — acquire locks, release at commit; correct but slow (lots of blocking, deadlocks).
  - *Serializable Snapshot Isolation (SSI)* — *optimistic*: let transactions run, detect conflicts at commit, abort losers. Good performance when contention is low (modern Postgres).

## 1.8 The trouble with distributed systems

Distributed = partial failure, and you often *can't tell* what failed.
- **Networks are unreliable** — a message may be lost, delayed, or the reply lost. You can't distinguish "node dead" from "node slow" from "network dropped it" — only timeouts, which are guesses.
- **Clocks are unreliable** — *time-of-day clocks* drift and can jump backward (NTP sync). **Never order events by wall-clock time.** Use *logical clocks*: Lamport timestamps (total order) or version vectors (causality). *Monotonic clocks* are fine for measuring elapsed durations only.
- **Process pauses** — a GC pause or VM migration can freeze a node for seconds; it wakes up thinking no time passed (dangerous if it still holds a lease/lock — use **fencing tokens**: a monotonically increasing number checked by the resource).
- **Byzantine faults** — nodes actively lie. Usually assumed away except in adversarial settings (blockchain, aerospace).

## 1.9 Consistency & consensus

- **Linearizability** — the strongest single-object guarantee: the system behaves as if there's *one copy* and every operation is atomic at some instant. Makes distributed data feel like a single machine. Cost: requires coordination, hurts availability under partitions, adds latency.
- **CAP theorem** — *when a network Partition happens*, you must choose *Consistency* (refuse requests to stay correct) or *Availability* (serve possibly-stale data). It says nothing when there's no partition. The real everyday trade-off is **latency vs consistency** (captured better by **PACELC**: if Partition then A-or-C, Else Latency-or-Consistency).
- **Causality** — a weaker, cheaper ordering: preserve cause→effect, allow concurrent events to differ. Often the sweet spot. Tracked with version vectors / Lamport clocks.
- **Consensus** — getting nodes to agree on one value despite failures. Equivalent to *total-order broadcast* and to a *linearizable compare-and-set*. Algorithms: **Paxos** (and Multi-Paxos), **Raft**, **Zab** (ZooKeeper), Viewstamped Replication. Used for leader election, distributed locks, config/membership. etcd and ZooKeeper are consensus-backed coordination services.
- **2PC (two-phase commit)** — atomic commit across multiple nodes: a coordinator asks all to *prepare* (promise they can commit), then tells all to *commit*. Problem: if the coordinator dies after prepare, participants block holding locks (it's a *blocking* protocol, and the coordinator is a SPOF).

## 1.10 Batch and stream processing

- **Batch** (MapReduce, Spark) — process a large *bounded* dataset; high throughput, not low latency. Joins via sort+shuffle (partition both sides by join key). Output is immutable, re-runnable.
- **Stream** (Kafka, Flink) — process *unbounded* data as it arrives. Treat the **event log** as the source of truth.
  - **CDC (Change Data Capture)** — turn a DB's writes into a stream to feed caches, search indexes, warehouses.
  - **Event sourcing** — store the immutable sequence of *events*; current state is a derived view (replayable, auditable).
  - **Exactly-once** processing = idempotency + atomically committing output-and-offset together.
- Mindset: most data systems are *derived data* (caches, indexes, materialized views) computed from a source of truth. Design the dataflow.

**Book-level takeaway:** think in terms of *which guarantees you actually need* (often weaker than linearizable), pick storage/replication/partitioning to match access patterns, and remember every property is bought with a trade-off.

---

# BOOK 2 — Fundamentals of Software Architecture (Richards & Ford)

This book defines *what architecture is* and how to think like an architect: not memorizing patterns, but reasoning about trade-offs.

## 2.1 The architect's mindset

- Architecture = the decisions that are **hard to change later** (structure, characteristics, styles). Detailed code design is cheaper to change.
- **First Law of Software Architecture:** *Everything is a trade-off.* If someone thinks they found a no-downside option, they haven't yet found the downside.
- **Second Law:** *Why is more important than how.* The reasoning behind a decision matters more than the decision.
- What an architect does: define architecture characteristics, choose a style, set technical direction, *guide* teams (not dictate), keep technically current, and mediate between business and engineering.

## 2.2 Architecture characteristics (the "-ilities")

These are the *non-functional requirements* the system must satisfy. Categories:
- **Operational** — availability, performance, scalability (handle more load), elasticity (handle *bursts*), recoverability, reliability.
- **Structural** — modularity, maintainability, deployability, testability, configurability, extensibility.
- **Cross-cutting** — security, usability, accessibility, privacy, compliance.

Rules:
- They are often *implicit* — you must extract them from vague business goals ("we're expanding to new markets" → internationalization + scalability).
- **Pick 3–7 driving characteristics.** Supporting all of them is impossible; many conflict (e.g., security vs performance, scalability vs simplicity). Prioritize explicitly.
- They should be *measurable* (define a fitness function for each — see below).

## 2.3 Measuring coupling (the core analytical tools)

- **Connascence** — two components are connascent if changing one requires changing the other. Strength, from weak to strong:
  - *Static* (visible in source): of Name, Type, Meaning (e.g., magic value 0=active), Position (argument order), Algorithm.
  - *Dynamic* (only at runtime, worse): of Execution (order), Timing, Value (two values must stay consistent), Identity (two refs to the same entity).
  - Guidance: minimize connascence overall; convert dynamic → static where possible; keep connascent things *close together* (locality).
- **Afferent coupling (Ca)** — number of things that depend *on* a component (incoming).
- **Efferent coupling (Ce)** — number of things a component depends *on* (outgoing).
- **Instability** `I = Ce / (Ce + Ca)` — 0 = stable (hard to change, many depend on it), 1 = unstable.
- **Abstractness** `A` = abstract types / total types.
- **Distance from main sequence** `D = |A + I − 1|` — components far from the `A + I = 1` line are in the *zone of pain* (concrete + stable → rigid) or *zone of uselessness* (abstract + unstable).

## 2.4 Architecture styles (with honest trade-offs)

Each style is a default set of trade-offs. Star ratings in the book are relative.

- **Layered (n-tier)** — presentation/business/persistence/database layers.
  - *Pros:* simplest, cheapest, familiar, good for small apps.
  - *Cons:* poor scalability/deployability/elasticity. The **architecture sinkhole anti-pattern**: requests that just pass straight through layers adding no value.
- **Pipeline (pipes & filters)** — data flows through stateless filters (Unix pipes, ETL).
  - *Pros:* composability, simplicity. *Cons:* not for rich interactive apps.
- **Microkernel (plugin)** — small core + plugins (IDEs, browsers, Eclipse).
  - *Pros:* extensibility, feature isolation. *Cons:* limited scalability; plugin contracts get messy.
- **Service-based** — a handful of coarse-grained *domain services* sharing one database, behind a UI.
  - *Pros:* a pragmatic middle ground — much of the benefit of microservices without the distributed-data pain. *Cons:* not independently scalable per fine-grained function.
- **Event-driven** — components communicate via async events.
  - *Pros:* high scalability, performance, responsiveness, decoupling. *Cons:* hard to test/debug, eventual consistency, complex error handling. Two topologies:
    - *Broker* — events flow node-to-node with no central coordinator; maximum decoupling and scale, but no place to handle/track overall workflow errors.
    - *Mediator* — a central event mediator orchestrates a multi-step workflow; better error handling and visibility, lower scalability, mediator is a bottleneck.
- **Space-based** — keep data in a replicated in-memory grid; remove the central DB from the request path; sync to DB asynchronously.
  - *Pros:* extreme, elastic scalability (handles unpredictable spikes — concert ticket sales). *Cons:* very complex, expensive, hard caching/consistency.
- **Microservices** — many small, independently deployable services, each owning its data, modeled on **bounded contexts**.
  - *Pros:* deployability, testability, scalability, evolvability, fault isolation. *Cons:* distributed-system complexity, network latency, hard data consistency, operational overhead.

## 2.5 Architect soft skills & governance

- **Diagramming & presenting** — C4 model (Context, Container, Component, Code) is a clean way to draw at multiple zoom levels.
- **Architecture Decision Records (ADRs)** — a short immutable doc per significant decision: *Title, Status (proposed/accepted/superseded), Context, Decision, Consequences*. Captures the *why* (Second Law). Never edit an old one — supersede it.
- **Fitness functions** — automated tests for architecture characteristics, run in CI. Examples: a test asserting the presentation layer never imports the persistence layer (ArchUnit); a performance test failing the build if p99 > 200ms; a check that no service has a cyclic dependency. This is the engine of **evolutionary architecture** — you can refactor freely because the fitness functions guard your characteristics.

**Book-level takeaway:** an architect's job is identifying the driving characteristics, choosing the style whose trade-offs best serve them, and encoding those decisions (ADRs) and constraints (fitness functions) so the system evolves without eroding.

---

# BOOK 3 — Software Architecture: The Hard Parts (Ford, Richards, Sadalage, Dehghani)

There are no best practices for the genuinely hard distributed decisions — only trade-off analysis. The book teaches a *method*: identify the hard problem, enumerate options, weigh trade-offs in context.

## 3.1 Pulling apart a monolith

- **Component-based decomposition** — break a monolith gradually: identify and size components, flatten them, measure coupling between them, group into domains, then extract domain services. Safer than a "big bang" rewrite.
- **Tactical forking** — instead of extracting a service out, *clone* the whole monolith for a team and delete everything they don't need. Sometimes deleting is easier than extracting.

## 3.2 Choosing service granularity (the central tension)

- **Disintegrators** — forces pushing you to split a service smaller:
  - service scope/function (single responsibility), code volatility (one part changes constantly), scalability (one part needs to scale alone), fault tolerance (isolate a crash-prone part), security (isolate sensitive data), extensibility (room to add).
- **Integrators** — forces pushing you to keep things together:
  - database transactions (need ACID across the data), workflow coupling (chatty back-and-forth = latency + fragility), shared code, tight data relationships.
- Right granularity = where disintegrator pressure outweighs integrator pressure. Too fine = "grain trap": death by network calls and distributed transactions.

## 3.3 Breaking apart the data

- **Five-step database decomposition:** (1) analyze table coupling, (2) split into logical schemas, (3) move to separate connections, (4) move to separate database servers, (5) fully independent databases. Doing it in stages de-risks it.
- **Polyglot persistence** — pick the right datastore per service (KV, document, graph, relational, search). Cost: more operational surface.
- After splitting, **joins across services disappear** — you now resolve relationships via service calls, replication, or denormalization.

## 3.4 Distributed transactions & sagas

- You lose ACID across services. Replace with a **saga**: a sequence of local transactions, each with a **compensating transaction** to undo it if a later step fails.
  - **Orchestrated saga** — a central orchestrator service drives each step and issues compensations. Easier to understand and to handle errors; introduces central coupling and a coordinator to maintain.
  - **Choreographed saga** — services emit and react to events with no central coordinator. Highly decoupled and scalable; *much* harder to trace, reason about, and recover.
- The book formalizes 8 saga patterns as a matrix of three axes: communication (sync/async) × consistency (atomic/eventual) × coordination (orchestrated/choreographed) — e.g., "Epic Saga" (sync + atomic + orchestrated, transactional but slow) vs "Anthology Saga" (async + eventual + choreographed, scalable but complex).
- Reality check: compensations themselves can fail; "undo" isn't always possible (you can't un-send an email). Design for it.

## 3.5 Contracts & coupling between services

- **Strict contracts** (gRPC/protobuf, fixed schema) — guarantees and tooling, but tight coupling; a change can break consumers (brittle, versioning pain).
- **Loose contracts** (JSON, GraphQL) — flexible and evolvable, but you lose guarantees; need **consumer-driven contract tests** to catch breakage.
- **Stamp coupling** — passing a big payload when the callee needs two fields: wastes bandwidth and couples consumers to the whole structure. Send only what's needed.

## 3.6 Distributed data ownership & access

- **Single writer ownership** — exactly one service may write a given data domain; others read via it. When multiple services seem to need write access, resolve with: table split, data-domain (shared bounded data set), delegation (one owner, others request changes), or service consolidation.
- **Distributed data access** patterns when a service needs data it doesn't own: synchronous inter-service call (simple, couples + adds latency), column replication (copy needed columns, eventual consistency), replicated cache (fast reads, memory + staleness), data domain (shared schema between specific services).
- **Code reuse** options: shared *library* (compile-time coupling, version skew) vs shared *service* (runtime coupling, latency) vs **sidecar / service mesh** (cross-cutting concerns like logging/auth/mTLS pushed into a sidecar container, keeping app code clean).

**Book-level takeaway:** for every hard distributed decision, write down the options and their trade-offs against *your* driving characteristics, and record the decision in an ADR. There is no universally correct answer.

---

# BOOK 4 — Building Microservices, 2nd ed (Sam Newman)

The practical, end-to-end guide to designing, splitting, deploying, and operating microservices.

## 4.1 What a microservice actually is

- An **independently deployable** service modeled around a **business domain**, owning its own data and exposing a network interface.
- The litmus test: *can you deploy one service to production without deploying any other?* If not, you have a **distributed monolith** — all the cost of distribution, none of the benefit.
- Model boundaries on **Domain-Driven Design bounded contexts**: hide the internal model, expose only an explicit contract. Information hiding at the service level.

## 4.2 Coupling, ranked (prefer the top)

- **Domain coupling** — service A needs service B's capability because the business genuinely connects them. Intrinsic and acceptable; keep it minimal.
- **Pass-through coupling** — A passes data through B to C; changes ripple.
- **Common coupling** — services share a mutable resource (the classic *shared database*); a schema change breaks everyone.
- **Content coupling** — A reaches into B's internals/database directly. The worst; destroys independent deployability. Forbid it.
- Goal everywhere: **high cohesion** (related behavior lives together) + **low coupling** (services know little about each other).

## 4.3 When (and when not) to do microservices

- Good reasons: independent deployability/scaling, team autonomy at scale, technology flexibility, fault isolation.
- Bad reasons: resume-driven, "everyone's doing it." For a new product with an unclear domain, **start with a monolith** — you don't yet know the right boundaries, and getting them wrong in a distributed system is very expensive.

## 4.4 Splitting a monolith incrementally

- **Strangler fig** — put a proxy in front of the monolith; redirect functionality to new services one slice at a time until the monolith is gone.
- **Branch by abstraction** — introduce an abstraction layer in the code, build the new implementation behind it, switch over, delete the old.
- **Parallel run** — run old and new implementations simultaneously, compare results, build confidence before cutting over (great for risky logic like billing).
- **Decorating collaborator** / **change data capture** — trigger new behavior off the monolith's calls or its database changes without modifying it.
- Splitting the database is the hard part: do it deliberately (shared static data, dedicated tables per context, split schema before splitting code).

## 4.5 Communication styles

- **Synchronous blocking** (request-response, REST/gRPC) — simple and intuitive, but creates *temporal coupling* (both must be up) and risks chains where one slow service stalls everything.
- **Asynchronous non-blocking** (events via a broker) — better decoupling and resilience; the producer doesn't wait. Costs: harder to follow flow, eventual consistency, broker to operate.
- Patterns: request/response, **event-driven** (services react to facts), common-data (shared store).
- Tech menu: REST (ubiquitous), gRPC (fast, schema, internal), GraphQL (client-driven aggregation across services), message brokers (Kafka, RabbitMQ).
- **Avoid** synchronous call chains A→B→C→D — latency and failure probability multiply.

## 4.6 Workflow across services

- **Orchestration** — one service is the "brain" telling others what to do (explicit, easy to follow, central coupling).
- **Choreography** — services react to events with no central brain (decoupled, scalable, harder to see the whole flow).
- For multi-step distributed business transactions, use **sagas** with compensations; do *not* attempt distributed two-phase commit (blocking, fragile).

## 4.7 Deployment & progressive delivery

- One service = one independently releasable artifact (container). Orchestrate with Kubernetes. Use infrastructure-as-code.
- Each service gets its own CI/CD pipeline.
- Separate **deployment** from **release** to reduce risk: feature flags, **canary** (route a small % of traffic to the new version, watch metrics), **blue-green** (two environments, flip traffic), **parallel run**.

## 4.8 Resilience (you must design for failure)

- A distributed system *will* have partial failures. Apply stability patterns (from Release It!): **timeouts** on every call, **retries with backoff + jitter** (only for idempotent ops), **bulkheads** (isolate resource pools), **circuit breakers** (stop calling a failing dependency).
- Understand your CAP posture per data flow; decide where you need consistency vs availability.

## 4.9 Observability (you can't SSH into 200 containers)

- **Three pillars:** structured **logs** with a **correlation ID** threaded through every hop; **metrics** (aggregate counters/gauges); **distributed traces** (one request's path across services with timing).
- Alert on **user-facing symptoms / SLOs** (error rate, latency), not on raw machine metrics — reduces noise and false alarms.

## 4.10 Security & teams

- **Zero-trust** instead of "internal network is safe": authenticate every service-to-service call (mTLS, JWT/service identity), least privilege, central secrets management. Beware the **confused deputy** (a service tricked into using its privileges on an attacker's behalf).
- **Conway's Law** — your architecture inevitably mirrors your org's communication structure. Use the **Inverse Conway Maneuver**: design teams (small, autonomous, **stream-aligned**, owning services end-to-end) to produce the architecture you want.

**Book-level takeaway:** microservices buy *independent deployability* at the price of *distributed-system complexity*. Only pay it when you need it, split incrementally, design for failure and observability from day one, and shape your teams to match your services.

---

# BOOK 5 — Understanding Distributed Systems (Roberto Vitillo)

A gentler, well-structured build-up of the same distributed concepts as DDIA — excellent for cementing the mental model. Organized as Communication → Coordination → Scalability → Resiliency → Operations.

## 5.1 Communication

- **TCP** gives reliable, ordered, connection-based delivery; **UDP** is fire-and-forget. **TLS** adds encryption, authentication (certs), and integrity on top.
- Higher-level styles: **RPC** (call a remote function — feels local, hides the network's dangers), **REST** (resources + HTTP verbs, stateless, cacheable), **messaging** (async via a broker).
- Crucial failure insight: when a request fails, the caller often *cannot know* whether the server processed it (failure could be before, during, or after, or just the reply was lost). Therefore:
- **Idempotency is mandatory** for anything you might retry — design operations so doing them twice equals doing them once (e.g., use a client-supplied request ID the server dedupes on).

## 5.2 Coordination

- **Failure detection** — heartbeats + timeouts. Fundamental limit: you cannot distinguish a *slow* node from a *dead* one.
- **Leader election** — pick one coordinator; requires consensus to be safe.
- **Raft** (taught in depth because it's understandable):
  - *Leader election* — time divided into *terms*; a node times out, becomes candidate, requests votes; majority wins.
  - *Log replication* — clients send commands to the leader; it appends to its log and replicates (AppendEntries) to followers; once a majority has an entry, it's **committed** and applied to the state machine.
  - *Safety* — only a node with an up-to-date log can become leader, guaranteeing committed entries are never lost.
- Consensus is the building block for **replicated state machines**, **distributed locks**, **leader/config/membership** services (etcd, ZooKeeper).

## 5.3 Replication & consistency models (strongest → weakest)

- **Linearizable / strong** — one-up-to-date-copy illusion; every read sees the latest write. Most expensive.
- **Sequential** — all nodes see operations in the same order (not necessarily real-time order).
- **Causal** — preserves cause→effect ordering; concurrent operations may differ. *Often the practical sweet spot* — cheap and intuitive.
- **Eventual** — replicas converge *if writes stop*; cheapest, but reads can be stale/out of order.
- **CALM theorem** — programs expressible with *monotonic* logic (only ever add facts, never retract) can be consistent **without coordination** — a principled way to know when you can drop expensive coordination.

## 5.4 Scalability toolkit

- **Functional decomposition** — split by capability: API gateway, **backend-for-frontend (BFF)** (a tailored API per client type), microservices.
- **Partitioning** — split data: range vs hash; **consistent hashing** to minimize movement when nodes change; watch for hot partitions.
- **Duplication** — copy data closer/wider:
  - **CDN** at the network edge for static assets.
  - **Caching** — *cache-aside* (app checks cache, loads on miss), *write-through* (write cache + DB together), *write-behind* (write cache now, DB async). Eviction by **TTL** and **LRU**. Watch cache **stampede** on expiry.
  - **Read replicas** for read scaling.
- **Asynchronism** — a **message queue** decouples producer and consumer, *buffers* spikes (load leveling), and enables retries + dead-letter queues. **Competing consumers** scale processing horizontally.
- **Load balancing** — DNS / L4 (transport) / L7 (HTTP-aware); health checks remove dead backends; **service discovery** lets services find each other dynamically.

## 5.5 Resiliency

- **Common failure causes:** single points of failure, *unbounded* resource consumption, cascading failures, slow dependencies, **retry storms** (everyone retries at once and finishes off a struggling service).
- **Protect downstream** (the thing you call): timeouts, retries with **exponential backoff + jitter**, **circuit breaker**, **bulkhead**.
- **Protect yourself from upstream** (callers): **load shedding** (reject early when overloaded), **rate limiting**, **backpressure** (signal callers to slow down).
- **Self-healing** — health checks + auto-restart, redundancy, autoscaling, so the system recovers without a human.

## 5.6 Operations

- **Observability** — metrics (RED: Rate, Errors, Duration; USE: Utilization, Saturation, Errors), logs, traces. Define **SLI** (measured indicator) → **SLO** (target) → **SLA** (contract); spend the **error budget** on change velocity.
- **Releases** — canary, blue-green, automated rollback. **Chaos/fault injection** to prove resilience. **Runbooks** for on-call.

**Book-level takeaway:** distributed systems are about communicating safely despite uncertainty, coordinating via consensus, scaling via partition+duplicate+async, and surviving via timeouts/breakers/bulkheads with real observability.

---

# BOOK 6 — Database Internals (Alex Petrov)

Two halves: how a single-node storage engine works, and how distributed databases coordinate. This is the "under the hood" depth that makes you reason about DB performance correctly.

## 6.1 Storage engines (single node)

**B-Tree family** (read-optimized, in-place updates):
- A balanced tree with high **fan-out** (each node holds many keys → shallow tree → few disk seeks; lookups are O(log n) with a tiny base).
- **B+Tree** (what real databases use): keys in internal nodes for routing only; *all actual data lives in leaf nodes*, and leaves are linked in a list → efficient range scans.
- Disk layout is **page-based** (fixed-size blocks). **Slotted pages** store variable-length records: a header with slot pointers grows from one end, data from the other.
- Node **splits** (on insert overflow) and **merges/rebalances** (on delete underflow) keep it balanced.
- **Write-ahead log (WAL)**: append the change to a durable log *before* modifying pages, so a crash can be recovered by replaying the log. Uses **checkpoints** to bound recovery time, and **steal/no-force** buffer policies. This is what gives B-tree DBs durability.
- **Copy-on-write B-trees** (LMDB): never overwrite a page in place — write new versions and atomically swap the root. Gives lock-free reads and crash safety without a WAL.

**LSM-Tree family** (write-optimized, append-only):
- Buffer writes in an in-memory sorted **memtable**; flush full memtables to immutable on-disk **SSTables**.
- **Compaction** merges SSTables in the background, discarding overwritten/deleted keys. Strategies: *size-tiered* (merge similarly-sized files; write-cheap, more space/read cost) vs *leveled* (non-overlapping levels; read/space-cheap, more write cost).
- Read helpers: **bloom filters** (skip files that definitely lack the key) and **fence pointers / sparse index** (jump to the right block).

**The three amplifications (the unifying lens):**
- *Read amplification* — extra reads per logical read (LSM may check many files).
- *Write amplification* — extra writes per logical write (B-tree page rewrites; LSM compaction rewrites).
- *Space amplification* — extra storage vs logical data (LSM keeps obsolete versions until compaction).
- **RUM conjecture:** you can optimize for at most two of *Read*, *Update*, *Memory(space)* — the third pays. Every engine is a point in this space.
- The **buffer pool / page cache** keeps hot pages in memory; eviction via **LRU** or **CLOCK**.

## 6.2 Distributed databases (the coordination half)

- **Failure detectors** — heartbeats, **phi-accrual** (outputs a suspicion level instead of a binary up/down), **gossip** (nodes randomly exchange state — epidemic spread, scales well).
- **Leader election** algorithms — bully, ring.
- **Replication & consistency** — single-leader/multi-leader/leaderless, quorums (`w + r > n`), **read repair**, **hinted handoff** (temporarily store writes for a down node), **anti-entropy**.
- **Anti-entropy with Merkle trees** — a hash tree over data ranges lets two replicas compare top-down and find exactly which ranges diverge, syncing only those (cheap reconciliation).
- **Distributed transactions:**
  - **2PC** — prepare then commit; blocks if coordinator dies.
  - **3PC** — adds a pre-commit phase to be non-blocking in theory; still fails under network partitions.
  - **Calvin** — deterministic transaction ordering avoids coordination cost.
  - **Spanner** — combines Paxos replication with **TrueTime** (GPS+atomic clocks giving bounded time uncertainty) to provide global linearizable transactions by waiting out the uncertainty window.
- **Consensus** — Paxos (single-decree and Multi-Paxos for a log), Raft, Zab; failure scenarios like **dueling proposers** (two nodes endlessly competing) motivate having a stable leader.

**Book-level takeaway:** a database's whole performance personality comes from its storage engine's position on the read/write/space trade-off plus its replication/consensus choices. Know the engine, predict the behavior.

---

# BOOK 7 — Release It! 2nd ed (Michael Nygard)

The book about what actually breaks systems *in production* and the patterns that keep them alive. Written from real outage post-mortems. This maps directly onto reliability/self-healing work.

## 7.1 Stability anti-patterns (the things that cause outages)

- **Integration points** — every call to another system (DB, API, socket) is the #1 source of instability: it can hang, return garbage, or never reply. Every integration point needs protection.
- **Chain reactions** — in a horizontally scaled pool, when one node dies its load shifts to the survivors, pushing them over the edge too, one by one.
- **Cascading failures** — a failure in one tier propagates *across* tiers through integration points (a dead DB takes down every service that calls it). Circuit breakers stop the spread.
- **Users** — real traffic is hostile: too many of them, expensive sessions (memory per session × users = OOM), bots/scrapers. Model the worst case.
- **Blocked threads** — the most common way apps hang: all request threads stuck waiting on a lock, a slow downstream call, or an exhausted connection pool. The app is "up" but serves nothing.
- **Slow responses** — *worse than outright failures* because they tie up the caller's resources (threads, connections) while it waits. Better to **fail fast** than to be slow.
- **Unbounded result sets** — code assumes a query returns a handful of rows; one day it returns 10 million and blows up memory. Always paginate / LIMIT.
- **Dogpile / thundering herd** — many clients hit at the same instant (cache expiry, cron at :00, fleet restart, retry sync), spiking load.

## 7.2 Stability patterns (the toolkit that prevents the above)

- **Timeouts** — put a timeout on *everything*: network calls, connection-pool checkout, lock acquisition. Never wait forever.
- **Circuit Breaker** — wrap a risky integration point. After N consecutive failures it **trips open** and fails fast immediately (no waiting on the dead dependency); after a cooldown it goes **half-open** to test one request; success **closes** it. Protects both the caller (stops wasting resources) and the callee (stops the hammering).
- **Bulkheads** — partition resources so one failure can't sink everything (named after ship compartments). E.g., separate thread/connection pools per downstream dependency, or per tenant, so one bad dependency starves only its own pool.
- **Steady State** — the system must run indefinitely without human cleanup: purge old data, rotate/limit logs, cap cache sizes. Any unbounded growth is a future outage.
- **Fail Fast** — check resource availability and validate inputs *before* doing expensive work, so you reject quickly instead of failing slowly halfway through.
- **Let It Crash** — sometimes the cleanest recovery is to kill the component and restart it to a known-good state fast (Erlang/supervisor model), rather than trying to repair in place.
- **Handshaking / Backpressure** — let a receiver tell senders to slow down before it's overwhelmed.
- **Test Harness** — test against *nasty* failures, not polite ones: connections that accept but never respond, slow byte-by-byte responses, garbage payloads. Production failures are rarely clean.
- **Shed Load** — under overload, reject early (HTTP 429) to protect the core rather than collapsing under everything.
- **Governor** — slow down dangerous automated actions (e.g., an autoscaler scaling *down* too aggressively), giving humans time to react.

## 7.3 Designing for production

- **Transparency** — build in observability from the start: structured logs, metrics, health-check endpoints, so you can see inside a running system.
- **Recovery-oriented** — fast restarts, blue-green deploys, feature flags, decoupled releases so you can roll back instantly.
- **Networking realities** — virtual IPs, DNS caching gotchas, multihomed hosts binding to the wrong interface.
- **Capacity** — find the binding constraint, load-test to the *breaking point* (not just expected load), and know your scaling ceiling before traffic finds it for you.

**Book-level takeaway:** systems run for years and fail in partial, ugly ways. Treat every external call as a liability, bound every resource, fail fast, and isolate blast radius with timeouts, circuit breakers, and bulkheads. Design for the 3am incident.

---

# BOOK 8 — Designing Distributed Systems (Brendan Burns)

A catalog of *reusable patterns* for distributed systems, framed around containers/Kubernetes. Think of it as design patterns (GoF) but for distributed operations — it gives you vocabulary and building blocks.

## 8.1 Single-node patterns (multiple containers cooperating in one pod)

- **Sidecar** — add a helper container alongside the main app container to extend it without touching its code: TLS termination, logging/metrics shipping, config sync, a proxy. This is the pattern underneath a **service mesh** (each pod gets a proxy sidecar handling mTLS, retries, routing).
- **Ambassador** — a sidecar that brokers the app's *outbound* connections to the world: sharding requests to the right database shard, service discovery, request throttling. The app just talks to localhost; the ambassador handles the messy outside.
- **Adapter** — a sidecar that *normalizes* the app's interface to the outside: convert its log format or expose metrics in a standard shape (e.g., Prometheus) so heterogeneous apps look uniform to your tooling.

## 8.2 Multi-node serving patterns

- **Replicated load-balanced service** — N identical stateless replicas behind a load balancer; use **readiness probes** so traffic only goes to ready instances; handle session affinity if needed. The default way to scale stateless work.
- **Sharded service** — when state is too big for one node, partition it across shards by a sharding function; deal with hot shards and add a replicated cache in front.
- **Scatter/Gather** — fan a request out to all shards in parallel (a *root* node to many *leaf* nodes), then combine the partial results (search engines work this way). Latency is bound by the *slowest* leaf (tail-latency sensitive).
- **Functions / FaaS** — event-triggered, ephemeral, stateless compute. Great for glue and spiky event handling; bad for chatty, long-lived, or stateful workloads (cold starts, cost, no local state).
- **Ownership election** — use leader election to guarantee exactly one instance owns a singleton responsibility (e.g., one scheduler) even with replicas running.

## 8.3 Batch computational patterns

- **Work queue** — a queue of discrete tasks fed to a pool of stateless workers (containers as interchangeable "task transformers"); scale by adding workers.
- **Event-driven coordination** — compose pipelines from small reusable stages: *copier, filter, splitter, sharder, merger*.
- **Coordinated batch** — multi-stage batch with **joins** and **reduce** steps and a **barrier** between stages (stage 2 waits until all of stage 1 finishes).

**Book-level takeaway:** most operational concerns (TLS, logging, sharding, discovery, fan-out) are recurring patterns you can implement *outside* your app code using containers — giving you a shared vocabulary and keeping business logic clean.

---

# BOOK 9 — System Design Interview Vol 1 & 2 (Alex Xu)

Applies all the theory above to *designing real systems under a time limit*. Two assets: a repeatable framework, and a library of building blocks + worked problems.

## 9.1 The 4-step framework (use this verbatim in interviews and design reviews)

1. **Understand the problem & scope it.** Ask clarifying questions. Separate **functional** requirements (what it does) from **non-functional** (scale, latency, consistency, availability). State assumptions explicitly. Do **back-of-envelope estimates**: DAU, QPS, storage/day, bandwidth. Don't design before you know the scale.
2. **Propose high-level design.** Draw the main boxes (clients, LB, services, DB, cache, queue), define the **APIs**, and sketch the **data model**. Get agreement before drilling down.
3. **Deep dive.** Pick the interesting/bottleneck pieces and go deep: data partitioning, the hot path, a specific algorithm, failure handling. Discuss trade-offs out loud.
4. **Wrap up.** Identify bottlenecks and how you'd scale further, add monitoring/observability, discuss edge cases and failure recovery, recap the design.

## 9.2 Estimation reflexes

- **QPS** = DAU × (actions per user per day) ÷ 86,400. **Peak QPS** ≈ 2–3× average.
- **Storage** = items/day × item size × replication factor × retention days.
- **Latency numbers to know:** memory read ~100 ns; SSD random read ~100 µs; same-DC network round trip ~0.5 ms; cross-region RTT ~50–150 ms; disk seek ~10 ms. Implication: serve from memory/cache whenever latency matters; minimize cross-region round trips.
- Use powers of two for storage (1 KB, 1 MB, 1 GB…) and round aggressively — the goal is the right order of magnitude.

## 9.3 Building blocks (these recur in almost every problem)

- **Scaling path** (memorize the progression): single server → split web/DB tiers → add a **load balancer** + multiple stateless app servers → **DB replication** (read replicas) → **cache** + **CDN** → make the app tier **stateless** (move sessions out) → **multi-datacenter** → **message queue** for async work → **database sharding** for write scaling.
- **Caching** — cache-aside (most common), read-through, write-through, write-behind; eviction by LRU + TTL; beware stampede on expiry (use locking or staggered TTLs); accept staleness for speed.
- **Database scaling** — read replicas scale *reads*; **sharding** scales *writes* (split data across DBs by a shard key); **consistent hashing** to add/remove shards with minimal data movement; **denormalization** to avoid cross-shard joins.
- **Consistent hashing** — map nodes and keys onto a ring; a key goes to the next node clockwise; adding/removing a node only remaps its neighbor's keys (not everything). **Virtual nodes** even out the distribution.
- **Message queue** — decouple producer/consumer, absorb spikes (load leveling), retry with **dead-letter queues**.
- **Rate limiting** — token bucket (allows bursts), leaking bucket (smooths output), fixed-window (simple, edge bursts), sliding-window (accurate).
- **Unique ID generation** — UUID (no coordination, large, unsorted), **Snowflake** (64-bit: timestamp + machine ID + sequence — sortable by time, distributed), ticket server (central counter, SPOF).
- **Proximity / geo** — **geohash** (encode lat/long into a string prefix; nearby points share prefixes) or **quadtree** (recursively subdivide space) for "find nearby" queries.

## 9.4 Pattern playbook (how to reason about any problem fast)

- **Read-heavy** (feeds, profiles) → cache + CDN + read replicas; precompute/denormalize.
- **Write-heavy** (logging, metrics, IoT) → sharding + message queue + LSM-tree store.
- **Strong consistency needed** (payments, inventory) → quorum/locks/ledger, single-writer ownership.
- **Eventual consistency acceptable** (likes, view counts, feeds) → async processing + denormalized counters.
- Always raise: single points of failure, replication, monitoring, and the trade-offs of each choice — that's what distinguishes a senior answer.

## 9.5 The classic problems (each is an application of the blocks above)

URL shortener (hashing + KV store), rate limiter, news feed (fan-out on write vs read), chat system (websockets + message ordering), search autocomplete (trie + caching), web crawler (BFS + politeness + dedup), notification system (queues + fan-out), YouTube/Netflix (CDN + transcoding pipeline), Google Drive (chunking + dedup + metadata), and Vol 2: proximity service, nearby friends, distributed message queue, metrics monitoring, ad-click aggregation, hotel reservation, S3-like object store, leaderboard, payment system, digital wallet, stock exchange.

**Book-level takeaway:** memorize the 4-step framework and the building blocks; then every "design X" question becomes assembling known parts and defending the trade-offs for *this* problem's read/write/consistency profile.

---

# BOOK 10 — AI Engineering (Chip Huyen)

How to build *reliable* products on top of foundation models (LLMs). This is the convergence layer — system design discipline applied to non-deterministic models, plus the security surface that comes with them.

## 10.1 Foundation model basics you must internalize

- A model is pre-trained on huge text, then **post-trained**: *supervised fine-tuning (SFT)* on curated examples, then preference alignment (*RLHF* or *DPO*) to make it helpful/safe.
- Input is split into **tokens**; the model predicts the next token. **Context window** = max tokens it can attend to. **Sampling** controls randomness: *temperature* (higher = more random), *top-p/top-k* (restrict the candidate pool).
- Core design constraint: outputs are **probabilistic and can hallucinate**. You architect *around* non-determinism — never assume a single call is correct or stable.

## 10.2 Evaluation — the hardest and most important part

- For open-ended outputs there's no single right answer, so evaluation is the central engineering problem. Methods:
  - *Exact match / functional correctness* — when there's a checkable answer (code runs, JSON parses).
  - *Similarity* — embedding similarity to a reference.
  - *LLM-as-a-judge* — use a model to score outputs against criteria; cheap and scalable but has biases (position bias, verbosity bias, self-preference) you must control for.
  - *Human eval* — gold standard, expensive; use for calibration.
- Build an **eval pipeline + golden dataset** *before* scaling usage — "eval-driven development." Without it you can't tell if a prompt/model change helped or hurt.
- Watch for **data contamination** (test data leaked into training) and **metric gaming**.

## 10.3 Prompt engineering

- *System prompt* (role/rules) vs *user prompt* (the request). **In-context learning**: zero-shot (just ask), few-shot (give examples). **Chain-of-thought** ("think step by step") improves reasoning. Request **structured output** (a JSON schema) when you need to parse the result.
- **Defensive prompting / security:** treat model output as **untrusted input**. Threats: **prompt injection** (malicious instructions hidden in user content or retrieved documents override your system prompt), **jailbreaks**, **data leakage** (model reveals secrets in context). Mitigate with input/output validation, instruction isolation, and least-privilege tool access.

## 10.4 RAG (Retrieval-Augmented Generation) — give the model your data

- Pipeline: **chunk** documents → **embed** each chunk into a vector → store in a **vector database** → at query time, **retrieve** the most relevant chunks (semantic vector search, often combined with keyword search = *hybrid*) → optionally **rerank** → stuff them into the prompt → **generate** an answer grounded in them.
- Tuning knobs: chunk size/overlap, embedding model quality, top-k retrieved, reranking, **query rewriting** (reformulate the user's question for better retrieval).
- Decision: **RAG** (inject knowledge, stays current, citeable) vs **long-context** (just paste everything — simpler, costs tokens, limited) vs **fine-tuning** (bake in behavior/format, *not* facts).

## 10.5 Agents

- An **agent** = an LLM in a loop that can call **tools** (functions/APIs), plan multi-step tasks, and keep **memory**.
- Failure modes to engineer against: **compounding errors** (a mistake early derails the whole chain), **infinite loops**, **tool misuse**, and **cost blowups** (each step is another paid call). Controls: step/iteration limits, guardrails on tool inputs/outputs, sandboxed tool execution, and human-in-the-loop for risky actions.

## 10.6 Fine-tuning

- Use it for: domain adaptation, enforcing a format/behavior/tone, or shrinking cost by training a *small* model to match a big one's behavior on your task. **Do not** use it to add facts — use RAG for facts.
- **PEFT / LoRA** — parameter-efficient fine-tuning: train small adapter weights instead of the whole model; cheap, fast, and you can swap adapters. **Quantization** (lower-precision weights) shrinks models for cheaper serving.

## 10.7 Inference optimization & cost

- Latency metrics: **TTFT** (time to first token) and **TPOT** (time per output token). Trade against **throughput** and **cost**.
- Techniques: **batching** requests, **KV-cache** (reuse attention computation), **speculative decoding** (a small model drafts, big model verifies), **quantization**, and **model routing** (try a cheap model first, escalate to an expensive one only when needed).
- **Cost engineering** is a first-class concern: token budgets, prompt compression, response caching, right-sizing the model per task. Runaway inference spend is both an ops and a security problem (abuse can rack up bills).

## 10.8 Production architecture (the convergence with system design + security)

- Layered: **model layer** (one or more models, possibly routed) → **orchestration** (prompts, RAG, agent logic, tool calls) → **eval & observability** → **guardrails** → application.
- **Observability for AI:** log every prompt/response/trace, monitor output quality, **drift**, latency, and cost over time.
- **Security surface (own this):** prompt injection, output filtering before acting on model output, **sandboxed tool execution**, PII handling and data governance, and rate/cost abuse prevention. This is exactly the AI + security intersection that's scarce and high-leverage.

**Book-level takeaway:** treat the model as a probabilistic, untrusted component. Wrap it in evaluation, retrieval, guardrails, observability, cost controls, and security — the same reliability discipline as any distributed system, plus a new adversarial surface (prompt injection) you must defend.

---

# UNIVERSAL CHEAT SHEET (everything the books agree on)

## The trade-offs that appear everywhere
- **Consistency vs Availability vs Latency** — CAP (under partition) and PACELC (else, latency vs consistency).
- **Read vs Write vs Space amplification** — RUM conjecture; every storage engine picks a corner.
- **Latency vs Throughput** — optimizing one usually costs the other (batching helps throughput, hurts latency).
- **Coupling vs Autonomy** — looser coupling = more independence but more moving parts.
- **Simplicity vs Flexibility** — every abstraction you add for flexibility costs simplicity.

## Reusable mechanisms (the master list)
```
Caching        cache-aside / write-through / write-behind; TTL + LRU; beware stampede
Partitioning   range vs hash; consistent hashing + virtual nodes; watch hot keys
Replication    single-leader / multi-leader / leaderless quorum (w + r > n)
Consistency    linearizable > sequential > causal > eventual (pick the weakest you can)
Consensus      Raft / Paxos for leader election, locks, config (etcd, ZooKeeper)
Async          message queue, event-driven, CDC, event sourcing, sagas + compensation
Idempotency    mandatory for any retry-safe operation (dedupe on request ID)
Resilience     timeout + retry(backoff+jitter) + circuit breaker + bulkhead + backpressure + load shedding
Self-healing   health checks, auto-restart, redundancy, autoscaling, steady-state cleanup (TTL, log rotation)
Observability  metrics (RED/USE) + logs (correlation IDs) + traces; SLI/SLO/error budget
Deploy safely  canary, blue-green, feature flags, parallel run, instant rollback
Decomposition  bounded contexts (DDD); high cohesion + low coupling; single-writer data ownership
```

## Latency ladder (order of magnitude — memorize)
```
L1 cache                  ~1 ns
Main memory               ~100 ns
SSD random read           ~100 µs
Network RTT (same DC)      ~0.5 ms
Disk seek (HDD)            ~10 ms
Network RTT (cross-region) ~50–150 ms
```

## Estimation reflexes
- QPS = DAU × actions/user/day ÷ 86,400; peak ≈ 2–3× average.
- Storage = items × size × replication × retention.
- 80/20: ~20% of data gets ~80% of traffic → size caches for the hot set.

## Suggested reading order (and why)
```
1. Fundamentals of Software Architecture   mindset: trade-offs + characteristics
2. System Design Interview Vol 1            vocabulary + building blocks
3. DDIA                                     depth: data, replication, consistency
4. Understanding Distributed Systems        reinforces DDIA, gentler framing
5. Release It!                              reliability + self-healing (your domain)
6. The Hard Parts                           distributed trade-offs, sagas
7. Building Microservices                   service decomposition + ops
8. Database Internals                       storage-engine depth
9. Designing Distributed Systems            reusable operational patterns
10. AI Engineering                          convergence: AI + security
```
Pace: ~1 chapter/week with notes. Principles compound across the whole career; tools change, trade-offs don't.
