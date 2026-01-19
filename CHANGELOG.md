### dbspec 1.0

* Initial release.
* Defined a portable core interface (`get` / `set`) for document-oriented storage.
* Established query predicates, logical operators, options, streaming, and mutation semantics.
* Introduced explicit adapter extensions and a non-portable escape hatch (`data`).
* Clarified that some behaviors (e.g. sorting on missing fields, `$nin` on missing fields) are implementation-defined to allow native backend execution.
