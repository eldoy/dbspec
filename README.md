Below is a **clean reset** to **dbspec v1.0**, rewritten to reflect the new philosophy:

* **Single portable query language, as much as possible**
* **Reference implementation defines semantics**
* **No over-specification that forces client-side emulation**
* **Backend-native behavior is allowed where portable**
* **Escape hatch (`data`) is explicit and intentional**
* **No mention of CouchDB, Mango, MongoDB, SQLite, or any backend by name**

This is a *grounded*, *executable* spec.

---

# dbspec v1.0

A backend-agnostic query and mutation specification for document-oriented storage.

This document is **normative**.
All behavior described is required unless explicitly marked **implementation-defined** or **optional**.

---

## 1. Purpose and Scope

dbspec defines a **portable query and mutation interface** for document-oriented databases.

The goal is to provide:

* A single query language usable across adapters
* Minimal application changes when switching adapters
* Native execution where possible
* Explicit escape hatches where portability ends

dbspec defines **interface and semantics**, not execution strategy.

---

## 2. Data Model

### 2.1 Document

A **document** is a JSON object with the following properties:

* Must be an object (`typeof === "object"`, not `null`)
* Must contain an `id` field of type `string`
* Field values may be:

  * JSON primitives (`string`, `number`, `boolean`, `null`)
  * Arrays
  * Objects
  * `Date` objects

### 2.2 Identity and Nullability

* A **missing field** is not equivalent to `null`
* `{ a: null }` and `{}` are distinct
* Equality and comparison operators MUST preserve this distinction

---

## 3. Logical Database Model

dbspec operates on **exactly one logical collection at a time**.

How a collection is selected is **host-defined**.

### 3.1 Collection Resolution

A dbspec-compliant host MUST expose at least one of the following models.

#### Model A — Single Collection

```js
db.get(...)
db.set(...)
```

All operations implicitly target the single collection.

#### Model B — Named Collections

```js
db(name).get(...)
db(name).set(...)
```

* `db(name)` returns a collection handle
* Each collection is an independent document space

#### Model C — Hybrid (Allowed)

Hosts MAY support both models simultaneously.

---

## 4. Core Collection Interface

The following methods are **required** on a collection handle:

```ts
get(query, options?, onBatch?)
set(queryOrDoc, values?)
```

These methods define the **entire portable surface area** of dbspec.

---

## 5. Query Semantics

### 5.1 Query Object Shape

```ts
Query := {
  [field: string]: Predicate | Value
}
```

Rules:

* `{}` matches all documents
* Multiple fields imply logical AND
* Field names are literal keys
* No dot-notation is implied unless the adapter documents support

---

### 5.2 Predicate Forms

```ts
Predicate :=
  | { $eq: Value }
  | { $ne: Value }
  | { $gt: Value }
  | { $gte: Value }
  | { $lt: Value }
  | { $lte: Value }
  | { $in: Value[] }
  | { $nin: Value[] }
  | { $regex: RegExp | string }
  | { $exists: boolean }
```

#### Shorthand Equality

```js
{ field: value }
```

Equivalent to:

```js
{ field: { $eq: value } }
```

---

### 5.3 Predicate Semantics

#### `$eq`

* Matches if field exists and values are equal
* Dates compare by millisecond timestamp

#### `$ne`

* Matches if field exists and value is not equal

#### `$gt`, `$gte`, `$lt`, `$lte`

* Field must exist
* Values compared by natural ordering
* Dates compare by timestamp

#### `$in`

* Field must exist
* Matches if value equals any element

#### `$nin`

* Field must exist
* Matches if value equals none of the elements
* Behavior for missing fields is **implementation-defined**

#### `$regex`

* Field must be a string
* String operand is compiled via `RegExp`
* Invalid patterns evaluate as non-matching
* No exceptions may be thrown

#### `$exists`

* `$exists: true` → field must be present
* `$exists: false` → field must be missing

---

### 5.4 Logical Operators

```ts
{ $and: Query[] }
{ $or: Query[] }
{ $not: Query }
```

Rules:

* `$and`: all subqueries must match
* `$or`: at least one subquery must match
* `$not`: negates the subquery
* Logical operators may appear at any level

---

## 6. Options Object

```ts
Options := {
  limit?: number        // default: 1000
  skip?: number         // default: 0
  sort?: { [field]: 1 | -1 }
  fields?: { [field]: boolean }
  count?: boolean
  batch?: number
}
```

### 6.1 limit / skip

* Applied after query evaluation
* `limit` caps total returned documents

### 6.2 sort

* Field → direction map
* Sort behavior for missing fields is **implementation-defined**
* Sort may fail if unsupported by the adapter

### 6.3 fields (Projection)

* `true` includes field
* `false` excludes field
* If any `true` is present, projection is inclusive
* `id` is included by default unless explicitly excluded

### 6.4 count

* If `true`, no documents are returned
* Return value is `{ count: number }`
* Streaming is disabled

### 6.5 batch

* Controls delivery size in streaming mode
* Does not imply execution strategy

---

## 7. Streaming Mode

Activated when `onBatch` is provided.

```js
get(query, options, onBatch)
```

Rules:

* Results are delivered via `onBatch(docs[])`
* `docs.length <= batch`
* Calls are sequential
* Internal buffering is **allowed**
* `get` returns `undefined`

Streaming defines **delivery**, not execution.

---

## 8. Mutation Semantics (`set`)

### 8.1 Insert

```js
set(document)
```

* Inserts document
* If `id` missing, host generates one
* Object MAY be mutated to attach `id`
* Returns inserted document

---

### 8.2 Bulk Insert

```js
set(documents[])
```

* Each document treated as independent insert
* Operation is atomic per document
* Returns inserted documents (same object references)

---

### 8.3 Update

```js
set(query, values)
```

* Updates all matching documents
* Shallow merge
* `undefined` removes field
* `null` sets field to `null`
* Returns `{ n: number }`

---

### 8.4 Delete

```js
set(query, null)
```

* Deletes all matching documents
* Returns `{ n: number }`

---

### 8.5 Clear

```js
set({}, null)
```

Deletes all documents in the current collection.

---

## 9. Adapter Extensions (Allowed)

Adapters MAY expose additional APIs outside dbspec, provided:

* Core semantics remain unchanged
* Extensions are clearly documented

Examples:

```js
db.drop()
db(name).drop()
db.compact(name)
db.info()
db.version
```

These are **out of scope** for dbspec.

---

## 10. Escape Hatch

Adapters MAY expose backend-native access via:

```js
db.data
```

Use of `data` is **explicitly non-portable**.

---

## 11. Versioning

```ts
dbspec.version = "1.0"
```

Backward-incompatible changes require a major version increment.

---

**End of dbspec v1.0**
