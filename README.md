# dbspec v1.0

A backend-agnostic query and mutation specification for document-oriented storage.

This document is **normative**.
All behavior described is required unless explicitly marked **implementation-defined**.

---

## 1. Purpose and Scope

dbspec defines a **portable, minimal, stable interface** for document databases.

Goals:

* Single query language
* Stable return shapes
* No semantic overloading
* No mode flags
* No feature creep
* Adapter-native execution where possible
* Explicit adapter responsibility where not

dbspec defines **semantics**, not execution strategy.

---

## 2. Data Model

### 2.1 Document

A **document** is a JSON object:

* Must be an object (`typeof === "object"`, not `null`)
* Must contain an `id: string`
* Field values may be:

  * JSON primitives
  * Arrays
  * Objects
  * `Date` objects

---

### 2.2 Identity and Nullability

* Missing field ≠ `null`
* `{ a: null }` and `{}` are distinct
* Equality, comparison, and updates MUST preserve this distinction

---

## 3. Logical Database Model

dbspec operates on **exactly one logical collection at a time**.

Collection resolution is **host-defined**.

Supported models:

```js
db.all(...)
db.get(...)
db.set(...)
```

or

```js
db(name).all(...)
db(name).get(...)
db(name).set(...)
```

---

## 4. Core Collection Interface

The **entire portable surface** consists of four methods:

```ts
all(query, options?) -> Document[]
get(query) -> Document | null
count(query) -> number
set(document | document[] | query, values?) -> { n: number }
```

No other core methods exist.

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
* No implicit dot-notation

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

* `$eq`
  Field must exist; values compared by identity
  Dates compare by timestamp

* `$ne`
  Field must exist; value must not equal operand

* `$gt`, `$gte`, `$lt`, `$lte`
  Field must exist; natural ordering
  Dates compare by timestamp

* `$in`
  Field must exist; matches any element

* `$nin`
  Field must exist; matches none
  Missing-field behavior is **implementation-defined**

* `$regex`
  Field must be string
  Invalid patterns never throw; evaluate as non-matching

* `$exists`
  `true` → field present
  `false` → field missing

---

### 5.4 Logical Operators

```ts
{ $and: Query[] }
{ $or: Query[] }
{ $not: Query }
```

* Operators may appear at any level
* Semantics are strictly logical

---

## 6. Options Object (for `all` only)

```ts
Options := {
  limit?: number        // default: 1000
  skip?: number         // default: 0
  sort?: { [field]: 1 | -1 }
  fields?: { [field]: boolean }
}
```

### 6.1 limit / skip

* Applied after query evaluation
* `limit` caps total returned documents

---

### 6.2 sort

* Field → direction
* Missing-field behavior is **implementation-defined**
* Unsupported sorts MAY fail

---

### 6.3 fields (Projection)

* `true` includes field
* `false` excludes field
* Any `true` → inclusive projection
* `id` is included by default
* Excluding `id` is **best-effort**
* Consumers requiring strict exclusion MUST post-process

---

## 7. Read Semantics

### 7.1 `all(query, options?)`

* Returns zero or more documents
* Always returns an array
* Empty result → `[]`

---

### 7.2 `get(query)`

* Returns the **first matching document**
* Returns `null` if none
* Never returns an array
* No options object
* No flags

---

### 7.3 `count(query)`

* Returns number of matching documents
* No document materialization required
* Adapter MAY iterate internally if backend lacks native count

---

## 8. Write Semantics (`set`)

`set` is the **only mutation primitive**.

### 8.1 Forms

```js
set(document)
set(document[])
set(query, values)
set(query, null)
```

---

### 8.2 General Rules

* `set` always **upserts**
* Shallow merge
* Returns `{ n }` = number of documents changed
* Atomicity is **per document**

---

### 8.3 Insert / Upsert

```js
set(document)
```

* Inserts document
* If `id` exists → replaces document
* If `id` missing → host generates one

```js
set(document[])
```

* Each document processed independently
* Per-document atomicity

---

### 8.4 Update / Upsert by Query

```js
set(query, values)
```

* All matching documents are updated
* If no documents match:

  * A new document is created
  * Equality fields from `query` and `values` are merged

#### Field rules

* `undefined` → field is **removed**
* `null` → field is stored as `null`

---

### 8.5 Delete

```js
set(query, null)
```

* Deletes all matching documents
* `{}` deletes all documents in the collection
* Returns `{ n }`

---

## 9. Adapter Responsibilities

Adapters MUST:

* Preserve all semantic distinctions
* Use backend-native operations where possible
* Perform fetch-merge-write where backend lacks partial updates
* Never expose backend artifacts (`_rev`, internal IDs)

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
