# Database Interface Specification

A document database interface with two core verbs: `get` and `set`.

Maps describe one document or query. Lists describe multiple documents or alternative queries. `null` selects counting in `get` and deletion in `set`. Argument shape and position determine the operation.

This specification is language-neutral. It defines operations, inputs, results, and behavior independently of programming language, storage engine, or execution model.

Examples use pseudocode: `{}` denotes a map with string keys, `[]` denotes an ordered list, and `null` denotes an explicit null value. Quoted text denotes strings, `=` denotes assignment, `user.id` reads a map field, and `//` introduces a comment. The `?` in signatures marks an optional argument.

Language bindings choose their native types, function or method syntax, receiver or namespace, synchronous or asynchronous execution, and error-reporting mechanism. No `db` prefix, `await` keyword, or promise type is required. Bindings must preserve the distinction between omitted arguments, explicit null values, empty maps, and empty lists. A binding using unordered maps must provide a way to preserve the specified sort-field order.

## Core grammar

| Call | Operation | Result |
| --- | --- | --- |
| `get(collection, id, options?)` | Read by ID | Document or `null` |
| `get(collection, query, options?)` | Read the first match | Document or `null` |
| `get(collection, selectors, options?)` | Read all matches | List of documents |
| `get(collection, selector, null)` | Count matches | Number |
| `set(collection, document, options?)` | Upsert and merge | Stored document |
| `set(collection, documents, options?)` | Upsert and merge many | List of stored documents |
| `set(collection, id, null, options?)` | Delete by ID | Number deleted |
| `set(collection, ids, null, options?)` | Delete by IDs | Number deleted |

A collection is a non-empty string. An ID is a non-empty string, unique within its collection. A query is a map containing field conditions. A selector is an ID, query, or list of selectors. Options, when present, are always last.

Only the combinations defined here are part of the core interface.

## Documents

Documents are maps with string keys and JSON-compatible values: strings, finite numbers, booleans, null, maps, and lists. The `id` field identifies the document.

- A document without an ID receives a generated UUIDv7 when saved.
- An explicitly supplied ID must be a non-empty string.
- Field values of `null` are stored as JSON null; they do not delete fields.
- Nested maps and lists are valid document values.
- Field names are literal top-level fields. Dots do not imply traversal.
- Returned documents include their IDs.

Collections are available on first use without a separate creation call. Reading a missing collection behaves like reading an empty collection.

## Get

### One document

An ID or query map returns one document, or `null` if nothing matches:

```js
get('user', 'user-id')
get('user', { active: true })
get('user', {})
get('user') // Same as an empty query
```

An empty query matches every document. Without sorting, which match comes first is unspecified.

### Multiple documents

A list selector returns a list of matching documents:

```js
get('user', ['first-id', 'second-id'])
get('user', [{ active: true }])
get('user', [{}]) // All documents
get('user', []) // No documents
```

List entries are combined with OR. A document matching multiple entries appears only once. Missing IDs are omitted. Without sorting, result order is unspecified, including for lists of IDs.

```js
get('user', [
  { active: true, country: 'NO' },
  { active: true, country: 'SE' }
])
```

This returns active users in either country. Fields within each map are combined with AND.

### Options

Read options are `sort`, `limit`, and `offset`:

```js
get('user', [{ active: true }], {
  sort: { age: -1, name: 1 },
  limit: 20,
  offset: 40
})
```

- Sort directions are `1` for ascending and `-1` for descending. Sort fields are applied in their supplied order.
- `limit` and `offset` must be integers from 0 through 9,007,199,254,740,991 (2^53 − 1).
- An omitted limit imposes no limit on list reads. Offset defaults to zero.
- Sorting and offset apply before selecting the returned documents.
- Single-document reads return at most one match, even when a larger limit is supplied.
- A limit of zero returns `[]` for list reads and `null` for single-document reads.

```js
get('user', {}, { sort: { age: -1 } })
get('user', {}, { sort: { age: -1 }, offset: 1 })
```

These return the oldest and second-oldest matching users, respectively.

### Count

A literal `null` in the options position requests the number of matching documents:

```js
get('user', {}, null) // Count all documents
get('user', { active: true }, null)
get('user', [{ active: true }], null) // Same count
get('user', [{ country: 'NO' }, { country: 'SE' }], null)
get('user', 'user-id', null) // 0 or 1
get('user', [], null) // 0
```

