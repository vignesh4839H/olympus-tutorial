# Opt-in soft delete with trash and restore for base collections

Deleting a record in PocketBase removes its row immediately and there is no way
to undo it. Add an opt-in soft delete mode so that a "base" collection can keep
its deleted records in a trash and bring them back later.

The mode is off by default. A collection that does not enable it must behave
exactly as it does today, down to its serialized JSON and its exported db row —
no new keys, no new fields, no new indexes.

## The collection option

Add two options to the "base" collection type, stored in the collection
`options` JSON and settable through the collection API:

- `softDelete` (boolean) turns the mode on for the collection.
- `trashRetention` (integer, seconds) makes trashed records expire. The zero
  value keeps them forever. Setting a non-zero retention while `softDelete` is
  off is a validation error.

While the mode is on, the collection carries a `deleted` system date field
holding the moment the record was trashed; turning the mode off removes that
field again. Enabling the mode leaves the existing records untouched and
visible, and so does disabling it. The `deleted` value is managed internally
and must be ignored when it arrives in a record create or update request body.

If a collection already has a field named `deleted` that is not a date field,
enabling the mode is a validation error.

## Deleting

Deleting a record of a soft delete collection stamps its `deleted` field
instead of removing the row. The regular record delete hooks still run, and the
in-memory record reflects the new state afterwards.

Deleting a record that is *already* in the trash removes it permanently. There
is also an explicit permanent delete that skips the trash for a record that is
still live:

- `App.PurgeRecord(record)` / `App.PurgeRecordWithContext(ctx, record)`

Both permanent paths keep the existing cascade delete behaviour of the
referencing records, including the referencing records that are themselves in
the trash, and the records they cascade into are removed permanently too.

## The trash is invisible

A trashed record must not be reachable through the ordinary read paths: the
record queries and every finder built on them, the list and view endpoints, and
the relation, back-relation and `@collection` filters. Expanding a relation
that points at a trashed record yields nothing.

Trashed records are reachable only through explicit opt-ins:

- `App.RecordQueryWithTrashed(collection)` — live and trashed records
- `App.TrashedRecordQuery(collection)` — only trashed records (matches nothing
  for a collection without the mode)
- `App.FindRecordByIdWithTrashed(collection, id, ...optFilters)`
- `App.FindTrashedRecordById(collection, id, ...optFilters)`
- `App.FindAllTrashedRecords(collection, ...exprs)`

The last two report an error for a collection that does not have the mode
enabled.

A record has `Record.IsTrashed()` and `Record.TrashedAt()`, and a record that
is in the trash cannot be updated — a validating save must fail. Saves that
skip validation are the internal path and stay allowed.

## Unique indexes

A trashed row is still in the table, so it would keep reserving its unique
values and block the creation of a new record with the same data. While the
mode is on, the collection unique indexes must therefore ignore the trashed
rows, and turning the mode off must give back exactly the index expressions the
collection had before.

A consequence to get right: once a record is trashed another record may take
its unique value, and restoring the trashed one then has to fail.

## Cascade and restore

Trashing a record also moves to the trash the records that reference it through
a `CascadeDelete` relation, recursively. The relation values themselves are
left alone — the record is still there and may come back. A referencing record
with a multi-value relation follows the main record only when the trashed
record is its last remaining reference.

Every record trashed by a single delete shares one `deleted` timestamp, and two
separate delete operations must never end up with the same one. That group is
what a restore brings back:

- `App.RestoreRecord(record)` / `App.RestoreRecordWithContext(ctx, record)`

restores the record and exactly the records that went to the trash with it. A
record that was trashed on its own earlier stays in the trash. Restoring a
record that is not trashed is a no-op; restoring one from a collection without
the mode is an error. A record cannot be restored while a record it
cascade-depends on is still in the trash.

## Emptying the trash

- `App.PurgeTrashedRecords(collection, before)` permanently deletes the trashed
  records of the collection that were trashed before the given time, returning
  how many were purged. A zero time purges the whole trash. A collection
  without the mode is an error.
- `App.PurgeExpiredTrashedRecords()` applies each collection's
  `trashRetention`, skipping the collections that do not have one.

The expired records are purged periodically in the background.

## HTTP API

- `GET /api/collections/{collection}/records` and
  `GET /api/collections/{collection}/records/{id}` accept a `trashed` query
  parameter: `with` returns live and trashed records, `only` returns just the
  trashed ones, and anything else than those two or an empty value is a 400.
  For a collection without the mode the parameter is accepted and has no
  effect.
- `DELETE /api/collections/{collection}/records/{id}` accepts `purge`. A truthy
  value deletes the record permanently and also finds a record that is already
  in the trash; a non-boolean value is a 400.
- `POST /api/collections/{collection}/records/{id}/restore` restores a trashed
  record and responds with it, guarded by the collection update rule. A record
  that is not in the trash is a 404 and a collection without the mode is a 400.
- `DELETE /api/collections/{collection}/trash` empties the collection trash and
  responds with `{"purged": <count>}`. It accepts an optional `before` datetime
  query parameter to keep the more recently trashed records. Because it is a
  bulk irreversible operation it is restricted to superusers, and a collection
  without the mode is a 400.
