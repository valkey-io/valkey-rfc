# PRFAQ: Valkey Graph Primitives

Status: supporting material for RFC PR
Date: 2026-05-02
Related artifacts:

- `GRAPH_PRIMITIVES.md`
- Valkey issue #3108: Exploring a bounded graph primitives module for Valkey

## Press Release

### Valkey community explores bounded graph primitives for safer relationship workloads

Today the Valkey community is exploring `valkey-graph-primitives`, an optional
module that makes small graph neighborhoods safe to create, expand, inspect,
export, and delete in Valkey.

Many teams already store graph-shaped relationships in Valkey using Sets,
Hashes, JSON, and application-side conventions. They use those relationships for
service dependency lookups, authorization and entitlement expansion, fraud and
risk signals, policy evaluation, and cached expansions from dedicated graph
databases. The problem is not that these teams lack a graph query language. The
problem is that hot-path relationship fanout can become operationally unsafe:
large replies, accidental fanout explosions, event-loop latency spikes, and
little visibility into how much graph work a request performed.

`valkey-graph-primitives` proposes a narrower answer: bounded graph operations,
not a graph database.

The module would add a `GP.` command family for directed edges, bounded batch
writes, degree checks, neighbor reads, breadth-first expansion, cursor-bounded
edge scanning, cursor-bounded graph deletion, invariant checking, per-graph
limits, and observability. Every traversal-style operation is explicitly bounded
by caller-supplied limits and non-overrideable operator ceilings. Replies return
metadata by default so applications can see whether a result was truncated, which
limit was hit, how many edges were scanned, and how long the command took.

The first release intentionally avoids Cypher, Gremlin, SPARQL, arbitrary
pattern matching, query planning, shortest path, weighted traversal, global graph
algorithms, cross-shard traversal, and custom graph storage. Graph data is stored
in native Valkey Sets and Hashes so operators can inspect, persist, migrate, and
repair it with familiar tools.

The design goal is simple:

> Make small graph neighborhoods safe to expand and safe to operate, without
> turning Valkey into a graph database.

This proposal is for teams that already use Valkey as a low-latency hot path,
but need safer primitives than ad hoc `SMEMBERS`, `SSCAN`, `HGETALL`, scripts,
or client-side traversal loops. Dedicated graph databases remain the right
answer for graph sources of truth, graph analytics, arbitrary graph query
languages, and global graph algorithms. `valkey-graph-primitives` is for bounded
relationship expansion near the application.

The RFC asks the Valkey community whether this is the right shape for an
upstream module: module-only, bounded by construction, observable by default,
cluster-slot-safe through a graph key anchor, and operationally complete enough
in v0.1 to create, inspect, export, check, and delete graph state safely.

## FAQ

### 1. Who is the customer?

The primary customer is the platform or application team that already keeps
relationship data in Valkey because it needs low-latency reads close to the
application.

Representative users include:

- a platform team expanding service dependencies during incident response;
- an authorization team checking group, role, tenant, or resource fanout;
- a fraud or risk team reading bounded relationship neighborhoods in a request
  path;
- an application team caching graph expansions from Neo4j, Neptune, FalkorDB, or
  another source of truth;
- an infrastructure team that wants safer primitives than application-side
  traversal over Sets and Hashes.

These users usually do not need a graph query language in Valkey. They need
bounded, predictable, observable fanout.

### 2. What customer problem does this solve?

Valkey users already model graph-like data with native structures. A common
pattern is one Set per adjacency list, plus optional reverse indexes and metadata
Hashes. That works until the application needs to expand more than one hop,
combine direction and type filters, enforce hard limits, and understand whether
the answer is complete.

Without module-level primitives, every application reinvents:

- edge key conventions;
- reverse adjacency maintenance;
- traversal loops;
- fanout limits;
- timeout behavior;
- truncation metadata;
- consistency checks;
- graph cleanup and export;
- operational counters.

Those reinventions are often less safe than a module command because the server
cannot consistently enforce operator ceilings or report traversal cost.

### 3. Why should Valkey care about graph-shaped workloads at all?

