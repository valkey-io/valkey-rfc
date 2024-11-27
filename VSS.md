---

## RFC: Status: Proposed


# Valkey Search RFC

## Abstract

This proposed Valkey Search module will introduce search algorithms to provide numeric, tag and vector search capabilities on both Valkey Cluster and Valkey Standalone. With search on Valkey, users can take advantage of low latency in-memory search in their familiar Valkey environment.

## Motivation

With the rising popularity of generative AI, developers have turned to in-memory data stores such as Valkey or Redis for ultra-low latency (p99 single digit ms) vector search for use cases like Retrieval Augmented Generation (RAG), recommendation systems, semantic search, semantic caching, and more. For developers already using Valkey or Redis, co-locating their vectors and search functionality alongside their underlying data will accelerate and simplify adoption. And for other “greenfield” users, the ultra-low latency of Valkey search may attract them to replace their existing  database for high performance use cases to achieve in-memory speeds.

## Design considerations

Here we list Critical User Journeys (CUJs) which we believe Valkey users will desire as part of a Search Module.

**Search Operations CUJ** \- as a user, I want to perform search operations using commonly available clients.

- Requirement: support index insertion/mutation/deletion/query operations  
- Requirement: maintain compatibility with RediSearch VSS APIs, Memorystore APIs, and MemoryDB APIs as much as possible

**Performance CUJ** \- As a user, I want to use in-memory search because it provides the lowest latency, and most performant option

- Requirement: low latency data ingestion  
- Requirement: low latency queries  
- Requirement: high throughput: query \+ ingestion  
- Requirement: high throughput at high levels of recall  
- Requirement: \[Scale up\] multi-threading & vertical scaling (query \+ ingestion)  
- Requirement: Negligible resource consumption (CPU, memory) for Valkey applications and operations/commands that do not use the search module.

**API Compatibility CUJ** \- As a user, I want a compatible API to RediSearch VSS, which is supported by common clients. The goal is to enable applications to be portable between Redisearch and Valkey search with minimal effort. The search module will strive for the highest level of compatibility with Redisearch API for core application operations that are valid, deterministic and error-free. Other items, e.g., management or configuration commands, timing dependent operations, programmer development commands, status commands and/or error messages for invalid operations may differ substantially.

Binary RDB and replication stream compatibility with Redisearch is not needed, meaning that online/live data migration will not be possible.

**HASH and JSON Data Type support CUJ** \- As a user, I want to store my vectors in a HASH or JSON data type, which is natively supported in Valkey.

**Scale Up & Out CUJ** \- As a user, I want to be able to both scale up and out

- Requirement: multi-threading for scale up  
- Requirement: Cluster for scale out & storing more data

**Consistency CUJ** \- As a user, I want to be able to select a consistency model that matches my needs.

- Requirement: A sequential consistency model must be supported.

**Memory footprint CUJ** \- As a user, I want the index to consume memory based on usage and dynamically grow

- Requirement: Lazy memory consumption based on the index entries and the dimensions.  
- Requirement: Index size should be allowed to grow dynamically with minimal performance overhead

**Replication CUJ** \- As a user, I want to rely on replication to provide high-availability and ensure that my data is protected by storing it across multiple nodes with low replication latency

- Requirement: Support fast bootstrap  
- Requirement: Support index full sync on replication lag.

**Monitoring CUJ** \- As a user, I need to be able to monitor indexes and re-indexing with simple observability metrics

- Requirement: Ability to monitor index base statistics  
- Requirement: Ability to monitor cross indexes, aggregated, view statistics.

**Backfill CUJ** \- As a user, I need to be able to create an index after the keyspace contains vector entries

- Requirement: Index creation triggers a backfill process which indexes the existing keyspace entries.

**Hybrid Query CUJ** \- As a user, I want to be able to perform hybrid queries to incorporate filtering criteria in addition to best-match vector search

- Requirement: Support for filtering numeric and tag-based filters.  
- Requirement: Support for boolean combinations of operators in combination with filtering (AND, OR, NOT)

**Result Set Management CUJ** \- As a user I want to dictate which fields are returned in a search

- Requirement: Support query controls to set the desired returned fields.

**ACLs CUJ** \- As a user of Access Control Lists (ACLs), I want to manage user-based permissions, specific command permissions, etc to manage vector search access

