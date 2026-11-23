# CLAUDE.md — MCell codebase wiki & maintenance

Portable, in-tree guidance for working on the MCell tree and on this wiki. It travels with the
`mcell_codebase_wiki/` folder to any machine (unlike `~/.claude/.../memory/`, which is
machine-local). A short signpost at the build root (`../CLAUDE.md`) auto-loads each session and
points here; this file holds the substance.

## `$ROOT` — resolving paths on your machine (read first)

The wiki carries **no machine-specific absolute paths**. Every source citation in every doc is
written **relative to a single anchor, `$ROOT`** — the **MCell build root**, the directory that
holds the sibling repos (`mcell/`, `libbng/`, `nfsim/`, `nfsimCInterface/`, `mcell_tools/`,
`mcell_python/`) **and** this `mcell_codebase_wiki/` folder. A citation like `mcell/src4/world.h:121`
means `$ROOT/mcell/src4/world.h`.

**`$ROOT` is self-locating — it is the parent of the folder this file lives in.** There is nothing
to configure and no need to ask anyone: wherever the tree sits, `$ROOT` is "one level up" from this
wiki directory. (The folder's name differs from machine to machine — do not rely on it.) From the
wiki directory:

```sh
ROOT=$(cd .. && pwd)
```

In a session whose working directory is `$ROOT`, the `mcell/...`, `libbng/...`, etc. citations are
already clickable as-is. The wiki docs referenced as `mcell_codebase_wiki/NN_*.md` are siblings of
this file.

## What this tree is

`$ROOT` is the MCell build root: the MCell source (`mcell/`) plus its sibling dependency
repos (`libbng/`, `nfsim/`, `nfsimCInterface/`), the build orchestrator (`mcell_tools/`),
and the runtime Python env (`mcell_python/`, a `uv` project on a statically-built Python 3.13).
The root itself is **not** a git repo; each sub-project is its own repo. This wiki folder is one
too: a **local** git repo (branch `main`, no remote), so wiki edits show up in its own
`git status` / `git diff`. (`wiki_check.py` lists it among the repos it scans; no `RULES` row
matches it, so that is harmless.)

**Activate the MCell Python venv from `$ROOT`:** `source mcell_python/.venv/bin/activate` — the
same command on every machine, and always this root's own venv (a personal shell alias such as
`mp` may point at another root's). Never hardcode the interpreter path in docs or commands; after
activating, use `python3` (e.g. `run_tests.py -t "$(command -v python3)"`). Details: doc 05 §7.

MCell builds **two artifacts from one tree**: the `mcell` executable (MCell3 C front-end, parses
MDL, can use NFsim) and `mcell.so` (the `import mcell` Python module, target `mcell4_so`,
nanobind — migrated from pybind11, doc 06). Both converge on the **MCell4** C++ engine in `mcell/src4/`.

## The wiki — start here for any structural question

Entry point: **`mcell_codebase_wiki/00_overview.md`** (architecture, build graph, integration map,
reading guide). Then `01_libbng.md`, `02_nfsim.md`, `03_nfsimCInterface.md`,
`04_mcell_core_and_dependencies.md` (engines + main loop + geometry analysis),
`05_mcell_python_api_and_build.md` (YAML→nanobind codegen + build + static-Py/uv); `06`,
`07_vtk_removal.md` and `08_vertex_index.md` record the nanobind migration, the VTK removal and the
vertex-index project. Every doc
carries `file_path:line` citations. **Prefer reading the relevant wiki doc before re-deriving
structure from source.**

Key facts worth holding (details + citations in the wiki):
- **Two engines:** `mcell/src/` = MCell3 (C-as-C++, MDL); `mcell/src4/` = MCell4 (C++17, the real
  engine). An MDL file takes one of two paths. MCell4's MDL path (`mcell -mcell4`) goes
  `MCell3's parser → struct volume → MCell4's check of the vertex indices → MCell3's initialization
  → src4/mcell3_world_converter → World`. MCell3's engine (`mcell file.mdl`) runs MCell3's parser
  and initialization and then its own simulation.
- **NFsim is MCell3-only.** `src4` never calls it; network-free MCell4 chemistry is libbng's job.
  MCell reaches NFsim only through the `extern "C"` wrapper `nfsimCInterface` (`NFsimC::NFsimC`).
- **The Python API is code-generated, manually:** `mcell/libmcell/definition/*.yaml` + `gen.py` →
  `mcell/libmcell/generated/gen_*.{h,cpp}` + `mcell.pyi`. Regenerate with
  `cd mcell/libmcell/definition && python3 gen.py` (NOT a CMake step). Never hand-edit
  `generated/`; behavior subclasses live in `libmcell/api/` (`X : public GenX`).
- **No VTK; connectivity from vertex indices.** The geometry code is in-tree and has no external
  dependency.
  - MCell4 takes each wall's neighbours only from the walls' vertex indices, never from
    coordinates. It checks every geometry object's mesh at initialization and writes a geometry
    report; errors stop the run, warnings don't, and no mesh is ever repaired.
  - The same code does the counted volumes, compartment nesting, volumes and areas, and the OBJ
    export.
  - It lives in `MCell::geometry` (exact predicates, `GeometryAnalysis`, and the `MeshTable` with
    the checks in `mesh_table`), `MeshUtils` and `CountedVolumeUtils`.
  - The geometry files are C++23, and all of MCell is built without FP contraction, so ARM64
    computes as x86-64 does (doc 04 §6; history: docs 07 and 08).
- **Gotchas:** gcc-8.3.0 is hard-blocked (segfaults NFsim); nauty must stay static + C++-compiled
  or `mcell.so` fails to link; the BNGL parser is non-reentrant; `find_package(Python3 3.13 EXACT)`.

## Keeping the wiki in sync (after changing code, or before trusting the wiki)

`mcell_codebase_wiki/MAINTENANCE.md` is the full playbook (change→doc map in §2, drift checklist
in §5, the helper in §6). The helper is **`mcell_codebase_wiki/wiki_check.py`** — stdlib-only,
lives in this dir, **never writes to any source repo's tracked files** (safe for a shared,
multi-maintainer tree), and hardcodes no paths.

