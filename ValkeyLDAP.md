---
RFC: (PR number)
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

The work done in this Redis Pull Request #11659 means authentication can be handed of to a Valkey module without affecting the Valkey core authentication.

## Design considerations

### Performance Impact

- Minimize authentication overhead through efficient caching
- Optimize LDAP queries and connection management
- Implement connection pooling to reduce latency
- Balance security with performance requirements
- Ensure minimal impact on Valkey command execution

### Scalability

- Support for large user bases and group hierarchies
- Efficient handling of concurrent authentication requests
- Horizontal scalability in clustered environments
- Resource usage optimization for high-load scenarios
- Graceful degradation under heavy load

### Security

- Zero trust security model implementation
- Secure credential handling and storage
- Protection against common attack vectors
- Audit trail for authentication events
- TLS 1.2+ required for AD communication
- Certificate validation
- Secure credential storage
- Encrypted cache entries

### Maintainability

- Clear separation of concerns
- Modular design for easy updates and inclusion of new protocols
- Comprehensive logging and monitoring
- Simple configuration management
- Easy troubleshooting and debugging

### Compatibility

- Support for various Valkey deployment models
- Backward compatibility with existing Valkey features
- Integration with different LDAP providers
- Support for multiple authentication methods
- Compatibility with existing Valkey clients

### High Availabilty

- Multiple identity provider server support
- Connection failover
- Cache replication
- Graceful degradation

### Error Handling

- Invalid credentials
- Network timeouts
- LDAP server unavailable
- Cache corruption
- Configuration errors
- Group resolution failures

## Specification

### Authentication flow

1. Client sends AUTH command
2. Module extracts username/password
3. Check credential cache
4. If not cached:
    - connect to LDAP server
    - perform bind operation
    - search for user group membership
    - cache successful result
    - set TTL
5. Pass AUTH/NOAUTH to Valkey core
6. Return auth response

### On module load

- Load configuration from Valkey config or default values
- Initialize data structures and connection pools
- Register command hooks with valkey core
- Establish initial LDAP server connections
- Load cached credentials if persistence enabled
- Register monitoring callbacks

### Commands 

1 LDAP.CONFIG SET <parameter> <value>

- Sets configuration parameters for the LDAP authentication module
- Parameters include server addresses, timeouts, cache settings, etc.
- Returns simple string reply: OK on success

2 LDAP.CONFIG GET <parameter>

- Gets current configuration for specified parameter
- Returns bulk string reply: parameter value

3 LDAP.RELOAD

- Reloads configuration from config file
- Reestablishes connections with LDAP servers
- Returns simple string reply: OK on success

4 LDAP.STATUS

- Returns health and statistics information
- Format: nested hash with connection states, cache stats, auth counts

5 LDAP.FLUSHCACHE

- Clears the credential cache
- Options to flush all or specific users
- Returns integer reply: number of entries flushed

### Valkey Command Hooks

#### Authentication Command Hook

- Intercepts AUTH command
- Processes credentials through LDAP binding
- Maintains original Valkey auth functionality for local accounts

#### ACL Command Hook

- Integrates with ACL commands
- Maps LDAP groups to Valkey ACLs
- Preserves ACL inheritance and evaluation logic

### Internal data structures

#### Configuration Store

- Hash table for configuration parameters
- Persistent across restarts via RDB or config file
- Thread-safe access with read-write locks

#### Credential Cache

- LRU cache with configurable TTL
- Encrypted storage with secure key management
- Size-limited with automatic eviction
- Concurrent access optimized

#### Group Mapping Table

- Hash table mapping LDAP groups to Valkey ACLs
- Configurable refresh interval
- Support for nested group resolution

#### Connection Pool

- LDAP connection objects with state tracking
- Priority queue based on connection health
- Automatic connection cycling
- Size limits and idle timeouts

### Configuration

#### Server Configuration

- LDAP.servers: list of LDAP server addresses (comma-separated)
- LDAP.port: LDAP port (default: 636 for LDAPS)
- LDAP.use_ssl: whether to use LDAPS (default: true)
- LDAP.timeout.connect: connection timeout in ms (default: 1000)
- LDAP.timeout.search: search operation timeout in ms (default: 2000)
- LDAP.retry.max: maximum retry attempts (default: 3)
- LDAP.retry.delay: delay between retries in ms (default: 500)

#### Authentication Configuration

- LDAP.basedn: base DN for user searches
- LDAP.binddn: service account DN for initial bind
- LDAP.bindpw: service account password (stored securely)
- LDAP.user_filter: LDAP filter for user searches
- LDAP.group_filter: LDAP filter for group membership queries
- LDAP.default_permissions: default ACL for authenticated users

#### Cache Configuration

- LDAP.cache.enabled: enable credential caching (default: true)
- LDAP.cache.ttl: cache entry TTL in seconds (default: 300)
- LDAP.cache.size: maximum cache size (default: 10000)
- LDAP.cache.persist: persist cache across restarts (default: false)
- LDAP.cache.encryption: encryption method for cached credentials

#### Security Configuration

- LDAP.tls.cert: path to client certificate
- LDAP.tls.key: path to client key
- LDAP.tls.ca: path to CA certificate
- LDAP.tls.verify: verify server certificates (default: true)
- LDAP.tls.ciphers: allowed cipher suites
- LDAP.tls.version: minimum TLS version (default: TLS1.2)

### Metrics and Monitoring

#### Exported Metrics
- Authentication attempts (success/failure)
- Cache performance (hit ratio, size, evictions)
- LDAP connection status (latency, errors, timeouts)
- Group resolution performance
- Resource utilization (memory, CPU)

#### Health Indicators
- LDAP server connectivity
- Cache integrity
- Configuration consistency
- Error rates and patterns
- Response time percentiles

### Error Handling and Logging

#### Error Categories
- Configuration errors (invalid parameters, inconsistent settings)
- Connection errors (network failures, timeouts, rejected connections)
- Authentication errors (invalid credentials, expired accounts)
- Authorization errors (insufficient permissions, group mapping failures)
- Internal errors (memory allocation, thread management)

#### Logging Levels

- ERROR: critical issues requiring immediate attention
- WARN: potential issues that don't prevent operation
- INFO: normal operational events
- DEBUG: detailed debugging information
- TRACE: full protocol-level logging for troubleshooting

### Cluster mode

TBD :
- this should follow the existing authentication rules for Valkey Cluster
- configuration should be sync-ed across all nodes
- authentication cache should be consistent across nodes 

### Replication

TDB :
- this should follow the existing authentication rules for Valkey primaries and replicas
- authentication cache should be consistent across primary and replica

### Dependencies 

- [openLDAP library](https://git.openldap.org/openldap/openldap)

