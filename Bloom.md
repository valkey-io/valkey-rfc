---
RFC: 4
Status: Accepted
---

# ValkeyBloom Module RFC

## Abstract

The proposed feature is ValkeyBloom which is a Rust based Module that brings a native bloom filter data type into Valkey.

## Motivation

Bloom filters are a space efficient probabilistic data structure that can be used to "check" whether an element exists in
a set (with a defined false-positive), and to "add" elements to a set. While checking whether an item exists, false-positives
are possible, but false negatives are not possible. https://en.wikipedia.org/wiki/Bloom_filter

To utilize Bloom filters in their client applications, users today use client libraries that are compatible with the ReBloom
including jedis, redis-py, node-redis, nredisstack, rueidis, rustis, etc. This allows customers to perform bloom filter
based operations, e.g. add and set.

Redis Ltd.‘s ReBloom is published under a proprietary license and hence cannot be distributed freely with ValKey.

There is growing [demand](https://github.com/orgs/valkey-io/discussions?discussions_q=+bloom+) for an
(1) Open Source bloom filter feature in Valkey which is (2) compatible with the ReBloom API syntax and with
existing ReBloom based client libraries. ValkeyBloom will help address both these requirements.


## Design Considerations

The ValkeyBloom module brings in a bloom module data type into Valkey and provides commands to create / reserve
bloom filters, operate on them (add items, check if items exist), inspect bloom filters, etc.
It allows customization of properties of bloom filter (capacity, false-positive rate, expansion rate, specification of
scaling vs non-scaling filters etc) through commands and configurations. It also allows users to create scalable bloom
filters and back up & restore bloom filters (through RDB load and save).

ValkeyBloom provides commands (BF.*), configs, etc to operate on Bloom objects which are top-level structures containing
lower-level BloomFilter provided by an external crate. ValkeyBloom utilizes an open source (BSD-2) Bloom Filter Rust library
around which it implements a scalable bloom filter.

When a bloom filter is created, a bit array is created with a length proportional to the capacity (number of items the user
wants to add to the filter) and hash functions are also created. The number of hash functions are controlled by the
false-positive rate that the user configures.

When a user adds an item (e.g. BF.ADD) to the filter, the item is passed through the hash functions and the corresponding
bits are set to 1. When a user checks whether an item exists on a filter (e.g. BF.EXISTS), the item is passed through the
filters and if all the resolved bits have values as 1, we can say that the item exists with a false-positive rate of
X (specified by the user when creating the filter). If any of the bits are 0, the item does not exist and the BF.EXISTS
operation will return 0.

We have the following terminologies / properties:
* Bloom Object: The top level structure representing the data type. It contains meta data and a list of lower-level
                Bloom Filters (Implemented by an external crate) in case of scaling and a single lower-level bloom filter
                in case of non scaling.
* Bloom Filter: A single bloom filter (Implemented by an external crate).
* Capacity: The number of items we expect to add to a bloom filter. This controls the size of the filter.
* False Positive Rate: The accuracy the user expects when operating (set / check) on a filter. This controls the number
                        of hash functions.
* Expansion Rate: This is used in scalable bloom filters where multiple bloom filters are stacked to allow users to
                    continue using the same bloom object when it reaches capacity by adding another filter of larger
                    capacity (expansion rate * prev filter capacity).
* Hash Functions: Hash functions used for bit check and set operations underneath.
* Hash Key: Keys used by the hashing function.


### Module OnLoad

Upon loading, the module registers a new bloom filter module based data type, creates bloom filter (BF.*) commands,
bloom specific configurations and the bloom ACL category.

* Module name: bf
* Data type name: bloomfltr
* Module shared object file name: valkeybloom.so

With the Module name as "bf", ValkeyBloom is compatible with ReBloom in its Module name which is accessible by clients
through HELLO, MODULE LIST, and INFO commands. Also, metrics and configs will be prefixed with this name (by design for Modules).

Regarding the Module Data type name, because ValkeyBloom's Module Data type (the current version) is not compatible with
ReBloom, it is not named same as ReBloom's. We are naming it "bloomfltr" and it is exactly 9 characters as enforced by
core Valkey logic for Module data types.
This will allow us to create a new Module Data Type in the future which can be compatible with the ReBloom and this will
be named the same as ReBloom's Bloom Filter data type. When we do this, we will need to support both Data Types (current
version) and the future version and their names must be unique.

