---
RFC: 10
Status: <Proposed>
---

# ValkeyProxy RFC

## Abstract  (Required)

The proposed ValkeyProxy is to provide the features similar to redis-cluster-proxy. 


## Motivation (Required)

When a customer connects directly to the Valkey server, it may encounter the following situations:

a. In primary-replica  mode, the node is switched over from primary to replica, and the client needs to perform special processing

b. In cluster mode, if the primary node fails, the customer needs to perform special handling in the process of re-selecting the primary or the process of data migration

c. When the primary node processes a large key, it may fail to respond to other nodes for a long time, resulting in disconnection of the client

d. When the number of connections to a node exceeds the maximum value, the client may not be able to connect to the primary node

Therefore, we need a proxy to make the customer unaware of the above problems, and also to provide some additional Valkey services


## Design Considerations  (Required)

1. All Valkey commands should be supported
2. Provide a connection pool to facilitate client connection
3. Proxy can automatically handle the primary/replica switchover mode, and the customer is not aware of it
4. Primary/replica mode and cluster mode support multi-tenant
5. Support all Valkey commands to count the time consumption in memory: min/max/avg/p95/p99
6. Multi-database support in cluster mode
7. For the customer's persistent connection, the proxy will not actively disconnect, but only returns the error message; If the client is disconnected, the connection will be disconnected immediately
8. Support CLIENT LIST command aggregate return: return the client of all proxy nodes
9. Some commands such as set and zset are supported for aggregation processing across slot cluster connections 


## Specification

## Authentication and Authorization (Optional)
## If there are any changes around introducing new ACL command/categories for user access control.

## Configuration (Optional)
## If there are any configuration changes introduced to enable/disable/modify the behavior of the feature.

## Cluster mode (Optional)
## If there is any special handling for this feature (e.g., client redirection, Sharded PubSub, etc) in cluster mode or if there are any new cluster bus extensions or messages introduced, list out the changes.


## References
https://github.com/RedisLabs/redis-cluster-proxy
