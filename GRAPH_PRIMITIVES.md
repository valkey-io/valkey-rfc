---
RFC: 36
Status: Proposed
---

# Bounded graph primitives module

## Abstract

This RFC proposes an optional Valkey module, tentatively named
`valkey-graph-primitives`, that provides bounded adjacency and traversal
operations for graph-shaped data stored in Valkey.

The module is not a graph database and is not a RedisGraph successor. It does
not introduce Cypher, Gremlin, SPARQL, query planning, shortest path, global
graph algorithms, cross-shard traversal, or a custom graph data type. The goal is
to make small graph neighborhoods safe to create, expand, inspect, export, and
delete in Valkey using explicit limits, operator ceilings, and observable
metadata.

## Motivation

Many Valkey users already store graph-shaped relationships with native Sets,
Hashes, JSON values, and application-specific key conventions. Common examples
include:

- service and dependency graphs;
- authorization and entitlement graphs;
- cached expansions from dedicated graph databases;
- fraud, policy, risk, and relationship signals;
- hot-path relationship fanout where the source of truth lives elsewhere.

These workloads usually do not need a full graph query language in Valkey. They
need bounded relationship expansion close to the application.

Today, each application tends to reinvent:

- edge key naming;
- reverse adjacency maintenance;
- fanout limits;
- traversal loops;
- timeout behavior;
- truncation metadata;
- graph export and deletion;
- consistency checks;
- operational counters.

This makes graph-shaped workloads more dangerous than they need to be. A client
or script can accidentally perform large fanout, generate large replies,
monopolize server time, or return a partial answer without clear metadata.

The product line for this proposal is:

> Make small graph neighborhoods safe to expand and safe to operate, without
> turning Valkey into a graph database.

## Design considerations

### Module-only scope

The proposal is for a module, not Valkey core. A module keeps the feature
optional, lets the command surface mature independently, and avoids expanding
core Valkey before the community has proven demand, safety, benchmarks, and
maintainability.

The module is expected to use existing module capabilities:

- custom commands;
- command flags and key positions;
- ACL categories;
- module configuration;
- INFO handlers;
- replication APIs;
- native Valkey data structures.

No new Valkey core command or module API is required for the initial proposal.

### Non-goals

Version 0.1 intentionally excludes:

- Cypher;
- Gremlin;
- SPARQL;
- arbitrary pattern matching;
- query planning;
- shortest path;
- weighted traversal;
- PageRank and other global graph algorithms;
- unbounded recursive traversal;
- cross-shard graph traversal;
- transactional snapshot isolation for traversal;
- a custom native graph data type;
- graph-level TTL;
- unbounded graph dump or drop commands.

These are design guardrails, not temporary omissions.

### Safety first

Every traversal-style operation must be bounded by explicit caller limits and
non-overrideable operator ceilings.

For example, `GP.EXPAND` requires:

```text
DEPTH <n>
LIMIT <n>
MAXVISIT <n>
```

`MAXEDGES <n>` is optional in command syntax, but an effective edge-scan ceiling
is always enforced by either the caller-supplied value or the operator-configured
`graph-max-edges-scanned` value.

Caller values may lower the effective budget, but may not raise it above module
configuration. Explicit over-ceiling values should be rejected rather than
silently clamped:

```text
ERR TIMEOUT exceeds graph-hard-timeout-us
ERR MAXEDGES exceeds graph-max-edges-scanned
```

### Observable by default

Bounded traversal is not enough. Callers need to know whether a response is
complete and what work was performed.

Traversal and bounded lifecycle commands should return metadata by default. A
`NOMETA` option may request a compact reply for hot paths where the application
accepts the risk of interpreting partial results.

Metadata should include fields such as:

```text
truncated
limit_hit
visited
returned
edges_scanned
elapsed_us
```

### Native Sets and Hashes first

Version 0.1 stores graph data in ordinary Valkey Sets and Hashes.

This makes graph state:

- inspectable with ordinary Valkey commands;
- persisted through ordinary RDB/AOF mechanisms;
- easier to debug during incidents;
- easier to migrate;
- easier to repair manually in emergencies.

The tradeoff is that native structures can be mutated outside the module. Direct
mutation of internal graph keys is unsupported, and the module should provide
bounded invariant detection through `GP.CHECK`.

