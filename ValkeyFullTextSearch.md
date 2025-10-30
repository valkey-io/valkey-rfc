---
RFC: 24
Status: Proposed
---

# Title
Text Searching

## Abstract

The existing search platform is extended with a new field type, ```TEXT``` is defined. The query language is extended to support locating keys with text fields based on term, wildcard, fuzzy and exact phrase matching.

## Motivation

Text searching is useful in many applications.

This text search implementation targets developers building low-latency applications with fast and frequently modifying data. We expect dramatically shorter query and ingestion times than other libraries.

Characteristics of the full text search engine:
1. Lower Cost per query due to time-optimized data structures and reduced parsing/command processing overhead.
2. Incremental update machinery. Data structures support incremental updates without GC.
3. Current queries, i.e., query operations include the most recent mutations, i.e., mutation to query visibility in less than a millisecond.

Use cases:
* Chat / Session History Search
 * Per-user session data loaded at login/web session initiation
 * Enables searching through user interaction history, chat, commands, or activity logs
 * Provides fast retrieval of relevant past sessions or actions

* RAG (Retrieval-Augmented Generation) Operations
 * Complex text content / filtering that can't be predicted or converted to TAG operations
 * Handles unpredictable phrasing and natural language queries
 * Combines with vector similarity search for hybrid workloads

## Terminology

In the context of this RFC.

| Term        | Meaning                                                                                                                                                                                                                     |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| key         | A Valkey key. Depending on the context of usage could be the text of the key name or the contents of the HASH or JSON key.                                                                                                  |
| field       | A component of an index. Each field has a type and a path.                                                                                                                                                                  |
| index       | A collection of fields and field-indexes. The object created by the ```FT.CREATE``` command.                                                                                                                                |
| field-index | A data structure associated with a field that is intended to accelerate the operation of the search operators for this field type.                                                                                          |
| character   | An Unicode character. A character may occupy 1-4 bytes of a UTF-8 encoded string.                                                                                                                                           |
| word        | A syntactic unit of text consisting of a list of characters. A word is delimited by un-escaped punctuation and/or whitespace.                                                                                             |
| token       | same as a word                                                                                                                                                                                                              |
| text        | A UTF-8 encoded string of bytes.                                                                                                                                                                                            |
| stemming    | A process of mapping similar words into a common base word. For example, the words _drives_, _drove_ and _driven_ would be replaced with the  _drive_.                                                                      |
| term        | The output of stemming a word.                                                                                                                                                                                              |
| prefix tree | A mapping from input string to an output object. Insert/delete operations are O(length(input)). Additional operations include the ability to iterate over the entries that share a common prefix in O(length(prefix)) time. |

## Design considerations

The text searching facility provides machinery that decomposes text fields into terms and field-indexes them.
The query facility of the ```FT.SEARCH``` and ```FT.AGGREGATE``` commands is enhanced to select keys based on combinations of term, fuzzy and phrase matching together with the existing selection operators of Tag, Numeric range and Vector searches.

### Tokenization process

A tokenization process is applied to strings of text to produce a list of terms.
Tokenization is applied in two contexts.
First as part of the ingestion process for text fields of keys.
Second to process query string words and phrases.

The tokenization is language dependent (based on language specified in index creation) and currently supports
English. Logic will need to be extended to support other languages in the future.

The tokenization process has four steps.

1. For most languages (e.g. english), the text is tokenized, removing punctuation and redundant whitespace, generating a list of words. For other languages (CJK, etc.), tokenization requires
language aware boundary detection. This will be done using ICU's BreakIterator that accepts a custom dictionary.
2. (Depending on the language), Upper-case characters are converted to lower-case.
3. (Depending on the language), Stop words are removed from the list. A default stopword list is maintained for all relevant languages.
4. (Depending on the language), Words with more than a fixed number of characters are replaced with their stemmed equivalent term according to the selected language.

