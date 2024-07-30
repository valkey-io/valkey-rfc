---
RFC: 4
Status: Proposed
---

# ValKeyBloom Module RFC

## Abstract

The proposed feature is ValKeyBloom which is a Rust based Module that brings a native bloom filter data type into Valkey.
The module should be API-compatible with Redis Ltd.’s ReBloom Module.

## Motivation

Bloom filters are a space efficient probabilistic data structure that can be used to “check” whether an element exists in a set (with a defined false positive), and to “add” elements to a set. While checking whether an item exists, false positives are possible, but false negatives are not possible. https://en.wikipedia.org/wiki/Bloom_filter

To utilize Bloom filters in their client applications, users today use client libraries that are compatible with the ReBloom including jedis, redis-py, node-redis, nredisstack, rueidis, rustis, etc. This allows customers to perform bloom filter based operations, e.g. add and set.

Redis Ltd.‘s ReBloom is published under a proprietery license and hence cannot be distributed freely with ValKey.

There is growing [demand](https://github.com/orgs/valkey-io/discussions?discussions_q=+bloom+) for an (1) Open Source bloom filter feature in Valkey which is (2) compatible with the ReBloom API syntax and with existing ReBloom based client libraries. ValkeyBloom can help address both these requirements.


## Design Considerations

The ValkeyBloom module brings in a bloom module data type into Valkey and provides commands to create / reserve bloom filters, operate on them (add items, check if items exist), inspect bloom filters, etc. It allows customization of properties of bloom filter (capacity, false positive rate, expansion rate, specification of scaling vs non-scaling filters etc) through commands and configurations. It also allows users to create scalable bloom filters and back up & restore bloom filters (through RDB load and save).

ValkeyBloom provides commands (BF.*), configs, etc to operate on Bloom objects which are top-level structures containing lower-level BloomFilter provided by an external crate. ValkeyBloom utilizes an open source (BSD-2) Bloom Filter Rust library around which it implements a scalable bloom filter.

When a bloom filter is created, a bit array is created with a length proportional to the capacity (number of items the user wants to add to the filter) and hash functions are also created. The number of hash functions are controlled by the false positive rate that the user configures.

When a user adds an item (e.g. BF.ADD) to the filter, the item is passed through the filter and corresponding bit are set to 1. When a user checks whether an item exists on a filter (e.g. BF.EXISTS), the item is passed through the filters and if all the resolved bits have values as 1, we can say that the item exists with a false positive rate of X (specified by the user when creating the filter). If any of the bits are 0, the item does not exist and the BF.EXISTS operation will return 0.

We have the following terminologies / properties:
* Bloom Object: The top level structure representing the data type. It contains meta data and a list of lower-level Bloom Filters (Implemented by an external crate) in case of scaling and a single lower-level bloom filter in case of non scaling.
* Bloom Filter: A single bloom filter (Implemented by an external crate).
* Capacity: The number of items we expect to add to a bloom filter. This controls the size of the filter.
* False Positive Rate: The accuracy the user expects when operating (set / check) on a filter. This controls the number of hash functions.
* Expansion Rate: This is used in scalable bloom filters where multiple bloom filters are stacked to allow users to continue using the same bloom object when it reaches capacity by adding another filter of larger capacity (expansion rate * prev filter capacity).
* Hash Functions: Hash functions used for bit check and set operations underneath.
* Hash Key: Keys used by the hashing function.


### Module OnLoad

Upon loading, the module registers a new bloom module based data type, creates bloom filter (BF.*) commands, bloom specific configurations and the bloom ACL category.

* Module name: bloom
* Data type name: bloom

### Persistence

ValKeyBloom implements persistence related Module data type callbacks for the Bloom data type:

* rdb_save: Serializes bloom objects to RDB.
* rdb_load: Deserializes bloom objects from RDB.
* aof_rewrite: Emits commands into the AOF during the AOF rewriting process.

### RDB Compatibility with ReBloom:

During RDB Save of a bloom object, we will need to save the number of filters, expansion rate, false positive rate. And for every underlying bloom filter in this object, we store its hash keys used by hashing functions, number of hashing functions, number of bits of the bit array, bytes of the bit array itself.

The data above is written into the RDB during save operations and can be restored from RDB load. However, the data that gets written to the RDB is specific to the data type's structure and struct members. Additionally, the data within the underlying bloom filter (from the external crate) is specific to the implementation of the bloom filter as the hash key (seed), raw bit array data, etc. can all vary.

Because of this, it is not possible to be RDB compatible with ReBloom. It might be possible if the RDB Save/Load of the ReBloom module is reverse engineered AND also the bloom filter implementation is changed to follow ReBloom Module's raw Bloom Filter structure syntax. Additionally, if there is any RDB change in ReBloom, this reverse engineering + structure updates will be needed once again.

For these reasons, ValkeyBloom is not RDB Compatible with ReBloom.

### AOF Rewrite handling:

Module data types (including bloom) can implement a callback function that will be triggered for Bloom objects to rewrites its data as command/s. From the AOF callback, we can handle AOF rewrite by saving a RESTORE command with the key, TTL, and serialized value of the corresponding bloom object.

There is an alternate to the approach above. It involves having the ValkeyBloom Module support BF.LOADCHUNK and BF.SCANDUMP commands and the AOF Rewrite callback can write BF.LOADCHUNK commands to restore a bloom object in interations. This approach was not chosen because we can already save and restore bloom object data from RDB data using the RESTORE command without having to support additional Module commands.

### Migrating workloads from ReBloom:

Customers that currently use ReBloom can move to ValkeyBloom using two approaches:

1. Create the bloom filter objects to have the same properties using BF.RESERVE / BF.INSERT on Valkey (with ValkeyBloom loaded). Re-populate the bloom filter objects by inserting items by moving the existing bloom workload to the Valkey server (with ValkeyBloom loaded). The workload can be moved without any errors since we are API Compatible.

Pros
* This is the simplest option in terms of effort - assuming the user is fine with recreating bloom objects and adding items on the Valkey server (with ValkeyBloom).

Cons
* The user will need to re-create the bloom objects (using BF.RESERVE/BF.INSERT) and populate these objects **AFTER** switching to Valkey (with ValkeyBloom).

2. Users can generate an AOF file from a server (that has the ReBloom module loaded) and with an on-going bloom workload that creates bloom filters & inserts items into them. Next, this can be re-played on a Valkey Server (with ValkeyBloom loaded). Then, the user can move their existing bloom workload to the Valkey server (with ValkeyBloom loaded).

Pros
* Bloom objects can be created on the user's existing system (with Rebloom) and the user can also populate the objects by adding items (BF.ADD/BF.MADD) **BEFORE** switching to Valkey (with ValkeyBloom).

Cons
* The user will need to manage AOF file generation on the server (with ReBloom) and ensure that the contents do not get re-written (e.g. as a result of BGREWRITEAOF) into a RDB file. This is because the RDB file with bloom object data generated by ReBloom is not compatible with ValkeyBloom. The main drawback here is that the AOF can exceed a size limit and get truncated / re-written.

### Memory Management

The bloom data type also supports memory management related callbacks:

* free: Frees the bloom filter object when it is deleted, expired or evicted.
* defrag: Supports active defrag for Bloom filter objects
* mem_usage: Reports bytes used by a Bloom object and is used by the MEMORY USAGE command
* copy: Supports a deep copy of bloom filter objects and is used by the COPY command
* free_effort: Determine whether the bloom object's memory needs to be lazy reclaimed or synchronously freed. We return the number of filters in the bloom object as the free effort and this is similar to how the core handles free_effort for aggregated objects.


### Replication

Every Bloom Filter based write command will be replicated to replica nodes.

### Keyspace Event Notification

Every bloom filter based write command will be made to publish a keyspace event after the data is mutated.
* Event type: VALKEYMODULE_NOTIFY_GENERIC
* Event name: command name in lowercase. e.g., BF.ADD command publishes event "bloom.add".

Users can subscribe to the bloom events via the standard keyspace event pub/sub. For example,

```text
1. enable keyspace event notifications:
    valkey-cli config set notify-keyspace-events KEA
2. suscribe to keyspace & keyevent event channels:
    valkey-cli psubscribe '__key*__:*'
```

### Scalable filters / Non Scaling filters

Bloom Filters can either be configured as scalable (default) or non-scalable through specification with the BF.RESERVE or BF.INSERT command.

When non-scaling filters reach their capacity, if a user tries to add items to the bloom object, an error is returned.

When scaling filters reach their capacity, if a user adds an item to the bloom object, a new bloom filter is created and added to the list of bloom filters within the same bloom object. This new bloom filter will have a larger capacity (previous bloom filter's capacity * expansion rate of the bloom object).

When we want to check whether an item exists on a bloom object (BF.EXISTS/BF.MEXISTS), we look through each filter (from oldest to newest) in the object's filter list and perform a check operation on each one. Similarly, to add a new item to the bloom object, we check through all the filters to see if the item already exists and if not, the item is added to the current filter.

When a bloom object has a larger number of bloom filters, it will result in reduced performance.

Additionally, the default expansion rate is 2 and auto scaling of filters is enabled by default. The table below shows the total capacity (across all filters) when there are x additional (scaled out) filters and the starting filter has a capacity of 1.

| x   | Capacity with x additional filters    |
|-----|---------------------------------------|
| 0   | 1                                     |
| 1   | 3                                     |
| 2   | 7                                     |
| 3   | 15                                    |
| 4   | 31                                    |
| 5   | 63                                    |
| 10  | 2047                                  |
| 15  | 65535                                 |
| 20  | 2097151                               |
| 25  | 67108863                              |
| 30  | 2147483647                            |
| 35  | 68719476735                           |
| 40  | 2199023255551                         |
| 45  | 70368744177663                        |
| 50  | 2251799813685247                      |
| 55  | 72057594037927934                     |
| 60  | 2305843008139952126                   |
| 65  | 73786976294838205662                  |
| 70  | 2361183241434822608686                |
| 75  | 75557862452714323477986               |
| 100 | 2535301200456458804968143894986       |

Practically, we can expect scaling to stop well before the 50 range. With the default capacity of 100,000 and the default expansion rate of 2, the total items inserted (across all filters) would exceed the 64-bit unsigned integer limit before reaching 50 filters.


### Choice of Bloom Filter Library

We evaluated the following libraries:
* https://crates.io/crates/bloomfilter
* https://crates.io/crates/growable-bloom-filter
* https://crates.io/crates/fastbloom

Here are some important factors when evaluating libraries:
* Is it actively maintained?
* Has there been any breaking changes in serialization and deserialization of bloom filters over version updates?
* Performance
* Popularity

https://crates.io/crates/bloomfilter was chosen as it provides all the required APIs for controlling a bloom filter. This library has also been stable through the releases.

Certain libraries (e.g. fastbloom) are no longer compatible with older versions (after updates) resuling in serialization / deserialization incompatibility.

growable-bloom-filter is a bloom filter library that implements a scalable bloom filter. However, it does not allow control over when the scaling occurs and does not expose APIs to check the number of filters that it has scaled out to. This crate also has had a breaking change across versions and bloom filters of the older version are not loadable on the newer versions.


### Large BloomFilter objects

Create and Delete operations on large bloom filters take longer durations and will block the main thread for this duration. 
Because of this, the following operations can be handled differently:
* defrag callback: If the memory used is greater than X MB (configurable with `bf.bloom_large_item_threshold`), we can skip defrag operations on this bloom object.
* free_effort callback: This callback decides the free effort for the bloom object. If it is greater than X MB (configurable with `bf.bloom_large_item_threshold`), we can return 0 to use async free logic.
* create operations (BF.ADD/MADD/INSERT): We can compute the estimated size of the bloom filter that will be created as a result of the operation. We can consider rejecting requests in case of unavailable memory.


## Specification

### Bloom Filter Command API

The following Bloom Filter commands are supported and their API syntax is compatible with ReBloom:
BF.ADD <key> <item>
BF.EXISTS <key> <item>
BF.MADD <key> <item> [<item> ...]
BF.MEXISTS <key> <item> [<item> ...]
BF.CARD <key>
BF.INFO <key> [CAPACITY | SIZE | FILTERS | ITEMS | EXPANSION]
BF.RESERVE <key> <false_positive_rate> <capacity> [EXPANSION <expansion>] | [NONSCALING]
BF.INSERT <key> [CAPACITY <capacity>] [ERROR <fp_error>] [EXPANSION <expansion>] [NOCREATE] [NONSCALING] ITEMS <item> [<item> ...]

Currently following commands (from ReBloom) are not supported:
BF.LOADCHUNK <key> <iterator> <data>
BF.SCANDUMP <key> <iterator>

The BF.SCANDUMP command is used to perform an incremental save on specific Bloom filter object.
The BF.LOADCHUNK is used to incrementally load / restore a bloom filter object from data from the BF.SCANDUMP command.

The reason for not implementing these two commands is because the Module provides the ability to load and save BloomModule data type items during RDB load and save. BF.LOADCHUNK and BF.SCANDUMP are APIs to load bloom filter objects through commands, but since we will provide RDB save & load, having specific commands for the same purpose was not considered as required.

Note: Based on the AOF Rewrite discussion, we will decide on whether these commands should be supported.

### Configs

The default properties using which Bloom Filter objects are created can be controlled using configs.

The following module configurations are added to customize bloom filters:
1. bf.bloom_capacity: Controls the default capacity. When create operations (BF.ADD/MADD) are used, bloom objects created will use the capacity specified by this config. This controls the number of items that can be added to a bloom filter object before it is scaled out (scaling) OR before it rejects add operations due to insufficient capacity on the object (nonscaling).
2. bf.bloom_expansion_rate: Controls the default expansion rate. When create operations (BF.ADD/MADD) are used, bloom objects created will use the expansion rate specified by this config. This controls the capacity of the new filter that gets added to the list of filters as a result of scaling.
3. bf.bloom_max_filters: Controls the maximum number of filters that any bloom object is allowed to scale out to.
4. bf.bloom_fp_rate: Controls the default false positive rate that new bloom objects created (from BF.ADD/MADD) will use.
5. bf.bloom_large_item_threshold: Memory usage of a bloom object beyond which bloom objects are exempted from defrag and are forced to be async freed (when deleted).


### ACL

The ValkeyBloom module will introduce a new ACL category - @bloom.

There are 4 existing ACL categories which are updated to include new BloomFilter commands: @read, @write, @fast, @slow.

### Info metrics

Info metrics are visible through the “info bloom or “info modules” command:
* Number of bloom filter objects
* Total bytes used by bloom filter objects


## References
* [ValkeyBloom GitHub Issue on the valkey project](https://github.com/valkey-io/valkey/issues/407)
* [ValkeyBloom GitHub Repo](https://github.com/KarthikSubbarao/valkey-bloom)
* [BloomFilter discussions on valkey-io](https://github.com/orgs/valkey-io/discussions?discussions_q=+bloom+)
