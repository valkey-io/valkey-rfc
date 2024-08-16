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

A description of the design constraints and requirements for the proposal, along with comparisons to similar features in other projects.

## Specification (Required)

A more detailed description of the feature, including the reasoning behind the design choices.

### Commands (Optional)

If any new commands are introduced:

1. Command name
   - **Request**
   - **Response**

### Authentication and Authorization (Optional)

If there are any changes around introducing new ACL command/categories for user access control.

### Append-only file (Optional)

If there are any changes around the persistence mechanism of every write operation.

### Configuration (Optional)

If there are any configuration changes introduced to enable/disable/modify the behavior of the feature.

### Keyspace notifications (Optional)

If there are any events to be introduced or modified to observe activity around the dataset.

### Cluster mode (Optional)

If there is any special handling for this feature (e.g., client redirection, Sharded PubSub, etc) in cluster mode, list out the behavior.

### Module API (Optional)

If any new module APIs are needed to implement or support this feature.

### RDB (Optional)

If there are any changes in snapshotting mechanisms like new data type, version, etc.

### Replication (Optional)

If there are any changes required in the replication mechanism between a primary and replica.

### Networking (Optional)

If there are any changes introduced in the RESP protocol (RESP), client behavior, new server-client interaction mechanism (TCP, RDMA), etc.

### Dependencies (Optional)

If there are any new dependency libraries required to support the feature. Existing dependencies are jemalloc, lua, etc.

### Metrics (Optional)

If there are any new metrics to be introduced to observe behavior or measure the performance of the feature.

### Benchmarking (Optional)

If there are any benchmarks performed and preliminary results are available to share or a set of scenarios identified to measure the feature's performance.

### Testing (Optional)

If there are any test scenarios planned to ensure the feature's stability and validate its behavior.

### Debug mechanism (Optional)

If there is any debug mechanism introduced to support admin/operators for maintaining the feature.

## Appendix (Optional)

Links to related material such as issues, pull requests, papers, or other references.
