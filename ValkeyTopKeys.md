---
RFC: (PR number)
Status: Proposed
---

# Top Keys analysis

## Abstract (Required)

Hot key is a generic term for a key whose presence or access pattern leads to elevated consumption of cluster resources such as CPU, network bandwidth, or memory allocation within a slot.
Typically, hot keys arise due to application design issues or server-side bugs.  

This document describes the mechanisms to identify server-side bottlenecks caused by hot keys and proposes a strategic design for **on-demand hot key detection** in Valkey, 
focusing on access frequency and key cardinality.

---

## Motivation

The goal is to provide operators with **real-time ability to identify keys that are primary contributors to system load**, 
allowing them to perform targeted mitigations such as key deletion, redistribute slots or scaling.  

Some examples where a specific key can contribute to resource consumption include:  
1. Extremely large hash tables can generate large network spikes when commands like `HGETALL` are executed.  
2. Very large sets can cause extended server unresponsiveness when executing commands such as `SDIFFSTORE`.  
3. Single hot keys accessed disproportionately frequently can overload a node (high CPU usage, increased latency).  
4. Very large keys can contribute to memory imbalance across cluster slots, impacting slot placement decisions during scaling.
5. Unbalanced cluster slots memory can be a result of application side bug causing a specific key to become much bigger than the average key size.    

Per slot statistics can also assist in identifying cluster wide hotspots in a Valkey cluster, however they are limited in their ability to pin-point the specific item
which is the top contributor.
using external client tools can also be used in order to identify hot-keys, big-keys and mem-keys, by scanning all of the cluster dataset (eg `valkey-cli`). 
However there are several issues with this method:
1. Scanning through the entire dataset can take relatively long time and can result in a multi hours wait time, until the full statistics are finalized.
2. hot-keys analysis is currently only possible when the eviction policy is set to LFU (when access frequency is managed on per-key level)
3. This require client side implementation or application logic which needs to be built and maintained.   

In this proposal We will provide a top contributor analysis, rather than a filtered set or distribution analysis.
Existing Valkey observability tools like `SLOWLOG` and `COMMANDLOG` already support tracking slow commands or network bursts.
Therefore, this proposal focuses on **hot key access frequency**, **key cardinality** and **key memory** , which are the most actionable signals during incidents.

### Do we need to provide top contributor analysis based on key's memory footprint?
Key cardinality can be considered as a good estimation for memory consumption, since it is expected that big keys will also consume more memory.
Memory is more important in the slot level, where a potential control plane implementation might need to take scaling decisions, based on memory distribution across slots.
For this reason it might be a low hanging fruit to support per key memory tracking which can be followed by a memory top-contributor analysis.

## Design considerations

Slot-Level Granularity - Dataset-level metadata SHALL be maintained at **slot granularity** to avoid performance bottlenecks due to locks or async slot operations (e.g., `FLUSHSLOT`, atomic slot migration, or command offloading).

## Specification (Required)


### 1. On-Demand Availability

- Users typically discover a problem first and only then need to debug it.  
   Hot key statistics SHALL be retrievable **on-demand** .
- The feature MUST NOT require advance opt-in or long warm-up periods to be useful during an ongoing    
   incident.

---

### 2. Configuration

We cannot require the user to plan for the specific debug case he did not expect, Since these are mostly used in order to debug UNEXPECTED events.
For this reason, activating the feature should not be a blocker to identify an already-ongoing situation.
Fine tuning configurations, can be considered, but also cannot become a blocker to identify an already ongoing issue.

---

### 3. Integrability

- Output MUST be suitable for aggregation into cluster-wide or database-wide views.
- Node-level results are acceptable.
- Results MUST support deterministic merging by external clients.

---

### 4. Performance

- Top-N queries MUST execute in bounded constant time.
- Memory overhead for tracking statistics MUST be bounded and independent of total key count.

---

### 5. Accuracy

- Approximate statistics are acceptable.
  However, True heavy contributors MUST converge into Top-N results after a predictable observation period.
