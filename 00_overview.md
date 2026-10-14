# MCell Codebase Wiki — Overview & Integration Map

> **Audience:** maintainers and developers of MCell who need to understand, build, extend,
> and debug the whole system — including its three bundled C/C++ dependencies, VTK, and the
> code-generated Python API — without re-reading the entire tree each time.
>
> **Project root (`$ROOT`):** the MCell build root — the parent of this `mcell_codebase_wiki/`
> folder. All source citations below are relative to it (e.g. `mcell/src4/world.h:120`); see
> `CLAUDE.md` §"`$ROOT`" for how to resolve it on your machine.
> This root holds the MCell source (`mcell/`) **and** its sibling dependency repos
> (`libbng/`, `nfsim/`, `nfsimCInterface/`, `VTK/`), plus the build orchestrator
> (`mcell_tools/`) and the runtime Python environment (`mcell_python/`).

---

## 1. How to read this wiki

| # | Document | Scope |
|---|----------|-------|
| **00** | `00_overview.md` (this file) | Whole-system architecture, build graph, integration map, reading guide |
| 01 | [`01_libbng.md`](01_libbng.md) | **libbng** — BioNetGen-Language library (`BNG::`): species/complex graphs, rule application, BNGL parser, nauty canonicalization |
| 02 | [`02_nfsim.md`](02_nfsim.md) | **NFsim** — the network-free stochastic simulator (vendored C++ engine) |
| 03 | [`03_nfsimCInterface.md`](03_nfsimCInterface.md) | **nfsimCInterface** — the `extern "C"` bridge MCell3 uses to drive NFsim |
| 04 | [`04_mcell_core_and_dependencies.md`](04_mcell_core_and_dependencies.md) | **MCell core** — the two engines (MCell3 `src/`, MCell4 `src4/`), the main loop, and how libbng / NFsim / **VTK** are consumed (the VTK section is self-contained) |
| 05 | [`05_mcell_python_api_and_build.md`](05_mcell_python_api_and_build.md) | **Python API generation & build** — the YAML→C++/nanobind codegen, `mcell_tools` build orchestration, static-Python 3.13 + uv |
| 06 | [`06_nanobind_migration_feasibility.md`](06_nanobind_migration_feasibility.md) | **pybind11→nanobind migration** — feasibility study + the **executed** port (done & validated on Linux/macOS/Windows, `libs/pybind11` removed); MI is a non-issue; gains are build/dev ergonomics, not runtime |
| — | [`MAINTENANCE.md`](MAINTENANCE.md) | **How to keep this wiki current** — change→document map, regeneration triggers, drift-detection checklist. Read this before editing the wiki or after changing the code. |

**Suggested entry paths:**
- *"I need to change the simulation physics"* → 04 (MCell4 engine, `src4`).
- *"I need to add/modify a Python API class"* → 05 (YAML codegen) then 04 (engine objects it wraps).
- *"I need to touch chemistry / species / reactions"* → 01 (libbng) then 04 §"How MCell4 uses libbng".
- *"I need to fix network-free (NFsim) behavior"* → 03 (C boundary) then 02 (engine) — note this is an **MCell3-only** path.
- *"I need to change geometry handling, meshes, or OBJ export"* → 04 §VTK.
- *"I need to build/package/release"* → 05 Part B.

---

## 2. What MCell is, and what it produces

**MCell** ("Monte Carlo Cell") is a 3-D, spatially-resolved, particle-based reaction–diffusion
simulator. The build produces **two artifacts from one source tree**:

1. **`mcell`** — a standalone **executable**. Front-end: the legacy **MCell3** code (C compiled
   as C++) which parses **MDL** text files. It can run the old MCell3 engine directly or convert
   the parsed model into MCell4 and run that. It is the only path that can use **NFsim**.
2. **`mcell.so`** — a **Python module** (`import mcell`), CMake target `mcell4_so`, built with
   **nanobind** (migrated from pybind11 — doc 06). Front-end: the code-generated Python API in `libmcell/`. It builds an MCell4
   `World` directly from Python objects — no MDL, no NFsim.