### Module Unload

Once the Module has been loaded, the `MODULE UNLOAD` will be rejected since Module Data type is created on load.
Valkey does not allow unloading of Modules that exports a module data type.

```
127.0.0.1:6379> MODULE UNLOAD bloom
(error) ERR Error unloading module: the module exports one or more module-side data types, can't unload
```

### Persistence

ValkeyBloom implements persistence related Module data type callbacks for the Bloom data type:

* rdb_save: Serializes bloom objects to RDB.
* rdb_load: Deserializes bloom objects from RDB.
* aof_rewrite: Emits commands into the AOF during the AOF rewriting process.

### RDB Save and Load

During RDB Save of a bloom object, the Module will save the number of filters, expansion rate, false-positive rate.
And for every underlying bloom filter in this object, number of hashing functions, number of bits of the bit array,
bytes of the bit array itself.

We do not save the seed of the hash function used by Bloom Filters in the RDB because the Module uses a fixed seed.
During RDB Load, we restore and re-create the Bloom object using the RDB data and the fixed seed.
The main benefits to using fixed seed are that it reduces the RDB size and it simplifies the RDB save and load.

### RDB Compatibility with ReBloom

ValkeyBloom is not RDB Compatible with ReBloom.

The meta data that gets written to the RDB is specific to the Module data type's structure and struct members.
Additionally, the data within the underlying bloom filter (from the external crate) is specific to the implementation of
the bloom filter as the hash key (seed), hashing algorithm/s, raw bit array data, etc. can all vary.

Restoring a bloom filter means that items will need to resolve (through hash functions) to the same indexes of the bit
array. The same hash seed, hashing algorithms, and number of hash functions, bit array will need to be used in order for
previously "added" items to the bloom filter to be resolved through "exists" operations after restoration.

Because of this, it is not possible to be RDB compatible with ReBloom.

### AOF Rewrite handling

Module data types (including bloom) can implement a callback function that will be triggered for Bloom objects to rewrite
its data as command/s. From the AOF callback, we will handle AOF rewrite by saving a BF.LOAD command with the key, TTL, and
serialized value of the corresponding bloom object.

### Migrating workloads from ReBloom:

Customers that currently use ReBloom can move to ValkeyBloom using two approaches:

1. Create the bloom filter objects to have the same properties using BF.RESERVE / BF.INSERT on Valkey (with ValkeyBloom loaded).
Re-populate the bloom filter objects by inserting items by moving the existing bloom workload to the Valkey server
(with ValkeyBloom loaded). The workload can be moved without any errors since we are API Compatible.

Pros
* This is the simplest option in terms of effort - assuming the user is fine with recreating bloom objects and adding
    items on the Valkey server (with ValkeyBloom).

Cons
* The user will need to re-create the bloom objects (using BF.RESERVE/BF.INSERT) and populate these objects **AFTER**
    switching to Valkey (with ValkeyBloom).

2. Users can generate an AOF file from a server (that has the ReBloom module loaded) and with an on-going bloom workload
that creates bloom filters & inserts items into them. Next, this can be re-played on a Valkey Server (with ValkeyBloom loaded).
Then, the user can move their existing bloom workload to the Valkey server (with ValkeyBloom loaded).

Pros
* Bloom objects can be created on the user's existing system (with Rebloom) and the user can also populate the objects
    by adding items (BF.ADD/BF.MADD) **BEFORE** switching to Valkey (with ValkeyBloom).

Cons
* The user will need to manage AOF file generation on the server (with ReBloom) and ensure that the contents do not get
    re-written (e.g. as a result of BGREWRITEAOF) into a RDB file. This is because the RDB file with bloom object data
    generated by ReBloom is not compatible with ValkeyBloom. The main drawback here is that the AOF can exceed a size
    limit and get truncated / re-written.

### Memory Management

On Module Load, the Rust Module overrides the memory allocator that will delegates the allocation and deallocation tasks
to the Valkey server using the existing Module APIs: ValkeyModule_Alloc and ValkeyModule_Free. This API panics if unable
to allocate enough memory.

The bloom data type also supports memory management related callbacks:

* free: Frees the bloom filter object when it is deleted, expired or evicted.
* defrag: Supports active defrag for Bloom filter objects
* mem_usage: Reports bytes used by a Bloom object and is used by the MEMORY USAGE command
* copy: Supports a deep copy of bloom filter objects and is used by the COPY command
* free_effort: Determine whether the bloom object's memory needs to be lazy reclaimed or synchronously freed. We return
    the number of filters in the bloom object as the free effort and this is similar to how the core handles free_effort
    for aggregated objects.

### Replication

Every Bloom Filter based write operation (bloom object creations, scaling, setting of an item on a filter which returns 1
indicating a new entry, etc) will be replicated to replica nodes. Attempts of adding an already existing item to a bloom
object (which returns 0) will not replicated.

Note: When BF.ADD/BF.MADD/BF.INSERT commands (containing one or more items) are executed on a scalable bloom object which
is at full capacity on primary node, we check whether the item exists or not. This check is based on the configured false
positive rate. If the item is not found, the command results in scaling out by adding a new filter to the bloom object,
adding the item to it, and then replicating the command verbatim to replica nodes. However, the replicated command can
result in a false-positive when it checks whether the item exists. In this case, the scale out does not occur on the bloom
object on the replica. This can result in a slight different memory usage between primary and replica nodes which is more
apparent when bloom objects have large filters.

### Non Scalable filters

When non-scaling filters reach their capacity, if a user tries to add items to the bloom object, an error is returned. This
default behavior is based on ReBloom. This helps keep the false-positive error rate of the Bloom object to be what the user
requested when creating the bloom object.

A configuration can be used to provide an alternative behavior of allowing bloom objects to be saturated by allowing add
operations (BF.ADD/BF.MADD/BF.INSERT) to continue without being rejected even when a filter is at full capacity. This will
increase the false-positive error rate, but a user can opt into this behavior to allow add operations to "succeed".

### Scalable filters

Bloom Filters can either be configured as scalable (default) or non-scalable through specification with the BF.RESERVE or BF.INSERT command.