A custom module data type may be reconsidered later if benchmarks show that
native structures cannot meet memory, latency, or integrity goals.

### Operational lifecycle in version 0.1

A minimal module that can create graph state but cannot safely export, inspect,
check, or delete it is operationally incomplete.

Version 0.1 therefore includes source and destination catalogs so lifecycle
operations can be cursor-bounded and do not require unbounded keyspace scans.

The extra write amplification is intentional. It supports:

- `GP.ESCAN` for bounded export/debug;
- `GP.DROP` for cursor-bounded graph deletion;
- `GP.CHECK` for cursor-bounded invariant checks;
- `GP.INFO` for graph-level metadata and counters.

### Cluster keying

Every graph command takes a real Valkey key, `<graph-key>`, as its first
argument. The graph key is the cluster slot anchor. Internal keys are derived by
appending suffixes while preserving any hash tag.

Example:

```text
gp:{services}
gp:{services}:gp:v1:out:default:api-gateway
gp:{services}:gp:v1:in:default:auth-service
gp:{services}:gp:v1:srcs
gp:{services}:gp:v1:dsts
gp:{services}:gp:v1:types
```

Because `{services}` is preserved, the graph and its internal keys share the
same cluster slot.

Cross-shard traversal is out of scope.

### Managed service adoption

The proposal should be designed with managed Valkey services in mind, but should
not assume that users can load arbitrary modules into every managed service.

Some managed services restrict module commands or engine-level configuration. A
credible managed-cloud adoption path is therefore provider-bundled support, not
arbitrary user-loaded shared libraries.

That raises the bar for:

- event-loop safety;
- conservative defaults;
- exact command flags and key specs;
- ACL categories;
- replication and persistence behavior;
- upgrade compatibility;
- observability;
- unsupported eviction policy handling.

## Specification

### Module identity

Tentative values:

```text
Module name: gp
Command prefix: GP.
Initial version: 0.1.0
Target Valkey: >= 8.0
Implementation language: Rust
SDK: valkeymodule-rs
License: BSD-3-Clause
```

### Graph key and IDs

All commands use `<graph-key>` as the first user key argument.

`<graph-key>` must be valid as a Valkey key and must not end with an internal
module suffix such as `:gp:v1`. Vertex IDs and edge type IDs use a stricter
human-readable grammar in version 0.1.

Allowed vertex and type ID characters:

```text
A-Z a-z 0-9 _ - . : @ /
```

IDs must be non-empty and bounded in length by module configuration.

Binary-safe vertex IDs may be added later behind an explicit per-graph encoding
mode.

### Internal storage model

The module stores directed adjacency in native Sets.

For a graph key `gp:{services}`, type `default`, source `api-gateway`, and
destination `auth-service`, internal keys are shaped like:

```text
gp:{services}:gp:v1:out:default:api-gateway
gp:{services}:gp:v1:in:default:auth-service
```

The module also maintains native Hashes/Sets for:

- graph metadata;
- edge type counts;
- source catalogs;
- destination catalogs;
- aggregate counters.

Reverse adjacency is mandatory in version 0.1 so `IN` and `BOTH` reads are
bounded by known keys rather than reverse scans.

### Commands

#### `GP.EADD`

- **Request**

```text
GP.EADD <graph-key> <src> <dst> [TYPE <type>] [NX]
```

- **Response**

```text
(integer) 1  edge was added
(integer) 0  edge already existed and NX was supplied
```

Adds a directed edge. If `TYPE` is omitted, the edge type is `default`.

The command updates forward adjacency, reverse adjacency, type counts, graph
metadata, and source/destination catalogs atomically.

`XX` is not part of version 0.1 because edges do not yet have mutable payloads.

#### `GP.EDEL`

- **Request**

```text
GP.EDEL <graph-key> <src> <dst> [TYPE <type>]
```

- **Response**

```text
(integer) 1  edge was removed
(integer) 0  edge did not exist
```

Deletes a directed edge and updates reverse adjacency, catalogs, type counts,
and graph metadata.

#### `GP.EADDM`

- **Request**

