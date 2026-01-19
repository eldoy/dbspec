# dbspec v1.0

A backend-agnostic query and mutation specification for document-oriented storage.

This document is **normative**. All behavior described is required unless explicitly marked optional.

---

## 1. Data Model

### 1.1 Document

A **document** is a JSON object with the following properties:

- Must be an object (`typeof === "object"`, not `null`)
- Must contain an `id` field of type `string`
- Field values may be:
  - JSON primitives (`string`, `number`, `boolean`, `null`)
  - Arrays
  - Objects
  - `Date` objects

### 1.2 Identity and Nullability

- A **missing field** is not equivalent to `null`
- `{ a: null }` and `{}` are distinct
- Field comparison must preserve this distinction

---

## 2. Host Interface

dbspec defines **only semantics**. Invocation is handled by the host.

Required host methods:

```ts
get(query, options?, onBatch?)
set(queryOrDoc, values?)
````

dbspec defines:

* Query structure
* Predicate semantics
* Mutation semantics
* Option behavior
* Streaming behavior

---

## 3. Query Semantics

### 3.1 Query Object Shape

```ts
Query := {
  [field: string]: Predicate | Value
}
```

Rules:

* An empty query `{}` matches all documents
* Multiple fields imply an implicit logical AND
* Field names are literal keys (no dot-notation implied unless implemented)

---

### 3.2 Predicate Forms

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

#### 3.2.1 Shorthand Equality

```js
{ field: value }
```

Is equivalent to:

```js
{ field: { $eq: value } }
```

---

### 3.3 Predicate Evaluation Rules

#### `$eq`

* True if field exists and values are equal
* Equality is **value-based**, not reference-based
* Dates compare by millisecond timestamp

#### `$ne`

* True if field does not exist **or** value is not equal

#### `$gt`, `$gte`, `$lt`, `$lte`

* Compare normalized values
* Date values compare by timestamp
* If field is missing, predicate fails

#### `$in`

* True if field value equals any element in array
* Equality uses `$eq` semantics

#### `$nin`

* True if field value equals none of the elements
* Missing fields evaluate as true

#### `$regex`

* If value is not a string → false
* If operand is `RegExp` → used directly
* If operand is string → compiled using `new RegExp(string)`
* If compilation fails → predicate evaluates false
* No exceptions may be thrown

#### `$exists`

* `$exists: true` → field must be present (even if `null`)
* `$exists: false` → field must be missing

---

### 3.4 Logical Operators

Logical operators operate on full queries.

```ts
{ $and: Query[] }
{ $or: Query[] }
{ $not: Query }
```

Rules:

* `$and`: all subqueries must match
* `$or`: at least one subquery must match
* `$not`: negates the subquery result
* Logical operators may appear at any level
* Logical operators do not short-circuit semantic correctness

---

## 4. Options Object

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

### 4.1 limit

* Maximum number of documents returned
* Applies to the entire operation, not per batch

### 4.2 skip

* Number of matching documents to discard before returning results

### 4.3 sort

* Ordered map of field → direction
* `1` = ascending, `-1` = descending
* Fields are compared in declaration order
* Missing fields sort after present fields

### 4.4 fields (Projection)

* `true` includes field
* `false` excludes field
* If omitted, all fields are returned
* `id` is always included unless explicitly excluded

### 4.5 count

* If `true`, no documents are returned
* Return value is `{ count: number }`
* Streaming is disabled

### 4.6 batch

* Enables batching in streaming mode
* Maximum number of documents per batch

---

## 5. Streaming Mode

Activated when `onBatch` is provided.

```js
get(query, options, onBatch)
```

Rules:

* Results are delivered via `onBatch(docs[])`
* `docs.length <= batch`
* `onBatch` is called sequentially
* No full result buffering allowed
* `get` returns `undefined`
* `limit` applies across all batches

---

## 6. Mutation Semantics (`set`)

### 6.1 Insert

```js
set(document)
```

Behavior:

* Inserts document
* If `id` missing, host generates one
* Document is stored as-is
* Returns inserted document

---

### 6.2 Update

```js
set(query, values)
```

Behavior:

* All matching documents are updated
* Update is a **shallow merge**
* For each key in `values`:

  * `undefined` → remove field
  * `null` → set field to `null`
  * other → overwrite value
* Returns `{ n: number }` (number of updated documents)

---

### 6.3 Delete

```js
set(query, null)
```

Behavior:

* Deletes all matching documents
* Returns `{ n: number }`

---

### 6.4 Clear

```js
set({}, null)
```

Deletes all documents in the collection.

---

## 7. Evaluation Guarantees

* All predicates are **pure**
* No user-defined functions
* No observable side effects
* Backends may:

  * Reorder predicates
  * Short-circuit evaluation
  * Push down filters
* Results must remain semantically identical

---

## 8. Backend Capabilities

Backends may expose:

```ts
capabilities = {
  indexed: boolean
  streaming: boolean
  countOptimized: boolean
}
```

These flags are informational only and do not affect semantics.

---

## 9. Escape Hatch

Backends may expose raw storage via:

```js
db.data
```

dbspec imposes no constraints on this interface.

---

## 10. Versioning

```ts
dbspec.version = "1.0"
```

Backward-incompatible changes require a major version increment.

---

**End of dbspec v1.0**
