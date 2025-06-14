---
RFC: 10
Status: <Proposed>
---

# ValkeyProxy RFC

## Abstract  (Required)

The proposed ValkeyProxy is to provide the features similar to redis-cluster-proxy. 


## Motivation (Required)

When a client connects to the Valkey server directly, it may encounter the following situations:

a. In standalone mode, when a node is switched over from primary to replica, the client needs to perform special processing.

b. In cluster mode, if the primary node fails, the client need to perform special handling in the process of re-selecting the primary or the process of data migration.

c. When the primary node processes a large key, it may fail to respond to other nodes for a long time, resulting in disconnection of the client.

d. When the number of connected clients reaches the 'maxclient' value, the client may not be able to connect to the server.

Therefore, we need a proxy to make the client unaware of the above problems, and also provide some additional Valkey services.


## Design Considerations  (Required)

1.	Support most Valkey commands. 

2.	Shield the functional/API differences between cluster mode and standalone mode, so that clients do not need to care about the underlying operating mode when using it.

        Now, If the client connects to the server directly, then in standalone mode and cluster mode, the client may need to call different code logic in the following scenarios:
        a. The client needs to save sharding metadata information in cluster mode.
        b. When failover happens in these 2 different modes, the client needs to do different processing.
        c. In cluster mode, if data migration occurs, the client must handle it.

        If the client connects to the Proxy directly instead of server node, when the client sends a command, it does not need to pay too much attention to the underlying connection mode. The Proxy will handle the execution of different modes for the client.

3.	By connecting to Proxy, clients have no perception for server failover, slot migration, adding/removing server nodes, proxy nodes and other exceptions.

        Through Proxy, we can avoid the client being directly connected to the underlying server stage. All client connections, reading and writing request will only be aware of the Proxy, so the client does not need the underlying Valkey server or how data is storaged. 

        Therefore, no matter whether failover happens, re-electing the primary node, resharding after adding a server node, or even adding a Proxy node, clients don't need to worry about it. 

        It greatly simplifies the programming complexity of the client, allowing the client to focus more on business logic without caring too much about changes in the underlying architecture.

4.      Performance optimization

        We can use Proxy to optimize in different ways for different clients. For example, we can support Pipeline technology on the Proxy side to improve the overall system response efficiency.

        For some time-consuming or high CPU cost commands, we can make statistics in Proxy side, and for some results, such as min/max/avg/p95/p99 from memtier_benchmark or valkey_benchmark, clients can getthese statistical information timely.


5.	Support audit log 

        Audit log is a log that records the operations of clients accessing Proxy and Valkey server, and can provide storage, query, and analysis functions. With the audit log function, Proxy will automatically log the read and write requests through the proxy and standardize various information such as system security events, user access records, system operation logs, and system operation status in the information system. 

        Cloud providers or system administrators can utilize these kinds of  information to perform rich log statistical summary and correlation analysis functions to achieve a comprehensive audit of information system logs.

6.      Provide reading/writing separation features

        In production environments, most scenarios involve more reading requests and less writing requests. By enabling read-write separation feature in the Proxy, we can let the primary node focus on processing write operations and let the replica handle read operations. It is very suitable for scenarios with relatively large read operations and can reduce the pressure on the primary.

        The disadvantage of read-write separation is that there is no guarantee that there will be no delay in the primary node. If there are large batches of data operations, the replica may experience a large delay. At this time, the data obtained by the read operation may be inconsistent with the primary data.


7.	Provide a connection pool to facilitate client connection.

        Connection pooling can reduce the overhead associated with opening and closing connections and with keeping many connections open simultaneously. This overhead not only includes memory needed to handle each new connection, also involves CPU overhead to close each connection and open a new one. 

        Connection pooling simplifies client application logic. Clients need to write application code to minimize the number of simultaneous open connections.
   
        We can also provide Valkey a feature similar to the connection multiplexing in RDS, also known as connection reuse. With multiplexing, the RDS can perform all the operations for a transaction using one underlying database connection, then it can use a different connection for the next transaction. Clients can open many simultaneous connections to the proxy of RDS, and the proxy keeps a smaller number of connections open to the DB instance or cluster. Doing so further minimizes the memory overhead for connections on the database server. This technique also reduces the chance of "too many connections" errors.

        The client's persistent connection will not be disconnected actively, rather than returning the error message; If the client is disconnected actively, the connection will be disconnected immediately.

8.	Support CLIENT LIST etc commands aggregate return: return the client of all proxy nodes.

9.	Some commands such as set and zset are supported for aggregation processing across slot cluster connections.

10.     Support traffic control feature within the proxy.

11.     Support Redisson Distrbution Lock if client works on cluster mode.


## References
https://github.com/RedisLabs/redis-cluster-proxy
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.howitworks.html
https://redisson.org/docs/data-and-services/locks-and-synchronizers/
