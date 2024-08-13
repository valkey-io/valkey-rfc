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

If any new commands are introduced:

1. Command name
   - **Request**
   - **Response**

### Authentication and Authorization (Optional)

If there are any changes around authentication and authorization (ACL) access for commands/keyspace/channels.

### Append-only file (Optional)

If there are any changes around the persistence mechanism of every write operation.

### Configuration (Optional)

If there are any configuration changes introduced to enable/disable/modify the behavior of the feature.

### Keyspace notifications (Optional)

If there are any events to be introduced or modified to observe activity around the dataset.

### Cluster mode (Optional)

If there are any changes introduced for cluster mode related activities like cluster bus protocol change, slot migration, etc.

### Module API (Optional)

If any module API is modified or introduced to support dynamic libraries.

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

If there are any test scenarios planned out to support the feature's stability and validate behavior.

### Debug mechanism (Optional)

If there is any debug mechanism introduced to support admin/operators for further maintaining the feature.

## Appendix (Optional)

Links to related material such as issues, pull requests, papers, or other references.