Because those workloads are already in Valkey.

The proposal does not ask Valkey to become a graph database. It asks whether
Valkey should provide safer primitives for a class of workloads users already
run with ordinary commands. The relevant comparison is not "Valkey graph
primitives versus a full graph database." The comparison is "bounded module
commands versus ad hoc client traversal and scripts over Sets and Hashes."

### 4. What is the product line?

The product line is:

> Safe bounded relationship expansion in Valkey, not graph database
> expressiveness.

That line should be repeated in the RFC, docs, examples, benchmarks, and
maintainer discussion. It prevents the proposal from drifting into RedisGraph
successor territory.

### 5. Why not just use a dedicated graph database?

Use a dedicated graph database when the graph is the source of truth, when users
need Cypher or Gremlin, when queries require arbitrary pattern matching, when
workloads need shortest path or global graph algorithms, or when the graph spans
large partitions.

This module serves a different layer:

- hot-path graph cache;
- bounded neighborhood expansion;
- operational dependency and entitlement fanout;
- relationship signals used inside application requests;
- low-latency reads from graph state that is sourced elsewhere.

The module should complement dedicated graph systems, not compete with them.

### 6. How is this different from RedisGraph?

RedisGraph was a general-purpose property graph database module with a query
language and a more ambitious execution surface. Redis announced RedisGraph end
of life in 2023.

This proposal learns the opposite lesson: do less.

`valkey-graph-primitives` does not include Cypher, graph query planning,
arbitrary pattern matching, shortest path, PageRank, graph analytics, or custom
graph storage in v0.1. It proposes small bounded commands over native Valkey
structures.

The point is not to replace RedisGraph. The point is to avoid re-creating the
maintenance and expectation burden of a full graph database inside Valkey.

### 7. How is this different from FalkorDB?

FalkorDB is the right comparison when a user wants a graph database compatible
with RedisGraph-style Cypher workflows.

`valkey-graph-primitives` is deliberately narrower. It should be judged as a
server-side primitive layer for bounded relationship expansion, not as an
alternative to FalkorDB for property graph workloads.

### 8. Why a module instead of Valkey core?

A module is the right first home because:

- the feature is optional;
- the command surface can mature without expanding Valkey core prematurely;
- module APIs already support custom commands, command flags, ACL categories,
  module configuration, replication hooks, INFO handlers, and blocking clients;
- a module can prove demand, safety, benchmarks, and maintainability before any
  core discussion.

The RFC should explicitly not ask for Valkey core inclusion.

### 9. Why make v0.1 richer instead of shipping only EADD and EXPAND?

A minimal v0.1 that can create graph state but cannot safely export, check, or
delete it is operationally incomplete.

The richer v0.1 includes:

- `GP.EADD` and `GP.EDEL`;
- `GP.EADDM` and `GP.EDELM`;
- `GP.EXISTS`;
- `GP.DEGREE`;
- `GP.NEIGHBORS`;
- `GP.EXPAND`;
- `GP.ESCAN`;
- `GP.DROP`;
- `GP.CHECK`;
- `GP.LIMITS`;
- `GP.INFO`;
- `GP.STATS`;
- `GP.STATS.RESET`.

This is still bounded and narrow. The additional commands are not graph database
features. They are lifecycle and safety features.

### 10. What is the v0.1 command surface?

