---
RFC: 21
Status: Proposed
---

# ValkeyAUDIT Module RFC

## Abstract

The proposed Valkey Audit module, named ValkeyAudit supports the auditing of Valkey connections and commands. The module provides integration with standard external enterprise audit collection software and endpoints, e.g. syslog and splunk. The design allows for a highly custimisable deployment through configuration options of the types of events and commands to audit. 

Arguably, the work would be better placed in Valkey core but a separate Audit module allows for 
- a faster and more isolated implementation 
- less impact on core performance
- the number of Valkey users that would require auditing does not warrant a core implementation

## Motivation

Organizations running Valkey in enterprise environments frequently need to provide information about which users and clients connect, attempt to connect and disconnect to Valkey as well as information on which users ran which command. This module addresses these challenges by providing the options to log the following and provide this information in standard formats to external systems:

- data manipulation operations
- config operations
- connections and disconnections
- authentication requests

## Design considerations

There are three concerns that need to be considered in the implementation of this module: Configurability, Performance, and Security.

### Configurability

Regarding the configurability concern, there are two configuration groups: the audit log output configuration, and the audit events configuration.

The module should support several protocols to store the audit logs, and the choice of which protocol to use should be dynamically configurable by the module.

A filesystem audit log file protocol, and the [syslog](https://www.rfc-editor.org/rfc/rfc5424.html) protocol, will be the first protocols supported by the audit module.

The format of the audit events should also be configurable by the user.

Regarding the audit events configuration, the module should allow to toggle the following event categories:
- connections and disconnections
- auth requests with redaction of password
- config commands
- key operations

For key operations, the following options should be configurable:
- disable logging of operation payload
- configurable size of payload


### Performance

Regarding the perfomance concern, it's important that the flush of the audit events log does not happen in the critical path of the command being audited.

The logging of audit events must be done as fast as possible, and therefore the event log must be kept in memory, and be asynchronously flushed using one of the configured output protocols.

The memory buffer used to store the audit events must be limited by a configurable parameter. When the memory buffer is near full due to the flush not being able to keep up, the logging of events must be throttled.


### Security 

Regarding the security concern, there are a few aspects that will be taken into consideration:
- removal of potentially sensitive payloads
- removal of sensistive authentication information
- encryption of audit log data in transit (only applies to network protocols)

The removal of sensitive information should be configurable per command, but should be easy to apply the same configuration to all commands, or groups of commands.

## Specification

Valkey modules can subscribe to Valkey server events. Client events like connections and disconnections can be handled through the `ValkeyModuleEvent_ClientChange` server event. 

Valkey server command execution can be plugged into by registering command filters. The filter applies in all execution paths including:

1. Invocation by a client.
2. Invocation through [`ValkeyModule_Call()`](https://valkey.io/topics/modules-api-ref/#ValkeyModule_Call) by any module.
3. Invocation through Lua `server.call()`.
4. Replication of a command from a primary.

When the installed hooks are invoked an event is generated and stored in a circular memory buffer.

Each event has a well defined structure, There are several event types to be logged:
- Connect
- Disconnect
- Authentication
- Command

For each event there is a set of fields that are commmon to all types:
- Timestamp
- Source IP/port
- Target IP/port
- Connection ID
- Event type

Then there is a set of fields specific to each event type:
- *Connect Type*
  - Connection status (success/failure)
  - Failure reason (if applicable)
- *Disconnect Type*
  - Disconnect status (graceful/error)
  - Disconnect reason (if available)
- *Authentication Type*
  - Username
  - ACL rules used for verification (TODO: why do we need this)
  - Authentication status (success/failure)
  - Failure reason (if applicable)
- *Command Type*
  - Database ID
  - Command name
  - Command key
  - Command args (might be empty if payload capture is disabled, and payload length limit is configurable)
  - Command status (success/failure)

To flush the events from the memory buffer to the configured destinations, a background thread is spawned for each destination.

The background thread is responsible for consuming the events from the memory buffer and send them to the respective destination, either using the syslog protocol, or writing to a filesystem file.

At any point in time, the event logging procedure must know what events have been already flushed by all background threads in order to add new events in the circular buffer.


### Module Configuration

The configuration options for this module will be registered using the `ValkeyModule_RegisterStringConfig` API function, which will allow the user to set and get the options using the `CONFIG SET` and `CONFIG GET` commands.

The list of configuration options is the following:

#### General options
- `audit.enabled`: whether the logging of audit events is enabled.
- `audit.format`: the format of audit log messages
- `audit.payload_disable`: whether or not to log the payload of key commands (default: `false`)
- `audit.payload_maxsize`: the length in characters of the payload in the case that payload logging is enabled
- `audit.always_audit_config`: enable or disable the auditing of config commands regardless of per user events setting. This allows the logging of config commands for a user even if the user is in the exclusion list.

#### Protocol and options
- `audit.protocol`: file, syslog or tcp
  E.g.
  - `audit.protocol file /tmp/audit.log`
  - `audit.protocol syslog local0`
  - `audit.protocol tcp 127.0.0.1:532`
- `audit.tcp_host` : the target host for TCP destination
- `audit.tcp_port` : the target port for TCP destination
- `audit.tcp_buffer_on_disconnect` : whether or not to keep the messages in the buffer if TCP disconnected (limited to buffer size)
- `audit.tcp_timeout_ms` : timeout in ms for establishing TCP connection
- `audit.tcp_max_retries` : maximum number of times to retry sending
- `audit.tcp_retry_interval_ms`: retry interval for establishing TCP connections

#### Exclusion
- `audit.events` : event categories to audit (connections,auth,config,keys,other,none,all)
- `audit.excluderules` : Specific usernames and/or IP addresses to exclude from auditing

### Monitoring 

#### `INFO AUDIT`

Returns the information about the module configuration, the status of the memory circular buffer, and some statistics of the background threads, in the form of a dictionary value.

Example:

info audit

\# audit_audit

`audit_total_events:3`

`audit_total_errors:0`

`audit_exclusion_hits:0`

`audit_uptime_seconds:10`

`audit_file_status:connected`

`audit_syslog_status:disconnected`

`audit_tcp_status:disconnected`

`audit_error_rate_percent:0`