- Requirement: Existing ACls functionality should extend to vector search commands

**Persistence CUJ** \- As a user I want to be able to leverage RDB and/or AOF for persisting and loading indexes

- Requirement: Index metadata should be persisted leveraging RDB  
- Requirement: Index types that take significant time to rebuild, such as vector indexes, must support persistence of the index payload.

**Tuning CUJ** \- As a user who is also managing the infrastructure, I want to tune my configurations, like number of threads, to optimize my search performance

- Requirement: Configs like number of threads should be exposed and tunable to users

## Specification

The search module allows a user to create any number of indexes. Each index consists of any number of typed fields and covers a specified subset of the keys within a Valkey database.

The creation of an index initiates a backfill process in the background that will scan the keyspace and insert all covered keys into the newly created index. Mutations of a key are automatically reflected in the index. This process is automatic after the creation of the index. Mutation of keys is explicitly allowed during the backfill process. The current state of a backfill may be interrogated by a client using the FT.INFO command.

A query operation is a two step process. In the first step, a query is executed that generates a set of keys and distances. In the second step, that list of keys and distances is used to generate a response.

The first step is performed in a background thread, while the second step must be performed by the mainthread. If a key produced by step 1 is mutated before step 2 is executed by the mainthread, the result of the query may not be consistent with the latest state of the keys.

### Commands

```
FT.CREATE <index-name>
    ON HASH
    [PREFIX <count> <prefix> [<prefix>...]]
    SCHEMA 
        (
            <field-identifier> [AS <field-alias>] 
                  NUMERIC 
                | TAG [SEPARATOR <sep>] [CASESENSITIVE] 
                | VECTOR [HNSW | FLAT] <attr_count> [<attribute_name> <attribute_value>]+)
        )+
```

The `FT.CREATE` command creates an empty index and initiates the backfill process. Each index consists of a number of field definitions. Each field definition specifies a field name, a field type and a path within each indexed key to locate a value of the declared type. Some field type definitions have additional sub-type specifiers.

For indexes on HASH keys, the path is the same as the hash member name. The optional `AS` clause can be used to rename the field if desired. This is especially useful when the member name contains special characters.

- **\<index-name\>** (required): This is the name you give to your index. If an index with the same name exists already, an error is returned.  
    
- **ON HASH** (optional): Only keys that match the specified type are included into this index. If omitted, HASH is assumed.  
    
- **PREFIX \<prefix-count\> \<prefix\>** (optional): If this clause is specified, then only keys that begin with the same bytes as one or more of the specified prefixes will be included into this index. If this clause is omitted, all keys of the correct type will be included. A zero-length prefix would also match all keys of the correct type.  
    
- Field types:  
    
  - **TAG**: A tag field is a string that contains one or more tag values.  
      
    - **SEPARATOR \<sep\>** (optional): One of these characters `,.<>{}[]"':;!@#$%^&*()-+=~` used to delimit individual tags. If omitted the default value is `,`.  
    - **CASESENSITIVE** (optional): If present, tag comparisons will be case sensitive. The default is that tag comparisons are NOT case sensitive

    

  - **NUMERIC** Field contains a number.  
      
  - **VECTOR**: Field contains a vector. Two vector indexing algorithms are currently supported: HNSW (Hierarchical Navigable Small World) and FLAT (brute force). Each algorithm has a set of additional attributes, some required and other optional.  
      
    - **FLAT:** The Flat algorithm provides exact answers, but has runtime proportional to the number of indexed vectors and thus may not be appropriate for large data sets.  
      - **DIM \<number\>** (required): Specifies the number of dimensions in a vector.  
      - **TYPE FLOAT32** (required): Data type, currently only FLOAT32 is supported.  
      - **DISTANCE\_METRIC \[L2 | IP | COSINE\]** (required): Specifies the distance algorithm  
      - **INITIAL\_CAP \<size\>** (optional): Initial index size.  
    - **HNSW:** The HNSW algorithm provides approximate answers, but operates substantially faster than FLAT.  
      - **DIM \<number\>** (required): Specifies the number of dimensions in a vector.  
      - **TYPE FLOAT32** (required): Data type, currently only FLOAT32 is supported.  
      - **DISTANCE\_METRIC \[L2 | IP | COSINE\]** (required): Specifies the distance algorithm  
      - **INITIAL\_CAP \<size\>** (optional): Initial index size.  
      - **M \<number\>** (optional): Number of maximum allowed outgoing edges for each node in the graph in each layer. on layer zero the maximal number of outgoing edges will be 2\*M. Default is 16 Maximum is 512\.  
      - **EF\_CONSTRUCTION \<number\>** (optional): controls the number of vectors examined during index construction. Higher values for this parameter will improve recall ratio at the expense of longer index creation times. The default value is 200\. Maximum value is 4096\.  
      - **EF\_RUNTIME \<number\>** (optional):  Sets the count of vectors to be examined during a query operation. The default is 10, and the max is 4096\. You can set this parameter value for each query you run. Higher values increase query times, but improve query recall.

