---
RFC: <PR number>
Status: <Proposed>
---

# Title (Required)

Valkey Vector Search

## Abstract (Required)

This proposed Valkey Vector Search module will introduce KNN (“Flat”) and ANN (“HNSW”) vector similarity search algorithms to provide vector search capabilities on both Valkey Cluster and Valkey Standalone. With vector search on Valkey, users can take advantage of low latency in-memory vector search in their familiar Valkey environment. 

In the current status, this RFC is meant as a placeholder and not up for review by the TSC yet. Google Cloud and AWS both have vector search implementations with different design specs, as such the design section is not fully complete yet. We will publish performance benchmarks once complete and adjust the design section to ask the TSC for feedback on both of the vector search implementations on Valkey.

The original RFC is a collaborative effort between Google Cloud (Kyle Meggs, Yair Gottdenker) and AWS (Sanjit Misra, Allen Samuels). We welcome comments and suggestions to improve the RFC.

## Motivation (Required)

With the rising popularity of generative AI, developers have turned to in-memory data stores such as Valkey or Redis for ultra-low latency (p99 single digit ms) vector search for use cases like Retrieval Augmented Generation (RAG), recommendation systems, semantic search, semantic caching, and more. For developers already using Valkey or Redis, co-locating their vectors and search functionality alongside their underlying data will accelerate and simplify adoption. And for other “greenfield” users, the ultra-low latency of Valkey vector search may attract them to replace their existing vector database for high performance use cases to achieve in-memory speeds at the highest levels of recall and throughput.

## Design considerations (Required)

Here we list Critical User Journeys (CUJs) which we believe Valkey users will desire as part of a Vector Search Module. 

Vector Search Operations CUJ - as a user, I want to perform vector search operations using commonly available clients.
Requirement: support vector insertion/mutation/deletion/query operations
Requirement: maintain compatibility with RediSearch VSS APIs, Memorystore APIs, and MemoryDB APIs as much as possible 

Performance CUJ - As a user, I want to use in-memory vector search because it provides the lowest latency, and most performant option
Requirement: low latency data ingestion
Requirement: low vector query latency
Requirement: high throughput: query + ingestion
Requirement: high throughput at high levels of recall
Requirement: [Scale up] multi-threading & vertical scaling (query + ingestion)
Requirement: cheap and fast updates to the non-vector data

Mixed Workloads CUJ - As a user with “mixed” (both vector search and non-vector search) workloads, I want to use a portion of my resources for storing and searching vectors and a portion for “normal” Valkey operations (e.g. GET, SET)
Requirement: vector search functionality can contend with resources but should not block, break, or significantly degrade performance or functionality of native Valkey operations

VSS API Compatibility CUJ - As a user, I want a compatible API to RediSearch VSS, which is supported by common clients. 
 [FT.CREATE] Index Creation
Requirement: support HNSW and FLAT vector search index using.  
Requirement: support index configurations. 
HNSW - Dimensions, Distance metrics [cosine, ip, l2], Data type 32bit, Initial capacity, M, EF_RUNTIME, EF_CONSTRUCTION.
FLAT - Dimensions, Distance metrics [cosine, ip, l2], Data type 32bit, Initial capacity.
[FT.SEARCH] Query
Requirement: support setting runtime query parameters, such as EF_RUNTIME for HNSW
[FT.DROPINDEX] Drop Index
Requirement: support dropping an index.
[FT.INFO] Info
Requirement: support exposing index statistics.

HASH and JSON Data Type support CUJ - As a user, I want to store my vectors in a HASH or JSON data type, which is natively supported in Valkey (2 RFCs are open for JSON in Valkey OSS).

Scale Up & Out CUJ - As a user, I want to be able to both scale up and out
Requirement: multi-threading for scale up
Requirement: Cluster for scale out & storing more data

Consistency CUJ - As a user, I want that the vector index follows similar consistency model as Valkey
Requirement: an index is being synchronously updated with the vector mutation command execution, HSET/HDEL/etc

Memory footprint CUJ - As a user, I want the index to consume memory based on usage and dynamically grow
Requirement: Lazy memory consumption based on the index entries and the dimensions.
Requirement: Index size should be allowed to grow dynamically with minimal performance overhead 

Replication CUJ- As a user, I want to rely on replication to provide high-availability and ensure that my data is protected by storing it across multiple nodes with low replication latency
Requirement: Support fast bootstrap
Requirement: Support index full sync on replication lag.

Monitoring CUJ - As a user, I need to be able to monitor indexes and re-indexing with simple observability metrics
Requirement: Ability to monitor index base statistics
Requirement: Ability to monitor cross indexes, aggregated, view statistics.

Backfill CUJ - As a user, I need to be able to create an index after the keyspace contains vector entries
Requirement: Index creation triggers a backfill process which indexes the existing keyspace entries.

Hybrid Query CUJ - As a user, I want to be able to perform hybrid queries to incorporate filtering criteria in addition to best-match vector search 
Requirement: Support for filtering numeric and tag-based filters (with future text and geo-filtering)
Requirement: Support for boolean combinations of operators in combination with filtering (AND, OR, NOT)

Result Set Management CUJ - As a user I want to dictate which fields are returned in a search
Requirement: Support query controls to set the desired returned fields.

Mixed Workloads CUJ - As a user with “mixed” (both insert and search) workloads, I want to support running searches during index mutation
Requirement: Support an option for users to enable search during index mutation

ACLs CUJ - As a user of Access Control Lists (ACLs), I want to manage user-based permissions, specific command permissions, etc to manage vector search access
Requirement: Existing ACls functionality should extend to vector search commands

Persistence CUJ - As a user I want to be able to leverage RDB and/or AOF for persisting and loading indexes
Requirement: Index metadata should be persisted leveraging RDB
Requirement: Index types that take significant time to rebuild, such as vector indexes, must support persistence of the index payload.

Tuning CUJ - As a user who is also managing the infrastructure, I want to tune my configurations, like number of threads, to optimize my search performance
Requirement: Configs like number of threads should be exposed and tunable to users


## Specification (Required)

Google Cloud, AWS, and the OSS community are working to create a unified codebase based on the available commercial modules. 

## Appendix (Optional)

Google
Blogs
https://cloud.google.com/blog/products/databases/memorystore-for-redis-vector-search-and-langchain-integration
https://cloud.google.com/blog/products/databases/memorystore-for-redis-cluster-updates-at-next24
Docs
https://cloud.google.com/memorystore/docs/redis/about-vector-search
AWS
Blogs
https://aws.amazon.com/blogs/aws/vector-search-for-amazon-memorydb-is-now-generally-available/
https://aws.amazon.com/about-aws/whats-new/2023/11/vector-search-amazon-memorydb-redis-preview/
Docs
https://docs.aws.amazon.com/memorydb/latest/devguide/vector-search.html