```text
GP.EADD <graph-key> <src> <dst> [TYPE <type>] [NX]
GP.EDEL <graph-key> <src> <dst> [TYPE <type>]
GP.EADDM <graph-key> [TYPE <type>] [NX] <src-1> <dst-1> [<src-2> <dst-2> ...]
GP.EDELM <graph-key> [TYPE <type>] <src-1> <dst-1> [<src-2> <dst-2> ...]
GP.EXISTS <graph-key> <src> <dst> [TYPE <type>]
GP.DEGREE <graph-key> <vertex> [OUT|IN|BOTH] [TYPE <type>]
GP.NEIGHBORS <graph-key> <vertex> LIMIT <n> [MAXEDGES <n>] [TIMEOUT <ms>] [OUT|IN|BOTH] [TYPE <type> ...] [WITHMETA|NOMETA]
GP.EXPAND <graph-key> <start> DEPTH <n> LIMIT <n> MAXVISIT <n> [MAXEDGES <n>] [OUT|IN|BOTH] [TYPE <type> ...] [TIMEOUT <ms>] [WITHMETA|NOMETA] [INCLUDESTART]
GP.ESCAN <graph-key> <cursor> COUNT <n> [TYPE <type>|TYPE *] [WITHMETA|NOMETA]
GP.DROP <graph-key> <cursor> COUNT <n> [WITHMETA|NOMETA]
GP.CHECK <graph-key> <cursor> COUNT <n> [TYPE <type>|TYPE *] [WITHMETA|NOMETA]
GP.LIMITS <graph-key> GET
GP.LIMITS <graph-key> SET [MAXDEPTH <n>] [MAXLIMIT <n>] [MAXVISIT <n>] [MAXEDGES <n>] [TIMEOUT <ms>] [MAXTEMPBYTES <n>]
GP.LIMITS <graph-key> RESET
GP.INFO <graph-key>
GP.STATS
GP.STATS.RESET
```

### 11. Why include batch writes?

Most graph workloads ingest relationship changes in small bursts: user added to
groups, service dependencies refreshed, risk links updated, policy edges synced,
or cache state rebuilt from a source of truth.

Without bounded batch writes, clients must pipeline many single-edge operations.
That is workable, but it makes atomicity, counters, catalogs, and error reporting
more awkward. `GP.EADDM` and `GP.EDELM` give applications a bounded, observable
way to update multiple edges while still obeying `graph-max-edges-per-write`.

### 12. Why include source and destination catalogs?

Catalogs are the price of an operationally complete v0.1.

If the module only stores adjacency sets, it cannot safely enumerate all graph
keys without a keyspace scan. Source and destination catalogs let the module
implement bounded:

- edge scan/export through `GP.ESCAN`;
- graph deletion through `GP.DROP`;
- invariant checking through `GP.CHECK`;
- graph information through `GP.INFO`.

The tradeoff is extra write amplification. That tradeoff is worth surfacing to
maintainers directly because it determines whether v0.1 is only a traversal toy
or a usable operational module.

### 13. How are commands kept safe?

The safety model has several layers:

- traversal commands require caller bounds;
- operator-configured ceilings cannot be exceeded by callers;
- explicit over-ceiling values are rejected, not silently clamped;
- traversal and lifecycle replies include metadata by default;
- hot-path callers can opt into `NOMETA`;
- traversal, scan, check, and drop commands use `deny-script`;
- command flags and key positions are declared explicitly;
- temporary memory is preflighted against `graph-max-temp-memory-bytes`;
- graph deletion, export, and checking are cursor-bounded.

The design should make dangerous behavior mechanically hard, not merely
documented as a best practice.

### 14. Why reject over-ceiling caller limits instead of clamping them?

Rejecting over-ceiling values makes misconfiguration visible.

If a caller asks for `LIMIT 1000000` and the operator ceiling is `10000`, silent
clamping can hide a serious application bug. A clear error lets the application
team correct its assumptions and prevents dashboards from showing "successful"
queries that were never allowed to run as requested.

### 15. Why return metadata by default?

Graph truncation is a correctness signal, not just an operational detail.

If a traversal stops because it hit `LIMIT`, `MAXVISIT`, `MAXEDGES`, or
`TIMEOUT`, the caller often needs to know whether the answer is partial. Returning
metadata by default makes safe behavior the default. `NOMETA` exists for hot paths
where the application deliberately accepts compact replies and has another way to
interpret incomplete results.

### 16. How does cluster behavior work?

Every command takes `<graph-key>` as the first argument. That key is the cluster
slot anchor. Internal keys append suffixes to the graph key while preserving the
hash tag.

Example:

```text
gp:{services}
gp:{services}:gp:v1:out:default:api-gateway
gp:{services}:gp:v1:in:default:auth-service
gp:{services}:gp:v1:types
gp:{services}:gp:v1:srcs
gp:{services}:gp:v1:dsts
```

