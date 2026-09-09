# Olympus submission — property list support for yq

Everything in this folder is ready to paste into the submission form.

## 1. Repository

| Field | Value |
| --- | --- |
| Repository | `https://github.com/mikefarah/yq` |
| Commit | `8b5af0694bb82b41d4ae180fac9972029066f90a` |
| Commit date | 2026-08-25 |
| Language | Go |
| Category | Feature request |

### Why it qualifies

- Public repository, MIT licensed (permissive).
- 15,937 stars, well past the 500 floor.
- Actively maintained; the pinned commit is from August 2026 and the project is pushed to weekly.
- Production grade: the `yq` binary is a widely deployed CLI with a layered codebase — CLI (`cmd/`), an expression lexer/parser, ~90 operators, and fifteen pluggable encoder/decoder pairs behind a format registry.
- Established test suite: `go test ./...` runs 496 tests in about six seconds.
- Deterministic and offline: the unit suites need no network, no credentials and no external services. The integration suite that needs a real binary is gated behind `INT=1` and is not used here.

## 2. Title

> Read and write Apple property lists, both XML and binary

## 3. Task prompt

See [`task_prompt.md`](./task_prompt.md). 259 words, plain prose, no URLs, pure ASCII.

## 4. Dockerfile

See [`Dockerfile`](./Dockerfile). Starts from the Go base image, `WORKDIR /app`, downloads all
modules and pre-builds during the build phase, runs no tests during the build, and ends with
`CMD ["/bin/bash"]`. It builds from a clean checkout — neither patch is needed.

## 5. Test patch

See [`test.patch`](./test.patch). It adds three files and touches no production code:

| File | Purpose |
| --- | --- |
| `test.sh` | the `base` / `new` harness |
| `test/formats/property_list_test.go` | 35 hidden tests |
| `scripts/junitreport/main.go` | converts `go test -json` into JUnit XML |

The tests live in their own package (`test/formats`) so that `base` can run every existing suite
without picking them up. They only use exported API — `FormatFromString`, the encoder/decoder
interfaces and `CandidateNode` — so they test the observable contract rather than the reference
implementation. Binary fixtures were produced with Python's `plistlib` and are embedded as base64.

`test.sh` modes:

- `base` → `go test . ./cmd/... ./pkg/... ./test` (every existing suite, property list suite excluded)
- `new` → `go test ./test/formats/...`

## 6. Solution patch

See [`solution.patch`](./solution.patch). Six files under `pkg/yqlib`, no test files touched:

| File | Meaningful LOC |
| --- | --- |
| `decoder_plist.go` (new) | 201 |
| `decoder_plist_binary.go` (new) | 245 |
| `encoder_plist.go` (new) | 100 |
| `encoder_plist_binary.go` (new) | 207 |
| `format.go` (modified) | 8 |
| `no_plist.go` (new) | 3 |
| **Total** | **764** |

Counted from the added lines of the diff, excluding blank lines, comments, `package` lines,
import blocks and their contents, brace- and punctuation-only lines, and trivial statements.

The change follows the repo's own conventions: the format registry entry mirrors the existing
formats, the encoder/decoder files carry the `!yq_noplist` build tag, and a `no_plist.go` stub
keeps the registry compiling when the tag is set — the same shape as `no_toml.go`, `no_ini.go`
and the rest.

## 7. Verification

Every result below was produced by actually running the commands.

### Test contract

| State | `base` | `new` |
| --- | --- | --- |
| clean checkout + `test.patch` | **PASS** — 496 tests, 0 failures | **FAIL** — 35 tests, 35 failures |
| clean checkout + `test.patch` + `solution.patch` | **PASS** — 496 tests, 0 failures | **PASS** — 35 tests, 0 failures |

All 35 new tests fail on the base commit and all 35 pass with the solution.

### Determinism

- `new` run six times without the solution: 35 failures every run.
- `new` run six times with the solution: exit 0 every run.
- `base` run three times in both states: exit 0 every run.
- `go test -race ./test/formats/...`: clean.

### Quality

- `gofmt -l`: clean.
- `go vet ./...`: clean.
- `go build ./...`: OK.
- `go build -tags yq_noplist ./...`: OK.
- Both patches apply cleanly to the pinned commit with `git apply --check`.

### Independent correctness check

The encoders were validated against CPython's `plistlib`, which is a separate implementation of
the same format:

- yq's XML output is parsed by `plistlib` and compares equal to the original object.
- yq's binary output is parsed by `plistlib` and compares equal to the original object.
- A 400-key document with dates, long strings and blobs written by `plistlib` round-trips through
  yq's binary encoder **byte for byte identically** (11,745 bytes in, 11,745 bytes out).

### Offline / Docker

Simulated with a cold module cache: `go mod download` plus `go build ./...` during the build
phase is enough for both test modes to then run with `GOPROXY=off`, i.e. with no network.

## 8. Known risks

1. **The Docker image was not built.** This environment's egress policy blocks every container
   registry blob CDN (403 on `public.ecr.aws` and `docker.io` alike), so the image could not be
   pulled or built here. The Dockerfile was instead validated by reproducing its steps natively
   against a cold module cache. Run **Build Image** on the platform first and check it before
   spending anything on the later steps.
