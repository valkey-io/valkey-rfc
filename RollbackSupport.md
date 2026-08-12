---
RFC: 32
Status: Proposed
---

# Rollback Support Using Data Versioning for Valkey

## Abstract

This RFC proposes implementing rollback support for Valkey through a dual-value data versioning system. The system enables atomic rollback capabilities for multi-step operations by maintaining both committed and uncommitted states for data modifications. This provides a foundation for robust error handling in complex scenarios such as Lua script execution, write-ahead logging, multi-command transactions, and other operations that require all-or-nothing semantics with the ability to safely undo changes when failures occur.

## Motivation

Current Valkey operations are immediately visible and permanent once executed, but many advanced features require the ability to tentatively modify data and later commit or rollback those changes atomically. This creates several challenges across different use cases:

### Primary Use Cases

1. **Lua Script Rollback**: When a Lua script encounters an error or is explicitly terminated, all data modifications within the script should be rolled back to maintain data integrity
2. **Write-Ahead Log Consistency**: Ensuring data modifications are only visible after they are durably committed to the WAL, with automatic rollback on WAL failures
3. **Multi-Command Transactions**: Supporting atomic execution of command sequences beyond the current MULTI/EXEC limitations
4. **Conditional Operations**: Enabling complex conditional logic where modifications are applied only if all conditions are met
5. **Administrative Operations**: Allowing safe execution of potentially destructive operations with rollback capability

### Current Limitations

Without data versioning, these scenarios suffer from:
- **Partial State Corruption**: Failed operations leave the database in an inconsistent intermediate state
- **Manual Cleanup Complexity**: Applications must implement complex compensation logic to undo partial changes
- **Race Conditions**: Concurrent access during multi-step operations can observe inconsistent states
- **Limited Error Recovery**: No built-in mechanism to automatically restore previous consistent state
- **Complex Failure Handling**: Each operation type requires custom rollback logic

### Solution Benefits

Data versioning with rollback support addresses these issues by:
- **Atomic Rollback**: Instantly restore data to its previous consistent state on any failure
- **Isolation During Execution**: Hide tentative changes from other operations until commit
- **Simplified Error Handling**: Uniform rollback mechanism across all operation types
- **Improved Reliability**: Guarantee that operations either complete entirely or have no effect

### Cross-Cutting Applications

This infrastructure enables numerous advanced features:
- **Safe Lua Script Execution**: Scripts can modify data freely, knowing any error will trigger complete rollback
- **Transactional APIs**: Building higher-level transactional interfaces on top of Valkey
- **Administrative Tools**: Safe bulk operations with preview and rollback capabilities
- **Replication Safety**: Ensuring replicas never observe partially committed operations
- **Backup Consistency**: Creating consistent snapshots even during ongoing operations

## Design considerations

### Requirements

1. **Backward Compatibility**: Existing clients and commands must work unchanged when versioning is disabled
2. **Universal Applicability**: Versioning system must support all Valkey data types and operation patterns
3. **Performance**: Minimal overhead for operations not using versioning features
4. **Memory Efficiency**: Dual-value storage must be lightweight and space-efficient
5. **Atomic Operations**: Commit and rollback operations must be atomic across all modified keys
6. **Crash Recovery**: System state must be recoverable after unexpected shutdowns

### Constraints

1. **Memory Overhead**: Each key with uncommitted changes requires additional storage
2. **Coordination Complexity**: Atomic commits across multiple data types require careful orchestration
3. **Rollback Scope**: Clear boundaries for what operations participate in rollback
4. **Error Propagation**: Failed rollbacks must be handled gracefully without data corruption

Valkey's dual-value versioning design adapts these concepts for key-value operations while maintaining simplicity and broad applicability across different use cases.

## Specification

### Core Concepts

#### Dual-Value Storage
Each key maintains two values when there are uncommitted changes:
- **Committed Value**: The last successfully committed value, visible to all readers
- **Uncommitted Value**: The new value that has been written but not yet committed

#### State Transitions
Keys can be in one of two states:
- **COMMITTED**: Only committed value exists, reads return this value
- **UNCOMMITTED**: Both committed and uncommitted values exist, reads return uncommitted value for the same client session that created it, committed value for all other clients 

#### Rollback Support
If an operation fails or needs to be rolled back:
- Uncommitted value is discarded
- Committed value remains unchanged and continues to be served
- Key returns to COMMITTED state

#### Commit Support
When an operation finishes successfully:
- Uncommitted values become the new committed values
- Old committed values are discarded
- Keys return to the COMMITTED state

