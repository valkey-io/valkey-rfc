---
RFC: (PR number)
Status: (Change to Proposed when it's ready for review)
---

# Title
Security Model for Search Indexes

## Abstract

The existing Valkey ACL-based security model must be extended to cover the Search module's concept of an index which spans multiple keys.

## Motivation

Users that desire to enforce data visibility and immutability need a security model that's easy to understand and configure.

## Design considerations

The key design considerations are:

1. Overall performance should be unaffected if ACLs are not being used.
2. The security model should be simple to understand and therefore be unlikely to cause accidental misconfigurations.
3. The security model should be simple to implement to reduce odds of a bug.

## Specification

ACLs have two separate portions. One portion is granting/inhibiting access to commands either individually as part of a category.
No changes to the mechanism are required and it will be extended to cover the new search module commands and the new @search category.

The second portion of the ACL machinery enables/disable access subsets of the keyspace for a user.
The search module creates indexes which contain data from a subset of the keyspace that is defined by one or more key prefixes (see the definition of FT.CREATE in the search RFC).
In order to enforce the the ACL keyspace restrictions, a user may only access or otherwise use a search index 
if that user has read access to 100% of the keys that _might_ be in the index. 
In other words, if the user's ACL prohibits access to any key that might be present in the index (any current or any potential future key), then access is blocked. Thus the search module cannot be used to access the contents of any key that the user is prohibited from directly reading herself.

The implementation of this model is very straightforward. 
Each search command that has an index (FT.CREATE, FT.SEARCH, FT.AGGREGATE, FT.DROPINDEX, FT.INFO), performs a security check on each keyspace (prefix) that is part of the index definition.
The security check is to ask the question whether the current user has access to 100% of the keys that could begin with the specific prefix.
If any of those checks fail then the command fails with a security error. 
This is done at the start of each and every search command. 
Thus, consistent with the Valkey ACL usage, if an ACL for a user is modified, then the next command issued by that user will be subjected to the new definition.

No security information is ever cached or retained within the search module itself.

### Authentication and Authorization

No changes to the existing ACL machinery are proposed. Rather as described above, the search module ensures that no search command can be used to obtain access to keys which are not directly readable by the current user. 

### Implementation

The search module will implement two methods of performing the required prefix access checks. 
There is no functional difference between the two methods the only difference is the performance of search commands.

The first method will be used when the Valkey core does not export the Module API proposed below. 
This mode enables the Search module ACL enforcement to be fully operable with the existing 7.x and 8.x Valkey releases.
This method implements the security check by performing a module CALL operation to fetch the ACL description of the current user and then to parse that result to determine the keyspace patterns enabled for the current user.

The second method is used when the proposed module API below is present. In which case, the security check is done within the core in a more efficient manner.

### Module API

The following module API is proposed to be implemented in the next Valkey release:

```
enum KeyPrefixCheckFlags {
    VALKEYMODULE_PREFIX_CHECK_READ_ACCESS = 1,
}

int ValkeyModule_AclCheckKeyPrefixPermissions(
    ValkeyModuleContext *ctx, 
    ValkeyModuleString **prefixes,
    size_t prefix_count,
    size_t flags)
```

The ```prefixes``` parameter points to an array of byte strings to check. The number of elements in the array is in the prefix_count parameters.
The ```flags``` indicates the type of access for the check. Currently only one flag is defined and it must be specified.

The return value indicates whether access is permitted (C_OK) or not (C_ERR) for the current user.
This API doesn't log anything or generate any security error message, that's the responsibility of the caller.

The implementation of this command is straightforward. Each prefix is matched against each key pattern in turn, if the pattern doesn't match the entire prefix (either with explicit characters or with globing) or if there are non-wildcard characters after the prefix then the match fails and C_ERR is returned.
Only if all of the patterns pass the test is C_OK returned.
