---
RFC: 10
Status: <Proposed>
---

# ValkeyProxy RFC

## Abstract  (Required)

The proposed ValkeyProxy is to provide the features similar to redis-cluster-proxy. 


## Motivation (Required)

When a customer connects directly to the Valkey server, it may encounter the following situations:

a. In primary-replica mode, the node is switched over from primary to replica, and the client needs to perform special processing.

b. In cluster mode, if the primary node fails, the customer needs to perform special handling in the process of re-selecting the primary or the process of data migration.

c. When the primary node processes a large key, it may fail to respond to other nodes for a long time, resulting in disconnection of the client.

d. When the number of connections to a node exceeds the maximum value, the client may not be able to connect to the primary node.

Therefore, we need a proxy to make the customer unaware of the above problems, and also to provide some additional Valkey services.


## Design Considerations  (Required)

1.	Support all existing Valkey commands. 

2.	Shield the functional/API differences between cluster mode and primary-replica mode, so that clients do not need to care about the underlying operating mode when using it.

        Now, If the client is directly connected to the server, then in primary-replica mode and cluster mode, the client may need to execute different code logic for the two modes in the following scenarios:
        a. The client needs to save sharding metadata information in cluster mode.
        b. When failover happens in these 2 different modes, the client needs to do different processing.
        c. In cluster mode, if data migration occurs, the client must handle it.

        So if the client is only connected to the Proxy instead of server node, then as long as the client is not connected inSentinel mode, then when the client sends a command, it does not need to pay too much attention to the underlying connection mode. The Proxy will handle the execution of different modes for the client.

3.	Support multi-database mode even server runs as cluster mode.

4.	By using Proxy, clients have no perception for server failover, slot migration, adding/removing server nodes, proxy nodes and other exceptions.

        Through Proxy, we can avoid the client being directly connected to the underlying server stage. All client connections, reading and writing request will only be aware of the Proxy, so the client does not need the underlying Valkey server or 
how data is storaged. 

        Therefore, no matter whether failover happens, re-electing the primary node, resharding after adding a server node, or even adding a Proxy node, clients don't need to worry about it. 

        It greatly simplifies the programming complexity of the client, allowing the client to focus more on business logic without caring too much about changes in the underlying architecture.

5.      Performance optimization

        We can use Proxy to optimize in different ways for different clients. For example, we can support Pipeline technology on the Proxy side to improve the overall system response efficiency.

        For some time-consuming or high CPU cost commands, we can make statistics in Proxy side, and for some results, such as min/max/avg/p95/p99 from memtier_benchmark or valkey_benchmark, clients can get timely statistical information.


6.	Support audit log 

        Audit log is a log that records the operations of clients accessing Proxy and Valkey server, and can provide storage,
query, and analysis functions. With the audit log function, Proxy will automatically log the read and write requests through 
the proxy and standardize various information such as system security events, user access records, system operation logs, and system operation status in the information system. 

        Cloud providers or system administrators can utilize this information to perform rich log statistical summary and 
correlation analysis functions to achieve a comprehensive audit of information system logs.

7.      Provide reading/writing separation features

        In production environments, most scenarios involve more reading requests and less writing requests. By enabling
read-write separation feature in the Proxy, we can let the primary node focus on processing write operations and let the 
replica handle read operations. It is very suitable for scenarios with relatively large read operations and can reduce the 
pressure on the primary.

        The disadvantage of read-write separation is that there is no guarantee that there will be no delay in the primary 
node. If there are large batches of data operations, the replica may experience a large delay. At this time, the data 
obtained by the read operation may be inconsistent with the primary data.


8.	Provide a connection pool to facilitate client connection.

        Connection pooling can reduce the overhead associated with opening and closing connections and with keeping many 
connections open simultaneously. This overhead not only includes memory needed to handle each new connection, also involves 
CPU overhead to close each connection and open a new one. 

        Connection pooling simplifies client application logic. Clients need to write application code to minimize the number of simultaneous open connections.
   
        We can also provide Valkey a feature similar to the connection multiplexing in RDS, also known as connection reuse. 
With multiplexing, the RDS can perform all the operations for a transaction using one underlying database connection, then it can use a different connection for the next transaction. Clients can open many simultaneous connections to the proxy of RDS, and the proxy keeps a smaller number of connections open to the DB instance or cluster. Doing so further minimizes the memory overhead for connections on the database server. This technique also reduces the chance of "too many connections" errors.

        For the client's persistent connection, the connection will not actively disconnect, but only returns the error message; If the client is disconnected actively, the connection will be disconnected immediately.

9.	Support CLIENT LIST etc commands aggregate return: return the client of all proxy nodes.

10.	Some commands such as set and zset are supported for aggregation processing across slot cluster connections.

11.     Support traffic control feature within the proxy.

12.     Support Redisson Distrbution Lock if customer works on cluster mode.


## References
https://github.com/RedisLabs/redis-cluster-proxy
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.howitworks.html
https://redisson.org/docs/data-and-services/locks-and-synchronizers/