```text
GP.EADDM <graph-key> [TYPE <type>] [NX]
  <src-1> <dst-1> [<src-2> <dst-2> ...]
```

- **Response**

```text
[
  added: <integer>,
  existed: <integer>
]
```

Adds a bounded batch of directed edges. The number of edge pairs must not exceed
`graph-max-edges-per-write`.

#### `GP.EDELM`

- **Request**

```text
GP.EDELM <graph-key> [TYPE <type>]
  <src-1> <dst-1> [<src-2> <dst-2> ...]
```

- **Response**

```text
[
  deleted: <integer>,
  missing: <integer>
]
```

Deletes a bounded batch of directed edges. The number of edge pairs must not
exceed `graph-max-edges-per-write`.

#### `GP.EXISTS`

- **Request**

```text
GP.EXISTS <graph-key> <src> <dst> [TYPE <type>]
```

- **Response**

```text
(integer) 1  edge exists
(integer) 0  edge does not exist
```

Checks directed edge existence.

#### `GP.DEGREE`

- **Request**

```text
GP.DEGREE <graph-key> <vertex> [OUT|IN|BOTH] [TYPE <type>]
```

- **Response**

```text
(integer) <degree>
```

Returns the degree for a vertex and direction. `OUT` is the default direction.

#### `GP.NEIGHBORS`

- **Request**

```text
GP.NEIGHBORS <graph-key> <vertex>
  LIMIT <n>
  [MAXEDGES <n>]
  [TIMEOUT <milliseconds>]
  [OUT|IN|BOTH]
  [TYPE <type> ...]
  [WITHMETA|NOMETA]
```

- **Response**

With metadata:

```text
[
  neighbors: [<vertex> ...],
  meta: {
    truncated: <0|1>,
    limit_hit: <none|limit|maxedges|timeout>,
    returned: <integer>,
    edges_scanned: <integer>,
    elapsed_us: <integer>
  }
]
```

With `NOMETA`:

```text
[<vertex> ...]
```

Returns bounded neighbors for a vertex.

#### `GP.EXPAND`

- **Request**

```text
GP.EXPAND <graph-key> <start>
  DEPTH <n>
  LIMIT <n>
  MAXVISIT <n>
  [MAXEDGES <n>]
  [OUT|IN|BOTH]
  [TYPE <type> ...]
  [TIMEOUT <milliseconds>]
  [WITHMETA|NOMETA]
  [INCLUDESTART]
```

- **Response**

With metadata:

```text
[
  vertices: [<vertex> ...],
  meta: {
    truncated: <0|1>,
    limit_hit: <none|depth|limit|maxvisit|maxedges|timeout>,
    depth_reached: <integer>,
    visited: <integer>,
    returned: <integer>,
    edges_scanned: <integer>,
    elapsed_us: <integer>
  }
]
```

With `NOMETA`:

```text
[<vertex> ...]
```

Performs bounded breadth-first expansion.

#### `GP.ESCAN`

- **Request**

```text
GP.ESCAN <graph-key> <cursor>
  COUNT <n>
  [TYPE <type>|TYPE *]
  [WITHMETA|NOMETA]
```

- **Response**

With metadata:

```text
[
  cursor: <next-cursor>,
  edges: [[<src>, <dst>, <type>] ...],
  meta: {
    returned: <integer>,
    elapsed_us: <integer>
  }
]
```

Scans graph edges using catalogs. `COUNT` must not exceed
`graph-max-scan-count`.

#### `GP.DROP`

- **Request**

```text
GP.DROP <graph-key> <cursor>
  COUNT <n>
  [WITHMETA|NOMETA]
```

- **Response**

```text
[
  cursor: <next-cursor>,
  keys_deleted: <integer>,
  done: <0|1>,
  meta: {
    elapsed_us: <integer>
  }
]
```

Deletes graph state in bounded pages. `COUNT` must not exceed
`graph-max-drop-count`.

Concurrent writes during `GP.DROP` are unsupported and may return an error.

#### `GP.CHECK`

- **Request**

```text
GP.CHECK <graph-key> <cursor>
  COUNT <n>
  [TYPE <type>|TYPE *]
  [WITHMETA|NOMETA]
```

- **Response**