In case of query parsing, we will need to parser will need to differentiate between query syntax characters and the actual text content.
This is done by detecting query syntax boundaries (start and end).
Additionally, for multi-language support, the parser checks the current position of the query syntax string to see if the bytes is either an ASCII character (0x00-0x7F) or
UTF-8 start byte (≥0xC0), avoiding false positive matches on UTF-8 continuation bytes (0x80-0xBF) that could be mistaken for query syntax characters. Once the
```
┌─────────────────┐     ┌───────────────┐     ┌──────────────┐     ┌───────────────┐
│  Input Text     │     │ Tokenization  │     │ Case         │     │ Stop Word     │
│  "The Running   │ ──> │ ["The",       │ ──> │ ["the",      │ ──> │ ["running",   │
│   Searches cat" │     │  "Running",   │     │  "running",  │     │  "searches",  │
└─────────────────┘     │  "Searches",  │     │  "searches", │     │  "cat"]       │
                        │  "cat"]       │     │  "cat"]      │     └───────┬───────┘
                        └───────────────┘     └──────────────┘             │
                                                                           V
                                                                      ┌───────────────┐
                                                                      │ Stemming      │
                                                                      │ ["run",       │
                                                                      │  "search",    │
                                                                      │  "cat"]       │
                                                                      └───────────────┘
```

Step 1 of the tokenization process is controlled by a specified list of punctuation characters. 
Sequences of one or more punctuations characters delimit word boundaries.
Note that individual punctuation characters can be escaped using the normal backslash prefix syntax, causing the next character to be treated as non-punctuation.

### Text Field-Index Structure

The text field-index is a mapping from a term to a list of locations.
Depending on the configuration, the list of locations may be just a list of keys or a list of key/term-offset pairs. The list of locations is also known as a postings or a postings list.

The mapping structure is built from one or two prefix trees. When present, the second prefix tree is built from each inserted term sequenced in reverse order, effectively providing a suffix tree.