Because `{services}` is preserved, all internal keys for the graph stay in the
same cluster slot.

v0.1 does not provide cross-shard graph traversal.

### 17. What are the consistency guarantees?

Individual module commands are atomic with respect to Valkey command execution.

Traversal commands are bounded best-effort reads over native data structures.
They do not provide transactional snapshot isolation. Concurrent writes may or
may not be reflected during traversal. That is acceptable for the target use
cases, but must be explicit in the RFC and docs.

If a use case needs snapshot isolation over large graph queries, it needs a
dedicated graph database or a future module design with safe snapshot mechanics.

### 18. Why native Sets and Hashes instead of a custom graph type?

Native structures are valuable in v0.1 because they are:

- inspectable with ordinary Valkey commands;
- persisted through ordinary RDB/AOF mechanisms;
- easier to debug during incidents;
- easier to migrate;
- easier to repair manually if needed.

The cost is that native structures can be mutated outside the module. The RFC
should be candid about this. Direct mutation of internal keys is unsupported, and
`GP.CHECK` exists to detect bounded classes of invariant drift.

A custom module data type can be reconsidered if benchmarks show native
structures cannot meet latency, memory, or integrity goals.

### 19. What eviction policies are safe?

The safe default is `noeviction`.

`volatile-*` policies can be supported only if the module never sets TTLs on
internal graph keys. `allkeys-*` policies should be treated as unsupported for
graph correctness because they can evict one internal key and leave the graph in
an inconsistent state.

This is a real managed-cloud concern because some managed services default to an
eviction policy other than `noeviction`. The module should expose this clearly in
docs and `GP.INFO`.

### 20. What prevents AWS or GCP adoption?

The main blocker is not API shape. It is managed-service control.

In self-managed Valkey, modules can be loaded at startup or with `MODULE LOAD`.
In managed services, customers often do not control arbitrary module loading or
engine-level configuration. For example, Google Cloud Memorystore for Valkey
documents `MODULE LOAD`, `MODULE LOADEX`, `MODULE LIST`, and `MODULE UNLOAD` as
blocked commands. Amazon MemoryDB documents the `module` command as unavailable.

That means the adoption story should not assume customers can upload an arbitrary
`.so` file into AWS or GCP managed Valkey. The realistic path is:

1. prove the module in open source/self-managed Valkey;
2. make the module operationally boring enough for provider review;
3. pursue provider-bundled or provider-supported packaging;
4. ensure command flags, key specs, metrics, config defaults, and upgrade
   behavior meet managed-service standards.

AWS MemoryDB's JSON support is useful precedent for the shape of this path:
provider-bundled data-type functionality can be made available automatically
when running supported engine versions. It does not prove graph primitives will
be accepted, but it shows the right adoption model is provider-bundled support,
not arbitrary customer module loading.

### 21. What would AWS/GCP ask before adopting this?

They would likely ask:

- Can any command monopolize the event loop?
- Are command flags, ACL categories, and key specs exact?
- Does the module behave correctly during failover, replication, AOF replay, RDB
  restore, backup, and engine upgrade?
- Are memory ceilings explicit and enforceable?
- Does the module expose metrics that fit CloudWatch or Cloud Monitoring?
- Are defaults conservative enough for multi-tenant managed services?
- Can the module reject unsupported eviction policies or at least report them?
- Does it need OS-level dependencies or unsafe runtime loading behavior?
- Is the data layout stable across versions?
- Can providers test compatibility without reverse-engineering workload
  semantics?

The spec now leans into those questions rather than treating them as afterthoughts.

### 22. Should this be async?

Not in v0.1.

Valkey module APIs support blocking clients, but async traversal introduces
harder questions: snapshot safety, concurrent mutation semantics, memory
ownership, cancellation, and consistency. The conservative v0.1 should use
synchronous commands with strict ceilings.

Async can come later if the module introduces a safe snapshot, an auxiliary
index, or a custom data type that makes long-running traversal safe.

### 23. Why no `GP.DUMP` in v0.1?

`GP.ESCAN` is the safer primitive.

