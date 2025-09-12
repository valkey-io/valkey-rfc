---
RFC:
Status: Draft
---

## **Abstract**

Valkey clusters optimize for low latency: a primary executes a write and acknowledges immediately;
replicas receive the update asynchronously. This approach delivers excellent throughput and tail
latency for many workloads, but it also creates a window where an **acknowledged** write can be
lost if the primary fails before replicas make the write durable. The existing `WAIT <replicas>
<timeout>` command provides a client-side synchronization mechanism for clients to block until a
specified number of replicas have recorded the write. However, this propagation-based
acknowledgement does not provide a design-level guarantee of durability across all topology
changes, including failover, replica additions/removals, and horizontal scaling via slot
migration.


## **Motivation**

Production users increasingly run workloads where **loss of an acknowledged write is
unacceptable**, not just in steady state but also during failover and cluster changes. In
practice, users have two distinct needs for durability controls:



1. **Explicit, Per-Command Control**: Users require that the existing `WAIT` command provides a
meaningful durability guarantee. When a client explicitly invokes `WAIT` after a write **with a
replica count sufficient to form a quorum**, its successful return must signify that the write
will be preserved across topology changes. This allows applications to make deliberate, fine-
grained tradeoffs between latency and durability,
2. **Connection-Level Opt-In**: For ease of use and safety, users need a way to configure a
connection to be durable by default. This allows an application to establish its durability
requirement once, ensuring all subsequent writes on that connection are treated as durable without
needing to add `WAIT` after every command. This approach reduces application complexity by removing
the burden on developers to correctly identify every critical synchronization point in their logic.
It allows them to make a deliberate tradeoff, accepting higher write latency in exchange for
simpler code and stronger durability guarantees.


## **Key Durability Challenges**

While Valkey allows a client to receive an acknowledgement that a write has been accepted by the
primary, this acknowledgement is not bound to the future authoritative history of the shard.
Several legitimate cluster behaviors can cause an acknowledged write to be lost, even without bugs
or operator errors:



* **Leader Election**: A write may be acknowledged by the primary and propagated to a subset of
replicas, but a different replica—one that never received the write—can be legitimately promoted to
primary. After promotion, the shard's authoritative history will exclude that write.

* **Slot Migration Rollback**: A data loss scenario occurs when the source shard broadcasts a
higher config epoch after the migration has completed, forcing the target shard to roll back the
ownership transfer and discard any writes it accepted in the interim. This can be triggered by a
failover in the source shard or by a concurrent migration conflict where the source shard bumps its
own epoch.

* **Two-Node Shard Tradeoffs**: In a 1-primary/1-replica topology, any durability guarantee
inherently trades availability for durability. Under a single-node failure, requests expecting a
durable acknowledgement must fail clearly and predictably.

The fundamental problem is that a success from the primary today confirms **acceptance**, not a
durable **commitment** that will survive a topology change. Solving this requires a mechanism for
**quorum-based writes**, where an acknowledgement is only sent after a majority of nodes have
durably recorded the operation. This concept of a quorum-based write is embodied by both the `WAIT`
command (when used with a replica count sufficient to form a quorum) and the connection-level
durability opt-in.


## **Solution Requirements**

Based on the durability gaps and user needs identified, any proposed solution must adhere to the
following principles and constraints:

### **Core Durability Principles**

* **Authoritative Acknowledgement**: A durable write's acknowledgement must be tied to a single,
authoritative decision at the shard level, ensuring it remains valid through any topology change
like a leader election or slot migration.
* **Membership-Awareness**: During a cluster change like a slot migration, a durability
confirmation must come from the nodes that will be the **final, authoritative owners** of the
data. This prevents a "lame duck" quorum, a group of nodes that is about to lose authority, from
confirming a write that the new owner has never seen, which would otherwise lead to data loss.
* **Clear Failure Modes**: If a client requests a durable write and the server cannot confirm it
within the timeout, the server must return an explicit error, not an ambiguous success. This
prevents a "silent downgrade" in durability where a client might incorrectly assume a write was
successful. This clear error signal informs the client that the write's state is unknown, enabling
the application to safely retry the operation, which should be designed to be idempotent.

### **Interface and Usability Constraints**

* **Preserve <code>WAIT</code> Command Interface**: The syntax (`WAIT <replicas> <timeout>`) and
integer return value of the existing command must not be changed.

* **Flexible Durability Controls**: The solution must also provide a connection-level opt-in for
durable-by-default behavior.

### **Architectural Constraints**

* **No Performance Regression for Non-Durable Workloads**: The chosen durability mechanism must be
implemented in a way that does not prevent workloads that do not require strong durability from
retaining their existing high-performance characteristics. The system must continue to offer a path
that avoids the I/O and network latency costs of a durable, quorum-based commit.

* **Foundation for Future Features**: The chosen durability mechanism should provide a clear and
extensible foundation for future high-value features, most notably **data tiering**. This would
enable Valkey to cost-effectively manage datasets that are significantly larger than available
memory.