Here is a visual example. Note that in the drawing, the sections noted as "same postings as _xxxx_" mean that the pointer actually points to the same memory area as referenced. In other words, when a suffix tree is present, the postings list is referenced by both (think shared_ptr).

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                  TextIndex                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│ ┌──────────────────┐                                ┌──────────────────┐     │
│ │   Prefix Tree    │             Postings           │   Suffix Tree    │     │
│ └──────────────────┘                                └──────────────────┘     │
│ │                  │                                │                  │     │
│ │  ┌────────────┐  │          ┌────────────┐        │  ┌────────────┐  │     │
│ │  │   apple    │──┼─────────>│ key1: [0]  │<───────┼──│   elppa    │  │     │
│ │  └────────────┘  │          │ key2: [3]  │        │  └────────────┘  │     │
│ │                  │          └────────────┘        │                  │     │
│ │                  │                                │                  │     │
│ │  ┌────────────┐  │          ┌────────────┐        │  ┌────────────┐  │     │
│ │  │   banana   │──┼─────────>│ key1: [1]  │<───────┼──│   ananab   │  │     │
│ │  └────────────┘  │          │ key3: [2]  │        │  └────────────┘  │     │
│ │                  │          └────────────┘        │                  │     │
│ │                  │                                │                  │     │
│ │  ┌────────────┐  │          ┌────────────┐        │  ┌────────────┐  │     │
│ │  │   cherry   │──┼─────────>│ key2: [0]  │<───────┼──│   yrrehc   │  │     │
│ │  └────────────┘  │          │ key3: [8]  │        │  └────────────┘  │     │
│ │                  │          └────────────┘        │                  │     │
│ │                  │                                │                  │     │
│ └──────────────────┘                                └──────────────────┘     │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```


### Search query operators

While the syntax for Vector, Tag and Numeric query operators requires the presence of a field specifier, it is optional for text query operators.
If a field specifier is omitted for a text query operator then this is syntactic sugar for specifying all of the text fields.

As with the other query operators, the text query operators fit into the and/or/negate/parenthesized query language structure.

Here are the main type of queries:
1. Term
2. Prefix
3. Suffix
4. Infix
5. Fuzzy
6. Proximity (Includes Exact Phrase)

#### Term Matching

Term matching operates on the normalized terms after pre-processing (tokenization, lowercasing, stopword removal, stemming if enabled on the index and if the FT.SEARCH does not specify VERBATIM) and only the matched keys are returned.

| Example | Interpretation | Required Index Options |
|:-:|---|:-:|
| `'abcd'` | This is either stemmed search | none |
| `'abcd'` | This is an exact match search | NOSTEM |
| `'"abcd"'` | This is an exact match search | none |
| `'abcd'` VERBATIM | This is an exact match search | none |


#### Prefix Matching

| Example | Interpretation | Required Index Options |
|:-:|---|:-:|
| `'a*'`   | matches any term starting with `a` | none |
| `'fred*'` | matches any term starting with `fred` | none |

#### Suffix Matching

| Example | Interpretation | Required Index Options |
|:-:|---|:-:|
| `'*z'` | matches any term ending with `z` | SUFFIXTRIE |
| `'*red'` | matches any term ending with `zred` | SUFFIXTRIE |

#### Infix Matching

| Example | Interpretation | Required Index Options |
|:-:|---|:-:|
| `'*ab*'`  | matches any term that contains `ab` in the middle | SUFFIXTRIE |

#### Fuzzy Matching

Fuzzy word matching is specified by enclosing a word in pairs of percent signs ```%```. This term matches words within one [Levenshtein edit distance](https://en.wikipedia.org/wiki/Levenshtein_distance) edit distance for each pair of percent signs from the specified word.

| Example | Interpretation |
|:-:|---|
| `'%abc%'` | Matches any term within one edit-distance of `abc` |
| `'%%abc%%'` | Matches any term within two edit-distances of `abc` |

#### Proximity Matching

Proximity queries involve two or more text tokens in the query and have a concept of slop and inorder properties.
slop of 1 means we allow upto 1 intervening word between tokens in the query.

Double quotes can be used to enclose term tokens (not wildcard / fuzzy) to imply a
slop of 0 and the inorder requirement. This is called an "exact phrase" query. One additional nuance to the exact phrase operation is that stopwords are not removed from the tokens enclosed within the double quotes.
Hence, to be able to match results, stopwords should not be used the exact phrase because they were not indexed.

Alternatively, the `INORDER` / `SLOP` arguments can be explicitly provided (as FT.SEARCH command arguments or Query Modifying Attributes) to apply the proximity constraints and this can be done even on wildcard / fuzzy tokens as shown in the examples below. However, when doing this, the field specifiers for text tokens must be the same because we cannot derive an positional allignment across text content in different fields within a document.

All proximity matching queries require the `WITHOFFSETS` index creation option.

| Example | Interpretation |
|:-:|---|
| `'"a b"'` | Exact phrase matches term `a` immediately followed by term `b`.  |
| `'a %b%"' INORDER` | Matches any document that contains `a` immediately followed by any term within one edit-distance of `b`. |
| `'a %b%"' INORDER SLOP 3` | Matches any document that contains `a` immediately followed by any term within one edit-distance of `b` allowing up to 3 intervening words. |
| `'*ab* c' INORDER SLOP 2` | Matches any document that contains `ab` followed by the term `c` allowing up to 2 intervening words. |
| `'"a b c"'` | Exact phrase match on the terms `a`, `b` and `c` in that order with no intervening words. |
| `'@title:"a b c"'` | Exact phrase match on any document that contains the terms `a`, `b` and `c` in that order with no intervening words in the field named `title`. |
| `'(a b c) => [$inorder:false]'` | Query Modifying attributes (QMA) are applied on matches of the terms `a`, `b`, and `c` such that there is no order requirement or requirement on number of intervening words |
| `'(a b) => [$inorder:false, $slop:1]'` |  Query Modifying attributes (QMA) are applied on matches of the terms `a` and `b` such that there is no order requirement but there is a requirement on number of allowed intervening words being 1  |

### Extension of the ```FT.SEARCH``` command

Aside from the updates to support new query operations within the query string, there are new command arguments being supported in the FT.SEARCH command.
* INORDER - Adds an order constraint in the search to ensure the text tokens matched are in the order specifed in the query string.
* SLOP - Specifies the number intervening words allowed in between the matched tokens in documents. This cannot be used to override the slop of individual exact phrase opertations within a query string as they use slop of 0.
* VERBATIM - Specifies that the term tokens processed from the query string are not stemmed.

### Search Query Operations Flow

The following diagram illustrates the flow of a text search operation from query to results:

```
┌──────────────────┐
│ Search Query     │
│ e.g., "quick     │
│ brown fox"       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     ┌──────────────────────────────────────────┐
│ Query Parser     │────▶│ Parsed Query Tree                        │
└────────┬─────────┘     │                                          │
         │               │    ┌─────────┐                           │
         │               │    │ PHRASE  │                           │
         │               │    └────┬────┘                           │
         │               │         │                                │
         │               │  ┌──────┴──────┼─────────────┐           │
         │               │  │             │             |           │
         │               │  ▼             ▼             ▼           │
         │               │ ┌─────┐     ┌─────┐     ┌─────┐          │
         │               │ │quick│     │brown│     │ fox │          │
         │               │ └─────┘     └─────┘     └─────┘          │
         │               └──────────────────────────────────────────┘
         ▼