```sh
cd mcell_codebase_wiki
./wiki_check.py                     # audit: citation drift + which docs changed source maps to
./wiki_check.py --since origin/main # what changed on this branch vs main → docs to update
./wiki_check.py --staged --strict   # gate mode (nonzero exit) — for a local pre-commit hook
./wiki_check.py --mark-stale --staged   # flag implicated docs with a "possibly stale" banner
./wiki_check.py --clear-stale       # remove the banners once docs are updated
```

After editing the YAML/codegen or any `src4`/`libmcell`/dependency area, re-check the wiki doc(s)
the helper names and refresh their prose + `file_path:line` citations. For a subsystem-level
rewrite, re-running the multi-agent mapping pass for the affected doc is cleaner than hand-patching
(see MAINTENANCE.md §1 and §7).

### CAVEAT — how the citation check behaves (don't over-trust it)
`wiki_check.py`'s citation check is deliberately **conservative to avoid false positives**: it flags
a cited file as `MISSING` only when that **basename exists nowhere** in the tree. A citation that
points at the *wrong directory* but whose filename still exists somewhere is **not** flagged. So:
- It reliably catches deleted/renamed files and out-of-range line numbers.
- It will **not** catch a path-prefix mistake (right filename, wrong folder). Use the change→doc
  reminder + a human spot-check (MAINTENANCE.md §5) for those.
It also skips shorthand/placeholders (`foo.h/.cpp`, `gen_x.h`) and MCell's output paths under
`reports/` (the samples of doc 04 §6.3), and resolves repo-relative paths
(`api/bindings.cpp` → `mcell/libmcell/api/bindings.cpp`) via a whole-tree file index.

## How to modify the tooling (when docs or watched areas change)

- **Add / rename a wiki doc, or watch a new source area:** edit the **`RULES`** list near the top
  of `wiki_check.py` (the `(repo, path-regex, [doc-prefixes], reason)` rows) **and** the §2
  change→doc table in `MAINTENANCE.md` together — they are intentional mirrors. Doc prefixes
  (`"04"`, `"05"`, …) are resolved by globbing `NN_*.md`, so renaming a doc's *title* is fine;
  renaming its *number prefix* means updating `RULES`.
- **Change the stale-banner text/markers:** edit `banner_text()` / `BANNER_START` / `BANNER_END`
  in `wiki_check.py`. Keep insert/remove symmetric — there is a byte-perfect round-trip invariant:
  `--mark-stale` (twice) then `--clear-stale` must leave docs byte-identical (it self-heals stray
  blank-line runs via the `\n{3,}→\n\n` collapse in `strip_banner`).
- **Tune what counts as a citation:** `CITATION_RE` / `SRC_EXT` / `_is_placeholder()` /
  `PRUNE_DIRS` in `wiki_check.py`.
- The script is intentionally dependency-free (Python 3 stdlib only) so it runs on any machine
  with no setup. Keep it that way.

## Conventions

- Treat a wiki update as part of "done" for any change touching `src4/`, `src/`, `libmcell/`, the
  dependencies, or the build — same as updating a changelog.
- Don't commit anything into `mcell/` (or any dependency repo) for wiki/tooling purposes; the wiki
  and its helper live at the build-root level on purpose. A pre-commit hook, if wanted, goes in a
  maintainer's local `.git/hooks/` (uncommitted) — see MAINTENANCE.md §6.
- If you move/rename this `mcell_codebase_wiki/` folder, update the signpost at `../CLAUDE.md`.
