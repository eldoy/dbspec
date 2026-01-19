### dbspec 1.1

* Clarified collection scoping: hosts may expose single-collection (`db.get/set`), multi-collection (`db(name).get/set`), or hybrid models.
* Explicitly distinguished core dbspec semantics from adapter-specific extensions.
* Documented allowance for adapter lifecycle and maintenance functions (e.g. `drop`, `compact`, `info`, `version`).
* No changes to query, mutation, or evaluation semantics.

### dbspec 1.0

* Initial release.
* Defined core document model, query semantics, predicates, options, streaming behavior, and mutation semantics (`get` / `set`).
* Established backend-agnostic, normative specification for document-oriented storage.
