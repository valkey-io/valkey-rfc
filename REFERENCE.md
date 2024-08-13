
---
RFC: <PR number>
Status: <Proposed>
---

# Title (Required)

## Abstract (Required)

A few sentences describing the feature.

## Motivation (Required)

What the feature solves and why the existing functionality is not enough.

## Design considerations (Required)

A description of the design constraints and requirements for the proposal. Comparisons with similar features in other projects.

## Specification (Required)

A more detailed description of the feature, including why certain details in the design have been chosen.

### Commands (Optional)

If any new commands are introduced.

1. Command name
**Request**
**Response**

### Authentication and Authorization (Optional)

If any changes are around authentication and authorization (ACL) access for commands/keyspace/channels.

### Append only file (Optional)

If any changes are introduced around persistence mechanism of every write operation.

### Configuration (Optional)

If any configuration changes are introduced to enable/disable/modify the behavior of the feature.

### Keyspace notifications (Optional)

If any event to be introduced or modified to observe activity around the dataset.

### Cluster mode (Optional)

If any change introduced for cluster mode related activity like cluster bus protocol change,slot migration, etc.

### Module API (Optional)

If there any module API modified or introduced to support dynamic libraries.

### RDB (Optional)

If any changes in snapshotting mechanism like new data type, version, etc.

### Replication (Optional)

If any changes required in replication mechanism between a primary and replica.

### Networking (Optional)

If any changes introduced in RESP protocol (RESP), client behavior, new server client interaction mechanism (TCP, RDMA), etc.

### Dependencies (Optional)

If any new dependency libraries required to support the feature. Existing dependencies are jemalloc, lua, etc.

### Metrics (Optional)

If any new metrics to be introduced to observe behavior or measure performance of the feature.

### Benchmarking (Optional)

If any benchmark performed and preliminary results are available to share or set of scenario identified to measure the feature performance.

### Testing (Optional)

If any test scenario planned out to support the feature stability and validate behavior.

### Debug mechanism (Optional)

If any debug mechanism introduced to support admin/operators for further maintaining the feature.

## Appendix (Optional)

Links to related material such as issues, pull requests, papers, or other references.