#### Data Structure
Each key maintains both committed and uncommitted values along with state information:
- **Committed Value**: The current visible value
- **Uncommitted Value**: The tentative new value (null if no uncommitted changes)
- **State**: Either COMMITTED (only committed value) or UNCOMMITTED (both values exist)

#### Read-Your-Writes Semantics
The system provides read-your-writes consistency within a client session:
- **Same Session**: A client that writes an uncommitted value will read that uncommitted value on subsequent operations
- **Other Sessions**: All other clients will continue to read the committed value until the uncommitted value is committed
- **Session Tracking**: The system must track which client session created each uncommitted value
- **Isolation**: This provides write isolation between sessions while allowing writers to see their own changes

### Implementation Approaches

The dual-value storage architecture requires careful consideration of implementation strategies, as each Valkey data type will need a custom implementation. There are two primary approaches, each with distinct trade-offs:

#### Approach 1: External Uncommitted Table (Centralized)

In this approach, uncommitted values are stored in a separate, centralized data structure indexed by a unique identifier for each data item.

**Conceptual Structure:**
- Main storage contains committed values indexed by key
- External uncommitted table contains uncommitted values indexed by unique data item identifiers
- Data item identifiers uniquely identify each piece of data across all types

**Advantages:**
- **Memory Efficiency**: No memory overhead for keys without uncommitted changes
- **Type Independence**: Same uncommitted storage mechanism across all data types

**Disadvantages:**
- **Access Overhead**: Two lookups required for keys with uncommitted changes
- **ID Management**: Need unique identification scheme across data types
- **Memory Fragmentation**: Uncommitted table may be sparsely populated

#### Approach 2: Co-located Storage (Embedded)

In this approach, uncommitted values are stored directly alongside committed values within each data structure.

**Conceptual Structure:**
- Simple types store committed and uncommitted values together with state information
- Complex types store dual values at the appropriate granularity (e.g., per hash field)
- Collections types store dual values in each element entry

**Advantages:**
- **Access Efficiency**: Single lookup provides both committed and uncommitted values
- **Type-Specific Optimization**: Each data type can optimize storage layout
- **Locality**: Related data stored together
- **Simplified Management**: No external ID management needed

**Disadvantages:**
- **Memory Overhead**: Every data item carries uncommitted storage overhead
- **Complex Implementation**: Each data type needs custom dual-value logic
- **Size Inflation**: Data structures grow even when no uncommitted changes exist


#### Performance Considerations

**Memory Usage Patterns:**
- External table overhead grows with the number of uncommitted operations
- Co-located overhead grows with the total number of keys
- Crossover point depends on the ratio of uncommitted operations to total keys

**Access Time Characteristics:**
- External table requires additional lookup for uncommitted values
- Co-located storage provides single-lookup access for both committed and uncommitted values

**Cache Efficiency:**
- External table offers better cache locality for commit operations but worse for individual key access
- Co-located storage provides better cache locality for individual operations but may cause cache pollution

### API Changes

#### Transaction Management

To support commit or rollback of a group of values, we introduce the concept of a **transaction context**. This context tracks all uncommitted changes made during command execution and provides methods to commit or rollback those changes atomically.

**Transaction Lifecycle:**
- **Creation**: A transaction context is created when executing commands that require versioning
- **Tracking**: All data type modifications are associated with the transaction context
- **Resolution**: The transaction is either committed (making changes permanent) or rolled back (discarding changes)

**Implementation Impact:**
Command execution functions will need to:
1. Create a transaction context at the start of command execution
2. Pass the transaction context to all data type modification functions
3. Call commit or rollback on the transaction context based on command success/failure

#### Data Type Interface Extensions

Data types need a new method to declare versioning support. This method should return a boolean value indicating whether the data type implements native dual-value versioning capabilities.

**For Versioning-Enabled Data Types:**
- Handle committed/uncommitted values internally with optimal memory usage
- Implement type-specific dual-value storage strategies
- Provide efficient commit/rollback operations

**For Non-Versioning Data Types:**
- System automatically uses copy-on-write (COW) fallback
- Deep copy created when transitioning to uncommitted state: original value becomes committed version, modified value becomes uncommitted version
- Ensures backward compatibility with all existing modules and external data types
- Higher memory overhead but no code changes required

**Session Tracking:**
All uncommitted values must be associated with the client session that created them to support read-your-writes semantics. The transaction context serves as the bridge for this association by linking uncommitted values to their originating client session.