┌──────────────────┐
│ Query Execution  │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Term Dictionary Lookup                                               │
│                                                                      │
│ ┌─────────────┐         ┌─────────────┐         ┌─────────────┐      │
│ │Term: "quick"│         │Term: "brown"│         │Term: "fox"  │      │
│ └──────┬──────┘         └──────┬──────┘         └──────┬──────┘      │
│        │                       │                       │             │
│        ▼                       ▼                       ▼             │
│ ┌──────────────┐        ┌──────────────┐        ┌──────────────┐     │
│ │Postings List │        │Postings List │        │Postings List │     │
│ └──────┬───────┘        └──────┬───────┘        └──────┬───────┘     │
│        │                       │                       │             │
└────────┼───────────────────────┼───────────────────────┼─────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                                 ▼
┌───────────────────────────────────────────────────────────────────┐
│ Position Filtering (for phrase matching)                          │
│                                                                   │
│ Check if terms appear in correct sequence with proper spacing     │
│                                                                   │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ Document: key1                                              │   │
│ │                                                             │   │
│ │ [quick:pos2]───┐                                            │   │
│ │                │                                            │   │
│ │ [brown:pos3]───┼─── Sequential? YES                         │   │
│ │                │                                            │   │
│ │ [fox:pos4]─────┘                                            │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                   │
└───────────────────────────────────────────────────────┬───────────┘
                                                        │
                                                        ▼
                                             ┌───────────────────┐
                                             │ Results           │
                                             │ [key1, key5, ...] │
                                             └───────────────────┘
```

## System Architecture

The following diagram illustrates the overall architecture and data flow of the text search system:

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                        Text Search System Architecture                        │
├───────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌───────────────┐        ┌────────────────┐         ┌────────────────────┐   │
│  │ Client        │        │ Command        │         │ Search Query       │   │
│  │ Applications  │───────▶│ Processing     │────────▶│ Execution Engine   │   │
│  └───────────────┘        │ (FT.SEARCH)    │         └──────────┬─────────┘   │
│                           └────────────────┘                     │            │
│                                    ▲                             │            │
│                                    │                             │            │
│                                    │                             ▼            │
│                                    │                  ┌────────────────────┐  │
│                                    │                  │ Text Index         │  │
│                                    │                  │                    │  │
│                                    │                  │ - Prefix Tree      │  │
│                                    │                  │ - Suffix Tree      │  │
│                                    │                  │ - Postings Lists   │  │
│                                    │                  └──────────┬─────────┘  │
│                                    │                             │            │
│                           ┌────────┴───────┐                     │            │
│  ┌───────────────┐        │ Result         │◀────────────────────┘            │
│  │ Client        │◀───────│ Formatting     │                                  │
│  │ Applications  │        │                │                                  │
│  └───────────────┘        └────────────────┘                                  │
│                                                                               │
│                                                                               │
│  ┌───────────────┐        ┌────────────────┐         ┌────────────────────┐   │
│  │ Client        │        │ Command        │         │ Text Index         │   │
│  │ Applications  │───────▶│ Processing     │────────▶│ Builder            │   │
│  └───────────────┘        │ (FT.CREATE)    │         └──────────┬─────────┘   │
│                           └────────────────┘                     │            │
│                                                                  │            │
│                                                                  ▼            │
│                                                       ┌────────────────────┐  │
│                                                       │ Tokenization       │  │
│                                                       │ Pipeline           │  │
│                                                       │                    │  │
│                                                       │ 1. Split on punct. │  │
│                                                       │ 2. Case conversion │  │
│                                                       │ 3. Stop word filter│  │
│                                                       │ 4. Stemming        │  │
│                                                       └──────────┬─────────┘  │
│                                                                  │            │
│                                                                  ▼            │
│                                                       ┌────────────────────┐  │
│                                                       │ Term Storage in    │  │
│                                                       │ Prefix/Suffix Trees│  │
│                                                       └────────────────────┘  │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

```
## Specification