2. **`go.mod` requires Go 1.25.** If the base image ships something older, the build relies on the
   go command fetching the newer toolchain; `GOTOOLCHAIN=go1.25.0` pins it to the exact version
   go.mod names, so this happens during the build while there is still a network. If the build
   fails on toolchain resolution, that env line is the thing to look at.
3. **Difficulty is unmeasured.** Reading and writing the binary flavour is the hard half and no
   agent has attempted this yet. Run a **Quick Check** before committing to a full batch: if it
   comes back solved easily the task is too easy, and if it comes back blocked on something
   ambiguous rather than something hard, the prompt needs a sentence.

## 9. Precheck results

Run on 2026-09-09. Everything passed; three warnings came back and all three have
been addressed.

| Group | Result |
| --- | --- |
| GitHub Repository | 1/1 |
| Problem Description & Tests | 12/12 |
| Plagiarism Review | 1/1 — not a duplicate |
| Dockerfile | 2/2 |
| Solution Patch | 1/1 |

The repository carries a "used in 39 submissions by 12 other contributors" notice, but
the plagiarism and duplicate checks came back clean, which is the check that actually
decides novelty.

### Warnings and what was done

1. **Dockerfile — "version pinning cannot be verified."** yq does commit both `go.mod`
   and a 90 line `go.sum`, and there is no vendor directory, so the dependencies were
   already pinned; the checker simply could not confirm that from the Dockerfile alone.
   Added an explicit `RUN go mod verify`, which checks every downloaded module against
   the committed hashes and fails the build on a mismatch. Verified locally against a
   cold module cache: "all modules verified".

2. **Description — "contains only necessary information."** Both suggestions taken.
   Dropped "both usable for input and for output" (the following sentence already covers
   reading with either name, and the third paragraph already covers writing with each)
   and dropped the filler "on its own".

3. **Problem and tests — coverage gap and over-specification.** Both taken.
   - Added `TestPropertyListNamesAcceptEitherFlavour`, which reads an XML document
     through the `bplist` name and a binary document through the `plist` name. This was
     a real gap: the old suite only proved binary could be read through either name.
   - The exact-output tests pin dictionary key order, which the description had not
     stated. Rather than weaken the tests, the requirement is now stated: dictionary
     entries are written "in the order they are held rather than sorted". yq is
     order-preserving throughout, so this is the repo's own behaviour, and an
     implementation that sorts keys is now failing a stated requirement rather than an
     unstated one.
   - Also replaced "reads back unchanged" with "reads back with the same values" to
     settle the byte-identical versus value-identical ambiguity the reviewer raised.
     The tests check values, not bytes, so that no particular binary object layout is
     forced on the solver.

`solution.patch` was not touched by any of this, so the meaningful LOC count is still 764.

## 10. Second precheck round

Two warnings came back, both addressed.

1. **Dockerfile — `GOTOOLCHAIN=auto` hurts reproducibility.** Fair point: `auto` lets the
   toolchain drift over time. Changed to `GOTOOLCHAIN=go1.25.0`, the exact version
   `go.mod` names. Verified against a cold cache: it fetches precisely go1.25.0 during
   the build, `go mod verify` reports all modules verified, and both test modes then run
   offline.

2. **Description — "contains only necessary information" (verdict `request_changes`).**
   Worth noting this one carried no high-severity items, and the check's own text says
   only high severity is blocking, so it would not have stopped the submission. All five
   suggestions were taken anyway, cutting the prompt from 319 to 259 words:
   - dropped "Dictionaries become maps and arrays become sequences"
   - dropped the basic scalar mappings, keeping the non-obvious ones the reviewer asked
     to keep (data to `!!binary`, date to `!!timestamp` as RFC 3339 UTC)
   - replaced the enumerated XML error cases with a general rule that still names the
     binary misalignment case: "whether the document is malformed XML, structurally
     invalid, or a binary document whose header, root reference or offset table does not
     line up"
   - dropped "wherever the existing formats are"
   - dropped "and text escaped as XML requires"

### Residual fairness risk from that trim

Five behaviours the hidden tests assert are no longer stated outright and now rest on
being discoverable from the repository:

| Assertion | Now rests on |
| --- | --- |
| dict to mapping node, array to sequence node | the only sensible mapping, and what every other yq decoder does |
| `!!str` / `!!int` / `!!float` / `!!bool` tags | `createScalarNode` in the repo already maps these types this way |
| eight specific XML error cases | the general "not a well formed property list" rule |
| both names registered for input and output | "the formats yq can read and write" plus the writing paragraph |
| XML text escaping | an unescaped ampersand is not a well formed XML document |

Each is defensible, but this is the trade the trim buys: a shorter prompt against slightly
more reliance on inference. If a rollout later fails an agent on one of these rather than
on the genuinely hard part (the binary flavour), put the relevant sentence back — the
length check passes comfortably at either size.

`solution.patch` and `test.patch` were not touched this round. Meaningful LOC is still 764,
and the contract still holds: 35 new tests fail on the clean commit, all pass with the solution.