Non-list selectors are automatically wrapped in a list for counting. A map therefore counts all matches rather than selecting only the first. Each matching document counts once, even if it matches several list entries.

Counting returns a number without returning documents. There are no pagination options in the count form.

## Query conditions

### Equality and comparisons

A scalar value is shorthand for `$eq`:

```js
get('user', { age: 30 })
get('user', { age: { $eq: 30 } })
```

These queries are equivalent. Query scalar values are strings, finite numbers, booleans, or `null`. Comparisons are type-sensitive: `30` does not equal `'30'`, and `true` does not equal `1`.

| Operator | Meaning |
| --- | --- |
| `$eq` | Equal to the supplied value |
| `$ne` | Not equal to the supplied value |
| `$gt` | Greater than |
| `$gte` | Greater than or equal to |
| `$lt` | Less than |
| `$lte` | Less than or equal to |
| `$in` | Equal to any supplied scalar value |
| `$nin` | Equal to none of the supplied scalar values |
| `$exists` | Whether the field is present; operand must be a boolean |
| `$not` | Negate a field condition or query |

Multiple operators on a field are combined with AND:

```js
get('user', [{ age: { $gte: 18, $lt: 65 } }])
get('user', [{ role: { $in: ['admin', 'editor'] } }])
get('user', [{ role: { $nin: ['guest', 'blocked'] } }])
```

`$in` and `$nin` compare the whole field to scalar values; they do not search inside stored lists. An empty `$in` matches nothing; an empty `$nin` matches everything.

### Logic, null, and missing fields

Map fields imply AND. Query lists imply OR. `$and` and `$or` are not supported.

`$not` can negate a field condition or a complete query:

```js
get('user', [{ age: { $not: { $lt: 18 } } }])
get('user', [{ $not: { active: true } }])
```

Equality requires the field to exist. Explicit `null` differs from a missing field. `$exists: true` includes null values:

```js
get('user', [{ email: null }]) // Explicit null only
get('user', [{ email: { $exists: true } }])
get('user', [{ email: { $exists: false } }])
get('user', [{ email: { $ne: null } }]) // Non-null or missing
```

Positive comparisons do not match missing fields. Negation includes missing fields when the positive condition would not match. Ordering comparisons against `null` match nothing.

## Set

### Upsert and merge

`set` always upserts:

- Without an ID, create a document with a generated ID.
- With an unknown ID, create a document with that ID.
- With an existing ID, merge supplied top-level fields into that document.

```js
user = set('user', { name: 'John', age: 30 })
set('user', { id: user.id, age: 31 })
set('user', { id: 'jane', name: 'Jane', age: 28 })
```

Omitted fields remain unchanged. Supplied nested maps and lists replace their previous values rather than merging recursively. A supplied field value of `null` is stored:

```js
set('user', {
  id: 'jane',
  profile: { country: 'NO' },
  tags: ['editor'],
  email: null
})
```

Each call returns the complete stored document, including unchanged fields and its ID. `set` does not update documents by query; an ID selects an existing document to merge.

### Multiple documents

A list saves multiple documents and returns their stored values in input order:

```js
set('user', [
  { id: 'jane', age: 29 },
  { name: 'John', age: 30 }
])

set('user', []) // []
```

Each entry follows the same upsert rules. Entries are applied in input order. A save call is atomic, including its lookup and merge: if an entry fails, the entire call is rolled back.

### Delete

An ID followed by `null` deletes that document. A list of IDs followed by `null` deletes those documents:

```js
set('user', 'user-id', null)
set('user', ['first-id', 'second-id'], null)
set('user', [], null) // 0
```

Deletion returns the number of documents deleted. Missing IDs are ignored, and duplicate IDs count once. Delete selectors must be IDs, not query maps. A delete call validates all IDs before removing documents.

### Options

Save and delete options currently accept only an empty map. Options remain last:

```js
set('user', { id: 'jane', age: 30 }, {})
set('user', 'jane', null, {})
```

## Errors

Invalid arguments, unsupported operators, and unsupported options produce an error through the language binding’s error-reporting mechanism. They must not be silently ignored. A normal read with no matches returns `null` or `[]`; a count or deletion with no matches returns `0`.

## Optional extensions

An implementation may expose `alter` as an optional extra for collection and index management. It is outside the core `get`/`set` specification; its signature and behavior are implementation-defined.

Index management is not required by this specification. Implementations may scan documents, manage indexes automatically, or offer explicit management separately. Connection setup, closing connections, and storage configuration are also outside the core interface.
