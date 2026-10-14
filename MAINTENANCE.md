# Keeping This Wiki Current

> **Purpose:** this is the maintenance playbook for the wiki itself. After you change the
> codebase, use it to decide *which* document needs updating and *how* to refresh it — so the
> wiki keeps matching the tree instead of rotting into stale notes.
>
> **Core rule:** the wiki is a *derived artifact* of the source. Every doc cites real
> `file_path:line` locations, and **line numbers drift on almost any edit**. Treat a wiki update
> as part of "done" for any change that touches the areas below — the same way you'd update a
> changelog or a test.

---

## 1. The fastest way to refresh: re-run the mapping pass

The wiki was produced by a multi-agent mapping pass, and the cleanest way to update it after
substantial changes is to **re-run that pass** (or the relevant slice of it) rather than hand-patch.
Ask the assistant:

> "Re-map `<area>` and update the corresponding wiki doc(s) in `mcell_codebase_wiki/`, preserving
> the structure and refreshing all `file_path:line` references."

Each document is independent and regenerable on its own, so you can re-map just the part that
changed. For a small, surgical code change, a hand-edit is fine — see §3.

---

## 2. Change → document map

Find the area you touched on the left; update the doc(s) on the right.

| If you changed… | Update | Why |
|---|---|---|
| `libbng/` (BNG classes, BNGL parser, nauty, its `CMakeLists.txt`) | **01** + the libbng sections of **04 §"How MCell4 uses libbng"** and **00 §5.1** | libbng's API surface and the MCell call sites both live across these |
| `nfsim/` (NFcore engine, `NFinput`, its `CMakeLists.txt`) | **02** (+ **00 §4** if targets change) | engine internals + target list |
| `nfsimCInterface/src/` (the `extern "C"` functions, structs) | **03** + **04 §NFsim** + **00 §5.2** | the C boundary is cited from all three |
| `mcell/src4/` (MCell4 engine: World, Scheduler, events, `.inl`, geometry) | **04** (+ **00 §2/§5** if the architecture/main loop changes) | this is the engine doc |
| `mcell/src/` (MCell3, MDL parser, `*nfsim*.c`, `mcell.c`, `init.c`) | **04** (legacy-path + entry-point sections) | MCell3 front-end lives here |
| `mcell/src4/vtk_utils.*` or any VTK use; VTK module list in `mcell/CMakeLists.txt` | **04 §VTK** + **00 §4 "VTK"** | the VTK-usage reference is in 04, summarized in 00 |
| `mcell/libmcell/definition/*.yaml`, `gen.py`, `constants.py`, `doc.py` | **05 Part A** (+ **00 §5.3/§6**) | the codegen schema/flow doc |
| `mcell/libmcell/api/*` (hand-written API subclasses, `bindings.cpp`, `exports.cpp`) | **05 Part A** (generated↔handwritten split, module assembly) | the inheritance/binding pattern |
| `mcell/CMakeLists.txt`, `mcell/libmcell/CMakeLists.txt`, `src4/CMakeLists.txt` | **00 §3 (build graph)** + **05 Part B** + the affected dep doc | the build graph is described in 00 and 05 |
| `mcell_tools/` (`run.py`, `scripts/*.py`) or the VTK build path | **05 Part B** | build orchestration |
| `mcell_python/` (`pyproject.toml`, `.python-version`, deps) or the Python version | **05 Part B (static Py / uv)** + **00 §7 (version coupling)** | runtime env + the `Python3 EXACT` coupling |
| Added/removed a top-level dependency or subdir | **00 §3, §4** + a **new `NN_<dep>.md`** + add it to the **§1 index table** | the integration map is the source of truth for "what's wired in" |

> **Note:** doc **00 (overview)** is the integration map — almost any *structural* change (new
> dependency, changed call boundary, new build target, engine-flow change) should be reflected
> there even if the detail lives in another doc.

---

## 3. Updating a single doc by hand (small changes)

For a localized change (renamed a function, moved a file, added one API class):

1. **Open the doc** from the §2 table.
2. **Grep for the old reference** in the doc and the new location in the source. Example:
   ```sh
   cd "$ROOT"        # the build root: holds mcell/, libbng/, … and mcell_codebase_wiki/
   grep -n "old_function_name" mcell_codebase_wiki/03_nfsimCInterface.md
   grep -rn "new_function_name" nfsimCInterface/src
   ```
3. **Fix the prose + the `file_path:line`.** Re-confirm the line number against the current file
   (line numbers are the first thing to go stale).
4. If the change affects a **call boundary** (who calls what) or a **build target**, also patch
   **00**'s integration map / build graph.
5. Update the "Generated …" date stamp at the bottom of the doc if you did a meaningful refresh.

---

## 4. Special trigger: the codegen (most common drift source)

The Python API in `libmcell/generated/` is **generated, not authored**. The docs describe it as it
stands, so they drift whenever the API is regenerated.

**Whenever you change `definition/*.yaml` and regenerate:**
```sh
cd "$ROOT/mcell/libmcell/definition"   # $ROOT = the build root (see CLAUDE.md)
python3 gen.py            # regenerates generated/gen_*.{h,cpp}, gen_constants, mcell.pyi, RST docs
```
…then check **doc 05 Part A**:
- Did the **YAML schema** gain a new key (a new item type, method form, enum shape)? → update the
  schema section + the annotated example.
- Did the **set of generated outputs** change (new aggregate file, new stub)? → update the
  "outputs" list and the definition-file→outputs table.
- Did you add a **new API class**? → it shows up as new `gen_<class>.{h,cpp}` + a new
  `api/<class>.{h,cpp}` subclass; mention it where classes are enumerated, and note any new
  engine object it wraps in **doc 04**.

