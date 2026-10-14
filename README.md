# MCell Codebase Wiki

A maintainer-oriented **codebase wiki** for [MCell](https://github.com/mcellteam/mcell)
("Monte Carlo Cell", [mcell.org](https://mcell.org)) — the 3-D, particle-based spatial
reaction–diffusion simulator — together with its bundled C/C++ dependencies, its VTK usage, and its
code-generated Python API.

It maps how the tree is structured: the two engines (legacy MCell3 in `src/`, the MCell4 C++ engine
in `src4/`) and the MCell4 main loop; how the engine consumes libbng, NFsim (through
nfsimCInterface), and VTK; the YAML→C++/nanobind code generation behind `import mcell`; and the
`mcell_tools` build — so you don't have to re-read the whole tree each time. Every doc carries
`file_path:line` citations into the real source.

Docs 00–05 were produced by a multi-agent mapping pass (one agent per domain document); 06 and the
design/TODO notes record later work, chiefly the pybind11→nanobind migration. The wiki is kept in
sync with the code via the maintenance workflow below.

## Documents

Start with **`00_overview.md`** — the architecture overview, build graph, integration map, and
reading guide (with suggested entry paths by task).

| # | Document | Scope |
|---|----------|-------|
| **00** | [`00_overview.md`](00_overview.md) | Whole-system architecture, build graph, integration map, cross-cutting gotchas, reading guide |
| 01 | [`01_libbng.md`](01_libbng.md) | **libbng** — the BioNetGen-Language library (`BNG::`): species/complex graphs, rule application, BNGL parser, nauty canonicalization |
| 02 | [`02_nfsim.md`](02_nfsim.md) | **NFsim** — the network-free stochastic simulator (vendored C++ engine) |
| 03 | [`03_nfsimCInterface.md`](03_nfsimCInterface.md) | **nfsimCInterface** — the `extern "C"` bridge MCell3 uses to drive NFsim |
| 04 | [`04_mcell_core_and_dependencies.md`](04_mcell_core_and_dependencies.md) | **MCell core** — the two engines (MCell3 `src/`, MCell4 `src4/`), the main loop, entry points, and how libbng / NFsim / VTK are consumed |
| 05 | [`05_mcell_python_api_and_build.md`](05_mcell_python_api_and_build.md) | **Python API & build** — the YAML→C++/nanobind codegen, `mcell_tools` orchestration, the CMake build graph, static Python 3.13 + `uv` |
| 06 | [`06_nanobind_migration_feasibility.md`](06_nanobind_migration_feasibility.md) | **pybind11→nanobind migration** — the feasibility study and the executed, validated port (Linux/macOS/Windows; `libs/pybind11` removed) |

Notes:
- [`DESIGN_geometry_vec3_representation.md`](DESIGN_geometry_vec3_representation.md) — design
  proposal: store large geometry (`vertex_list`, `wall_list`) contiguously instead of as nested
  `vector<vector<…>>`.
- [`TODO_track_b_nutmeg_error_message.md`](TODO_track_b_nutmeg_error_message.md) — tracker for the
  last nanobind follow-up (the nutmeg `2000_shorter_error_message` test); its header gives the
  current status.

Plus:
- [`MAINTENANCE.md`](MAINTENANCE.md) — how to keep the wiki current (change→doc map, drift checklist, the helper).
- [`CLAUDE.md`](CLAUDE.md) — in-tree operating guide (also read by AI coding assistants); defines the `$ROOT` convention.
- [`wiki_check.py`](wiki_check.py) — stdlib-only drift checker (citation resolution + change→doc reminder).

## Using it — the `$ROOT` convention & placement

The wiki carries **no machine-specific absolute paths**. Every citation is written relative to a
single anchor, **`$ROOT`** — the MCell build root, the directory that holds the MCell source, its
sibling repos, and this `mcell_codebase_wiki/` folder. So a citation like `mcell/src4/world.h:120`
means `$ROOT/mcell/src4/world.h`.

`$ROOT` is **self-locating**: it is always the parent of this wiki folder. To make the citations
resolve (and for `wiki_check.py` to find the source), keep this folder **in the build root, next to
the repos it documents** (`mcell_tools` checks those out side by side — doc 05 §5d):

```
$ROOT/
├── mcell/                 # github.com/mcellteam/mcell — src/ (MCell3), src4/ (MCell4), libmcell/ (Python API)
├── libbng/                # doc 01
├── nfsim/                 # doc 02
├── nfsimCInterface/       # doc 03
├── VTK/                   # doc 04 §6 (built out-of-tree by mcell_tools)
├── mcell_tools/           # build orchestrator — doc 05 §5
├── mcell_python/          # runtime Python env — from $ROOT: source mcell_python/.venv/bin/activate (doc 05 §7)
├── …                      # other repos mcell_tools checks out (mcell_tests, cellblender, …)
└── mcell_codebase_wiki/   # this wiki (its own git repo)
```

```sh
# from inside mcell_codebase_wiki/
ROOT=$(cd .. && pwd)
```

(The wiki is fully readable on its own; the placement only matters when you want to follow the
`file_path:line` citations into the actual source or run the drift checker.)

## Keeping it in sync

The wiki is a *derived artifact* of the source, and line numbers drift on almost any edit. After
changing MCell or one of its dependencies, update the implicated doc(s) — see
[`MAINTENANCE.md`](MAINTENANCE.md) for the change→document map and drift checklist. The helper
automates it:

```sh
cd mcell_codebase_wiki
./wiki_check.py                            # audit: citation drift + which docs changed source maps to
./wiki_check.py --since origin/mcell4_dev  # what changed vs mcell4_dev → docs to update
./wiki_check.py --staged --strict          # gate mode (nonzero exit) — for a local pre-commit hook
```

`--since REF` diffs every repo under `$ROOT` against `REF`; a repo that lacks `REF` is silently
skipped, so pick a ref that exists (the repos have `origin/master` and `origin/mcell4_dev`, not
`origin/main`).

`wiki_check.py` is stdlib-only (no dependencies) and never writes to any source repo's tracked
files. It reports cited paths that no longer exist or whose line numbers are out of range, and maps
changed source files to the wiki docs that likely need updating. Its citation check is deliberately
conservative — see the caveat in [`CLAUDE.md`](CLAUDE.md).

## License

The wiki documents the MCell source and its dependencies, each under its own license: MCell,
libbng, NFsim, nfsimCInterface, and mcell_tools are MIT-licensed; VTK is BSD-3-Clause.
