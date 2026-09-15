Add an opt-in soft delete mode to "base" collections: deleting a record moves it to a restorable trash.

Add per-collection `softDelete` (boolean) and `trashRetention` (seconds; 0 keeps indefinitely). Setting retention with `softDelete` disabled is invalid. When enabled, collections add a system date field `deleted` holding the trash timestamp, and lose it when disabled. Client input for `deleted` is ignored, but the field appears in API outputs (empty while live). The option appears in the serialized options only while enabled. Refuse to enable if a custom `deleted` field already exists. Toggling either way keeps every existing record, and with the field gone the trashed ones read as live again.

`App.Delete` on a trashed record permanently removes it. `App.PurgeRecord` permanently deletes a live record. Both recursively cascade to referencing records, already trashed ones included.

Trashed records are excluded from normal reads, relations, back-relations, `@collection` filters, and relation expands. A filter naming a relation misses a trashed target however it is spelled, `rel` as well as `rel.id`. Access them via `App.FindRecordByIdWithTrashed`, `App.FindTrashedRecordById`, and `App.FindAllTrashedRecords` (the last two error without `softDelete`). Records answer `IsTrashed` and `TrashedAt`, and saving a trashed record fails validation.

With `softDelete` enabled, unique indexes exclude trashed rows while keeping index definitions unchanged; disabling gives the earlier expressions back.

Cascade-delete relations follow soft deletes recursively, though trashing preserves stored references, cascading or not, so a restore brings the old matches back. A multi-value relation follows only if the trashed record was its last visible target. Trashed records from a single delete share a timestamp and separate deletes never share one; `App.RestoreRecord` gathers that group from the record it is given and restores all of it, whatever order its members depend on each other in. Restoring an untrashed record does nothing, restoring without `softDelete` errors, and only a cascade dependency left trashed outside that group blocks a restore. The restored record comes back live in place, ready to change and save.

`App.PurgeTrashedRecords` clears records trashed before a `types.DateTime` cutoff (zero value purges all) and reports how many went; a collection without `softDelete` errors. `App.PurgeExpiredTrashedRecords` applies collection retentions, skipping those without one.

List and view endpoints accept `trashed=with` (live and trashed) or `trashed=only` (trashed only); other values return 400. Without `softDelete`, the parameter has no effect. The batch endpoint supports restore actions.

`DELETE /api/collections/{collection}/records/{id}` takes a boolean `purge` (`true` permanently deletes; non-boolean returns 400). Without `purge`, trashed IDs return 404.

`POST /api/collections/{collection}/records/{id}/restore` uses the collection update rule and returns the restored record (403 if rule is nil, 404 if rule fails or untrashed, 400 without `softDelete`).

`DELETE /api/collections/{collection}/trash` requires superuser access (401 for guests), accepts an optional `before` datetime (400 if malformed), returns the count under `purged`, and errors with 400 without `softDelete`.