**RESPONSE** OK or error.

```
FT.DROPINDEX <index-name>
```

The specified index is deleted. It is an error if that index doesn't exist.

- **\<index-name\>** (required): The name of the index to delete.

**RESPONSE** OK or error.

```
FT.INFO <index-name>
```

Detailed information about the specified index is returned.

- **\<index-name\>** (required): The name of the index to return information about.

**RESPONSE**

An array of key value pairs.

- **index\_name**	(string)	The index name  
- **num\_docs**	(integer)	Total keys in the index  
- **num\_records**	(integer)	Total records in the index  
- **hash\_indexing\_failures**	(integer)	Count of unsuccessful indexing attempts  
- **indexing**	(integer)	Binary value. Shows if background indexing is running or not  
- **percent\_indexed**	(integer)	Progress of background indexing. Percentage is expressed as a value from 0 to 1  
- **index\_definition**	(array)	An array of values defining the index  
  - **key\_type**	(string)	HASH. This is the only available key type.  
  - **prefixes**	(array of strings)	Prefixes for keys  
  - **default\_score**	(integer) This is the default scoring value for the vector search scoring function, which is used for sorting.  
  - **attributes**	(array)	One array of entries for each field defined in the index.  
    - **identifier**	(string)	field name  
    - **attribute**	(string)	An index field. This is correlated to a specific index HASH field.  
    - **type**	(string)	VECTOR. This is the only available type.  
    - **index**	(array)	Extended information about this internal index for this field.  
      - **capacity**	(integer)	The current capacity for the total number of vectors that the index can store.  
      - **dimensions**	(integer)	Dimension count  
      - **distance\_metric**	(string)	Possible values are L2, IP or Cosine  
      - **data\_type**	(string)	FLOAT32. This is the only available data type  
      - **algorithm**	(array)	Information about the algorithm for this field.  
        - **name**	(string)	HNSW or FLAT  
        - **m**	(integer)	The count of maximum permitted outgoing edges for each node in the graph in each layer. The maximum number of outgoing edges is 2\*M for layer 0\. The Default is 16\. The maximum is 512\.  
        - **ef\_construction**	(integer)	The count of vectors in the index. The default is 200, and the max is 4096\. Higher values increase the time needed to create indexes, but improve the recall ratio.  
        - **ef\_runtime**	(integer)	The count of vectors to be examined during a query operation. The default is 10, and the max is 4096\.

```
FT._LIST
```

Lists the currently defined indexes.

**RESPONSE**

An array of strings which are the currently defined index names.

```
FT.SEARCH <index> <query>
  [NOCONTENT]
  [TIMEOUT <timeout>]
  [PARAMS nargs <name> <value> [ <name> <value> ...]]
  [LIMIT <offset> <num>]
  [DIALECT <dialect>]
```

Performs a search of the specified index. The keys which match the query expression are returned.

- **\<index\>** (required): This index name you want to query.  
- **\<query\>** (required): The query string, see below for details.  
- **NOCONTENT** (optional): When present, only the resulting key names are returned, no key values are included.  
- **TIMEOUT \<timeout\>** (optional): Lets you set a timeout value for the search command. This must be an integer in milliSeconds.  
- **PARAMS \<count\> \<name1\> \<value1\> \<name2\> \<value2\> ...** (optional): `count` is of the number of arguments, i.e., twice the number of value name pairs. See the query string for usage details.  
- **LIMIT \<offset\> \<count\>** (optional): Lets you choose a portion of the result. The first `<offset>` keys are skipped and only a maximum of `<count>` keys are included. The default is LIMIT 0 10, which returns at most 10 keys.  
- **DIALECT \<dialect\>** (optional): Specifies your dialect. The only supported dialect is 2\.

