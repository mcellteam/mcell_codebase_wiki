# ✅ DONE — nutmeg `2000_shorter_error_message` fix (nanobind)

> **☑ RESOLVED (2026-10-14).** This was the one remaining item from the nanobind migration
> (companion to [`06_nanobind_migration_feasibility.md`](06_nanobind_migration_feasibility.md) §12).
> The fix is **applied** on the `mcell_tests` branch **`mcell4_nanobind`** (commit `3b0094ba`,
> pushed to `origin`) and verified with the real test runner on the nanobind build: **PASSED**.
>
> **Left for merge time:** when `mcell`'s `mcell4_nanobind` merges into `mcell4_dev`, merge
> `mcell_tests`' `mcell4_nanobind` into its `mcell4_dev` along with it — the two branches are a
> pair. Until then the fix must stay off `mcell_tests`' `mcell4_dev` (see next section).

**Branch context:** `mcell` on `mcell4_nanobind` (nanobind is the sole Python binding lib; pybind11
removed); `mcell_tests` on its matching `mcell4_nanobind` branch (remote
`git@github.com:mcellteam/mcell_tests.git`).

## Why the fix was deferred, and how that was resolved

`mcell_tests` originally had only the shared branch `mcell4_dev`, and the **pybind11 `mcell4_dev`
build is still actively tested** against it (Track A was verified on both builds). The correct
nanobind pattern does **not** match the pybind11 error text, so committing it to `mcell4_dev` would
flip this test PASS→FAIL on the pybind11 build. The fix was therefore only documented here, to be
applied at merge.

Resolved by giving `mcell_tests` its own **`mcell4_nanobind`** branch, paired with `mcell`'s. The
fix lives only there, so the pybind11 build (tested against `mcell_tests` `mcell4_dev`) is
unaffected.

## The test

```
mcell_tests/tests/nutmeg4_pymcell4/2000_shorter_error_message/
├── model.py        # imports geometry
├── geometry.py     # line ~2214: m.GeometryObject(name=..., vertex_list=..., element_connections=..., surface_regions=[])
└── test_description.toml
```

`geometry.py` deliberately calls `m.GeometryObject(..., element_connections=<walls>, ...)` — the
API keyword is actually **`wall_list`**, not `element_connections`. This is intentional: the wrong
kwarg raises `TypeError` at construction, `exit(1)`, and the test asserts the error text names the
offending kwarg. The test name ("shorter_error_message") comes from the pybind11 era, when MCell
**patched pybind11's dispatcher** (`pybind11.h`) to print *truncated argument values* instead of
dumping the full multi-thousand-element `vertex_list`/`element_connections` into the error.

### `test_description.toml` (pybind11-era, before the fix)

```toml
includes = ["exit_code_1"]

[[checks]]
  # order of the printouts seems pretty random,
  # behavior obtained by modification of pybind11.h - dispatcher - see MCell comment
  matchPattern = "element_connections=\\[\\[0, 1, 2\\], \\[1, 3, 4\\], \\[5, 0, 6\\], \\[7, 5, 8\\], ..."
  dataFile = "err_%d.0.log"
  numMatches = 1
  testType = "FILE_MATCH_PATTERN"

[[checks]]
  testType = "CHECK_EXIT_CODE"
  exitCode = 1  # exit(1)

[run]
  pyfile = "model.py"
```

## Root cause of the failure

The two binding libraries format the "incompatible arguments" `TypeError` differently:

- **pybind11 (MCell-patched dispatcher):** prints the *truncated values* of the supplied args, so
  stderr contains a line like `... element_connections=[[0, 1, 2], [1, 3, 4], [5, 0, 6], [7, 5, 8], ...`.
- **nanobind:** prints the *types* of the supplied args, not their values. stderr contains
  (single line):

  ```
  Invoked with types: nanobind.nb_type, kwargs = { name: str, vertex_list: list, element_connections: list, surface_regions: list }
  ```

  Full nanobind stderr for reference:

  ```
  TypeError: __new__(): incompatible function arguments. The following argument types are supported:
      1. __new__(cls: object, name: str, vertex_list: mcell.VectorVectorFloat, wall_list: mcell.VectorVectorInt, is_bngl_compartment: bool = False, surface_compartment_name: str = 'unset', surface_regions: mcell.VectorSurfaceRegion = mcell.VectorSurfaceRegion([]), surface_class: mcell.SurfaceClass | None = None, initial_surface_releases: mcell.VectorInitialSurfaceRelease = mcell.VectorInitialSurfaceRelease([]), initial_color: mcell.Color | None = None, node_type: mcell.RegionNodeType = RegionNodeType.UNSET, left_node: mcell.Region | None = None, right_node: mcell.Region | None = None) -> mcell.GeometryObject

  Invoked with types: nanobind.nb_type, kwargs = { name: str, vertex_list: list, element_connections: list, surface_regions: list }
  ```

The pybind11-era `matchPattern` (`element_connections=[[0, 1, 2], ...`) never appears in nanobind's
output, so the `FILE_MATCH_PATTERN` check finds **0** matching lines while `numMatches = 1`, giving
`FAILED_NUTMEG_SPEC`. Exit code is still `1`, so only the pattern check fails.

This is a **message-format parity difference only** — the error is still correctly raised and the
process still exits `1`. It is not a functional regression. (Consistent with the migration theme:
nanobind reports arg *types* where MCell's patched pybind11 reported truncated *values*.)

## How the check is evaluated (so the pattern is correct)

`mcell_tests/scripts/tester_nutmeg.py::check_match_pattern`:

1. `dataFile = "err_%d.0.log"` resolves to the run's stderr log (`mcell_stderr.log` in the test
   work dir).