```text
[
  cursor: <next-cursor>,
  issues: [[<code>, <detail> ...] ...],
  meta: {
    checked: <integer>,
    issue_count: <integer>,
    elapsed_us: <integer>
  }
]
```

Performs bounded invariant detection. Version 0.1 detects problems only; repair
is deferred.

#### `GP.LIMITS`

- **Request**

```text
GP.LIMITS <graph-key> GET

GP.LIMITS <graph-key> SET
  [MAXDEPTH <n>]
  [MAXLIMIT <n>]
  [MAXVISIT <n>]
  [MAXEDGES <n>]
  [TIMEOUT <milliseconds>]
  [MAXTEMPBYTES <n>]

GP.LIMITS <graph-key> RESET
```

- **Response**

`GET` returns the effective per-graph limits. `SET` stores per-graph limits that
may only lower global module ceilings. `RESET` removes per-graph overrides.

#### `GP.INFO`

- **Request**

```text
GP.INFO <graph-key>
```

- **Response**

Returns graph metadata, approximate counts, configured limits, and detected
operational warnings.

#### `GP.STATS`

- **Request**

```text
GP.STATS
```

- **Response**

Returns module-wide counters, including command counts, truncations, timeout
hits, maximum observed traversal work, check issues, drop progress counters, and
temporary memory rejection counts.

#### `GP.STATS.RESET`

- **Request**

```text
GP.STATS.RESET
```

- **Response**

Returns the previous module-wide counters and resets resettable counters.

### Authentication and Authorization

The module should declare an ACL category such as `graph`.

Suggested ACL categories:

```text
GP.EADD         write graph
GP.EDEL         write graph
GP.EADDM        write graph
GP.EDELM        write graph
GP.EXISTS       read graph
GP.DEGREE       read graph
GP.NEIGHBORS    read graph
GP.EXPAND       read graph
GP.ESCAN        read graph
GP.DROP         write graph
GP.CHECK        read graph
GP.LIMITS GET   read graph
GP.LIMITS SET   write graph
GP.LIMITS RESET write graph
GP.INFO         read graph
GP.STATS        read graph
GP.STATS.RESET  admin graph
```

Suggested command flags:

```text
GP.EADD         write deny-oom
GP.EDEL         write
GP.EADDM        write deny-oom
GP.EDELM        write
GP.EXISTS       readonly
GP.DEGREE       readonly
GP.NEIGHBORS    readonly deny-oom deny-script
GP.EXPAND       readonly deny-oom deny-script
GP.ESCAN        readonly deny-oom deny-script
GP.DROP         write deny-script
GP.CHECK        readonly deny-oom deny-script
GP.LIMITS       write deny-oom
GP.INFO         readonly
GP.STATS        readonly
GP.STATS.RESET  admin deny-script
```

Commands with graph state should declare `<graph-key>` as the only user key.
`GP.STATS` and `GP.STATS.RESET` declare no keys.

### Append-only file

Write commands should persist as deterministic mutations to native Valkey Sets
and Hashes.

The implementation should use module replication APIs consistently so each
logical graph mutation is replicated and replayed atomically. In particular,
updates to forward adjacency, reverse adjacency, catalogs, metadata, and counters
must not be partially represented in AOF.

### RDB

Version 0.1 introduces no custom module data type. Graph state is stored in
native Sets and Hashes and is therefore captured by ordinary RDB snapshotting.

Future versions that introduce a custom data type would need a new RDB encoding
and versioning scheme.

### Configuration

Initial module configuration should include conservative ceilings:

```text
graph-max-depth
graph-max-limit
graph-max-visit
graph-max-edges-scanned
graph-hard-timeout-us
graph-max-edges-per-write
graph-max-scan-count
graph-max-drop-count
graph-max-check-count
graph-max-id-bytes
graph-max-type-bytes
graph-max-types-per-command
graph-max-temp-memory-bytes
```

Per-graph limits may be configured through `GP.LIMITS`, but may only lower these
global ceilings.

### Keyspace notifications

Version 0.1 does not require new keyspace notification event types.

The module should not promise a stable mapping from graph commands to native Set
or Hash keyspace notifications, because internal storage layout may change.

### Cluster mode

Graph commands are single-slot by construction. The first argument is the graph
key and all internal keys preserve its hash tag.