**RESPONSE**

The command returns either an array if successful or an error.

If `NOCONTENT` is specified, then the output is .....

If `NOCONTENT` is not specified, then the output is an array of (2\*N)+1 entries, where N is the number of keys output from the search. The first entry in the array is the value N which is followed by N pairs of entries, one per key found. Each pair of entries consists of the key name followed by an array which is the result value for that key.

The result value for a key consists of a set of name/value pairs. The first name/value pair is for the distance computed. The name of this pair is constructed from the vector field name prepended with "\_\_" and appended with "\_score" and the value is the computed distance. The remaining name/value pairs are the members and values of the result key.

The query string conforms to this syntax:

```
(* | <filter_expression>)=>[ KNN <K> @<vector_field_name> $<vector_parameter_name> ]
```

Where:

- **\<vector\_field\_name\>** The name of a vector field within the specified index.  
- **\<K\>** The number of nearest neighbor vectors to return.  
- **\<vector\_parameter\_name\>** A PARAM name whose corresponding value provides the query vector for the KNN algorithm. Note that this parameter must be encoded as a 32-bit IEEE 754 binary floating point in little-endian format.  
- **\*  or \<filter\_expression\>** A \* indicates no filtering and all vectors within the index are searched. A filter expression can be provided to designate a subset of the vectors to be searched.

**Filter Expression**

A filter expression is constructed as a logical combination of Tag and Numeric search operators.

**Tag**

The tag search operator is specified with one or more strings separated by the `|` character. A key will satisfy the Tag search operator if the indicated field contains any one of the specified strings.

```
@<field_name>:{<tag>}
or
@<field_name>:{<tag1> | <tag2>}
or
@<field_name>:{<tag1> | <tag2> | ...}
```

For example, the following query will return documents with blue OR black OR green color.

`@color:{blue | black | green}`

As another example, the following query will return documents containing "hello world" or "hello universe"

`@color:{hello world | hello universe}`

**Numeric Range**

Numeric range operator allows for filtering queries to only return values that are in between a given start and end value. Both inclusive and exclusive range queries are supported. For simple relational comparisons, \+inf, \-inf can be used with a range query.

The syntax for a range search operator is:

```
@<field_name>:[ [(] <bound> [(] <bound>]
```

where \<bound\> is either a number or \+inf or \-inf

Bounds without a leading open paren are inclusive, whereas bounds with the leading open paren are exclusive.

Use the following table as a guide for mapping mathematical expressions to filtering queries:

```
min <= field <= max         @field:[min max]
min < field <= max          @field:[(min max]
min <= field < max	        @field:[min (max]
min < field < max	        @field:[(min (max]
field >= min	            @field:[min +inf]
field > min	                @field:[(min +inf]
field <= max	            @field:[-inf max]
field < max	                @field:[-inf (max]
field == val	            @field:[val val]
```

**Logical Operators**

Multiple tags and numeric search operators can be used to construct complex queries using logical operators.

**Logical AND**

To set a logical AND, use a space between the predicates. For example:

```
query1 query2 query3
```

**Logical OR**

To set a logical OR, use the `|` character between the predicates. For example:

```
query1 | query2 | query3
```

**Logical Negation**

Any query can be negated by prepending the `-` character before each query. Negative queries return all entries that don't match the query. This also includes keys that don't have the field.

For example, a negative query on @genre:{comedy} will return all books that are not comedy AND all books that don't have a genre field.

The following query will return all books with "comedy" genre that are not published between 2015 and 2024, or that have no year field:

@genre:\[comedy\] \-@year:\[2015 2024\]

**Examples of Combining Logical Operators**

Logical operators can be combined to form complex filter expressions.

The following query will return all books with "comedy" or "horror" genre (AND) published between 2015 and 2024:

`@genre:[comedy|horror] @year:[2015 2024]`

The following query will return all books with "comedy" or "horror" genre (OR) published between 2015 and 2024:

`@genre:[comedy|horror] | @year:[2015 2024]`

The following query will return all books that either don't have a genre field, or have a genre field not equal to "comedy", that are published between 2015 and 2024:

`-@genre:[comedy] @year:[2015 2024]`

### Configuration 

