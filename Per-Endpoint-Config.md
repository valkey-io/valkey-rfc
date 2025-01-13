RFC: 16
Status: Proposed

# PER ENDPOINT CONFIG RFC

## Abstract

I would like to introduce a new configuration parameter configure-endpoint to accommodate the combination for all existing configurable ports in Valkey.conf, including port, tls-port, cluster-port, rdma-port, maxclient config parameter, and other parameters. By enabling this configuration parameter, the cloud providers and internal administrators in a company could customize how to config each endpoint, such as the endpoint type, the maximum number of the connection, etc.

## Motivation

Currently, there are following disadvantages when users config the server endpoint

1.Now there are four types of port: port, tls-port, cluster-port, and rdma-port. But users can specify only one port per type. They have no way to create more than one tcp port in a server to accept the connections. 

2.Users can only specify the maximum number of connections through maxclient parameter for the server. They can not apply the maxclient parameters to one specific port or endpoint. 

3.The properties related to endpoint are independent.

4.If the connection number of clients reaches the maxclient and there is any emergence issue in the server side, administrators can not find a way to connect the server.

## Design Considerations

The new configuration parameter configure-endpoint can take multiply key-value pairs as the properties for one endpoint. Here are some examples:

configure-endpoint name=user-tcp type=plaintext maxclients-number=8000 cluster-broadcast=no
configure-endpoint name=user-tcp type=plaintext maxclients-number=20000 cluster-broadcast=no
configure-endpoint name=user-tls type=tls maxclients-number=10000 cluster-broadcast=no
configure-endpoint name=admin type=plaintext maxclients-number=10 cluster-broadcast=no
configure-endpoint name=cluster-bus type=plaintext maxclients-number=100 cluster-broadcast=yes

Through any example above, users or administrators can easily find the configuration parameters for any port and its type. 

## Reference:
https://github.com/valkey-io/valkey/pull/1120#issuecomment-25815490473

