---
RFC: 2
Status: Accepted
---

# ValkeyJSON RFC

## Abstract

The proposed Valkey JSON module, named ValkeyJSON, supports the native JavaScript Object Notation (JSON) format to encode 
complex datasets inside Valkey. It should be compliant with [RFC7159](http://www.ietf.org/rfc/rfc7159.txt) and [ECMA-404](https://ecma-international.org/publications-and-standards/standards/ecma-404) 
JSON data interchange standard. With this feature, users can natively store, query, and modify JSON data structures in 
Valkey using the popular [JSONPath query language](https://www.ietf.org/archive/id/draft-goessner-dispatch-jsonpath-00.html). 
To help users migrate from Redis and RedisJSON, as well as capitalize on existing OSS RedisJSON client libraries, the module 
is designed to be API-compatible and RDB-compatible with Redis Ltd.’s RedisJSON v2.

## Motivation

JSON format is a widely used data exchange format and simplifies the development of applications that store complex data 
structures by providing powerful searching and filtering capabilities. However, [Valkey core](https://github.com/valkey-io/valkey) 
does not have a native data type for JSON. Redis Ltd.‘s RedisJSON is a popular Redis module, but not under a true 
open source license and hence cannot be distributed freely with Valkey. There's a demand in the Valkey 
community to have a JSON module that matches most of the features of RedisJSON and is as API-compatible as possible. 
See the [community discussions](https://github.com/orgs/valkey-io/discussions?discussions_q=is%3Aopen+JSON).

## Design Considerations

ValkeyJSON will introduce a new JSON data type for Valkey, and commands to insert, update, delete and query JSON data. 
To help users migrate from Redis and RedisJSON, as well as capitalize on existing OSS RedisJSON client libraries, ValkeyJSON 
aims to be a drop-in replacement of RedisJSON. Therefore, it is designed to be API-compatible and RDB-compatible with 
Redis Ltd.’s RedisJSON.

### RDB Compatibility

ValkeyJSON is RDB compatible with Redis Ltd.’s RedisJSON v1.0.8 or later.

### Choice of JSON Library

We have evaluated 7 open source JSON libraries - BSON, ION, MessagePack, yyjson, cJSON, Serde JSON, and RapidJSON. BSON, 
ION and MessagePack do not have write (insert/update/delete) API and therefore cannot be used for the project. Among the 
rest, RapidJSON stands out as both memory efficient and providing efficient insert/update/delete API.

### JSONPath Query Parser

[JSONPath](https://www.ietf.org/archive/id/draft-goessner-dispatch-jsonpath-00.html) is a query language for JSON with 
features similar to XPath for XML. JSONPath is used for selecting and extracting elements from a JSON document. It 
supports advanced query capabilities such as wildcard, filter expressions, slices, union, recursive search, etc.
Using JSONPath query language, users can natively store, query, and modify JSON data structures, either wholly or partially.

One of the main deficiencies of the RapidJSON library is that it does not support JSONPath query. RapidJSON only supports 
[JSON Pointer](https://datatracker.ietf.org/doc/html/rfc6901) query language, which is restricted to single-value selection. 
Single-value selection corresponds to the restricted query capability in RedisJSON v1 API. RedisJSON v2 API supports a 
much more powerful JSONPath syntax with wildcard, filter expressions, slices, union, recursive search, etc., and can select 
multiple JSON values. RedisJSON v2 API is a superset of v1 API.

We should extend RapidJSON to support a subset of the JSONPath query language that is compatible with RedisJSON v2 API. 
To achieve this, we should implement a JSONPath query parser that integrates with RapidJSON. JSONPath query expressions
should apply to all CRUD operations.

The query expression should be designed to work with selecting a vector of values instead of a single value, and 
compatible with both RedisJSON v1 and v2 query syntax. The query parser should automatically detect if the query is of 
v1 or v2 syntax.

### Tokenization of JSON Object Keys

JSON has two container types - Array and Object. The JSON Object is a key/value mapping where keys are restricted to 
be strings, whereas the value can be any valid JSON value, including another container value. Thus, a single JSON document 
can contain multiple JSON objects, each with its own unique namespace. It's quite common that a key repeatedly appears 
in the same JSON document or across documents. If the number of JSON documents is large, repeated key names can consume 
a significant amount of memory usage.

To remove duplicate copies of JSON object key names, the module tokenizes key names by maintaining a global data structure 
called KeyTable, which is a sharded hash table storing key tokens and reference counts. Access to the KeyTable is threadsafe.

### Document Size Limit

We added a limit on JSON key size as a good design practice. Without the limit, a malicious program could repeatedly call 
"json.arrinsert" or similar commands to make a JSON key grow indefinitely, potentially leading to an out-of-memory (OOM) condition.

The ValkeyJSON module will provide a configuration option, json.max-document-size, that allows users to set a size limit for
JSON key size.

However, the default value of json.max-document-size will be set to 0, meaning the size of JSON key will be unlimited. 
This decision is based on the observation that RedisJSON, a related project, does not have a built-in size limit, and 
the core data types in the Valkey system also do not have such a restriction.

The amount of memory consumed by a JSON document can be inspected by using the `JSON.DEBUG MEMORY` or `MEMORY USAGE` 
command. `JSON.DEBUG MEMORY <key> [<path>]` can also report the size of a JSON sub-tree.

### Nesting Depth Limit

When a JSON object or array has an element that is itself another JSON object or array, that inner object or array is 
said to “nest” within the outer object or array. To avoid stack overflow, it's good to have a limit on the depth.
ValkeyJSON will have a default path limit of 128 levels, configurable by module config json.max-path-limit. Any attempt 
to create a document of deeper than 128 levels will be rejected. 

## Specification

### Supported JSON Standard

[RFC7159](http://www.ietf.org/rfc/rfc7159.txt) and [ECMA-404](https://ecma-international.org/publications-and-standards/standards/ecma-404) 
JSON data interchange standard is supported. UTF-8 Unicode in JSON text is supported.

### Root JSON Value

In earlier RFC 4627, only objects or arrays were allowed as root values of JSON. Since [RFC 7159](http://www.ietf.org/rfc/rfc7159.txt), 
the root value of a JSON document can be of any type, scalar type (String, Number, Boolean, Null) or container type (Array, Object). 
ValkeyJSON will be compliant with [RFC 7159](http://www.ietf.org/rfc/rfc7159.txt).

### JSONPath Query Syntax

ValkeyJSON supports two kinds of JSONPath query syntaxes:
* Restricted syntax – Has limited query capabilities, compatible with RedisJSON v1.
* Enhanced syntax – Follows the [Goessner-style](https://goessner.net/articles/JsonPath/) JSONPath query syntax, as shown in the table below.

If a query path starts with '$', the enhanced syntax is used. Otherwise, the restricted syntax is used. A query using 
the enhanced syntax always returns an array of values, while a restricted-syntax query always returns a single value. 

Enhanced syntax:

| Symbol/Expression | Description                                                              |
|:------------------|:-------------------------------------------------------------------------|
| $                 | the root element                                                         |
| . or []           | child operator                                                           |
| ..                | recursive descent                                                        |
| *                 | wildcard. All elements in an object or array.                            |
| []                | array subscript operator. Index is 0-based.                              |
| [,]               | union operator                                                           |
| [start:end:step]  | array slice operator                                                     |
| ?()               | applies a filter expression to the current array or object               |
| @                 | used in filter expressions referring to the current node being processed |
| ==                | equal to, used in filter expressions.                                    |
| !=                | not equal to, used in filter expressions.                                |
| >                 | greater than, used in filter expressions.                                |
| >=                | greater than or equal to, used in filter expressions.                    |
| <                 | less than, used in filter expressions.                                   |
| <=                | less than or equal to, used in filter expressions.                       |
| &&                | logical AND, used to combine multiple filter expressions.                |
| &#124;&#124;      | logical OR, used to combine multiple filter expressions.                 |

Examples:

| JSONPath Expression                                           | Description                                 |
|:--------------------------------------------------------------| :-----------                                |
| $.store.book[*].author                                        | the authors of all books in the store       |
| $..author                                                     | all authors                                 |
| $.store.*                                                     | all members of the store                    |
| $["store"].*                                                  | all members of the store                    |
| $.store..price                                                | the price of everything in the store        |
| $..*                                                          | all recursive members of the JSON structure |
| $..book[*]                                                    | all books                                   |
| $..book[0]                                                    | the first book                              |
| $..book[-1]                                                   | the last book                               |
| $..book[0:2]                                                  | the first two books                         |
| $..book[0,1]                                                  | the first two books                         |
| $..book[0:4]                                                  | books from index 0 to 3 (ending index is not inclusive) |
| $..book[0:4:2]                                                | books at index 0, 2                         |
| $..book[?(@.isbn)]                                            | all books with isbn number                  |
| $..book[?(@.price<10)]                                        | all books cheaper than $10                  |
| '$..book[?(@.price < 10)]'                                    | all books cheaper than $10. (The path must be quoted if it contains whitespaces) |
| '$..book[?(@["price"] < 10)]'                                 | all books cheaper than $10             |
| '$..book[?(@.["price"] < 10)]'                                | all books cheaper than $10             |
| $..book[?(@.price>=10&&@.price<=100)]                         | all books in the price range of $10 to $100, inclusive |
| '$..book[?(@.price>=10 && @.price<=100)]'                     | all books in the price range of $10 to $100, inclusive. (The path must be quoted if it contains whitespaces) |
| $..book[?(@.sold==true&#124;&#124;@.in-stock==false)]         | all books sold or out of stock                         |
| '$..book[?(@.sold == true &#124;&#124; @.in-stock == false)]' | all books sold or out of stock. (The path must be quoted if it contains whitespaces)                 |
| '$.store.book[?(@.["category"] == "fiction")]'                | all books in the fiction category |
| '$.store.book[?(@.["category"] != "fiction")]'                | all books in non-fiction categories |

### JSON Command API

The API is compatible with RedisJSON v2. Note that API compatibility here means our command API is a superset of RedisJSON API. 
For example, we have command “JSON.DEBUG DEPTH” and “JSON.DEBUG FIELDS”, while they do not.

#### JSON.ARRAPPEND

Append one or more values to the array values at the path.

##### Syntax

```bash
JSON.ARRAPPEND <key> <path> <json> [json ...]
```

* key - required, JSON key
* path - required, a JSON path
* json - required, JSON value to be appended to the array

##### Return

* If the path is enhanced syntax:
    * Array of integers, representing the new length of the array at each path.
    * If a value at the path is not an array, its corresponding return value is null.
    * SYNTAXERR error if one of the input json arguments is not a valid JSON string.
    * NONEXISTENT error if the path does not exist.

* If the path is restricted syntax:
    * Integer, the array's new length.
    * If multiple array values are selected, the command returns the new length of the last updated array.
    * WRONGTYPE error if the value at the path is not an array.
    * SYNTAXERR error if one of the input json arguments is not a valid JSON string.
    * NONEXISTENT error if the path does not exist.

#### JSON.ARRINDEX

Search for the first occurrence of a scalar JSON value in the arrays at the path.

* Out of range errors are treated by rounding the index to the array's start and end.
* If start > end, return -1 (not found).

##### Syntax

```bash
JSON.ARRINDEX <key> <path> <json-scalar> [start [end]]
```

* key - required, JSON key.
* path - required, a JSON path.
* json-scalar - required, scalar value to search for. JSON scalar refers to values that are not objects or arrays.
  i.e., String, number, boolean and null are scalar values.
* start - optional, start index, inclusive. Defaults to 0 if not provided.
* end - optional, end index, exclusive. Defaults to 0 if not provided, which means the last element is included.
  0 or -1 means the last element is included.

##### Return

* If the path is enhanced syntax:
    * Array of integers. Each value is the index of the matching element in the array at the path. The value is -1 if not found.
    * If a value is not an array, its corresponding return value is null.

* If the path is restricted syntax:
    * Integer, the index of matching element, or -1 if not found.
    * WRONGTYPE error if the value at the path is not an array.
    

#### JSON.ARRINSERT

Insert one or more values into the array values at path before the index.

* Inserting at index 0 prepends to the array.
* A negative index values is interpreted as starting from the end.
* The index must be in the array's boundary.

##### Syntax

```bash
JSON.ARRINSERT <key> <path> <index> <json> [json ...]
```

* key - required, JSON key
* path - required, a JSON path
* index - required, array index before which values are inserted.
* json - required, JSON value to be appended to the array

##### Return

* If the path is restricted syntax:
    * Array of integers, representing the new length of the array at each path.
    * If a value is an empty array, its corresponding return value is null.
    * If a value is not an array, its corresponding return value is null.
    * OUTOFBOUNDS error if the index argument is out of bounds.

* If the path is restricted syntax:
    * Integer, the new length of the array.
    * WRONGTYPE error if the value at the path is not an array.
    * OUTOFBOUNDS error if the index argument is out of bounds.

#### JSON.ARRLEN

Get length of the array values at the path.

##### Syntax

```bash
JSON.ARRLEN <key> [path]
```

* key - required, JSON key
* path - optional, a JSON path. Defaults to the root path if not provided

##### Return

* If the path is enhanced syntax:
    * Array of integers, representing the array length at each path.
    * If a value is not an array, its corresponding return value is null.
    * Null if the document key does not exist.

* If the path is restricted syntax:
    * Integer, array length.
    * If multiple objects are selected, the command returns the first array's length.
    * WRONGTYPE error if the value at the path is not an array.
    * NONEXISTENT error if the path does not exist.
    * Null if the document key does not exist.

#### JSON.ARRPOP

Remove and return element at the index from the array. Popping an empty array returns null.

##### Syntax

```bash
JSON.ARRPOP <key> [path [index]]
```

* key - required, JSON key.
* path - optional, a JSON path. Defaults to the root path if not provided.
* index - optional, position in the array to start popping from.
    * Defaults -1 if not provided, which means the last element.
    * Negative value means position from the last element.
    * Out of boundary indexes are rounded to their respective array boundaries.

##### Return

* If the path is enhanced syntax:
    * Array of bulk strings, representing popped values at each path.
    * If a value is an empty array, its corresponding return value is null.
    * If a value is not an array, its corresponding return value is null.

* If the path is restricted syntax:
    * Bulk string, representing the popped JSON value
    * Null if the array is empty.
    * WRONGTYPE error if the value at the path is not an array.

#### JSON.ARRTRIM

Trim arrays at the path so that it becomes subarray [start, end], both inclusive.

* If the array is empty, do nothing, return 0.
* If start < 0, treat it as 0.
* If end >= size (size of the array), treat it as size-1.
* If start >= size or start > end, empty the array and return 0.

##### Syntax

```bash
JSON.ARRTRIM <key> <path> <start> <end>
```

* key - required, JSON key.
* path - required, a JSON path.
* start - required, start index, inclusive.
* end - required, end index, inclusive.

##### Return

* If the path is restricted syntax:
    * Array of integers, representing the new length of the array at each path.
    * If a value is an empty array, its corresponding return value is null.
    * If a value is not an array, its corresponding return value is null.
    * OUTOFBOUNDS error if an index argument is out of bounds.

* If the path is restricted syntax:
    * Integer, the new length of the array.
    * Null if the array is empty.
    * WRONGTYPE error if the value at the path is not an array.
    * OUTOFBOUNDS error if an index argument is out of bounds.

#### JSON.CLEAR

Clear the arrays or an objects at the path.

##### Syntax

```bash
JSON.CLEAR <key> <path>
```

* key - required, JSON key.
* path - optional, a JSON path. Defaults to the root path if not provided.

##### Return

* Integer, the number of containers cleared.
* Clearing an empty array or object accounts for 0 container cleared.
* Clearing a non-container value returns 0.
* If no array or object value is located by the path, the command returns 0.

#### JSON.DEBUG

Report information. Supported subcommands are:

* MEMORY <key> [path] - report memory usage in bytes of a JSON value. Path defaults to the root if not provided.
* DEPTH <key> - report the maximum path depth of the JSON document.
* FIELDS <key> [path] - report the number of fields at the specified document path. Path defaults to the root if not provided.
  Each non-container JSON value counts as one field. Objects and arrays recursively count one field for each of their
  containing JSON values. Each container value, except the root container, counts as one additional field.
* HELP - print help messages of the command.

##### Syntax

```bash
JSON.DEBUG <subcommand & arguments>
```

##### Return

Depends on the subcommand:

* MEMORY
    * If the path is enhanced syntax:
        * returns an array of integers, representing memory size (in bytes) of JSON value at each path.
        * returns an empty array if the JSON key does not exist.
    * If the path is restricted syntax:
        * returns an integer, memory size the JSON value in bytes.
        * returns null if the JSON key does not exist.
* DEPTH
    * returns an integer, the maximum path depth of the JSON document.
    * returns null if the JSON key does not exist.
* FIELDS
    * If the path is enhanced syntax:
        * returns an array of integers, representing number of fields of JSON value at each path.
        * returns an empty array if the JSON key does not exist.
    * If the path is restricted syntax:
        * returns an integer, number of fields of the JSON value.
        * returns null if the JSON key does not exist.
* HELP
    * returns an array of help messages

#### JSON.DEL

Delete the JSON values at the path in a JSON key. If the path is the root path, it is equivalent to deleting
the key from Valkey.

##### Syntax

```bash
JSON.DEL <key> [path]
```

* key - required, JSON key.
* path - optional, a JSON path. Defaults to the root path if not provided.

##### Return

* Number of elements deleted.
* 0 if the JSON key does not exist.
* 0 if the JSON path is invalid or does not exist.

#### JSON.FORGET
An alias of JSON.DEL

#### JSON.GET

Get the serialized JSON at one or multiple paths.

##### Syntax

```bash
JSON.GET <key>
         [INDENT indentation-string]
         [NEWLINE newline-string]
         [SPACE space-string]
         [NOESCAPE]
         [path ...]
```

* key - required, JSON key.
* INDENT/NEWLINE/SPACE - optional, controls the format of the returned JSON string, i.e., "pretty print". The default
  value of each one is empty string. They can be overidden in any combination. They can be specified in any order.
* NOESCAPE - optional, allowed to be present for legacy compatibility and has no other effect.
* path - optional, zero or more JSON paths, defaults to the root path if none is given. The path arguments must be
  placed at the end.

##### Return

* Enhanced path syntax:
    * If one path is given:
        * Return serialized string of an array of values.
        * If no value is selected, the command returns an empty array.
    * If multiple paths are given:
        * Return a stringified JSON object, in which each path is a key.
        * If there are mixed enhanced and restricted path syntax, the result conforms to the enhanced syntax.
        * If a path does not exist, its corresponding value is an empty array.

* Restricted path syntax:
    * If one path is given:
        * Return serialized string of the value at the path.
        * If multiple values are selected, the command returns the first value.
        * If the path does not exist, the command returns NONEXISTENT error.
    * If multiple paths are given:
        * Return a stringified JSON object, in which each path is a key.
        * The result conforms to the restricted path syntax if and only if all paths are restricted paths.
        * If a path does not exist, the command returns NONEXISTENT error

#### JSON.MGET

Get serialized JSONs at the path from multiple document keys. Return null for nonexistent key or JSON path.

##### Syntax

```bash
JSON.MGET <key> [key ...] <path>
```

* key - required, one or more JSON keys.
* path - required, a JSON path.

##### Return

* Array of Bulk Strings. The size of the array is equal to the number of keys in the command. Each element of the array
  is populated with either (a) the serialized JSON as located by the path or (b) Null if the key does not exist or the
  path does not exist in the document or the path is invalid (syntax error).
* If any of the specified keys exists and is not a JSON key, the command returns WRONGTYPE error.

#### JSON.MSET

Set JSON values for multiple keys. The operation is atomic. Either all values are set or none is set.

##### Syntax

```bash
JSON.MSET <key> <path> <value> [key <path> <value> ...]
```

* key - required, JSON key
* path - required, a JSON path
* value - JSON value

##### Return

* Simple String 'OK' on success
* Error on failure

#### JSON.NUMINCRBY

Increment the number values at the path by a given number.

##### Syntax

```bash
JSON.NUMINCRBY <key> <path> <number>
```

* key - required, JSON key
* path - required, a JSON path
* number - required, a number

##### Return

* If the path is enhanced syntax:
    * Array of bulk Strings representing the resulting value at each path.
    * If a value is not a number, its corresponding return value is null.
    * WRONGTYPE error if the number cannot be parsed.
    * OVERFLOW error if the result is out of the range of double.
    * NONEXISTENT if the document key does not exist.

* If the path is restricted syntax:
    * Bulk String representing the resulting value.
    * If multiple values are selected, the command returns the result of the last updated value.
    * WRONGTYPE error if the value at the path is not a number.
    * WRONGTYPE error if the number cannot be parsed.
    * OVERFLOW error if the result is out of the range of double.
    * NONEXISTENT if the document key does not exist.

#### JSON.NUMMULTBY

Multiply the number values at the path by a given number.

##### Syntax

```bash
JSON.NUMMULTBY <key> <path> <number>
```

* key - required, JSON key
* path - required, a JSON path
* number - required, a number

##### Return

* If the path is enhanced syntax:
  * Array of bulk Strings representing the resulting value at each path.
  * If a value is not a number, its corresponding return value is null.
  * WRONGTYPE error if the number cannot be parsed.
  * OVERFLOW error if the result is out of the range of double.
  * NONEXISTENT if the document key does not exist.

* If the path is restricted syntax:
    * Bulk String representing the resulting value.
    * If multiple values are selected, the command returns the result of the last updated value.  
    * WRONGTYPE error if the value at the path is not a number.
    * WRONGTYPE error if the number cannot be parsed.
    * OVERFLOW error if the result is out of the range of double.
    * NONEXISTENT if the document key does not exist.

#### JSON.OBJLEN

Get number of keys in the object values at the path.

##### Syntax

```bash
JSON.OBJLEN <key> [path]
```

* key - required, JSON key
* path - optional, a JSON path. Defaults to the root path if not provided

##### Return

* If the path is enhanced syntax:
    * Array of integers, representing the object length at each path.
    * If a value is not an object, its corresponding return value is null.
    * Null if the document key does not exist.

* If the path is restricted syntax:
    * Integer, number of keys in the object.
    * If multiple objects are selected, the command returns the first object's length.
    * WRONGTYPE error if the value at the path is not an object.
    * NONEXISTENT error if the path does not exist.
    * Null if the document key does not exist.

#### JSON.OBJKEYS

Get key names in the object values at the path.

##### Syntax

```bash
JSON.OBJKEYS <key> [path]
```

* key - required, JSON key
* path - optional, a JSON path. Defaults to the root path if not provided

##### Return

* If the path is enhanced syntax:
    * Array of array of bulk strings. Each element is an array of keys in a matching object.
    * If a value is not an object, its corresponding return value is empty value.
    * Null if the document key does not exist.

* If the path is restricted syntax:
    * Array of bulk strings. Each element is a key name in the object.
    * If multiple objects are selected, the command returns the keys of the first object.
    * WRONGTYPE error if the value at the path is not an object.
    * NONEXISTENT error if the path does not exist.
    * Null if the document key does not exist.

#### JSON.RESP

Return the JSON value at the given path in Redis Serialization Protocol (RESP).
If the value is container, the response is RESP array or nested array.

* JSON null is mapped to the RESP Null Bulk String.
* JSON boolean values are mapped to the respective RESP Simple Strings.
* Integer numbers are mapped to RESP Integers.
* Floating point numbers are mapped to RESP Bulk Strings.
* JSON Strings are mapped to RESP Bulk Strings.
* JSON Arrays are represented as RESP Arrays, where the first element is the simple string [,
  followed by the array's elements.
* JSON Objects are represented as RESP Arrays, where the first element is the simple string {,
  followed by key-value pairs, each of which is a RESP bulk string.
  
##### Syntax

```bash
JSON.RESP <key> [path]
```
* key - required, JSON key
* path - optional, a JSON path. Defaults to the root path if not provided

##### Return

* If the path is enhanced syntax:
    * Array of arrays. Each array element represents the RESP form of the value at one path.
    * Empty array if the document key does not exist.

* If the path is restricted syntax:
    * Array, representing the RESP form of the value at the path.
    * Null if the document key does not exist.

#### JSON.SET

Set JSON values at the path.

* If the path calls for an object member:
    * If the parent element does not exist, the command will return NONEXISTENT error.
    * If the parent element exists but is not an object, the command will return ERROR.
    * If the parent element exists and is an object:
        * If the member does not exist, a new member will be appended to the parent object if and only if the parent
          object is the last child in the path. Otherwise, the command will return NONEXISTENT error.
        * If the member exists, its value will be replaced by the JSON value.
* If the path calls for an array index:
    * If the parent element does not exist, the command will return a NONEXISTENT error.
    * If the parent element exists but is not an array, the command will return ERROR.
    * If the parent element exists but the index is out of bounds, the command will return OUTOFBOUNDS error.
    * If the parent element exists and the index is valid, the element will be replaced by the new JSON value.
* If the path calls for an object or array, the value (object or array) will be replaced by the new JSON value.

##### Syntax

```bash
JSON.SET <key> <path> <json> [NX | XX]
```

* key - required, JSON key.
* path - required, JSON path. For a new JSON key, the JSON path must be the root path ".".
* json - required, JSON representing the new value
* NX - optional. If the path is the root path, set the value only if the JSON key does not exist, i.e., insert a new document.
  If the path is not the root path, set the value only if the path does not exist, i.e., insert a value into the document.
* XX - optional. If the path is the root path, set the value only if the JSON key exists, i.e., replace the existing document.
  If the path is not the root path, set the value only if the path exists, i.e., update the existing value.

##### Return

* Simple String 'OK' on success.
* Null if the NX or XX condition is not met.

#### JSON.STRAPPEND

Append a string to the JSON strings at the path.

##### Syntax

```bash
JSON.STRAPPEND <key> [path] <json_string>
```

* key - required, JSON key.
* path - optional, a JSON path. Defaults to the root path if not provided.
* json_string - required, JSON representation of a string. Note that a JSON string must be quoted, i.e., '"foo"'.

##### Return

* If the path is enhanced syntax:
    * Array of integers, representing the new length of the string at each path.
    * If a value at the path is not a string, its corresponding return value is null.
    * SYNTAXERR error if the input json argument is not a valid JSON string.
    * NONEXISTENT error if the path does not exist.

* If the path is restricted syntax:
    * Integer, the string's new length.
    * If multiple string values are selected, the command returns the new length of the last updated string.
    * WRONGTYPE error if the value at the path is not a string.
    * WRONGTYPE error if the input json argument is not a valid JSON string.
    * NONEXISTENT error if the path does not exist.

#### JSON.STRLEN 

Get lengths of the JSON string values at the path.

##### Syntax

```bash
JSON.STRLEN <key> [path]
```

* key - required, JSON key
* path - optional, a JSON path. Defaults to the root path if not provided

##### Return

* If the path is enhanced syntax:
    * Array of integers, representing the length of string value at each path.
    * If a value is not a string, its corresponding return value is null.
    * Null if the document key does not exist.

* If the path is restricted syntax:
    * Integer, the string's length.
    * If multiple string values are selected, the command returns the first string's length.
    * WRONGTYPE error if the value at the path is not a string.
    * NONEXISTENT error if the path does not exist.
    * Null if the document key does not exist.

#### JSON.TOGGLE 

Toggle boolean values between true and false at the path.

##### Syntax

```bash
JSON.TOGGLE <key> [path]
```

* key - required, JSON key
* path - optional, a JSON path. Defaults to the root path if not provided

##### Return

* If the path is enhanced syntax:
    * Array of integers (0 - false, 1 - true) representing the resulting boolean value at each path.
    * If a value is a not boolean, its corresponding return value is null.
    * NONEXISTENT if the document key does not exist.

* If the path is restricted syntax:
    * String ("true"/"false") representing the resulting boolean value.
    * NONEXISTENT if the document key does not exist.
    * WRONGTYPE error if the value at the path is not a boolean.

#### JSON.TYPE

Report type of the values at the given path.

##### Syntax

```bash
JSON.TYPE <key> [path]
```

* key - required, JSON key
* path - optional, a JSON path. Defaults to the root path if not provided

##### Return

* If the path is enhanced syntax:
    * Array of strings, representing type of the value at each path. The type is one of {"null", "boolean", "string", "number", "integer", "object" and "array"}.
    * If a path does not exist, its corresponding return value is null.
    * Empty array if the document key does not exist.

* If the path is restricted syntax:
    * String, type of the value
    * Null if the document key does not exist.
    * Null if the JSON path is invalid or does not exist.

### ACL

ValkeyJSON introduces a new ACL category - @json. The category includes all JSON commands. No existing Valkey commands 
are members of the @json category. 

There are 4 existing ACL categories which are updated to include new JSON commands: @read, @write, @fast, @slow. The 
table below has a column for each of these categories and a row for each command. If the cell contains a “y” then that 
command must be added into that category. All other command members of those categories remain unchanged.

| JSON Command   | @json | @read | @write | @fast | @slow |
|:---------------|:------|:------|:-------|:------|:------|
| JSON.ARRAPPEND | y     |       | y      | y     |       |
| JSON.ARRINDEX  | y     | y     |        | y     |       |
| JSON.ARRINSERT | y     |       | y      | y     |       |
| JSON.ARRLEN    | y     | y     |        | y     |       |
| JSON.ARRPOP    | y     |       | y      | y     |       |
| JSON.ARRTRIM   | y     |       | y      | y     |       |
| JSON.CLEAR     | y     |       | y      | y     |       |
| JSON.DEBUG     | y     | y     |        |       | y     |
| JSON.DEL       | y     |       | y      | y     |       |
| JSON.FORGET    | y     |       | y      | y     |       |
| JSON.GET       | y     | y     |        | y     |       |
| JSON.MGET      | y     | y     |        | y     |       |
| JSON.MSET      | y     |       | y      |       | y     |
| JSON.NUMINCRBY | y     |       | y      | y     |       |
| JSON.NUMMULTBY | y     |       | y      | y     |       |
| JSON.OBJKEYS   | y     | y     |        | y     |       |
| JSON.OBJLEN    | y     | y     |        | y     |       |
| JSON.RESP      | y     | y     |        | y     |       |
| JSON.SET       | y     |       | y      |       | y     |
| JSON.STRAPPEND | y     |       | y      | y     |       |
| JSON.STRLEN    | y     | y     |        | y     |       |
| JSON.TOGGLE    | y     |       | y      | y     |       |
| JSON.TYPE      | y     | y     |        | y     |       |

### Info Metrics

Info metrics are visible through the “info json” or “info modules” command.

| Info Name               | 	Description                                                   |
|:------------------------|:------------------------------------------------------------------|
| json_total_memory_bytes | Total amount of memory allocated to JSON documents and meta data. |
| json_num_documents      | Number of JSON keys.                                              | 

### Module Configs

| Config Name            | Default Value | Unit | 	Description                                     |
|:-----------------------|:--------------|:-----|:------------------------------------------------------|
| json.max-document-size | 64            | MB   | Maximum memory allowed for a single JSON document.    |
| json.max-path-limit    | 128           |      | Maximum nesting levels within a single JSON document. |

### Module API

ValkeyJSON shall be implemented via the [Valkey modules API](https://valkey.io/topics/modules-intro/).

#### Module OnLoad

Upon loading, the module registers a new JSON data type. All operations such as query, insert, update and delete are 
efficiently performed on the in-memory document objects, as opposed to JSON text.

* Module name: json
* JSON data type name: ReJSON-RL (Note: We use the same name as the one in RedisJSON for the sake of RDB compatibility.)

#### Persistence

ValkeyJSON hooks into Valkey's persistence API via the module type callbacks:

* rdb_save: Serializes document objects to RDB. Serialized JSON string is saved in RDB.
* rdb_load: Deserializes document objects from RDB.
* aof_rewrite: Emits commands into the AOF during the AOF rewriting process.

#### Memory Management

The JSON data type also supports memory management related callbacks:

* free: Deallocates a key when it is deleted, expired or evicted.
* defrag: Supports active defrag for JSON keys
* mem_usage: Reports JSON document size (AKA “memory usage” command)
* copy: Supports copy of JSON key

#### Keyspace Event Notification

Every JSON write command publishes a keyspace event after the data is mutated.
* Event type: REDISMODULE_NOTIFY_GENERIC
* Event name: command name in lowercase. e.g., json.set command publishes event "json.set".

Users can subscribe to the JSON events via the standard keyspace event pub/sub. For example,

```text
1. enable keyspace event notifications:
    valkey-cli config set notify-keyspace-events KEA
2. subscribe to keyspace & keyevent event channels:
    valkey-cli psubscribe '__key*__:*'
```

#### Replication

Every JSON write command is replicated to replicas by calling ValkeyModule_ReplicateVerbatim.

## References

* [JSON Command API](https://docs.aws.amazon.com/memorydb/latest/devguide/json-list-commands.html)
* [JSONPath query syntax](https://docs.aws.amazon.com/memorydb/latest/devguide/json-document-overview.html#json-path-syntax)
* JSON blog: [Unlocking JSON workloads with ElastiCache and MemoryDB](https://aws.amazon.com/blogs/database/unlocking-json-workloads-with-elasticache-and-memorydb)