A dump command invites large replies and unclear operational expectations. A
cursor-bounded scan lets callers export graph state page by page while respecting
operator limits. A future `GP.DUMP` can be a thin bounded convenience wrapper if
real users need it.

### 24. Why no `XX` on `GP.EADD` in v0.1?

`NX` is enough for idempotent insertion. `XX` adds semantic surface without a
clear v0.1 need because an edge update has no mutable payload yet. If future
versions add weights, properties, or timestamps as user-visible attributes, `XX`
can be reconsidered.

### 25. What does success look like?

The module is successful if it makes existing graph-shaped Valkey workloads
safer without changing Valkey's product identity.

Early success metrics:

- predictable p50/p95/p99 latency under bounded fanout;
- traversal commands consistently reporting truncation metadata;
- no command exceeding operator hard ceilings in tests;
- no keyspace scans required for lifecycle operations;
- bounded drop/check/export usable on real-size graphs;
- benchmarked memory overhead versus raw Sets/Hashes;
- simple client-side integration using existing Valkey clients;
- operator confidence that the module will not become a graph database by
  accident.

### 26. What are the launch benchmarks?

Benchmarks should include:

- `GP.EADD` single-edge throughput;
- `GP.EADDM` throughput at batch sizes 10, 100, and 1000;
- `GP.EDEL` and `GP.EDELM` delete throughput;
- `GP.EXISTS` latency;
- `GP.DEGREE` latency by degree size;
- `GP.NEIGHBORS` latency at different limits;
- `GP.EXPAND` latency by depth, fanout, and limit;
- truncation overhead;
- metadata overhead versus `NOMETA`;
- `GP.ESCAN COUNT 100/1000`;
- `GP.CHECK COUNT 100/1000`;
- `GP.DROP COUNT 100/1000`;
- memory overhead per edge compared with raw Sets/Hashes;
- persistence and replay behavior for native structures.

Benchmarks should include adversarial fanout cases, not only friendly graphs.

### 27. What are the five questions we should force early?

1. **Is this valuable without a graph query language?**
   Yes, if the target customer is already doing bounded relationship expansion
   in Valkey. No, if the target customer expects a graph database.

2. **What makes v0.1 operationally complete?**
   Not just `EADD` and `EXPAND`. v0.1 needs batch writes, scan/export, drop,
   check, limits, stats, and catalogs so users can operate the graph state they
   create.

3. **Can safety be enforced mechanically?**
   It must be. Mandatory caller bounds, operator ceilings, explicit rejection,
   metadata-first replies, `deny-script`, and temp-memory preflight are not
   polish. They are the product.

4. **What would stop managed-cloud adoption?**
   Arbitrary customer module loading is commonly restricted. The module must be
   designed for provider-bundled review: boring defaults, exact command specs,
   observability, compatibility, and safe upgrade behavior.

5. **Can native structures stay correct under real operations?**
   Only if direct mutation is unsupported, catalogs are maintained atomically,
   eviction policy limitations are explicit, and `GP.CHECK` can detect bounded
   drift.

### 28. What would critics say?

Critic: "This is scope creep. v0.1 is too big."
Answer: The added commands are lifecycle safety, not graph expressiveness. A
module that can create graph state but cannot safely export, delete, or check it
is harder to adopt.

Critic: "RedisGraph already proved graph in Redis is a trap."
Answer: RedisGraph proved the risk of a full graph database with a query
language inside Redis-style infrastructure. This proposal deliberately excludes
that scope.

Critic: "Native Sets and Hashes are fragile because users can mutate them."
Answer: Correct. That is why direct mutation is unsupported and `GP.CHECK`
exists. Native structures are chosen for v0.1 inspectability and persistence,
not because they are perfect.

Critic: "The main thread will still be at risk."
Answer: That is the core risk. The design must prove through ceilings,
benchmarks, command flags, and adversarial tests that bounded synchronous
commands stay safe. If it cannot, the proposal should not proceed.

Critic: "Managed clouds will not let users load this."
Answer: Also correct for many services. The adoption plan should be
provider-bundled, not arbitrary upload. That raises the quality bar, but it is
the only credible path for AWS/GCP managed adoption.

