---
RFC: 26
Status: Proposed
---

# Title
External data storage (values only)

## Abstract
Currently Valkey allows to store data in memory only. The option to store data externally (on local or network disk, e.g. SSD) is proposed here.
Only values are about to be stored externally - option to store both keys and values externally is proposed in another RFC (#25).

## Motivation
1) Cost saving - disks are cheaper than memory.
2) Larger storage - memory ends up fast in some use cases.
3) NOT a data durability enhancement - this is out of scope of this initiative.

## Design considerations
0) The whole thing on this stage is about strings only - it's not about the impossibility or lack of demand for other Valkey data structures (sets, hashes, etc.), it's about the scope. Other data structures are supposed to be added to scope based on community requests after successful implementation of the feature for strings.
1) Keys are just stored as usual; values are stored neither in Valkey dict, nor in any Valkey directly controlled memory space.
2) Values externally stored are interacted via module - exact implementation of how this is done is up to module creator.

## Specification
VC<N> = Valkey config option with its number for reference between RFC parts
VF<N> = flag for existing Valkey command with its number for reference between RFC parts
* Passing value storage<->client should be made in a way to prevent OOM during loading large data to limited memory.

### GET flow
1) Client requests a key.
2) Valkey dict contains the key with attribute "memory-stored" OR missing dict when requested without flag (VF1) - end of story.
3*) Key attribute is "external-stored" - value is requested from external storage.
4) If value is missing: it's a replica and there's full or partial sync in progress - nothing done, otherwise delete the key; return none to client, end of story; 
5) If value exists - return it to client.

### SET flow
1) Client puts a key.
2) Value is big enough (VC1) to be stored to external storage instead of Valkey dict OR directly mentioned as to be externally stored (VF2).
3*) Value is put to external storage, key receives attribute "external-stored".
4) Value is deleted from Valkey dict, if the key was in memory before current SET.
5) Timestamp in terms of external storage on successful value write is returned to core.

### Expiration\DEL flow
1) On expire\DEL mark the value as to be deleted from external storage.

### SCAN/KEYS flow
1) No changes.

### FLUSHALL/FLUSHDB/SWAPDB flow
1) Requires storing different databases externally separately enough to quickly drop|swap its data.

### DBSIZE flow
1) Requires value dynamic counter for each database to quickly return it without recounting on each request.

### Backup flow
1) External storage data should not be incorporated into RDB - this allows running Valkey core and external storage with its underlying data on different disks, either local or network ones.
2) RDB and external storage backup should be synchronized to match keys in memory with external values - thus running (BG)SAVE in core should initiate DUMP command for local external storage, which should be performed only after currently running values changing operations are done and not before starting new ones (some lock seems to be unavoidable here).

### External storage value lifecycle
1) Value is stored in external storage (by size or directly).
2) Space is freed somehow by module implementation, it's up to module creator with these limitations:
- Valkey expiration mechanism doesn't touch external data anyhow.
- The maximum living time since last read|write for value external storage could be set in Valkey config (VC2), it's existence is not guaranteed after that (0=don't gc ever - only by direct DEL command, -1=gc any time).
- Value is not supposed to be returned back to memory from external storage on any occasion - this could be introduced separately later with some config options if that's really a latency problem for somebody in some usecases (doing that naively and by default could lead to ping-pong between memory and external storage slowing down database in whole for most or even all keys).
- Valkey key is not deleted right after value freeing initiated by external storage - it gets removed on accessing it (see GET flow).

### Commands
1. EXT_DATA DEBUG (see Debug mechanism lower)
2. EXT_DATA LOADED: list previuosly loaded modules that implement external storage functionality (to get module names for INIT command)
3. EXT_DATA INIT <db> <module>: initialize previously loaded module with name <module> for a specified database with name <db>
4. EXT_DATA DROP <db>: drop previously initialized for a database with name <db> external storage with possible complete data loss (i.e. without guarantee that any externally storage data persist after execution)
5. EXT_DATA STATS: get memory and disk used for external data (and any other necessary|useful stats either)
6. EXT_DATA DUMP [<slot>] [<timestamp>] [<target>]: create external storage snapshot for a certain <slot> (not specified by default=no slot specific) starting from <timestamp> (missing timestamp by default=full snapshot) on <target> external storage instance (no <target> by default=local instance), returns its <id>
7. EXT_DATA LOAD <id>: replace or append (i.e. full or incremental, this is supposed to be specified in backup metadata) current external storage state with state from provided backup identified by its <id>

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
(VC0) extdata-mode[string]: external storage option - could be used to choose between key-value, values only, any other possible mode or none[default] which turns this whole feature off
(VC1) extdata-store-by-size[int]: 0=[default]don't store data by size
(VC2) extdata-ttl[int]: 0=don't clean out data written to external storage ever - only by direct DEL command, -1=[default]data could be cleaned up any time after writing to external storage (i.e. depends on current module implementation)

### Keyspace notifications
1. Key is directly set to external storage.
2. Key is explicitly deleted from external storage.
3. Key is deleted because appropriate value was freed from external storage (see GET flow).

### Cluster mode
1) External storage needs to be aware of cluster slot for each value stored.
2) During processing commands for moving slots core initiates DUMP <slot> <target> external storage command which returns backup <id>.
3) On <target> LOAD <id> external storage command is initiated.

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
3) Use LOAD external storage command on replica instance to load external storage backup.

#### Full sync flow
1) Reusing (BG)SAVE mechanism (see Backup flow) to initiate external storage backup creation - but with DUMP <target> option, where <target> is replica's coordinates.
2) Use LOAD external storage command on replica instance to load external storage backup.

### Networking
1. No changes.

### Dependencies
1. No changes.

### Observability
1. Add "# External storage" block to INFO output with the following stats:
extdata_memory_used
extdata_memory_used_human
extdata_disk_used
extdata_disk_used_human
db<N>:keys=<K>

### Debug mechanism
1. EXT_DATA DEBUG SET: set a value for a certain key directly to external storage
2. EXT_DATA DEBUG DEL: delete a value for a certain key directly from external storage
3. EXT_DATA DEBUG BLOCK: block writing operations
4. EXT_DATA DEBUG UNBLOCK: unblock writing operations