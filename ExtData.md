---
RFC: 25
Status: Proposed
---

# Title
External data storage

## Abstract
Currently Valkey allows to store data in memory only. The option to store data externally (on local or network disk, e.g. SSD) is proposed here.
Two options for different external storage (ES) scope are proposed here:
- only-values (OV);
- keys-and-values (KV).

## Motivation
1) Cost saving - disks are cheaper than memory.
2) Larger storage - memory ends up fast in some use cases.
3) NOT a data durability enhancement - this is out of scope of this initiative.

## Design considerations
0) The whole thing on this stage is about strings only - it's not about the impossibility or lack of demand for other Valkey data structures (sets, hashes, etc.), it's about the scope. Other data structures are supposed to be added to scope based on community requests after successful implementation of the feature for strings.
1) Values are stored neither in Valkey dict, nor in any Valkey directly controlled memory space.
2) Values externally stored are interacted via module - exact implementation of how this is done is up to module creator.
3) Keys:
- [OV] are stored as usual;
- [KV] same as values in pp.1-2.
4) Client does not interact ES module directly - only via core. Passing value[OV] or key/value[KV] ES<->core<->client should be made in a way to prevent (e.g. via BIO-jobs):
- OOM during loading large data to limited memory (for ES->core and client->core);
- blocking main thread (for core<->ES).

## Specification
VC<N> = Valkey config option with its number for reference between RFC parts
VF<N> = flag for existing Valkey command with its number for reference between RFC parts

### [OV]GET flow
1) Client requests a key.
2) Valkey dict contains the key with attribute "memory-stored" OR missing dict when requested without flag (VF1) - end of story.
3) Key attribute is "external-stored" - value is requested from ES.
4) If value is missing:
4.1)
- it's a replica - send DEL key to master;
- full or partial sync in progress - create BIO-job to delete key later;
- else - delete the key.
4.2) return none to client, end of story.
5) If value exists - return it to client.

### [KV]GET flow
1) Client requests a key.
2) Valkey dict is missing the key.
3) Key is checked against the ES, if it's out OR requested without flag (VF1) - end of story.
4) It's in - value is requested from ES, returned to client.

### SET flow
1) Client puts a key.
2) Value is big enough (VC1) to be stored to ES instead of Valkey dict OR directly mentioned as to be externally stored (VF2).
3)
- [OV] Value is put to ES, key receives attribute "external-stored".
- [KV] Key and value are put to ES.
4) [OV]Value or [KV]Key-value/pair is deleted from Valkey dict, if the key was in memory before current SET.
5) Timestamp in terms of ES on successful [OV]value or [KV]key-value write is returned to core.

### [OV]Expiration flow
1) On expire mark the value as to be deleted from ES.

### [KV]Expiration flow
0) Viable for allkeys-* policies only (noeviction=always in memory, volatile-*=makes it excessively difficult to manage between mem and non-mem keys) with Valkey config option (VC3) to enable|disable that.
1) On expire SET key-value to ES instead of deleting completely.

### DEL flow
1) On DEL mark the [OV]value or [KV]key-value as to be deleted from ES.

### SCAN/KEYS flow
1)
- [OV]No changes.
- [KV]Getting keys without flag (VF3) is similar to existing; with flag requires iterator module API.

### FLUSHALL/FLUSHDB/SWAPDB flow
1) Requires storing different databases externally separately enough to quickly drop|swap its data.

### DBSIZE flow
1) Requires [OV]value or [KV]key-value dynamic counter for each database to quickly return it without recounting on each request.

### Backup flow
1) ES data should not be incorporated into RDB - this allows running Valkey core and ES with its underlying data on different disks, either local or network ones.
2) DUMP command to save current state of ES, LOAD command to restore this state later.
3) [OV]RDB and ES backup should be synchronized to match keys in memory with external values - thus running (BG)SAVE in core should initiate DUMP command for local ES, which should be performed only after currently running values changing operations are done and not before starting new ones (some lock seems to be unavoidable here).

### ES [OV]value or [KV]Key-value lifecycle
1) [OV]Value or [KV]Key-value is stored in ES (by size, directly or [KV]expiration).
2) Space is freed somehow by module implementation, it's up to module creator with these limitations:
- Valkey expiration mechanism doesn't touch external data anyhow.
- The maximum living time since last read|write for [OV]value or [KV]key-value ES could be set in Valkey config (VC2), it's existence is not guaranteed after that.
- [OV]Value or [KV]Key-value is not supposed to be returned back to memory storage from ES on any occasion - this could be introduced separately later with some config options if that's really a latency problem for somebody in some usecases (doing that naively and by default could lead to ping-pong between memory and ES slowing down database in whole for most or even all keys).
- [OV]Memory-stored key is not deleted right after its corresponding value freeing initiated by ES - it gets removed on accessing it (see GET flow).