Both front-ends converge on the **same MCell4 C++ engine** (`src4/`, static lib `MCell::mcell4`).

```
            MDL text file                       Python script  (import mcell)
                 │                                     │
        ┌────────▼─────────┐                  ┌────────▼─────────────┐
        │  MCell3 (src/)   │                  │  libmcell Python API │
        │  flex/bison MDL  │                  │  (generated + api/)  │
        │  parser → volume │                  │  nanobind module     │
        └────────┬─────────┘                  └────────┬─────────────┘
                 │ mcell3_world_converter                │  api/mcell4_converter
                 │ (src4)                                 │
                 └───────────────┬───────────────────────┘
                                 ▼
                    ┌─────────────────────────┐
                    │   MCell4 engine (src4)   │   class World, Scheduler,
                    │   event-driven main loop │   Partition, events, .inl hot paths
                    └─────────────────────────┘
                                 │ owns / delegates
                ┌────────────────┼─────────────────┐
                ▼                ▼                  ▼
            libbng (BNG::)     VTK            (NFsim — MCell3 path only,
         species/rxns/BNGL  geometry/mesh      via nfsimCInterface C API)
```

---

## 3. Build graph (from `mcell/CMakeLists.txt`)

`mcell/CMakeLists.txt` is the hub. It pulls in the siblings with `add_subdirectory` and a
pre-built VTK with `find_package`. (Details: doc 05 Part B; per-target line numbers in each dep doc.)

```
mcell/CMakeLists.txt
├── find_package(VTK …)                  # pre-built, from mcell_tools/work/build_vtk  (consumed, not built here)
├── add_subdirectory(libs/jsoncpp)       # vendored
├── find_package(nanobind CONFIG)        # vendored in libs/nanobind (libs/pybind11 removed)
├── add_subdirectory(../libbng)          # → BNG::bng , Nauty::nauty           [doc 01]
├── add_subdirectory(../nfsim)           # → NFsim::NFsim (+ _static/_pic/_EXE) [doc 02]
├── add_subdirectory(../nfsimCInterface) # → NFsimC::NFsimC                     [doc 03]
├── add_subdirectory(libmcell)           # → MCell::libmcell (the Python API)  [doc 05]
└── add_subdirectory(src4)               # → MCell::mcell4 (the engine)        [doc 04]

Targets produced here:
  ├── add_executable(mcell …)            # MCell3 C sources + MDL flex/bison + links everything below
  │      links: MCell::mcell4  MCell::libmcell_dummy  BNG::bng  NFsimC::NFsimC
  │             NFsim::NFsim  JsonCpp::JsonCpp  Nauty::nauty  ${VTK_LIBRARIES}  Python3::Python
  └── nanobind_add_module(mcell4_so …)   # entry libmcell/api/exports.cpp → output lib/mcell.so
         links: MCell::libmcell  nanobind-static  MCell::mcell4  BNG::bng  JsonCpp::JsonCpp
                Nauty::nauty  ${VTK_LIBRARIES}    (no NFsim; on macOS the module leaves Python symbols to dynamic_lookup)
```

**Project-wide knobs worth knowing** (all in `mcell/CMakeLists.txt`):
- `MCELL_CXX_STANDARD` (default **17**) is forced onto every first-party *and* bundled target via
  the `mcell_project_options` INTERFACE target — it raises the siblings' lower C++ floors.
- `-fpermissive` / `/Zc:strictStrings-` are required by the legacy C-compiled-as-C++ code and by
  vendored nauty/nfsim string literals.
- `BUILD_SHARED_LIBS OFF` + `POSITION_INDEPENDENT_CODE ON` are forced so vendored static libs
  (jsoncpp, nauty) can be linked into the shared `mcell.so`.
- **gcc-8.3.0 is hard-blocked** (FATAL_ERROR) — it miscompiles/segfaults NFsim.
- `PYTHON_VERSION` defaults to `3.13`; `find_package(Python3 3.13 EXACT … Interpreter Development)`.

---

## 4. The dependencies at a glance

