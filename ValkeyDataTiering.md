---
RFC: 27
Status: Proposed
---

# Valkey Data Tiering

## Abstract

This document proposes a data tiering feature for Valkey, allowing users to manage data across different storage tiers based on access patterns and performance requirements.

## Motivation

- Users need to store data in multiple storage tiers to balance cost and performance.
- Current Valkey deployment models do not support data tiering out of the box.
- Existing solutions often require significant changes to application logic or infrastructure.

## Design considerations

- The feature should be transparent to existing applications.
- It should support multiple storage backends (e.g., SSD, HDD, cloud storage).
- Users should be able to configure tiering policies based on key attributes such as access frequency, size, and TTL.
- The feature should work with existing Valkey commands, such as SET, GET, and DEL.
- Data should be automatically migrated between tiers based on the defined policy.
- Module API should be provided to allow users to interact with the tiering feature.
- Tiering Module use `anti-caching` model to external storage.

## Specification

At the beginning of detailed design, there are several key points:
1. Only implement value tiering, with keys residing in memory to reduce unnecessary performance loss.
2. The key object needs to distinguish whether it has already been tied.
3. During command processing, it is necessary to ensure that the value is in memory.

### Command Processing

There are three entry points for command processing: 
- processCommand
- scriptCall
- RM_Call

Tiering Module will intercept these entry points, and then check if the key is tied. If it is, the value will be fetched from the external storage. Otherwise, the command will be processed as usual.

The fetch action may be time-consuming, so it should be support optional synchronization and asynchrony.
- **Synchronous fetch**: The command will be blocked until the value is fetched from the external storage. like `lua` or `module`
- **Asynchronous fetch**: The command will be executed immediately, and the value will be fetched in the background, and the command will be blocked until the value is fetched.
- **Reject fetch**: The command will be rejected if the value is not in memory. like `lua` or `module`

The fetch API will have relevant flags:
- FETCH_CLIENT
- FETCH_MODULE
- FETCH_LUA

During the command execution process, it is necessary to pin the corresponding key to prevent it from being evicted by the tiering module.

### Eviction Policy

The module can register a timed event to evict the value. The eviction policy can be configured by users.

In the structure of the `serverObject`, a `tiered` needs to be added:

```c
struct serverObject {
    unsigned type : 4;
    unsigned encoding : 4;
    unsigned lru : LRU_BITS; /* LRU time (relative to global lru_clock) or
                              * LFU data (least significant 8 bits frequency
                              * and most significant 16 bits access time). */
    unsigned hasexpire : 1;
    unsigned hasembkey : 1;
    unsigned tiered : 1;
    unsigned refcount : OBJ_REFCOUNT_BITS;
    void *ptr;
};
```

When the value is tiered, `serverObject` ptr field will use `shared empty string` to save space and avoid unnecessary compatibility issues.

### Append-only file

1. No changes.

### RDB

RDB dump file will contain tiered data. Therefore, module adaptation is required at two key points:
1. Fork process
2. Fetch value in child process

#### Fork process

When forking, the parent process will notify the tiering module to generate snapshot.
Prevent data from being cleaned during the process of generating RDBs.

1. Notify tiering module to generate snapshot.
2. Wait for the snapshot to be generated.
3. Fork child process.
4. Generate RDB file success.
5. Notify tiering module that RDB file generation is complete.

#### Fetch value in child process

When the child process needs to fetch the value, it will call the fetch API of the tiering module.

We will continue to use the previous fetch api here and employ synchronous loading actions.

### Module API

The tiering module will provide the following APIs:

1. `Tiering_Register`: Register the tiering module.
2. `Tiering_Unregister`: Unregister the tiering module.
3. `Tiering_AcquireSnapshot`: Acquire snapshot.
4. `Tiering_FetchValue`: Fetch value.
5. `Tiering_ReleaseSnapshot`: Notify that RDB file generation is complete.

#### Basic Capabilities

- Support tiering for value and fetch value.
- Support eviction policy.
- Support Garbage Collection.
- Support Generate snapshot.

### Networking
1. No changes.

### Dependencies
1. No changes.

### Replication
1. No changes.
