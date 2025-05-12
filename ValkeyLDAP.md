---
RFC: #20
Status: Proposed
---

# ValkeyLDAP Module RFC

## Abstract

The proposed Valkey LDAP module, named ValkeyLDAP, supports authentication to Valkey through external to Valkey managed authentication mechanisms based on the LDAP protocol. The module provides seamless integration between Valkey's authentication system and external enterprise authentication infrastructure, allowing organizations to leverage existing identity management systems. The design emphasizes security, scalability, and minimal performance impact while providing robust user authentication and role-based access control.

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

The `bind` mode can be used when the username mostly matches the DN of user entries in the LDAP directory, while the `search+bind` mode allows for a much more flexible LDAP directory structure.

### Bind Mode Authentication
In the `bind` mode, the module will bind to the distinguished name constructed by prepending a configurable prefix and appending a configurable suffix to the username. Typically, the prefix parameter is used to specify `cn=`, or `DOMAIN\` in an Active Directory environment. The suffix is used to specify the remaining part of the DN in a non-Active Directory environment.

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
* Security
  * TLS 1.2+ support
* Maintainability
  * Comprehensive error handling
* Compatibility
  * Compatibility with existing Valkey clients
* High Availabilty
  * Multiple identity provider server support
  * Server connection failover
* Configuration
  * LDAP server connection (address, port, ssl)
  * LDAP authentication method (bind, or search)
  * LDAP Bind DN
  * LDAP Bind PW
  * LDAP search Base DN
  * LDAP User filter pattern
  * TLS cert configs

Future improvements will include:

* Perfomance
  * Memory caching of authentication authorizations
  * Connection pooling to reduce latency
* Security
  * Secure credential handling and storage
  * Audit trail for authentication events
* Maintainability
  * Comprehensive logging and monitoring
* Configuration
  * Server connection timeout
  * Search operation timeout


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
3. Check credential cache
4. If not cached:
    - connect to LDAP server
    - perform bind operation
    - search for user group membership
    - cache successful result
    - set TTL
5. Pass AUTH/NOAUTH to Valkey core
6. Return auth response

The operations done in step 4 should be executed asynchronously in order to track the time that is taking for authenticating the user.

If the authentication process is taking longer than a specific threshold, the module should make use of the Blocking Client Module APIs, in particular using the `ValkeyModule_BlockClientOnAuth` function, to continue the authentication process in the background and allow the Valkey main event loop to continue processing other client commands.

The revised authentication flow steps in a long running authentication process would be:

1. Client sends AUTH/HELLO command
2. Module auth callback is called with username/password
3. Check credential cache
4. If not cached, run asynhcronously in a side thread:
    - connect to LDAP server
    - perform bind operation
    - search for user group membership
    - cache successful result
    - set TTL
    - If client is blocked, call `ValkeyModule_UnblockClient`
5. If step 4 taking a long time:
    - Call `ValkeyModule_BlockClientOnAuth` and signal authentication thread
5. Pass AUTH/NOAUTH to Valkey core
6. Return auth response

During the authentication flow, if a network error occurs, the Valkey authentication request will fail, and more detailed error description will be logged in the server log.

A user can also issue the `LDAP.STATUS` command to check if the module was able to establish a connection with the configured servers.

### On module load

Upon the module load operation we'll run the following procedures:

1. Load configuration from Valkey config or default values
2. Initialize data structures and connection pools
3. Establish initial LDAP server connections
4. Register command hooks with valkey core

### Commands

The configuration options of this module will be set using the standard `CONFIG SET` command.

The only commands exported by this module are:

* `LDAP.RELOAD`
  - Reloads configuration from config file
  - Reestablishes connections with LDAP servers
  - Returns simple string reply: OK on success

* `LDAP.STATUS`
  - Returns health and statistics information
  - Format: nested hash with connection states, cache stats, auth counts

* `LDAP.FLUSHCACHE`
  - Clears the credential cache
  - Options to flush all or specific users
  - Returns integer reply: number of entries flushed


### Configuration

The configuration options for this module will be registered using the `ValkeyModule_RegisterStringConfig` API function, which will allow the user to set and get the options using the `CONFIG SET` and `CONFIG GET` commands.

The list of configuration options is the following:

- General options
  - `ldap.auth_method`: the authentication method used. Possible values `bind`, or `search+bind`.
  - `ldap.servers`: list of LDAP server addresses (space-separated) where each server address has following format `ldap[s]://<hostname>[:<port>]`
    - The default port for `ldap` protocol is 389
    - The default port for `ldaps` protocol is 636
  - `ldap.timeout_connect`: connection timeout in ms (default: 1000)
  - `ldap.timeout_search`: search operation timeout in ms (default: 2000)

- TLS options
  - `ldap.use_starttls`: whether upgrade to a TLS encrypted connection upon connection to a non-ssl LDAP instance
  - `ldap.tls_cert_path`: path to client certificate
  - `ldap.tls_key_path`: path to client key
  - `ldap.tls_ca_cert_path`: path to CA certificate

- Bind mode options
  - `ldap.bind_dn_prefix`: service account DN for initial bind
  - `ldap.bind_dn_suffix`: service account DN for initial bind

- Search+Bind mode options
  - `ldap.search_bind_dn`: service account DN
  - `ldap.search_bind_passwd`: service account password
  - `ldap.search_base`: base DN for user searches
  - `ldap.search_filter`: filter for user searches
  - `ldap.search_attribute`: the entry attribute used in search for matching the username
  - `ldap.search_scope`: the LDAP search scope
  - `ldap.search_dn_attribute`: the attribute that contains the DN of the user entry

- Cache options
  - `ldap.cache_enabled`: enable credential caching (default: true)
  - `ldap.cache_ttl`: cache entry TTL in seconds (default: 300)
  - `ldap.cache_size`: maximum cache size (default: 10000)


This module will listen for changes of the options above using the `ValkeyModule_SubscribeToServerEvent` API function, and will act accordingly to reconfigure the connections, or other parts of the module, automatically.

### Metrics and Monitoring

#### Exported Metrics
- Authentication attempts (success/failure)
- Cache performance (hit ratio, size, evictions)
- LDAP connection status (latency, errors, timeouts)

#### Health Indicators
- LDAP server connectivity
- Error rates and patterns
- Response time percentiles


### Dependencies 

- [openLDAP library](https://git.openldap.org/openldap/openldap)

