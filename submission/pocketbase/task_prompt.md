Add an opt-in soft delete mode to "base" collections: deleting a record moves it to a restorable trash.

Add per-collection `softDelete` (boolean) and `trashRetention` (seconds; 0 keeps indefinitely). Retention needs `softDelete`. When enabled, collections add a system date field `deleted` holding the trash timestamp, and lose it when disabled. Client input for it is ignored, but it appears in API outputs (empty while live); an ordinary `deleted` field on a collection without the mode stays writable. The option appears in the serialized options only while enabled. Refuse to enable if a `deleted` field of any kind, system or not, already exists. Toggling either way keeps every record, and with the field gone the trashed ones read as live.

Truncating a collection still empties it, trashed rows included. `App.Delete` on a trashed record permanently removes it. `App.PurgeRecord` permanently deletes a live record. Both cascade recursively to referencing records, already trashed ones included.

Trashed records are excluded from normal reads, relations, back-relations, `@collection` filters, and expands. Relation filters must exclude trashed targets, written as `rel` or `rel.id`. Access them via `App.FindRecordByIdWithTrashed`, `App.FindTrashedRecordById`, and `App.FindAllTrashedRecords` (the last two error without `softDelete`). Records answer `IsTrashed` and `TrashedAt`, and saving a record the trash holds fails validation.

With `softDelete` enabled, unique indexes exclude trashed rows while keeping index definitions unchanged; disabling gives the earlier expressions back.

Cascade-delete dependents are trashed recursively. Trashing leaves stored relation values alone, cascading or not, so restoring the target makes its prior matches visible again. A multi-value relation follows only if the trashed record was its last visible target. Every record one delete trashes shares that delete's timestamp; separate deletes never share one. `App.RestoreRecord` accepts any member of such a group, whichever end it is given, finds the others by that timestamp, and restores them together in any dependency order, each coming back live in place. A record cannot come back while one it points at through a cascading relation sits in the trash outside that group; trashed records pointing at it never hold it back. Restoring an untrashed record does nothing; without `softDelete` it errors.

`App.PurgeTrashedRecords` clears records trashed before a `types.DateTime` cutoff (zero purges all) and reports how many went; without `softDelete` it errors. `App.PurgeExpiredTrashedRecords` applies each retention, skipping collections without one, and returns only an error.

List and view take `trashed=with` for live and trashed records, or `trashed=only` for just those. Any other value, an empty one included, returns 400; without `softDelete`, ignore it.

`DELETE /api/collections/{collection}/records/{id}` takes a boolean `purge` (`true` permanently deletes; anything else, empty included, returns 400). Without `purge`, trashed IDs return 404.

`POST /api/collections/{collection}/records/{id}/restore` uses the update rule and returns the restored record (403 if rule is nil, 404 if rule fails or untrashed, 400 without `softDelete`). The batch endpoint allows it too.

`DELETE /api/collections/{collection}/trash` requires superuser access (401 for guests), takes an optional `before` datetime, rejecting an unparsable or empty one with 400 rather than reading it as absent, returns the count under `purged`, and errors with 400 without `softDelete`.
