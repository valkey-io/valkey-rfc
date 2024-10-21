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

1.	Support all existing Valkey commands 

2.	Shield the functional/API differences between cluster mode and primary-replica mode, so that clients do not need to care about the underlying operating mode when using it.

3.	Support multi-tenant mode

4.	Support multi-database mode even server runs as cluster mode

5.	By using Proxy, clients have no perception for server failover, slot migration, adding/removing server nodes, proxy nodes and other exceptions.

6.	Support some enhancement proxy commands to count the time consumption or high CPU cost in memory: min/max/avg/p95/p99

7.	Support audit log and reading/writing separation features

8.	Provide a connection pool to facilitate client connection

9.	For the customer's persistent connection, the proxy will not actively disconnect, but only returns the error message; If the client is disconnected, the connection will be disconnected immediately

10.	Support CLIENT LIST etc commands aggregate return: return the client of all proxy nodes

11.	Some commands such as set and zset are supported for aggregation processing across slot cluster connections


## Specification

## Authentication and Authorization (Optional)
## If there are any changes around introducing new ACL command/categories for user access control.

## Configuration (Optional)
## If there are any configuration changes introduced to enable/disable/modify the behavior of the feature.

## Cluster mode (Optional)
## If there is any special handling for this feature (e.g., client redirection, Sharded PubSub, etc) in cluster mode or if there are any new cluster bus extensions or messages introduced, list out the changes.


## References
https://github.com/RedisLabs/redis-cluster-proxy
