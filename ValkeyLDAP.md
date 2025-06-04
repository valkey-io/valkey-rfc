---
RFC: 20
Status: Proposed
---

# ValkeyLDAP Module RFC

## Abstract

The proposed Valkey LDAP module, named ValkeyLDAP, supports authentication to Valkey through external to Valkey managed authentication mechanisms based on the LDAP protocol. The module provides seamless integration between Valkey's authentication system and external enterprise authentication infrastructure, allowing organizations to leverage existing identity management systems. The design emphasizes security, scalability, and minimal performance impact while providing robust user authentication.

## Motivation

Organizations running Valkey in enterprise environments frequently need to integrate with existing identity management systems that support the LDAP protocol so that this can be centrally managed. This module addresses these challenges by:

- eliminating the need for external authentication proxies
- reducing operational complexity
- providing native Active Directory integration
- enabling centralized user and permission management
- maintaining Valkey's performance characteristics
- maintaining Valkey's existing authenthication workflow
- supporting compliance requirements for centralized authentication

The Valkey Module API already supports the offloading of user authentication to an external module, we will make use of this API to implement the LDAP authentication module.

## Design considerations

This module will support the two traditional approaches to authenticate users against an LDAP server:
* The *bind* approach.
* The *search+bind* approach.

The `bind` mode can be used when the username is part of the distinguish name (DN) of user entries in the LDAP directory, while the `search+bind` mode allows for a much more flexible LDAP directory structure.

### Bind Mode Authentication
In the `bind` mode, the module will bind to the DN constructed by prepending a configurable prefix and appending a configurable suffix to the username. Typically, the prefix parameter is used to specify `CN=`, or `DOMAIN\` in an Active Directory environment. The suffix is used to specify the remaining part of the DN in a non-Active Directory environment.

### Search+Bind Authentication
In the `search+bind mode`, the module first binds to the LDAP directory with a username and password of an account that has permissions to perform search operation in the LDAP directory. If no username and password is configured for the binding phase, an anonymous bind will be attempted to the directory.

After the binding phase, a search operation is performed over the subtree at a configurable base DN string, and will try to do an exact match of the username specified in the `AUTH` command against the value of a configurable entry attribute.

Once the user has been found in this search, the module re-binds to the LDAP directory as this user, using the password specified in the `AUTH` command, to verify that the login is correct.

This mode allows for significantly more flexibility in where the user objects are located in the directory, but will cause two additional requests to the LDAP server to be made.

### Minimum Viable Product

In a first stage of the module implementation, the module will support a minimal set of features and requirements to successfully handle LDAP authentication queries in a production evironment, namely:

* Scalability
  * Efficient handling of concurrent authentication requests
  * Asynchronous communication with LDAP servers
  * Connection pooling to reduce latency
* Security
  * TLS 1.2+ support
  * Secure credential handling and storage
* Maintainability
  * Comprehensive error handling
  * Comprehensive logging and monitoring
* Compatibility
  * Compatibility with existing Valkey clients
* High Availabilty
  * Multiple LDAPS servers support
  * Server connection failover
* Configuration
  * LDAP server connection (address, port, ssl)
  * LDAP authentication method (bind, or search)
  * LDAP Bind DN
  * LDAP Bind PW
  * LDAP search Base DN
  * LDAP User filter pattern
  * TLS cert configs

### LDAP and Valkey User Mapping

The way the Valkey module API supports external authentication mechanisms is by offloading the authorization part to the module but it requires the user to have been created in Valkey previously. A user account in Valkey is often referred as ACL user.

When the module performs the LDAP authentication process and gets the successful authorization it must call the `ValkeyModule_AuthenticateClientWithACLUser` API function, which associates the existing ACL user with the current client connection.

Therefore, there must exist a mapping between the LDAP user and the ACL user.

In a first stage of the module implementation we will just support the simplest mapping method that only requires an existing ACL user to exist with the same username as the LDAP user being authenticated.

System administrators must ensure that the LDAP users also exist in Valkey with the proper permissions configured.

In a later stage we can develop different kinds of mapping mechanisms, where, for instance, the mapping can be specified in a file.

Another possibility for the future is to create a module command that helps system administrators to create ACL users based on the list of users present in the LDAP server.

## Specification

### Authentication flow

1. Client sends AUTH/HELLO command
2. Module auth callback is called with username/password
3. Start an asynchronous authentication task and let Valkey core continue serving other clients
4. Get an LDAP connection from a connection pool
5. Perform the bind or search+bind operation
6. Notify Valkey core with the authentication result
7. Valkey processes authentication result and responds to the AUTH/HELLO command

Steps 4 to 6 are executed by an asynchronous task, which allows the module to process several authentication requests in parallel.

### On module load

Upon the module load operation we'll run the following procedures:

1. Load configuration from Valkey config or default values
2. Initialize data structures and connection pools
3. Establish initial LDAP server connections
4. Register command hooks with valkey core

### Commands

The configuration options of this module will be set using the standard `CONFIG SET` command.

The only command exported by this module is:

* `LDAP.STATUS`: returns health and statistics information for each LDAP server instance.

The format of the response of this command is a map where each key corresponds to the hostname of an LDAP server, and the value is a map with the keys `status`, `ping_time`, `error`.

```json
{
  "server1_hostname": {
    "status": "healthy",
    "ping_time(ms)": "1.23"
  },

  "server2_hostname": {
    "status": "unhealthy",
    "error": "an error message"
  }
}
```


### Configuration

The configuration options for this module will be registered using the `ValkeyModule_RegisterStringConfig` API function, which will allow the user to set and get the options using the `CONFIG SET` and `CONFIG GET` commands.

The list of configuration options is the following:

- General options
  - `ldap.auth_enabled`: whether the module should try to authenticate the user using LDAP (default: `yes`)
  - `ldap.auth_method`: the authentication method used. Possible values `bind`, or `search+bind` (default: `bind`)
  - `ldap.servers`: list of LDAP server addresses (space-separated) where each server address has following format `ldap[s]://<hostname>[:<port>]`
    - The default port for `ldap` protocol is 389
    - The default port for `ldaps` protocol is 636
  - `ldap.timeout_connection`: connection timeout in seconds (default: 10)
  - `ldap.timeout_ldap_operation`: LDAP operation (i.e., bind, search, etc...) operation timeout in seconds (default: 10)

