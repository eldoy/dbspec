# dbspec v1.1

A backend-agnostic query and mutation specification for document-oriented storage.

This document is **normative**.
All behavior described is required unless explicitly marked **optional** or **adapter-specific**.

---

## 1. Data Model

### 1.1 Document

A **document** is a JSON object with the following properties:

* Must be an object (`typeof === "object"`, not `null`)
* Must contain an `id` field of type `string`
* Field values may be:

  * JSON primitives (`string`, `number`, `boolean`, `null`)
  * Arrays
  * Objects
  * `Date` objects

### 1.2 Identity and Nullability

* A **missing field** is not equivalent to `null`
* `{ a: null }` and `{}` are distinct
* Field comparison must preserve this distinction

---

## 2. Logical Database Model

dbspec defines semantics over **exactly one logical document collection at a time**.

How that collection is selected is **host-defined**.

### 2.1 Collection Resolution (Normative)

A dbspec-compliant host MUST expose **at least one** of the following models:

#### Model A — Single-Collection Database

Used by in-memory, embedded, or simple stores.

```js
db.get(...)
db.set(...)
```

All operations implicitly target the single collection.

#### Model B — Multi-Collection Database

Used by CouchDB-like or namespaced backends.

```js
db(name).get(...)
db(name).set(...)
```

* `db(name)` resolves a **collection handle**
* The returned object implements the full dbspec interface
* Each collection is an independent document space

#### Model C — Hybrid (Allowed)

A host MAY support both:

* `db.get / db.set` → default collection
* `db(name).get / db(name).set` → named collections

This model is **explicitly allowed** and recommended for adapters.

---

## 3. Core Host Interface (Collection Scope)

The following methods are **required** on a resolved collection.

```ts
get(query, options?, onBatch?)
set(queryOrDoc, values?)
```

dbspec defines:

* Query structure
* Predicate semantics
* Mutation semantics
* Option behavior
* Streaming behavior

Invocation style is host-defined.

---

## 4. Query Semantics

### 4.1 Query Object Shape

```ts
Query := {
  [field: string]: Predicate | Value
}
```

Rules:

* `{}` matches all documents
* Multiple fields imply implicit logical AND
* Field names are literal keys
* No dot-notation is implied unless the host implements it

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

#### 4.2.1 Shorthand Equality

```js
{ field: value }
```

Equivalent to:

```js
{ field: { $eq: value } }
```

---

### 4.3 Predicate Evaluation Rules

(unchanged from v1.0 — omitted here for brevity, **identical semantics apply**)

---

### 4.4 Logical Operators

```ts
{ $and: Query[] }
{ $or: Query[] }
{ $not: Query }
```

Rules unchanged from v1.0.

---

## 5. Options Object

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

Semantics unchanged from v1.0.

---

## 6. Streaming Mode

Activated when `onBatch` is provided.

```js
get(query, options, onBatch)
```

Rules unchanged from v1.0.

---

## 7. Mutation Semantics (`set`)

### 7.1 Insert

```js
set(document)
```

Behavior unchanged from v1.0.

---

### 7.2 Update

```js
set(query, values)
```

Behavior unchanged from v1.0.

---

### 7.3 Delete

```js
set(query, null)
```

Behavior unchanged from v1.0.

---

### 7.4 Clear

```js
set({}, null)
```

Deletes all documents in the **current collection**.

---

### 7.5 Bulk Insert

```js
set(documents[])
```

Behavior unchanged from v1.0.

---

## 8. Evaluation Guarantees

Unchanged from v1.0.

---

## 9. Adapter-Specific Extensions (Non-Normative but Allowed)

Adapters MAY expose additional methods **outside the dbspec core**, provided:

* They do not alter dbspec semantics
* They do not change the meaning of `get` or `set`
* Their behavior is explicit and documented

Common allowed extensions include:

### 9.1 Lifecycle / Maintenance

```js
db.drop()          // drop all collections or default collection
db(name).drop()   // drop a specific collection
db.compact(name)  // backend compaction
db.info()         // backend metadata
```

These operations are **out of scope** for dbspec and have no required semantics.

### 9.2 Version / Capabilities

```js
db.version        // adapter version (string)
db.capabilities   // informational flags
```

These are informational only.

---

## 10. Escape Hatches

Backends MAY expose raw access:

```js
db.data
```

dbspec imposes no constraints on this interface.

---

## 11. Versioning

```ts
dbspec.version = "1.1"
```

Backward-incompatible changes require a major version increment.

---

**End of dbspec v1.1**
