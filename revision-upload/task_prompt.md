Add an opt-in soft delete mode to "base" collections: deleting a record moves it to a restorable trash.

Add per-collection `softDelete` (boolean) and `trashRetention` (seconds; 0 keeps indefinitely). Retention without `softDelete` is invalid. When enabled, collections add a system date field `deleted` holding the trash timestamp, and lose it when disabled. Client input for that field is ignored, but it appears in API outputs (empty while live); the same name on a collection without the mode stays an ordinary writable field. The option appears in serialized options only while enabled. Refuse to enable if a `deleted` field of any kind, system or not, already exists. Toggling keeps existing records, and removing the field makes trashed records read as live.

`App.Delete` on a trashed record permanently removes it. `App.PurgeRecord` permanently deletes a live record. Both cascade permanently and recursively to referencing records, trashed ones included.

Trashed records are excluded from normal reads, relations, back-relations, `@collection` filters, and relation expands. Relation filters must exclude trashed targets, written as `rel` or `rel.id`. `App.FindRecordByIdWithTrashed` looks past that exclusion, while `App.FindTrashedRecordById` and `App.FindAllTrashedRecords` return only the trashed ones. Records answer `IsTrashed` and `TrashedAt`, and saving a trashed record fails validation.

With `softDelete` enabled, unique indexes exclude trashed rows while keeping index definitions unchanged; disabling restores original expressions.

Cascade-delete dependents are trashed recursively. Trashing leaves stored relation values intact, so restoring a target makes prior matches visible again. A multi-value relation cascades only if the trashed record was its last visible target.

Records trashed by the same delete share a timestamp; separate deletes use distinct timestamps. `App.RestoreRecord` accepts any member of such a group, finds the others by that timestamp, and restores them together in any dependency order. A record cannot be restored while a target of a cascading relation remains in trash outside its group; trashed dependents do not block restoration. Calling it on an untrashed record is a no-op.

`App.PurgeTrashedRecords` clears records trashed before a `types.DateTime` cutoff (zero value purges all) and returns the purged count. `App.PurgeExpiredTrashedRecords` applies collection retentions, skipping collections without one, and reports an error and no count.

A collection without the mode has no trash to reach: `App.FindTrashedRecordById`, `App.FindAllTrashedRecords`, `App.PurgeTrashedRecords` and `App.RestoreRecord` error on one, and the restore and empty-trash routes answer 400.

List and view accept `trashed=with` (live and trashed) or `trashed=only` (trashed only). Other values return 400; ignore the parameter on collections without `softDelete`.

`DELETE /api/collections/{collection}/records/{id}` takes boolean `purge` (`true` permanently deletes; non-boolean returns 400). Without `purge`, trashed IDs return 404.

`POST /api/collections/{collection}/records/{id}/restore` uses the collection update rule and returns the restored record (403 if rule is nil, 404 if rule fails or untrashed). Add this route to the batch endpoint's allowed actions as well.

`DELETE /api/collections/{collection}/trash` requires superuser access (401 for guests), accepts an optional `before` datetime, rejecting an unparsable one with 400 rather than reading it as absent, returns the count under `purged`.