- TLS options
  - `ldap.use_starttls`: whether upgrade to a TLS encrypted connection upon connection to a non-ssl LDAP instance (default: `no`)
  - `ldap.tls_cert_path`: path to client certificate
  - `ldap.tls_key_path`: path to client key
  - `ldap.tls_ca_cert_path`: path to CA certificate

- Bind mode options
  - `ldap.bind_dn_prefix`: service account DN for initial bind (default: `"CN="`)
  - `ldap.bind_dn_suffix`: service account DN for initial bind.

- Search+Bind mode options
  - `ldap.search_bind_dn`: service account DN
  - `ldap.search_bind_passwd`: service account password
  - `ldap.search_base`: base DN for user searches
  - `ldap.search_filter`: filter for user searches (default: `"objectClass=*"`)
  - `ldap.search_attribute`: the entry attribute used in search for matching the username (default: `"uid"`)
  - `ldap.search_scope`: the LDAP search scope. Possible values `base`, `one`, `sub` (default: `sub`)
  - `ldap.search_dn_attribute`: the attribute that contains the DN of the user entry (default: `"entryDN"`)


### Scalability of authentication requests

Scalability will be achieve using two stratgegies:

* LDAP connection pooling and muiltiplexing
* Parallel authentication processes with asynchronous I/O

#### LDAP connection pool

The module should be able to limit the number of LDAP connections it uses and should use a connection pool to avoid the overhead of opening a new connection for each authentication request.

Since the module supports configuring multiple LDAP servers for redundancy, it should manage a connection pool for each LDAP server.

The number of connections present in the pool should be configurable, and be modifiable at runtime.

The LDAP protocol allows to use a single connection to perfom multiple `bind` operations, this means that we can safely re-use the connections from the pool to process multiple authentication requests from different users.

#### Parallelism of authentication processes

As described in section [Authentication flow](#authentication-flow), the LDAP authentication process should be done in the background to avoid blocking other commands, and other authentication requests, since the authentication operation duration might depend on network latencies.

The authentication requests should be processed by a thread from a thread pool, and make use of asynchronous I/O when running LDAP operations to allow even more parallelism without requiring too many threads.

The thread pool allows to limit the memory and CPU resources used by the module to avoid impacting the Valkey core performance.
