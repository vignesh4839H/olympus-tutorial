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

- **618 meaningful production LOC** added by the solution patch (blank lines,
  comments, imports, braces and test files excluded).
- **82 graded test cases** across `core` and `apis`.
- Verified in a clean checkout of the pinned commit:
  - test patch only -> `./test.sh base` **passes**, `./test.sh new` **fails**
  - test + solution -> `./test.sh base` **passes**, `./test.sh new` **passes**
- Full `go test ./...` after the solution: everything green except
  `TestRecordAuthWithOAuth2`, which already fails at the pinned commit because
  it needs outbound network.

## Feature

Base collections get a `softDelete` option. When it is on, deleting a record
stamps a `deleted` system field instead of removing the row, the record
disappears from every read path (queries, finders, list/view endpoints,
relation and back-relation filters, expand), and it can be restored later.

The difficulty lives in the interactions:

- `RecordQuery` is the base of every read path, so the exclusion has to be
  correct there and opt-out-able for the trash views.
- Unique indexes have to become partial while the mode is on and revert exactly
  when it is off, otherwise a trashed row blocks its own replacement.
- Trashing cascades through `CascadeDelete` relations and a restore has to
  bring back exactly that group, which requires the group timestamp to be
  unique per delete operation.
- Permanent deletion keeps the existing cascade, including through records that
  are themselves in the trash.
- Collections that do not enable the mode must serialize byte-identically to
  before, which the repository's own `TestCollectionDBExport` enforces.

## Files touched by the solution

`core/record_trash.go` (new), `core/record_query.go`, `core/collection_model.go`,
`core/record_model.go`, `core/collection_validate.go`,
`core/collection_model_base_options.go`, `core/record_field_resolver.go`,
`core/record_query_expand.go`, `core/app.go`, `core/base.go`, `core/field.go`,
`apis/record_crud.go`, `apis/realtime.go`, `apis/batch.go`,
`forms/record_upsert.go`, (test files live in the test patch).

## Sibling directory

`../rejected-yq-plist/` holds the earlier yq property list submission. It passed
every precheck but was **rejected at the Scope Gate as publicly-solved**:
`DHowett/go-plist` implements the same codec. Kept as a record of what the gate
rejects and why. This feature has no such public equivalent — the hard part is
PocketBase's own collection, record, query and cascade machinery.