Critic: "This will confuse users into thinking Valkey is a graph database."
Answer: The name, docs, command surface, non-goals, examples, and benchmarks
must all reinforce "bounded primitives," not "database." Avoid graph database
marketing language.

Critic: "Why not leave this to clients?"
Answer: Clients cannot enforce server-side operator ceilings, consistent
metadata, atomic catalog maintenance, or event-loop-aware limits as reliably as
a module command can.

### 29. What should be explicitly out of scope?

v0.1 should explicitly exclude:

- Cypher;
- Gremlin;
- SPARQL;
- arbitrary pattern matching;
- query planning;
- shortest path;
- weighted traversal;
- PageRank and global graph algorithms;
- cross-shard traversal;
- transactional snapshot isolation;
- custom graph storage;
- graph-level TTL;
- unbounded dump/drop;
- claims of source-of-truth graph database behavior.

These are not embarrassing omissions. They are the guardrails that make the
proposal coherent.

### 30. What should the RFC ask maintainers?

The RFC should ask maintainers to react to the shape of the product boundary:

- Is a bounded graph primitives module aligned with Valkey's module direction?
- Is the richer v0.1 lifecycle justified, or should any lifecycle commands be
  deferred?
- Are source/destination catalogs worth the write amplification?
- Should traversal metadata be default-on?
- Are explicit over-ceiling errors preferable to silent clamping?
- Are the command flags, key specs, and ACL categories appropriate?
- What additional constraints would make providers more comfortable bundling
  this in managed Valkey services?

### 31. What is the recommended RFC PR strategy?

Use three artifacts:

1. `GRAPH_PRIMITIVES.md` in the Valkey RFC repo.
2. The RFC PR description, adapted from the concise RFC comment.
3. This PRFAQ, linked from the PR as supporting rationale.

The RFC should be formal and technical. The PR description should be concise and
invite maintainer feedback. The PRFAQ should carry the customer narrative,
adoption argument, and critic answers.

### 32. What should not be said publicly?

Do not claim:

- that AWS, GCP, or any managed provider will support this;
- that customers can load it into every managed Valkey service;
- that this replaces RedisGraph, FalkorDB, Neo4j, Neptune, or other graph
  databases;
- that native Sets/Hashes guarantee integrity under direct mutation;
- that bounded traversal gives snapshot isolation;
- that v0.1 is safe before benchmarks prove it.

Say instead:

- self-managed Valkey can test modules directly;
- managed-service adoption likely requires provider-bundled support;
- this is a bounded primitive layer;
- correctness and safety must be validated through tests and benchmarks.

## Decision Summary

Recommended posture:

- Proceed with the RFC.
- Keep the product line narrow and repeat it often.
- Keep the richer v0.1 lifecycle because it is operationally necessary.
- Treat cloud adoption as a design constraint from day one.
- Do not implement graph database features.
- Make benchmarks and adversarial tests first-class acceptance criteria.

## Source Notes

These sources support the external claims in this PRFAQ:

- Valkey modules can be loaded at startup or with `MODULE LOAD` in self-managed
  Valkey: https://valkey.io/topics/modules-intro/
- Valkey issue #3108 describes the bounded graph-primitives direction:
  https://github.com/valkey-io/valkey/issues/3108
- Google Cloud Memorystore for Valkey documents `MODULE LOAD`, `MODULE LOADEX`,
  `MODULE LIST`, and `MODULE UNLOAD` as blocked commands:
  https://cloud.google.com/memorystore/docs/valkey/supported-commands
- Amazon MemoryDB documents the `module` command as unavailable:
  https://docs.aws.amazon.com/memorydb/latest/devguide/restrictedcommands.html
- Amazon MemoryDB JSON support shows provider-bundled data-type functionality
  can be made available automatically on supported engine versions:
  https://docs.aws.amazon.com/memorydb/latest/devguide/json-gs.html
- Redis announced RedisGraph end of life in 2023:
  https://redis.io/blog/redisgraph-eol/