2. The pattern string is pre-escaped: only `(` → `\(`, `)` → `\)`, `+` → `\+`. **Everything else is
   treated as a regex** (so `[`, `.`, etc. keep their regex meaning — that's why the old pattern
   escaped its brackets as `\\[`).
3. `re.compile(pattern)` then `matcher.search(line)` **per line**; `num_matches` counts matching
   lines.
4. PASS iff `num_matches == numMatches` (here `numMatches = 1` → exactly one line must match).

## The fix (applied)

Replace the `matchPattern` with one that matches nanobind's output on exactly one line. The kwarg
name `element_connections` appears on exactly **one** line of nanobind stderr (the "Invoked with
types" line); the supported-signature line lists `wall_list`, not `element_connections`, so there
is no second match.

### Applied — nanobind-precise (`mcell_tests` `mcell4_nanobind`, commit `3b0094ba`)

```toml
[[checks]]
  # nanobind reports argument TYPES in the "incompatible function arguments" TypeError
  # (not truncated values as MCell's patched pybind11 dispatcher did),
  # the wrong kwarg 'element_connections' (correct name: wall_list) appears once, as a type entry
  matchPattern = "element_connections: list"
  dataFile = "err_%d.0.log"
  numMatches = 1
  testType = "FILE_MATCH_PATTERN"
```

`element_connections: list` needs no escaping (no `(`, `)`, `+`, and `:`/space are literal in
regex). The `CHECK_EXIT_CODE` check and `[run]` section are unchanged.

### Not chosen — build-agnostic

Matching just the kwarg name would pass on **both** pybind11 and nanobind:

```toml
  matchPattern = "element_connections"
  numMatches = 1
```

Rejected because (1) pybind11 is gone on this branch, so there is no reason to weaken the check —
it drops the original "is-the-message-truncated" assertion; and (2) it is fragile against the
Python traceback: Python 3.13 currently elides the middle of the multi-line `m.GeometryObject(...)`
call (`...<2 lines>...`), but a traceback that printed the source line
`element_connections = Cube_element_connections, # wrong name` would make the bare name match
**2** lines and fail `numMatches = 1`. `element_connections: list` occurs only in nanobind's
"Invoked with types" line.

## Verification

1. **Pattern replay (diagnosis, no repo changes):** replayed `check_match_pattern`'s exact
   escaping + per-line `re.search` against the nanobind stderr of this test:

   | pattern | matching lines | result vs `numMatches=1` |
   |---|---|---|
   | `element_connections=\[\[0, 1, 2\], \[1, 3, 4\], ...` (pybind11-era) | 0 | FAIL (the bug) |
   | `element_connections: list` (nanobind-precise, **applied**) | 1 | PASS |
   | `element_connections` (build-agnostic) | 1 | PASS |

2. **Real runner, after the fix (2026-10-14, Linux, `mcell4_nanobind` build):** report
   `PASSED: 1, FAILED: 0`; the test's `nutmeg.log`:

   ```
   FILE_MATCH_PATTERN: Checking of pattern 'element_connections: list' in mcell_stderr.log: PASSED
   CHECK_EXIT_CODE: MCell exit code is 1 as expected.
   ```

With this, nanobind ≡ pybind11 on the tracked test set (this was the last of the 4 post-hash-fix
fails; Track A's 3 dynamic-geometry DM→MDL fixes were done in the cellblender repo).

## How to re-run this test

```bash
cd "$ROOT"                               # $ROOT = the build root (see CLAUDE.md)
source mcell_python/.venv/bin/activate   # this root's MCell Python venv
cd mcell_tests
python3 run_tests.py -s \
  -m "$ROOT/mcell_tools/work/build_mcell" \
  -t "$(command -v python3)" \
  -p 2000_shorter_error_message
```

- **`source mcell_python/.venv/bin/activate`** (from `$ROOT`) activates this root's MCell Python
  venv, so the wiki never hardcodes the interpreter path; after it, `python3` *is* the venv
  interpreter, and `-t "$(command -v python3)"` hands it to the runner (doc 05 §7).
- **`-m` must be an absolute path.** The runner exports it verbatim as `MCELL_PATH` and runs the
  model from the test's work dir, so a relative path (e.g. `../mcell_tools/work/build_mcell`)
  fails with `ModuleNotFoundError: No module named 'mcell'`. That still exits `1` — the exit-code
  check alone would pass — but the pattern check correctly reports `FAILED_NUTMEG_SPEC`.
- `-p` is a regex matched against the test path; the test name isolates this one test
  (`nutmeg4_pymcell4` would run the whole group).
- Needs `$ROOT/mcell_tests_private` cloned.
- Confirm the report shows `2000_shorter_error_message` → **PASSED**; per-check results are in
  `mcell_tests/work/nutmeg_pymcell4_mcell4/tests/nutmeg4_pymcell4/2000_shorter_error_message/nutmeg.log`.

## Checklist

1. ☑ Edit `mcell_tests/tests/nutmeg4_pymcell4/2000_shorter_error_message/test_description.toml`:
   `matchPattern = "element_connections: list"` + comment describing the nanobind types format
   (`mcell_tests` `mcell4_nanobind`, commit `3b0094ba`, pushed).
2. ☑ Re-run just this test on the nanobind build → **PASSED** (Verification §2).
3. ☐ At merge time: merge `mcell_tests` `mcell4_nanobind` → `mcell4_dev` together with `mcell`'s
   `mcell4_nanobind` → `mcell4_dev`.
