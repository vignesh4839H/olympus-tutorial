# PocketBase submission — in progress

## Repository

| Field | Value |
| --- | --- |
| Repository | `https://github.com/pocketbase/pocketbase` |
| Commit | `5684ee24f1e88f71a2ede73d20cf547fbd509e16` |
| Commit date | 2026-09-06 |
| Language | Go |
| Category | Feature request |
| Platform check | **eligible · MIT**, no repository reuse warning |

### Why it qualifies (all verified by running the commands)

- MIT licensed, ~45k stars, commit within days of pinning.
- **No cgo** — the SQLite driver is the pure Go `modernc.org/sqlite`, so `CGO_ENABLED=0`
  builds cleanly and there is no system library to install.
- 19 MB checkout, `go 1.27`.
- Multi-subsystem, which is what the feature needs: `core` 22,371 non-test LOC,
  `tools` 17,396, `apis` 8,722, plus `forms`, `migrations`, `plugins`, `cmd`, and
  `tools/{cron,search,subscriptions,filesystem,auth,hook,router}`.
- Tests are deterministic and offline: `core`, `forms` and `tools/search` run in 25s
  with `GOPROXY=off`.

## Chosen feature

**Opt-in soft delete with trash and restore for records.**

Verified genuinely absent at this commit — zero files mention soft delete, trash or a
deleted timestamp, and there is no issue, no pull request and no maintainer decline.
Already present, and therefore ruled out: rate limiting, MFA, OTP, batch API,
impersonation, backups.

The codebase has two extension points that make this the natural shape:

- `collectionBaseOptions` is an empty struct — a place for base collection options that
  the maintainers created and never filled.
- `initDefaultFields()` injects system fields per collection type (`initIdField`,
  `initPasswordField`, ...), so a `deleted` system field picks up the column creation,
  the filter/sort resolver and the JSON serialisation that already exist.

### Why it is hard

The trash flag has to be honoured by `RecordQuery`, which every read path in the
codebase is built on: `FindRecordById`, `FindRecordsByFilter`, relation expansion, the
list endpoint and auth record lookups. Cascade behaviour, unique constraints and
backwards compatibility for collections that have not enabled the option all have to
agree. That breadth is the difficulty, and it is also why it cannot be written quickly.

## Status

| Artifact | State |
| --- | --- |
| Repository + commit | **done** — eligible on the platform |
| Dockerfile | **done and verified** (see below) |
| Title + task prompt | drafted, to be finalised against the tests so the two cannot drift |
| Test patch | not started |
| Solution patch | not started |

### Dockerfile verification

Run against a cold module cache:

- `go mod download` + `go mod verify` -> "all modules verified"
- `go build ./...` -> OK, 75s total, `CGO_ENABLED=0`
- tests then run with `GOPROXY=off` in 25s, confirming the container works offline

## Next steps

Build in this order, running the suite at each stage:

1. `collectionBaseOptions.SoftDelete` + validation
2. `initDeletedField()` — the system field carrying the trash timestamp
3. Delete path: mark instead of remove, plus purge
4. Query layer: exclude trashed by default
5. API: restore, purge, include-trashed
6. Cascade semantics

Meaningful LOC gets measured after step 3 and reported before going further, so the
600+ requirement is confirmed with a real number rather than an estimate.

## Sibling directory

`../rejected-yq-plist/` holds the earlier yq property list submission. It passed every
precheck but was **rejected at the Scope Gate as publicly-solved**: `DHowett/go-plist`
implements the same XML and binary codec and would have covered 14 of 21 graded tests.
Kept as a record of what the gate rejects and why.