The module should reject graph keys or internal derivations that cannot preserve
cluster-slot locality. Cross-shard traversal is not supported in version 0.1.

### Module API

The proposal does not require new module APIs.

The implementation is expected to use existing module support for:

- command registration;
- command flags and key positions;
- ACL categories;
- module configuration;
- replication APIs;
- INFO handlers;
- memory allocation accounting.

Blocking clients are intentionally not required for version 0.1. Traversal and
lifecycle commands should run synchronously with strict hard ceilings.

### Replication

Replicas should receive the same deterministic native Set/Hash mutations as the
primary.

Logical graph writes must maintain atomicity across:

- forward adjacency;
- reverse adjacency;
- source catalogs;
- destination catalogs;
- type counts;
- graph metadata;
- module counters where applicable.

### Networking

No RESP protocol changes are required.

Commands should return RESP-compatible arrays, maps, integers, bulk strings, and
errors. Implementations should define stable reply shapes for RESP2 and RESP3
clients.

### Dependencies

The reference implementation is expected to be written in Rust using
`valkeymodule-rs`.

The RFC does not require Valkey to vendor new dependencies.

### Benchmarking

Before asking for approval, the module should publish benchmarks for:

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
- memory overhead per edge compared with raw Sets and Hashes;
- persistence and replay behavior for native structures.

Benchmarks should include adversarial high-fanout graphs, not only friendly
datasets.

### Testing

Testing should cover:

- graph key and ID validation;
- cluster hash tag preservation;
- forward and reverse adjacency maintenance;
- source and destination catalog maintenance;
- edge type counts;
- single-edge and batch writes;
- idempotent `NX` behavior;
- deletion cleanup;
- degree reads;
- neighbor reads with limits, type filters, direction filters, timeout, and
  metadata;
- BFS expansion with `DEPTH`, `LIMIT`, `MAXVISIT`, `MAXEDGES`, timeout, and
  `INCLUDESTART`;
- cursor behavior for `GP.ESCAN`, `GP.DROP`, and `GP.CHECK`;
- rejection of over-ceiling caller bounds;
- temporary memory preflight;
- ACL and command flags;
- replication/AOF replay;
- RDB save and restore;
- unsupported direct mutation detected by `GP.CHECK`;
- unsupported eviction policy warnings;
- malformed command input.

### Observability

`GP.STATS` and module INFO should expose counters such as:

```text
gp_eadd_calls
gp_eaddm_calls
gp_edel_calls
gp_edelm_calls
gp_neighbors_calls
gp_expand_calls
gp_escan_calls
gp_drop_calls
gp_check_calls
gp_truncated_total
gp_timeout_total
gp_maxedges_hit_total
gp_maxvisit_hit_total
gp_temp_memory_rejected_total
gp_check_issues_total
gp_edges_scanned_total
gp_max_edges_scanned_observed
```

Traversal replies should include per-command metadata by default.

### Debug mechanism

Version 0.1 includes:

- `GP.INFO` for graph-level metadata and warnings;
- `GP.STATS` for module-level counters;
- `GP.STATS.RESET` for resettable counters;
- `GP.ESCAN` for bounded export/debug;
- `GP.CHECK` for bounded invariant detection;
- `GP.DROP` for cursor-bounded cleanup.

Direct mutation of internal graph keys is unsupported, but `GP.CHECK` should
detect bounded classes of drift such as missing reverse edges, missing forward
edges, stale catalog entries, and type count mismatches.

## Appendix

Related material:

- Valkey issue #3108, "Exploring a bounded graph primitives module for Valkey":
  https://github.com/valkey-io/valkey/issues/3108
- Companion PRFAQ in this RFC pull request:
  `GRAPH_PRIMITIVES_PRFAQ.md`
- Valkey module introduction:
  https://valkey.io/topics/modules-intro/
- Valkey module API reference:
  https://valkey.io/topics/modules-api-ref/
- Valkey blocking module operations:
  https://valkey.io/topics/modules-blocking-ops/
- Valkey cluster specification:
  https://valkey.io/topics/cluster-spec/
- `valkeymodule-rs` crate:
  https://docs.rs/valkey-module/latest/valkey_module/