### Commands
1. EXT_DATA DEBUG (see Debug mechanism lower)
2. EXT_DATA LOADED: list previuosly loaded modules that implement ES functionality (to get module names for INIT command)
3. EXT_DATA INIT <db> <module>: initialize previously loaded module with name <module> for a specified database with name <db>
4. EXT_DATA DROP <db>: drop previously initialized for a database with name <db> ES with possible complete data loss (i.e. without guarantee that any externally storage data persist after execution)
5. EXT_DATA STATS: get memory and disk used for external data (and any other necessary|useful stats either)
6. EXT_DATA DUMP [<slot>] [<timestamp>] [<target>]: create ES snapshot for a certain <slot> (not specified by default=no slot specific) starting from <timestamp> (missing timestamp by default=full snapshot) on <target> ES instance (no <target> by default=local instance), returns its <id>
7. EXT_DATA LOAD <id>: replace or append (i.e. full or incremental, this is supposed to be specified in backup metadata) current ES state with state from provided backup identified by its <id>

### Authentication and Authorization
1. @extdatadebug: EXT_DATA DEBUG subcommands
2. @extdatainit: EXT_DATA INIT and DROP subcommands
3. @extdatainfo: EXT_DATA LOADED and STATS subcommands
4. @extdatabackup: EXT_DATA LOAD and DUMP subcommands

### Append-only file
1. See VF2.

### RDB
1. Every key should store new attribute ("memory-stored" or "external-stored").

### Configuration
(VC0) extdata-mode[string]: ES option - could be used to choose between key-value, values only, any other possible mode or none[default] which turns this whole feature off
(VC1) extdata-store-by-size[int]: 0=[default]don't store data by size
(VC2) extdata-ttl[int]: 0=don't clean out data written to ES ever - only by direct DEL command, -1=[default]data could be cleaned up any time after writing to ES (i.e. depends on current module implementation)
[KV](VC3) extdata-expire[bool]: enable=[default] expire key-value pair to ES; disable=expire key-value pair as usual.

### Keyspace notifications
1. Key is directly set to ES.
2. Key is explicitly deleted from ES.
3. [OV]Key is deleted because appropriate value was freed from ES (see GET flow).

### Cluster mode
1) ES needs to be aware of cluster slot for each value stored.
2) During processing commands for moving slots core initiates DUMP <slot> <target> ES command which returns backup <id>.
3) On <target> LOAD <id> ES command is initiated.

### Module API
1. RegisterExternalStorage: method to be used in EXT_DATA INIT
2. UnregisterExternalStorage: method to be used in EXT_DATA DROP
3. ValkeyModuleExternalStorageMethods: a struct with following attributes:
    uint64_t version; /* Version of this structure for ABI compat. */

    /* The callback function called when `SET` command is called in this storage. */
    ValkeyModuleExternalStorageSetFunc set;

    /* The callback function called when `GET` command is called in this storage. */
    ValkeyModuleExternalStorageGetFunc get;

    /* The callback function called when `DEL` command is called in this storage. */
    ValkeyModuleExternalStorageDelFunc del;

    /* The callback function called when `DEBUG SETRO` command is called in this storage. */
    ValkeyModuleExternalStorageSetReadonlyFunc set_readonly;

    /* The callback function called when `DEBUG DROPRO` command is called in this storage. */
    ValkeyModuleExternalStorageDropReadonlyFunc drop_readonly;

    /* The callback function called when `KEYS\SCAN` command is called in this storage. */
    ValkeyModuleExternalStorageIterateFunc iterate;
	
	/* The callback function called when `DBSIZE\INFO` command is called in this storage. */
    ValkeyModuleExternalStorageStatsFunc stats;
	
	/* The callback function called when `LOAD` command is called in this storage. */
    ValkeyModuleExternalStorageLoadFunc load;
	
	/* The callback function called when `DUMP` command is called in this storage. */
    ValkeyModuleExternalStorageDumpFunc dump;

### Replication
#### Partial sync flow
1) Store offset-timestamp (see SET flow, p.5) in circular linked list.
2) On receiving sucessful (not requiring full sync) PSYNC get lowest nearest timestamp from list above and run DUMP <timestamp> <target>, where <target> is replica's coordinates.
3) Use LOAD ES command on replica instance to load ES backup.

#### Full sync flow
1) Reusing (BG)SAVE mechanism (see Backup flow) to initiate ES backup creation - but with DUMP <target> option, where <target> is replica's coordinates.
2) Use LOAD ES command on replica instance to load ES backup.

### Networking
1. No changes.

### Dependencies
1. No changes.

### Observability
1. Add "# ES" block to INFO output with the following stats:
extdata_memory_used
extdata_memory_used_human
extdata_disk_used
extdata_disk_used_human
db<N>:keys=<K>

### Debug mechanism
1. EXT_DATA DEBUG SET: set a value for a certain key directly to ES
2. EXT_DATA DEBUG DEL: delete a value for a certain key directly from ES
3. EXT_DATA DEBUG BLOCK: block writing operations
4. EXT_DATA DEBUG UNBLOCK: unblock writing operations