| Dependency | Language / namespace | Built by | CMake targets MCell links | How MCell reaches it |
|------------|----------------------|----------|---------------------------|----------------------|
| **libbng** | C++17 / `BNG::` | mcell's CMake (`add_subdirectory`) | `BNG::bng`, `Nauty::nauty` | **Direct C++ API** from MCell4 (`src4`) and libmcell |
| **NFsim** | C++ / `NFcore::`,`NFapi` | mcell's CMake | `NFsim::NFsim` (+ static/pic/EXE) | **Indirect** — only through nfsimCInterface |
| **nfsimCInterface** | C ABI (`extern "C"`) | mcell's CMake | `NFsimC::NFsimC` | **Direct C calls** from MCell3 (`src/*nfsim*.c`) |
| **VTK** | C++ / `vtk*` | **built separately** by `mcell_tools` (`work/build_vtk`), consumed via `find_package` | 8 modules (see below) | **Direct C++ API** from `src4/vtk_utils.cpp` + libmcell geometry utils |

### VTK — only 8 modules are used
MCell requests exactly these (and gets the rest transitively):
`CommonCore`, `CommonDataModel`, `CommonTransforms`, `FiltersCore`, `FiltersModeling`,
`FiltersSources`, `IOExport`, `RenderingCore`. They are used **only** for geometry analysis and
export — building the counted-volume containment hierarchy (`vtkSelectEnclosedPoints`,
`vtkCollisionDetectionFilter`), mesh validation/measurement (`vtkFeatureEdges`,
`vtkMassProperties`, `vtkCleanPolyData`, `vtkTriangleFilter`), and Wavefront-OBJ export
(`vtkOBJExporter` + a minimal RenderingCore scene). There is **no interactive rendering**. Full
class-by-class map is in **doc 04 §VTK** (self-contained).

---

## 5. Integration map — who calls what

This is the cross-cutting view; each arrow is detailed in the linked doc.

### 5.1 MCell4 (`src4`) ⇄ libbng — *direct C++, hot path*  → doc 01, doc 04
- `class World` **owns** a `BNG::BNGEngine bng_engine` (`src4/world.h`).
- `class SimulationConfig : public BNG::BNGConfig` (the one inheritance bridge).
- Species/reaction **storage is delegated entirely to libbng**: `get_all_species()` →
  `BNG::SpeciesContainer`, `get_all_rxns()` → `BNG::RxnContainer`, `bng_engine.get_data()` →
  `BNG::BNGData` (molecule types, components, compartments, observables).
- BNGL text is parsed with `BNG::parse_bngl_file(...)` from several `libmcell/api` files.
- Most-used types: `BNG::Species`, `BNG::Cplx`, `BNG::RxnRule`, `BNG::RxnClass`,
  `compartment_id_t`. Network-free behavior **in MCell4** is handled inside libbng (not NFsim).
- **Gotchas:** nauty canonicalization is **not thread-safe**; the BNGL parser is **non-reentrant**
  (global context). nauty must be **static & C++-compiled** or `mcell.so` fails to link.

### 5.2 MCell3 (`src`) ⇄ NFsim via nfsimCInterface — *C ABI*  → doc 02, doc 03
- Only the **executable / MCell3 path** uses NFsim, enabled by the `-rules` / network-free flag.
- MCell3 includes only the C headers `nfsim_c.h` / `nfsim_c_structs.h` and calls C functions:
  `setupNFSim_c`, `constructNauty_c`, `initAndQueryByNumReactant_c`,
  `initAndQuerySystemStatus_c`, `getCompartmentInformation_c`, `queryObservables_c`, …
- Callers: `src/nfsim_func.c`, `src/react_trig_nfsim.c`, `src/react_outc_nfsim.c`,
  `src/react_util_nfsim.c`, `src/vol_util.c`.
- The C wrapper forwards to NFsim's C++ facade **`NFapi`** (single global `System*`), which drives
  `NFcore::System / Molecule / ReactionClass / Complex / Observable`.