- The design SHOULD avoid cases where a dominant key is permanently missing from results.

---

### 6. Replication

- Statistics SHOULD be available on both primaries and replicas.
- Steady state Replication should update statistics.
- RDB/AOF/FullSync load should NOT update **HotKey** statistics.

---

### Feature Overview

### Metrics Tracked
1. **Write Access Frequency**
   - Refers to the access rate of a key by a WRITE command.
   - Number of write operations per key over a configurable time window.
2. **Read Access Frequency**
   - Refers to the access rate of a key by a READ command.
   - Number of read operations per key over a configurable time window.
3. **Key Cardinality**
   - Refers to the cardinality of the key. This will be the number of bytes in a string type, number of items in a hash type, number of items in a stream type etc...
4. **Key Memory Usage**
   - Refer to the amount of memory consumed by a key. This should be identical to the output of the command `MEMORY USAGE <key>`

---

### Commands (Optional)
### HOTKEYS
Returns Top-N keys by access frequency.
```
HOTKEYS <READ | WRITE> TOP <N>
```
**Response:**
```
1) 1) "key_name"
   2) (float) <qps>
2) 1) "another_key"
   2) (float) <qps>
3) ...
```
---

### TOPKEYS

Returns Top-N keys by size characteristics.
```
TOPKEYS <CARD | MEMORY> TOP <N>
```
**Response:**
```
1) "string"
   2) 1) 1) "keyStrA"
         2) (integer) <value>
      2) 1) "keyStrB"
         2) (integer) <value>
      ...
2) "hash"
   2) 1) 1) "keyHashA"
         2) (integer) <value>
      2) 1) "keyHashB"
         2) (integer) <value>
      ...
3) "list"
   2) 1) 1) "keyListA"
         2) (integer) <value>
      2) 1) "keyListB"
         2) (integer) <value>
      ...
4) "set"
   2) ...
5) "stream"
   2) ...
```
**Value Semantics**
- `CARDINALITY`: Type-specific item count
- `MEMORY`: Bytes used (equivalent to `MEMORY USAGE`)

---

### Authentication and Authorization (Optional)

- Access to hot-key statistics SHALL respect existing ACL rules.  
- Optional: Introduce `HOTKEYS` category for fine-grained control.

---

### Append-only file (Optional)

- No changes are required to the append-only file (AOF) format.  
- Summaries are ephemeral and do not persist per-command data.

---

### RDB (Optional)

- No changes are required to snapshotting or RDB persistence.  
- Statistics are reconstructed on server restart.

---

### Configuration (Optional)

These are high-level suggested configurations. Additional tuning options may be added later.

### Hot Keys

`hotkeys-max-n <integer>` 
Maximum number of keys returned for HOTKEYS.
Controls memory usage and ensures Top-N lists remain bounded.
Default: 10

`hotkeys-read-access-threshold <integer>` - default 3000
`hotkeys-write-access-threshold <integer>` - default 2000
Threashold configuration.  Only keys exceeding these QPS thresholds appear in HOTKEYS output. Prevents low-activity keys from cluttering results.

### TopKeys 

`topkeys-min-card <integer>`
Keys must exceed this cardinality to appear in TOPKEYS CARDINALITY.
default 1024

`topkeys-min-memory <memory>`
Only keys consuming more memory than this appear in TOPKEYS MEMORY.
default 1mb


---

### Keyspace notifications (Optional)

- No changes are required. Operators can optionally subscribe to keyspace events for correlated debugging.

---

### Cluster mode (Optional)

- Slot-level summaries enable safe operation under:
  - Slot migration
  - Partial node failure
  - Resharding  
- Cluster-wide top-key statistics are computed via client-side merging of node/slot summaries.

---

### Module API (Optional)

Module types have no existing API to expose their cardinality. For this reason, an extension of the API is required.
We will introduce a module Type extension version which will include `moduleTypeCardFunc`, a way to retrieve a specific module type   
instance cardinality.