Each ```TEXT``` field has the a set of configurations that control the process to convert a string into a list of terms, others control the contents of the generated index.

| Metadata          | Type              | Meaning                                                                    | Default Value |
| ----------------- | ----------------- | -------------------------------------------------------------------------- | ------------- |
| Punctuation       | vector<character> | Characters that separate words during the tokenization process.            |
| Stop Words        | vector<String>    | List of stop words to be removed during decomposition                      |
| Stemming Language | Enumeration       | Snowball stemming library language control                                 |
| Suffix Tree       | Boolean           | Controls the presence/absence of a suffix tree.                            |
| Word Offsets      | Boolean           | Indicates if the postings for a word contain word offsets or just a count. |

### Extension of the ```FT.CREATE``` command

Clauses are provided to control the configuration. 
If supplied before the ```SCHEMA``` keyword then the default value of the clause is changed for any text fields declared in this command.
If supplied as part of an individual field declaration, i.e., after the ```SCHEMA``` keyword, then it sets the configurable for just that field.

```
[PUNCTUATION <string>]
```
The characters of this string are used to split the input string into words. Note, the splitting process allows escaping of input characters using the usual backslash notation. This string cannot be empty. Default value for an index with the language as english is: `,.<>{}[]\"':;!@#$%^&*()-+=~/\\|`.


```
[NOSTOPWORDS | STOPWORDS <count> [word ...]]
```

Controls the application and list of stop words. Note that ```STOPWORDS 0``` is equivalent to ```NOSTOPWORDS```. The default english stop words are:

**a,    is,    the,   an,   and,  are, as,  at,   be,   but,  by,   for,
 if,   in,    into,  it,   no,   not, of,  on,   or,   such, that, their,
 then, there, these, they, this, to,  was, will, with**.

```
[MINSTEMSIZE <size>]
```

This clause controls the minimum number of characters in a word before it is subjected to stemming. Default value is 4.

```
[NOSTEM | LANGUAGE <language>]
```

Controls the stemming algorithm. Supported languages are: 

**none, english**

The default language is **english**. Note: ```LANGUAGE none``` is equivalent to ```NOSTEM```.

```
[WITHSUFFIXTRIE | NOSUFFIXTRIE]
```

Controls the presence/absence of the second prefix tree in the field-index. Default is ```NOSUFFIXTRIE```.

```
[WITHOFFSETS | NOOFFSETS]
```

Controls whether term-offsets are tracked in the field-index. Default is ```WITHOFFSETS```.

### Limits

To avoid combinatorial explosion certain operations have configurable limits are listed below:

| Name                 | Default | Limit                                                                         |
| -------------------- | ------- | ------------------------------------------------------------------------------|
| max-fuzzy-distance   | 3       | The maximum edit distance for a fuzzy search.                                 |
| max-expansions       | 200     | Maximum number of words that a single wildcard / fuzzy match can generate     |
| max-search-results   | TBD.    | Maximum search results for a query search when limits are not applied         |

### Dependencies

snowball library https://snowballstem.org/ and https://github.com/snowballstem. Languages supported by this library are: arabic, armenian, basque, catalan, danish, dutch, english, estonian, finnish, french, german, greek, hindi, hungarian, indonesian, irish, italian, lithuanian, nepali, norwegian, porter, portuguese, romanian, russian, scripts, serbian, spanish, swedish, tamil, turkish, and yiddish.
Note: There are some languages not supported by this library such as CJK. These do not need any stemming.