When scaling filters reach their capacity, if a user adds an item to the bloom object, a new bloom filter is created and
added to the list of bloom filters within the same bloom object. This new bloom filter will have a larger capacity
(previous bloom filter's capacity * expansion rate of the bloom object).

When we want to check whether an item exists on a bloom object (BF.EXISTS/BF.MEXISTS), we look through each filter
(from oldest to newest) in the object's filter list and perform a check operation on each one. Similarly, to add a new
item to the bloom object, we check through all the filters to see if the item already exists and if not, the item is
added to the current filter.

When a bloom object has a larger number of bloom filters, it will result in reduced performance.

Additionally, the default expansion rate is 2 and auto scaling of filters is enabled by default. The table below shows
the total capacity (across all filters) when there are x additional (scaled out) filters and the starting filter has a
capacity of 1.

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

Practically, we can expect scaling to stop well before the 50 range. With the default capacity of 100,000 and the default
expansion rate of 2, the total items inserted (across all filters) would exceed the 64-bit unsigned integer limit before
reaching 50 filters.

Below are results from an example performance test scenario. We have one BloomObject that is configured with the following
properties:

* Expansion rate = 1 (Scaling enabled)
* Capacity per filter = 1000 (This will be the same capacity for every subsequent filter due to expansion of 1)
* False positive rate = 0.001

In the test run, we start up ValkeyServer on a 4 core machine.

Starting the server (pinned to 2 cores):

```
valkey-server --loadmodule <path_to_module>
sudo taskset -cp 0,1 <valkey-server pid>
```

Creating the BloomObject & Running the benchmark (pinned to 1 core):

```
127.0.0.1:6379> bf.reserve key 0.001 1000 expansion 1
# Add enough items to scale out such that N additional filters are added to the bloom object "key".
sudo taskset -c 2 /home/ec2-user/valkey-benchmark -n 1000000 BF.EXISTS key item
```

**Results averaged over 3 runs:**

| Total Capacity | Filters per Bloom Object | p50 AVG   | p95 AVG   | p99 AVG   | TPS AVG       |
|----------------|--------------------------|-----------|-----------|-----------|---------------|
| 1000           | 1                        | 0.263     | 0.30833   | 0.559     | 95218.58667   |
| 10000          | 10                       | 0.26033   | 0.311     | 0.57233   | 94765.19333   |
| 25000          | 25                       | 0.255     | 0.30033   | 0.55367   | 98458.69667   |
| 50000          | 50                       | 0.26833   | 0.50833   | 0.831     | 99669.16667   |
| 100000         | 100                      | 0.487     | 0.711     | 0.99367   | 83829.30333   |
| 250000         | 250                      | 0.96167   | 1.17233   | 1.30033   | 49318.69      |
| 500000         | 500                      | 1.991     | 2.31633   | 2.40167   | 25476.69333   |
| 1000000        | 1000                     | 4.45233   | 4.76433   | 4.879     | 12034.26667   |


### Choice of Bloom Filter Library

We evaluated the following libraries:
* https://crates.io/crates/bloomfilter
* https://crates.io/crates/growable-bloom-filter
* https://crates.io/crates/fastbloom

Here are some important factors when evaluating libraries:
* Is it actively maintained?
* Has there been any breaking changes in serialization and deserialization of bloom filters over version updates?
* Performance
* Popularity (More Crate Downloads / GitHub Stars)

https://crates.io/crates/bloomfilter was chosen as it provides all the required APIs for controlling a bloom filter.
This library has also been stable through the releases. It uses SipHash as the hashing algorithm ([SipHasher13](https://docs.rs/siphasher/latest/siphasher/sip128/struct.SipHasher13.html)).

Certain libraries (e.g. fastbloom) are no longer compatible with older versions (after updates) resulting in
serialization / deserialization incompatibility.

growable-bloom-filter is a bloom filter library that implements a scalable bloom filter. However, it does not allow
control over when the scaling occurs and does not expose APIs to check the number of filters that it has scaled out to.
This crate also has had a breaking change across versions and bloom filters of the older version are not loadable on the
newer versions.


### Large BloomFilter objects

Create and Delete operations on large bloom filters take longer durations and will block the main thread for this duration. 
Because of this, the following operations will be handled differently.

**defrag callback:**

If the memory used by any bloom filter within the bloom object is greater than 4 KB (`bloom_large_item_threshold`
constant), we will skip defrag operations on this bloom object. Otherwise, we will defrag the bloom object in iterations
for each bloom filter in the bloom object.

**free_effort callback:**

This callback decides the free effort for the bloom object. If it is greater than 4 KB
(`bloom_large_item_threshold` constant), we will return 0 to use async free on the bloom object.

**write operations (BF.ADD/MADD/INSERT/RESERVE):**

If the write operation requires creation of a new bloom filter on a particular bloom object, we will compute the memory
usage of the bloom filter that is about to be created (based on capacity and false-positive rate). If the memory usage
is greater than 64 MB (`bloom_filter_max_memory_usage` constant), the write operation will be rejected.

Scalable Bloom filters will grow in used memory after creation of the bloom object - but only as a result of a BF.ADD, BF.MADD,
or BF.INSERT operation and we will reject these requests if it requires a scale out operation that would result in a
creation of a filter greater than the allowed size as explained above.

It is possible for the user to create their bloom object with an expansion rate of 1. In this case, it is possible that
the bloom object consists of a vector of several bloom filters that are just below the `bloom_filter_max_memory_usage`
threshold. In this case, the Bloom object becomes similar to the Native List data type in that we allow multiple elements
in the list, but we enforce a limit (4 GB for Lists) for each individual element in the list.

## Specification

### RDB Format

During RDB save, the Module data type callback is invoked and we save required meta data and bloom filter specific data
across every element in the bloom object's vector of filters.

```
<number-of-bloom-filters-N>
<expansion-rate>
<false-positive-rate>
<byte-vector-of-filter-#1>
<number-of-hash-fns-in-filter-#1>
<number-of-bits-in-bitmap-of-filter-#1>
<capacity-of-filter-#1>
.
.
.
<byte-vector-of-filter-#N>
<number-of-hash-fns-in-filter-#N>
<number-of-bits-in-bitmap-of-filter-#N>
<number-of-items-in-filter-#N>
<capacity-of-filter-#N>
```

### Bloom Filter Command API

The following are supported Bloom Filter commands with API syntax compatible with ReBloom:

**`BF.ADD <key> <item>`**

This API can be used to add an item to an existing bloom object or to create + add the item.
Response is in the Integer reply format.

Item does not exist (based on false-positive rate) and was successfully to the bloom filter.
If a bloom object named <key> does not exist, the bloom object is created and the item will be added to it.
```
(integer) 1
```

Item already exists (based on false-positive rate).
```
(integer) 0
```

**`BF.EXISTS <key> <item>`**

This API can be used to check if an item exists in a bloom object.
Response is in the Integer reply format.

Item exists (based on false-positive rate).
```
(integer) 1
```

Item does not exist (based on false-positive rate).
```
(integer) 0
```

**`BF.MADD <key> <item> [<item> ...]`**

This API can be used to add item/s to an existing bloom object or to create + add the item/s.
Response is the Array reply format with one or more Integer replies (one for each item argument provided).

1 indicates the item does not exist (based on false-positive rate) yet and was added successfully to the bloom filter.
If a bloom object named <key> does not exist, the bloom object is created and the item will be added to it.
0 indicates the item already exists (based on false-positive rate).
```
(integer) 1
(integer) 1
(integer) 0
```

**`BF.MEXISTS <key> <item> [<item> ...]`**

This API can be used to check if item/s exist in a bloom object.
Response is the Array reply format with one or more Integer replies (one for each item argument provided).

1 indicates the item exists (based on false-positive rate). 0 indicates the item does not exist (based on false-positive rate).
```
(integer) 1
(integer) 1
(integer) 0
```

**`BF.CARD <key>`**

This API can be used to check the number of items added to the bloom object (across all the filters).
Response is in the Integer reply format.
```
(integer) 20
```

**`BF.INFO <key> [CAPACITY | SIZE | FILTERS | ITEMS | EXPANSION]`**

The API can be used to get info statistics on the particular bloom object across all its filters.
Response is in an Array reply format with one or more Integer replies (based on whether a specific info stat is provided).

"Capacity" is the number of items that can be stored on the bloom object across its vector of bloom filters.

"Size" is the total memory used by the bloom object (including memory allocated for each bloom filter).

"Filters" is the number of bloom filters that the bloom object has. 1 indicates no scale out has occurred. >1 indicates otherwise.

"Number of items inserted" is the number of items added to the bloom object across all the filters.

"Expansion rate" defines the auto scaling behavior. -1 indicates the bloom object is a non scaling. >=1 indicates scaling.

```
127.0.0.1:6379> BF.INFO bloomobject1
 1) Capacity
 2) (integer) 1000
 3) Size
 4) (integer) 1431
 5) Number of filters
 6) (integer) 1
 7) Number of items inserted
 8) (integer) 1
 9) Expansion rate
10) (integer) 2
127.0.0.1:6379> BF.INFO bloomobject1 SIZE
1) (integer) 1431
127.0.0.1:6379> BF.INFO bloomobject1 CAPACITY
1) (integer) 1000
127.0.0.1:6379> BF.INFO bloomobject1 FILTERS
1) (integer) 1
```

**`BF.RESERVE <key> <false_positive_rate> <capacity> [EXPANSION <expansion>] | [NONSCALING]`**

This API is used to create a bloom object with specific properties.
When the command is used, only either EXPANSION or NONSCALING can be used. If both are used, an error is returned.

"CAPACITY" is the number of items that the users wants to store on the bloom object. For non scaling, this is a limit on
the number of items that can be inserted. For scaling, this is the number of items that can be added after which scaling
occurs.

"EXPANSION" can be used to create a scaling enabled bloom object with the specifed expansion rate.

"NONSCALING" can be used to indicate that the bloom object should not auto scale once items are added such that it reaches
full capacity.

The response is a simple String reply with OK indicating successful creation.
```
OK
```

**`BF.INSERT <key> [CAPACITY <capacity>] [ERROR <fp_error>] [EXPANSION <expansion>] [NOCREATE] [NONSCALING] ITEMS <item> [<item> ...]`**

This API is used to create a bloom object with specific properties and add item/s to it.

"CAPACITY" is the number of items that the users wants to store on the bloom object. For non scaling, this is a limit on
the number of items that can be inserted. For scaling, this is the number of items that can be added after which scaling
occurs.

"ERROR" is the false-positive error rate.

"NOCREATE" can be used to specify that the command should not result in creation of a new bloom object if it does not exist.
If NOCREATE is used along with CAPACITY or ERROR, an error is returned.

"EXPANSION" can be used to create a scaling enabled bloom object with the specifed expansion rate.

"NONSCALING" can be used to indicate that the bloom object should not auto scale once items are added such that it reaches
full capacity. Only either EXPANSION or NONSCALING can be used. If both are used, an error is returned.

"ITEMS" can be used to list one or more items to add to the bloom object.

The response is an array reply with one or more Integer replies.
1 indicates the item does not exist (based on false-positive rate) yet and was added successfully to the bloom filter.
If a bloom object named <key> does not exist, the bloom object is created and the item will be added to it.
0 indicates the item already exists (based on false-positive rate).
```
(integer) 1
(integer) 1
(integer) 0
```

The following are NEW commands which are not included in ReBloom:

**`BF.LOAD <key> <serialized-value-dump> <TTL>`**

Response is in the Simple String reply format. Returns OK on a successful restoration of a bloom object.
This command is only used during AOF Rewrite and is written into the AOF file to help with restoration.

```
OK
```

Currently following commands (from ReBloom) are not supported:

**`BF.LOADCHUNK <key> <iterator> <data>`**

**`BF.SCANDUMP <key> <iterator>`**

The BF.SCANDUMP command is used to perform an incremental save on specific Bloom filter object.
The BF.LOADCHUNK is used to incrementally load / restore a bloom filter object from data from the BF.SCANDUMP command.

The reason for not implementing these two commands is because the ValkeyBloom Module provides the ability to load and save BloomModule
data type items as part of RDB load and save operations. Additionally, the BF.LOAD command is supported for AOF operations
to re-create the same Bloom object. Because of these reasons, the BF.LOADCHUNK and BF.SCANDUMP APIs will not be supported.

### Configurations

The default properties using which Bloom Filter objects are created can be controlled using configs. The values of the
configs below are only used on a bloom object if the user does not specify the properties explicitly. Example: Using
BF.INSERT or BF.RESERVE can override the default properties.

Supported Module configurations:
1. bf.bloom_capacity: Controls the default capacity. When create operations (BF.ADD/MADD) are used, bloom objects created
    will use the capacity specified by this config. This controls the number of items that can be added to a bloom filter
    object before it is scaled out (scaling) OR before it rejects add operations due to insufficient capacity on the object (nonscaling).
2. bf.bloom_expansion_rate: Controls the default expansion rate. When create operations (BF.ADD/MADD) are used, bloom
    objects created will use the expansion rate specified by this config. This controls the capacity of the new filter
    that gets added to the list of filters as a result of scaling.
3. bf.bloom_fp_rate: Controls the default false-positive rate that new bloom objects created (from BF.ADD/MADD) will use.

### Constants
1. bloom_large_item_threshold: Memory usage of a bloom object beyond which bloom objects are exempted from defrag operations
    and when deleted, the Module will indicate the object's free_effort as 0 to be async freed.
2. bloom_filter_max_memory_usage: The maximum memory usage of a particular bloom filter that is allowed. Creation of
    bloom filters larger than this will not be allowed.

### ACL

The ValkeyBloom module will introduce a new ACL category - @bloom.

There are 4 existing ACL categories which are updated to include new BloomFilter commands: @read, @write, @fast, @slow.

### Keyspace Event Notification

Every bloom filter based write command (that involves mutation as explained in the section above) will be made to publish
a keyspace event after the data is mutated. Commands include: BF.RESERVE, BF.ADD, BD.MADD, and BF.INSERT.

* Event type: VALKEYMODULE_NOTIFY_GENERIC
* Event name: One of the two event names will be published based on the command & scenario:
 * bloom.add: Any BF.ADD, BF.MADD, or BF.INSERT command that results in adding item/s to a bloom object.
 * bloom.reserve: Any BF.ADD, BF.MADD, BF.INSERT, or BF.RESERVE command that results in creation of a bloom object.


Users can subscribe to the bloom events via the standard keyspace event pub/sub. For example,

```text
1. enable keyspace event notifications:
    valkey-cli config set notify-keyspace-events KEA
2. suscribe to keyspace & keyevent event channels:
    valkey-cli psubscribe '__key*__:*'
```

### Info metrics

Info metrics are visible through the `info bloom` or `info modules` command:
* Number of bloom filter objects
* Total bytes used by bloom filter objects

In addition to this, we have a specific API (`BF.INFO`) that can be used to list information and stats on the bloom object.

## References
* [ValkeyBloom GitHub Issue on the valkey project](https://github.com/valkey-io/valkey/issues/407)
* [ValkeyBloom GitHub Repo](https://github.com/KarthikSubbarao/valkey-bloom)
* [BloomFilter discussions on valkey-io](https://github.com/orgs/valkey-io/discussions?discussions_q=+bloom+)
