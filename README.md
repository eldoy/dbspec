Below is a **complete, minimal, implementation-grade `dblang` specification**.

No prose, no rationale—only contract.

---

# dblang v1.0

A backend-agnostic query and mutation language for document-oriented storage.

---

## 1. Data Model

* Document: JSON object
* Required field: `id` (string)
* Field values: JSON primitives, arrays, objects, Date
* Missing field ≠ `null`

---

## 2. Core Operations

dblang is consumed only via host methods:

* `get(query, options?, onBatch?)`
* `set(queryOrDoc, values?)`

dblang defines **query**, **mutation**, and **options** only.

---

## 3. Query Object

### 3.1 Shape

```ts
Query := {
  [field: string]: Predicate | Value
}
```

Empty query `{}` matches all documents.

---

### 3.2 Predicates

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
{ field: value } ≡ { field: { $eq: value } }
```

---

### 3.3 Logical Operators

```ts
{ $and: Query[] }
{ $or: Query[] }
{ $not: Query }
```

Top-level implicit AND applies when multiple fields are present.

---

## 4. Options Object

```ts
Options := {
  limit?: number        // default: 1000
  skip?: number         // default: 0
  sort?: { [field]: 1 | -1 }
  fields?: { [field]: boolean } // projection
  count?: boolean       // return count only
  batch?: number        // batch size for streaming
}
```

Rules:

* `count: true` suppresses document materialization
* `batch` activates streaming mode
* `limit` applies per call, not per batch

---

## 5. Streaming Mode

Activated when `onBatch` is provided.

```js
get(query, options, onBatch)
```

Contract:

* `onBatch(docs[])` called sequentially
* `docs.length <= batch`
* No full result buffering
* Return value is `undefined`

---

## 6. Mutation Semantics (`set`)

### 6.1 Insert

```js
set(document)
```

* Inserts document
* `id` auto-generated if missing
* Returns inserted document

---

### 6.2 Update

```js
set(query, values)
```

* Updates all matching documents
* Shallow merge semantics
* `undefined` → unset field
* `null` → set field to null
* Returns `{ n: number }`

---

### 6.3 Delete

```js
set(query, null)
```

* Deletes all matching documents
* Returns `{ n: number }`

---

### 6.4 Clear

```js
set({}, null)
```

Deletes all documents.

---

## 7. Evaluation Rules

* All predicates are **pure**
* No user-defined functions in dblang
* Backend may:

  * Rewrite predicates
  * Push down filters
  * Short-circuit evaluation
* Semantics must be preserved

---

## 8. Backend Capability Levels

Adapters MAY expose:

```ts
capabilities = {
  indexed: boolean
  streaming: boolean
  countOptimized: boolean
}
```

dblang semantics MUST remain valid regardless.

---

## 9. Escape Hatch

Backends MAY expose raw client as:

```js
db.data
```

dblang imposes no constraints on `data`.

---

## 10. Versioning

```ts
dblang.version = "1.0"
```

Backward-incompatible changes require major version bump.

---

**End of specification.**
