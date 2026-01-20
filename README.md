## RFC: Database Adapter Interface Specification

---

## 1. Scope

This RFC defines a minimal, uniform database adapter interface.
The specification is **definition-only**.

---

## 2. Core Concepts

### 2.1 Document

* JavaScript object convertible to JSON
* `id` is the logical identifier
* `_id` is used transparently when required by the underlying client
* `rev` is optional and adapter-defined
* `null` values are removed before storage
* Date objects are converted to the underlying client representation

---

## 3. Adapter Surface

### 3.1 `insert`

```ts
insert(values: Array<JSONSerializableObject>)
  -> Array<{ id: ID, rev?: Rev }>
```

* Inserts multiple documents
* UUID generated if `id` missing
* Input order preserved

---

### 3.2 `update`

```ts
update(query: Query, values: JSONSerializableObject)
  -> { n: number }
```

* Updates all matching documents
* `null` properties removed

---

### 3.3 `remove`

```ts
remove(query: Query)
  -> { n: number }
```

* Removes all matching documents

---

### 3.4 `list`

```ts
list(query: Query, queryOptions?: QueryOptions)
  -> Array<Doc>
```

* Default limit: 100

---

### 3.5 `batch`

```ts
batch(
  query: Query,
  batchOptions: QueryOptions & { size?: number },
  handler: (docs: Array<Doc>) => void | Promise<void>
) -> void | Promise<void>
```

* Default `size`: 1000

---

### 3.6 `set`

```ts
set(values: JSONSerializableObject)
  -> Doc

set(query: Query, values: JSONSerializableObject | null)
  -> Doc | null
```

* Always returns full document
* `set(values)` with existing `id` returns existing doc without write
* `set(query, null)` deletes and returns deleted doc or `null`

---

### 3.7 `get`

```ts
get(query: Query, queryOptions?: QueryOptions)
  -> Doc | null
```

---

### 3.8 `count`

```ts
count(query: Query, queryOptions?: QueryOptions)
  -> number
```

---

### 3.9 `index`

```ts
index(...args: unknown[]) -> unknown
```

* Adapter-specific

---

## 4. Query Semantics

### 4.1 Query Shape

```ts
Query := {
  [field: string]: Predicate | Value
}
```

* `{}` matches all documents
* Multiple fields imply logical AND
* No implicit dot-notation

---

### 4.2 Predicate Forms

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

Shorthand:

```js
{ field: value } === { field: { $eq: value } }
```

---

### 4.3 Predicate Semantics

* `$eq`: field exists; identity comparison; dates by timestamp
* `$ne`: field exists; not equal
* `$gt/$gte/$lt/$lte`: natural ordering; dates by timestamp
* `$in`: matches any element
* `$nin`: missing-field behavior implementation-defined
* `$regex`: invalid patterns never throw
* `$exists`: presence test

---

## 5. Query Options

```ts
QueryOptions := {
  limit?: number        // default: 1000
  skip?: number         // default: 0
  sort?: { [field]: 1 | -1 }
  fields?: { [field]: boolean }
}
```

* `id` included by default if available

---

## 6. Adapter Extensions

* Adapter-specific functions allowed
* Underlying client exposed via `adapter.data`

---

## 7. Non-Goals

* Transactions
* Schemas
* Concurrency guarantees
* Storage assumptions

---

**End of RFC**