1\. **reader-threads:** (Integer) Controls the amount of threads executing queries.  
2\. **writer-threads:** (Integer) Controls the amount of threads processing index mutations.  
3\. **use-coordinator:** (boolean) Cluster mode enabler.

### Dependencies 

1. [Abseil](https://github.com/abseil/abseil-cpp): A collection of C++ libraries designed to augment the standard library.  
2. [gRPC](https://github.com/grpc/grpc): A high-performance, open-source universal RPC framework.  
   

Additionally, the codebase integrates a forked version of hnswlib.

### Testing 

1. **Unittesting:** code coverage currently stands at 90%.  
2. **Stability test framework:** fuzzy testing based on python.

### Observability 

The following metrics are added to the INFO command.

- **search\_total\_indexed\_hash\_keys**	(Integer)	Total count of HASH keys for all indexes  
- **search\_number\_of\_indexes**	(Integer)	Index schema total count  
- **search\_number\_of\_attributes**	(Integer)	Total count of attributes for all indexes  
- **search\_failure\_requests\_count**	(Integer)	A count of all failed requests  
- **search\_successful\_requests\_count**	(Integer)	A count of all successful requests  
- **search\_used\_memory\_human**	(Integer)	A human-friendly readable version of the search\_used\_memory\_bytes metric  
- **search\_used\_memory\_bytes**	(Integer)	The total bytes of memory that all indexes occupy  
- **search\_background\_indexing\_status**	(String)	The status of the indexing process. NO\_ACTIVITY indicates idle indexing.  
- **search\_hnsw\_create\_exceptions\_count**	(Integer)	Count of HNSW creation exceptions.  
- **search\_hnsw\_search\_exceptions\_count**	(Integer)	Count of HNSW search exceptions  
- **search\_hnsw\_remove\_exceptions\_count**	(Integer)	Count of HNSW removal exceptions.  
- **search\_hnsw\_add\_exceptions\_count**	(Integer)	Count of HNSW addition exceptions.  
- **search\_hnsw\_modify\_exceptions\_count**	(Integer)	Count of HNSW modification exceptions  
- **search\_modify\_subscription\_skipped\_count**	(Integer)	Count of skipped subscription modifications  
- **search\_remove\_subscription\_successful\_count**	(Integer)	Count of successful subscription removals  
- **search\_remove\_subscription\_skipped\_count**	(Integer)	Count of skipped subscription removals  
- **search\_remove\_subscription\_failure\_count**	(Integer)	Count of failed subscription removals  
- **search\_add\_subscription\_successful\_count**	(Integer)	Count of successfully added subscriptions  
- **search\_add\_subscription\_failure\_count**	(Integer)	Count of failures of adding subscriptions  
- **search\_add\_subscription\_skipped\_count**	(Integer)	Count of skipped subscription adding processes  
- **search\_modify\_subscription\_failure\_count**	(Integer)	Count of failed subscription modifications  
- **search\_modify\_subscription\_successful\_count**	(Integer)	Count of successful subscription modifications

## Appendix 

Google Cloud Blogs

- [https://cloud.google.com/blog/products/databases/memorystore-for-redis-vector-search-and-langchain-integration](https://cloud.google.com/blog/products/databases/memorystore-for-redis-vector-search-and-langchain-integration)  
- [https://cloud.google.com/blog/products/databases/memorystore-for-redis-cluster-updates-at-next24](https://cloud.google.com/blog/products/databases/memorystore-for-redis-cluster-updates-at-next24)

Google Cloud Docs

- [https://cloud.google.com/memorystore/docs/redis/about-vector-search](https://cloud.google.com/memorystore/docs/redis/about-vector-search)

AWS Blogs

- [https://aws.amazon.com/blogs/aws/vector-search-for-amazon-memorydb-is-now-generally-available/](https://aws.amazon.com/blogs/aws/vector-search-for-amazon-memorydb-is-now-generally-available/)  
- [https://aws.amazon.com/about-aws/whats-new/2023/11/vector-search-amazon-memorydb-redis-preview/](https://aws.amazon.com/about-aws/whats-new/2023/11/vector-search-amazon-memorydb-redis-preview/)

AWS Docs

- [https://docs.aws.amazon.com/memorydb/latest/devguide/vector-search.html](https://docs.aws.amazon.com/memorydb/latest/devguide/vector-search.html)
