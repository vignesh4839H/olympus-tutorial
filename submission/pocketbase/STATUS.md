# PocketBase submission — complete

## Deliverables

| Field | Value |
| --- | --- |
| Repository | `https://github.com/pocketbase/pocketbase` |
| Commit | `5684ee24f1e88f71a2ede73d20cf547fbd509e16` |
| Language | Go |
| Category | Feature request |
| Title | Opt-in soft delete with trash and restore for base collections |
| Task prompt | `task_prompt.md` |
| Test patch | `test.patch` |
| Solution patch | `solution.patch` |
| Dockerfile | `Dockerfile` |
| Platform check | **eligible · MIT**, no repository reuse warning |

## Measured results

- **738 meaningful production LOC** added by the solution patch (blank lines,
  comments, imports, braces and test files excluded).
- **100 graded test cases** across `core` and `apis`.
- Verified in a clean checkout of the pinned commit:
  - test patch only -> `./test.sh base` **passes**, `./test.sh new` **fails**
  - test + solution -> `./test.sh base` **passes**, `./test.sh new` **passes**
- Full `go test ./...` after the solution: everything green except
  `TestRecordAuthWithOAuth2`, which already fails at the pinned commit because
  it needs outbound network.

## Prior art (found by the Scope Gate)

No soft delete, trash or deleted timestamp exists in the code at the pinned commit,
but prior art does exist:

- closed PR #7462 implements a collection toggle, timestamp stamping, default query
  hiding and an include-deleted query
- issue #2866 has the maintainer deferring generalized soft delete "until a more
  prominent and clear use case arise"

The Scope Gate passed the submission anyway, rating #7462 at most 1 of 30 graded
functions and reading #2866 as a hedged deferral rather than a decline. Earlier
notes in this repo wrongly stated that no such PR or issue existed.

## Feature

Base collections get a `softDelete` option. When it is on, deleting a record
stamps a `deleted` system field instead of removing the row, the record
disappears from every read path (queries, finders, list/view endpoints,
relation and back-relation filters, expand), and it can be restored later.

The difficulty lives in the interactions:

- `RecordQuery` is the base of every read path, so the exclusion has to be
  correct there and opt-out-able for the trash views.
- A filter naming a relation compares the stored foreign key without touching
  the related table, so both `rel` and `rel.id` have to be rerouted through the
  filtered join once the target collection can hide rows.
- Unique indexes have to become partial while the mode is on and revert exactly
  when it is off, otherwise a trashed row blocks its own replacement.
- Trashing cascades through `CascadeDelete` relations and a restore has to
  bring back exactly that group, which requires the group timestamp to be
  unique per delete operation.
- A restore group is a dependency graph, not a list: one member can
  cascade-depend on another, so the group has to be collected and validated as
  a whole before anything is written. Validating each member as it is reached
  makes the visit order observable and can leave a valid group permanently
  unrestorable.
- Permanent deletion keeps the existing cascade, including through records that
  are themselves in the trash.
- Collections that do not enable the mode must serialize byte-identically to
  before, which the repository's own `TestCollectionDBExport` enforces.

## Files touched by the solution

18 files, 2576 diff lines:

`core/record_trash.go` (new), `core/record_query.go`, `core/collection_model.go`,
`core/record_model.go`, `core/collection_validate.go`,
`core/collection_model_base_options.go`, `core/collection_record_table_sync.go`,
`core/record_field_resolver.go`, `core/record_field_resolver_runner.go`,
`core/record_query_expand.go`, `core/app.go`, `core/base.go`, `core/field.go`,
`apis/record_crud.go`, `apis/realtime.go`, `apis/batch.go`,
`forms/record_upsert.go`, `plugins/jsvm/internal/types/generated/types.d.ts`.
Test files live in the test patch.

### The JSVM declaration file

`make jstypes` rewrites `types.d.ts` with randomized type-alias names, a
unix-timestamp header and a different namespace order on every run: a straight
regeneration produced 13878 changed lines of which only about 90 were soft
delete related. The file in the patch is therefore the generator's output for
the new declarations only, spliced into the committed file so the diff is the
245 lines that actually describe the feature (the collection options and
`isSoftDeleteEnabled`, `Record.isTrashed` / `trashedAt`, and the new app query,
restore and purge methods) and nothing else.

## Sibling directory

`../rejected-yq-plist/` holds the earlier yq property list submission. It passed
every precheck but was **rejected at the Scope Gate as publicly-solved**:
`DHowett/go-plist` implements the same codec. Kept as a record of what the gate
rejects and why. This feature has no such public equivalent — the hard part is
PocketBase's own collection, record, query and cascade machinery.