- **Data crosses the boundary as opaque `void*` handles** (`MapVectorMap`, `MapVector`) walked by
  accessor functions; the only "real" payload C→C++ is the nauty graph-pattern string + integer
  counts. **MCell4/`src4` does not use NFsim at all.**

### 5.3 Python API (`libmcell`) ⇄ MCell4 engine — *generated nanobind*  → doc 05, doc 04
- The Python class surface is **code-generated** from `libmcell/definition/*.yaml` by
  `gen.py` → `libmcell/generated/gen_*.{h,cpp}` (+ `mcell.pyi`).
- Pattern: hand-written `api/<class>.{h,cpp}` **inherits** generated `generated/gen_<class>.h`
  (`Gen<Class>`); the generated base holds data + nanobind bindings, the hand-written subclass adds
  behavior/validation. Extending the API = edit YAML, regenerate, implement new virtuals in `api/`.
- `NB_MODULE(mcell, m)` lives in `api/bindings.cpp`; `api/mcell4_converter.cpp` turns the
  Python `Model` into an MCell4 `World`.

### 5.4 Engine bridge: MCell3 ⇄ MCell4 — *in-process converter*  → doc 04
- `src4/mcell3_world_converter.cpp` reads the legacy `struct volume` (filled by `src/init.c` from
  the MDL parse) and constructs an equivalent MCell4 `World`.
- `src4/mcell4_iface_for_mcell3.{h,cpp}` is the C-callable shim letting the legacy C `main`
  (`src/mcell.c`) invoke the C++ engine.

---

## 6. The Python API generation & build pipeline (one-paragraph orientation)

`libmcell/definition/*.yaml` (11 files: model, subsystem, geometry, instantiation, observables,
introspection, simulation_setup, constants, checkpointing, callbacks, submodules) describe every
API class declaratively (attributes, types, defaults, methods, enums, constants, docs). The
**manual** step `cd mcell/libmcell/definition && python3 gen.py` regenerates
`libmcell/generated/gen_*.{h,cpp}`, `gen_constants.*`, `gen_vectors_bind.cpp`, the `mcell.pyi`
stub, and RST docs. It is **not** wired into CMake — regeneration is a deliberate developer action.
The build itself is driven by `mcell_tools/run.py` → `scripts/build.py`: it builds **VTK first**
(static, into `work/build_vtk`), then configures+builds MCell into `work/build_mcell`, then
bundles. The runtime Python is a **statically-built Python 3.13** pinned by `mcell_python/.python-version`;
`mcell_python/` is a **uv** project/venv (numpy/scipy/pandas/pyyaml/pyside6/…) that provides the
environment for `import mcell` and for running model scripts. Full details, the YAML schema, an
annotated class example, and the regeneration/build commands are in **doc 05**.

---

## 7. Cross-cutting gotchas (consolidated)

- **NFsim is MCell3-only.** Don't look for NFsim calls in `src4` — network-free MCell4 chemistry
  is libbng's job. (doc 03/04)
- **gcc-8.3.0 segfaults NFsim** — the build refuses it. (doc 00/02)
- **nauty must stay static + C++-compiled**, or `mcell.so` link fails (`undefined reference to
  updatecan`); this is why `BUILD_SHARED_LIBS` is forced OFF and PIC ON. (doc 00/01)
- **The codegen is a manual step.** Editing `generated/gen_*` by hand is wrong — edit the YAML and
  run `gen.py`. Hand-written behavior goes in `api/`, never in `generated/`. (doc 05)
- **The `.inl` files in `src4`** are hot-path code `#include`d into `.cpp` for cross-TU inlining —
  they are not standalone units; changing them recompiles their includers. (doc 04)
- **Exact Python version coupling:** `find_package(Python3 3.13 EXACT)` — the module is ABI-bound
  to the specific statically-built 3.13. (doc 05)
- **VTK is built and consumed out-of-tree** (`mcell_tools/work/build_vtk`); the path is hardcoded
  in `mcell/CMakeLists.txt`. A clean checkout must build VTK before MCell. (doc 05)

---

*Generated by a multi-agent mapping pass. Each linked document carries `file_path:line`
references to the real source; this overview is the index and the integration map that ties them
together.*