`gen.py` is **not** a CMake step — regeneration is a deliberate human action, which is exactly why
the wiki can fall behind: a code reviewer must remember to re-check doc 05 after any API change.

---

## 5. Drift-detection checklist (run before trusting the wiki)

A quick sanity sweep to catch the most likely staleness:

- [ ] **Citations resolve.** Spot-check a few `file_path:line` refs per doc — does the cited line
      still contain what the doc claims? (Line drift is the #1 failure.)
- [ ] **Build graph matches CMake.** Diff **00 §3** against the current `add_subdirectory` /
      `target_link_libraries` lines in `mcell/CMakeLists.txt`.
- [ ] **CMake target names match.** The alias targets (`BNG::bng`, `NFsim::NFsim`,
      `NFsimC::NFsimC`, `MCell::mcell4`, `MCell::libmcell`) named across the docs still exist.
- [ ] **API class list matches generated/.** `ls mcell/libmcell/generated/gen_*.h` vs the classes
      doc 05 enumerates.
- [ ] **VTK module list matches.** The 8 modules in **00 §4 / 04 §VTK** vs the
      `find_package(VTK … COMPONENTS …)` block in `mcell/CMakeLists.txt`.
- [ ] **Python version.** `mcell_python/.python-version` and `find_package(Python3 … EXACT)` still
      agree with what **05** states.

A handy one-liner to find every cited file path that no longer exists:
```sh
cd "$ROOT"        # the build root: holds mcell/, libbng/, … and mcell_codebase_wiki/
grep -rhoE '\b(mcell|libbng|nfsim|nfsimCInterface|src4?|libmcell)[A-Za-z0-9_./-]+\.(h|hpp|hh|cpp|c|cc|inl|py|yaml|txt)' mcell_codebase_wiki \
  | sort -u | while read -r f; do [ -e "$f" ] || echo "MISSING: $f"; done
```
(Paths in the docs are written relative to `$ROOT`; run the check from there. It flags
deleted/renamed files — line-number drift still needs the spot-check above.)

---

## 6. The `wiki_check.py` helper (automate §2 + §5)

`wiki_check.py` lives in this directory and mechanizes the change→doc map (§2) and the
drift-detection checklist (§5). It is **stdlib-only**, **takes the wiki path as an argument**
(nothing is hardcoded), and **never writes to any source repo's tracked files** — so it is safe
on a tree shared by several maintainers. Run it from anywhere:

```sh
cd mcell_codebase_wiki
./wiki_check.py                     # full audit: citation drift + change->doc reminder
./wiki_check.py --since origin/main # what changed on this branch vs main, and which docs to touch
./wiki_check.py --staged --strict   # gate mode: nonzero exit if drift/pending updates (for a hook)
./wiki_check.py --mark-stale --staged   # MODIFY: flag implicated docs with a "possibly stale" banner
./wiki_check.py --clear-stale       # MODIFY: remove all stale banners once docs are updated
```

What it does:
- **Citation check (§5):** scans every `*.md` for `path[:line]` references, resolves each against a
  file index of the whole tree, and reports files that no longer exist or line numbers now out of
  range. It auto-resolves paths cited relative to any repo (e.g. `api/bindings.cpp` →
  `mcell/libmcell/api/bindings.cpp`) and ignores shorthand/placeholders (`foo.h/.cpp`, `gen_x.h`).
- **Change→doc reminder (§2):** for each discovered source repo it lists changed files (staged /
  since a ref / vs HEAD) and prints exactly which wiki docs the `RULES` table says to update.
- **`--mark-stale` / `--clear-stale` (the "modify" half):** insert or remove an idempotent,
  byte-clean "possibly stale" banner at the top of the implicated docs, so a flagged doc is
  visibly marked until a human reviews and clears it.

Paths default to this script's directory (`--wiki`), its parent (`--root`), and every immediate
sub-repo of the root (`--repo`, auto-discovered); override any of them explicitly. The
change→doc rules live in the `RULES` list near the top of the script — **edit `RULES` whenever you
add/rename a wiki doc or a watched source area**, mirroring §2 of this file.

### Optional: wire it as a *local, uncommitted* pre-commit hook
This keeps automation per-clone and adds **nothing** to any repo's tracked tree (the `.git/hooks/`
directory is local and never committed, so other maintainers are unaffected). For example, in your
own clone of the `mcell` repo:

```sh
cat > mcell/.git/hooks/pre-commit <<'EOF'
#!/bin/sh
# advisory reminder; never blocks a commit (drop --strict to keep it non-blocking)
exec python3 "$(git rev-parse --show-toplevel)/../mcell_codebase_wiki/wiki_check.py" --staged
EOF
chmod +x mcell/.git/hooks/pre-commit
```

Use `--staged --strict` instead if you want the commit to *fail* until you address the wiki (a hard
gate). Because the script auto-discovers paths, the same one-liner works from any of the sibling
repos (`libbng`, `nfsim`, `nfsimCInterface`, `mcell_tools`, `mcell_python`) — adjust the relative
path to `mcell_codebase_wiki` if the repo sits at a different depth.

---

## 7. When to do a full re-map vs. a patch

- **Patch by hand** (§3): renamed/moved a few symbols, added one API class, fixed a target name.
- **Re-map the affected doc** (§1): you reworked a subsystem (e.g. the diffuse/react event,
  the codegen, the NFsim boundary) and the structure — not just line numbers — changed.
- **Re-map everything**: a major refactor crossing several areas (new engine path, dependency
  swap, build-system overhaul). Re-running the whole pass is cheaper and more reliable than
  reconciling every doc by hand.

---

*This note is itself part of the wiki — if you change the wiki's structure or file names, update
the §1 index in `00_overview.md` and this document's §2 table together.*
