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

The ValkeyBloom module brings in a new bloom module data type into Valkey and provides commands to create / reserve bloom filters, operate on them (add items, check if items exist), inspect bloom filters, etc. It allows customization of properties of bloom filter (capacity, false positive rate, expansion rate, specification of scaling vs non-scaling filters etc) through commands and configurations. It also allows users to create scalable bloom filters.

ValkeyBloom utilizes an open source (BSD-2) Bloom Filter Rust library.

### Module OnLoad

Upon loading, the module registers a new bloom module based data type and creates bloom filter (BF.* commands).

* Module name: bloom
* Data type name: bloom

### Persistence

ValKeyBloom implements persistence related Module data type callbacks for the Bloom data type:

* rdb_save: Serializes bloom objects to RDB.
* rdb_load: Deserializes bloom objects from RDB.
* aof_rewrite: Emits commands into the AOF during the AOF rewriting process.

RDB Compatibility:

AOF Rewrite handling:



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

### Scalale filters / Non Scaling filters

Bloom Filters can either be configured as scalable (default) or non-scalable.

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


### Max number of filters for scalable bloom filters

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
BF.INSERT <key> [ERROR <fp_error>] [CAPACITY <capacity>] [EXPANSION <expansion>] [NOCREATE] [NONSCALING] ITEMS <item> [<item> ...]

Currently following commands (from ReBloom) are not supported:
BF.LOADCHUNK <key> <iterator> <data>
BF.SCANDUMP <key> <iterator>

The BF.SCANDUMP command can be used to perform an incremental save on specific Bloom filter object.
The BF.LOADCHUNK can be used to incrementally load / restore a bloom filter object from data from the BF.SCANDUMP command. 

The reason for not implementing these two commands is because the Module provides the ability to load and save BloomModule data type items during RDB load and save. BF.LOADCHUNK and BF.SCANDUMP are APIs to load bloom filter objects through commands, but since we will provide RDB save & load, having specific commands for the same purpose was not considered as required.

Note: Based on the AOF Rewrite discussion, we will decide on whether these commands should be supported.

### Configs

The default properties using which Bloom Filter objects are created can be controlled using configs.

The following module configurations are added to customize bloom filters:
1. bf.bloom_capacity: Controls the default capacity. When create operations (BF.ADD/MADD) are used, bloom objects will created will use the capacity specified by this config. This controls the number of items that can be added to a bloom filter object before it is scaled out (scaling) OR before it rejects add operations due to insufficient capacity on the object (nonscaling). 
2. bf.bloom_expansion_rate: 
3. bf.bloom_max_filters:
4. bf.bloom_fp_rate:
5. bf.bloom_large_item_threshold:


### ACL

The ValkeyBloom module will introduce a new ACL category - @bloom.

There are 4 existing ACL categories which are updated to include new BloomFilter commands: @read, @write, @fast, @slow.

### Info metrics

Info metrics are visible through the “info bloom or “info modules” command:
* Number of bloom filter objects
* Total bytes used by bloom filter objects


## References
[ValkeyBloom GitHub Issue on the valkey project](https://github.com/valkey-io/valkey/issues/407)
[ValkeyBloom GitHub Repo](https://github.com/KarthikSubbarao/valkey-bloom)
[BloomFilter discussions on valkey-io](https://github.com/orgs/valkey-io/discussions?discussions_q=+bloom+)
