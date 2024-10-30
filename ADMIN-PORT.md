---
RFC: 3
Status: Proposed
---

# ADMIN PORT RFC

## Abstract

In Valkey.conf, Add a new config parameter: "admin-port" to allow cloud providers or internal administrators in a company or special users from public internet to connect to the Valkey server to execute some special commands or load specfic modules.


## Motivation

For adminstrators, they may want to execute some special commands in the server, which are usually defined in the module, and
other general clients are not allowed to run these kinds of commands. Thus connecting to an admin-port is a good way to implement this feature. 

Beside above mentioned, we could implement the following features in the future once admin-port is commited:
1. The connections through admin port are not limited to maxclient number, we can always guarantee the administrators can connect to the server.
2. Some status check commands could be sent and receied through admin port, we can do some special logic to make sure they have opportunity to be handle first, such as: PING, HELLO, ECHO, INFO etc


## Design Considerations

Before we propose the "admin-port" feature, we have the following 3 candidates:
1. ACL: Now Valkey have ACL feature to implement similar function, but it brings one disadvantages: username need to be predefined.

2. Run some special commands through the connected IP address: Before we run every commands, we need check if the client ip address is in a linkedlist, it costs CPU and extra memory, it causes side effect in performance

3. Iptables: It works on transport layer and IP address, but it can not apply to the specific commands of the Valkey

Based on all above 3 candidate defects, we decide to select "admin-port" to implement our goal.


## Specification

1. The admin client role is only decided by the cloud providers or internal administrators, whatever where the client comes from. Thus generally, admin-port should not be exposed to the public.
2. If a client connects via admin-port and ACL rules apply to this client as well, this client should follow the ACL to execute some commands
3. If a client connects via non-admin-port and ACL rules apply to this client as well, this client is not allowed to execute those special commands. The special commands could be new commands introduced by admin through modules, for example, setting/getting traffic/replication control threshold, encrypting the username and password in config file, etc

## References
Add a management-port https://github.com/valkey-io/valkey/issues/497
Trusted/un-trusted client feature https://github.com/valkey-io/valkey/issues/469, https://github.com/valkey-io/valkey/pull/666
iptables — a comprehensive guide https://sudamtm.medium.com/iptables-a-comprehensive-guide-276b8604eff1
An In-Depth Guide to iptables, the Linux Firewall https://www.booleanworld.com/depth-guide-iptables-linux-firewall/

