# 07 — The VTK Removal (record, 2026)

> **What happened.** MCell no longer uses VTK. VTK used to compute MCell's geometry analysis: the
> counted-volume hierarchy, compartment nesting, closedness, and the volumes for the BNGL export and
> `data_model_to_pymcell`. It also wrote MCell's OBJ export. In-tree code with no external
> dependencies replaced it, on `mcell` branch `mcell4_nanobind_novtk` (October–November 2026). That
> branch is to be merged into `mcell4_nanobind`; `mcell4_dev`, which still uses pybind11, gets the
> removal later.
>
> **This document** is the record: why, what was decided, what changed for users, how it was
> checked, the numbers, and what is left. **Doc 04 §6** is the reference for the code as it is now.
> The working records (the live plan, the handovers, the spike and every gate run) stay in the
> `mcell4_nanobind_novtk` build root, outside any repository (§9).

---

## 1. Why

- **Size.**
  - VTK was 13.5 MB of the 21.3 MB `mcell.so` (63.5%, from a linker map), with 61 VTK library
    entries linked.
  - After the removal, `mcell.so` is 6.4 MB (−70%, §7).
- **Build.**
  - `mcell_tools` built VTK from source before MCell: about 8 minutes at `-j32`, a 914 MB build
    directory and a 632 MB clone.
  - MCell itself had to find the prebuilt tree (`find_package(VTK)` with a hard-coded `VTK_DIR`).
- **Speed.** VTK's containment test cost a lot per pair of counted objects, and the number of pairs
  grows with the square of the number of objects:
  - initializing a container with 100 counted vesicles took 11.7 s in MCell, against 0.17 s with
    nothing counted;
  - the 70 counted spines of the private neuropil model took 16.3 s (§7).
- **Correctness.** Found in the spike and confirmed later:
  - *reversed nesting*: `vtkSelectEnclosedPoints` (random rays and voting) wrongly marked a few
    points as inside on large meshes. With the "any point inside" rule, that reversed the nesting of
    two nested spheres (82k and 328k triangles);
  - *missed contacts*: `vtkCollisionDetectionFilter` missed about 22k of 2M touching or coplanar
    contacts;
  - *float coordinates*: VTK stored the points as float. A 1e-9 gap counted as an intersection,
    vertices 1e-12 apart were merged (hiding cracks that MCell's double collision code sees), and
    the reported volumes and OBJ coordinates were float-rounded;
  - *open counted objects*, in the Python API, got only a warning. Counts were then wrong (1,000
    reported inside an open box, 307 really inside), and a second counted object made MCell crash;
  - *orientation was never checked*, although MCell decides from a wall's normal whether a molecule
    crossing a counted object's wall goes in or out.

## 2. Decisions (all by the user)

| Date | Decision |
|---|---|
| 2026-10-28 | Remove VTK completely. Replace it with a small in-tree module, with no external libraries (no libigl, CGAL, Geogram, Embree or GMP); vendoring a small public-domain file is allowed. |
| 2026-10-29 | **Double precision everywhere**, in computation and in reported numbers. This replaced a first plan that kept the reported numbers in float, which had given the same object different volumes in different tools. |
| 2026-10-29 | **Consolidation:** one mesh per object (the internal-unit doubles MCell simulates with), one definition of "closed", and `Region`'s volume formula, so that releases by concentration stay bit-identical. |
| 2026-10-29 | Vendor Shewchuk's exact predicates. Compare VTK and the new code inside MCell, behind a build option, before switching. Compile the volume code and the predicates without floating-point contraction. Write new `.cpp` files for C++23, per file, with headers that stay valid C++17. |
| 2026-10-29 | Leave MCell3's code unchanged (history, and backward compatibility with MCell3); new behaviour goes into MCell4 code only. |
| 2026-10-29, -30 | **Errors for counted objects.** Counting inside an open surface is an error. A surface with a boundary is fine to simulate, and only counting inside it is impossible. An edge shared by three or more triangles (a non-manifold edge, a different fault from an open surface) is an error, and takes precedence. |
| 2026-11-02 | **Edges are matched by exact coordinates**, as VTK's cleaning and MCell's wall linking do. Phase 4 adds a warning where vertex indices and coordinates disagree, and matching by vertex index is a later project. Every Python API object gets the non-manifold check. MDL models on the MCell4 engine get the counted-object errors, but MCell3 runs them as before. |
| 2026-11-02 | Names: `vtk_utils` / `VtkUtils` became `counted_volume_utils` / `CountedVolumeUtils`. New mesh code goes into `MeshUtils`. |
| 2026-10-29, 11-04 | **Mesh checks are warnings, never errors**, so that test runs never stop halfway. Each is one machine-parsable line, in five checks (A, B1, B2, C, D; doc 04 §6.5). Bow-tie and pinched vertices come later. |
| 2026-11-04 | Fix the counted-volume crash found while checking the warning texts (§8). |
| 2026-11-05 | The docs and the merge plan (this document); macOS and Windows checks on the user's machines. New header files use `.hpp`. After the macOS check: `bngl_exporter.cpp` without contraction, as part of the removal, and `diffuse_react_event.cpp` without contraction, as a separate fix (§8). |

## 3. Phases and commits

All in `mcell` on `mcell4_nanobind_novtk` unless noted. Each phase was gated by a full test-suite run
(§6).

| Phase | Content | Commits |
|---|---|---|
| 0 | VTK's points switched to double. A baseline on float and on double: no test changed status, and only reported numbers changed. | `b6f52721` |
| 1a | Shewchuk's `orient2d` and `orient3d`, ported. | `c67be5d9` |
| 1b | `GeometryAnalysis`: cleaning, closedness, containment, volume, OBJ writer, number formatting. | `912c1cb0` |
| 1c | A temporary A/B check inside MCell, comparing VTK and the module at all five call sites. | `d7b286de` |
| 2 | The five call sites switched to the module, with the new errors and the memoized parent search. A separate commit added the non-manifold check of every Python API object. Test 3120's OBJ reference became double. | `f2b3313c`, `8aedd02f`; `mcell_tests` `b32e1508` |
| 2b | Consolidation: compartments on the internal-unit mesh, one definition of closed, `Region`'s volume formula everywhere. | `4676b457` |
| 3 | VTK removed: code, CMake, the A/B check, and the build step (`mcell_tools` branch `mcell4_nanobind`). A separate commit renamed `vtk_utils` to `counted_volume_utils`. | `40c47842`, `f09b92e8`; `mcell_tools` `5fb3d0c` |
| 4 | Mesh checks as parsable warnings. A separate commit first moved `get_mesh` and `get_clean_mesh` into `MeshUtils`. | `58bb5563`, `81b63cdf` |
| fix | Counted volumes of intersecting objects that have a parent (§8). | `c3d77618`; `mcell_tests` `ba0ae972` |
| 5 | The new headers renamed to `.hpp` (a rename only). `bngl_exporter.cpp` built without contraction, so that exported volumes are the same on every platform (§8). These docs; macOS and Windows checks. | `476ca1e1`, `cd6cc248`; the rest in progress |
| 6 | Merge into `mcell4_nanobind`, and these docs into the wiki. | to do |

## 4. What replaced VTK

Doc 04 §6 has the full reference. In short:
- **`MCell::Predicates`** (`src4/predicates.{hpp,cpp}`): exact orientation signs.
- **`MCell::GeometryAnalysis`** (`src4/geometry_analysis.{hpp,cpp}`): mesh algorithms on plain
  arrays.
  - Every decision (closed, touching, inside) takes its signs from the exact predicates.
  - Triangle-triangle contact uses Guigue–Devillers.
  - Containment uses a bounding-volume hierarchy, and point-in-solid uses an exact on-surface test
    and then ray parity.
  - These two files form the static library `MCell::geometry`, compiled as C++23 without
    floating-point contraction.
- **`MCell::MeshUtils`** (`src4/mesh_utils.{hpp,cpp}`): the meshes of MCell's geometry objects, and
  the mesh-check warnings.
- **`MCell::CountedVolumeUtils`** (`src4/counted_volume_utils.{h,cpp}`): counted volumes, the
  containment mapping and parent search shared with compartments, the BNGL export volume, and the
  OBJ export.
- **Conventions:**
  - double throughout;
  - edges matched by exact coordinates;
  - "closed" means every edge in exactly two triangles;
  - volume by `Region`'s signed formula, area by MCell's wall formula;
  - containment rules unchanged from the VTK code.

The module reproduced double-precision VTK bit for bit until phase 2. That's why phase 2 changed
no simulation result. Only VTK's own errors, such as the reversed nesting, differ.

## 5. What users see differently

- **New errors** (phase 2):
  - In the Python API, including data model → Python, counting inside an open surface stops with
    "Counting inside an open surface is not possible, error for <name>." Before, VTK printed a
    warning, and the counts were wrong or MCell crashed.
  - Any Python API object with an edge of three or more walls raises `ValueError`:
    "Non-manifold mesh is not allowed, error for <name>."
  - A counted object or a BNGL compartment with such an edge gets the same message. This also
    covers MDL models on the MCell4 engine whose counted object has an even number of walls on one
    edge: MCell3's check passes those, and MCell3 still runs them. Before, the MCell4 engine warned
    and ran with one counted object, and crashed with a second.
- **The last digits of reported volumes.**
  - The switch to double (phase 0) changed exported volumes such as `1.0000000961751194` to `1`.
  - `Region`'s formula (phase 2b) changed them by at most 1.5e-14 relative, e.g. `VOLUME_EC
    38.999999999999993` to `39`. No volume changed sign.
  - The BNGL export tests run their exported model again, building the default compartment box from
    the exported volume. So the frames of that second run changed in 12 tests (two diverge:
    bngl/1040 and bngl/2500), with no status change.
  - Concentration releases, counts and compartment nesting didn't change in any test.
- **OBJ coordinates are written from doubles,** e.g. `1.3` where VTK wrote `1.2999999523162842`. Test
  3120's reference was updated after every new coordinate, rounded to float, was checked to give
  the old value.
- **Inside-out meshes have a negative volume** in the BNGL export and `data_model_to_pymcell`. VTK
  reported the absolute value. `Region`'s volume, which releases use, was always signed (phase 2b
  kept it bit-identical). So a release by concentration into an inside-out object already stopped
  with the INT_MAX release error; it now gets a `MESH_INSIDE_OUT` warning first.
- **`data_model_to_pymcell`** computes areas with MCell's wall formula (VTK used Heron's formula). For
  a non-manifold compartment, it prints its existing warning with the non-manifold message and
  writes a fallback value.
- **Mesh-check warnings** (phase 4):
  - one line each, `Warning: [CODE] name: text`, parsable with `^Warning: \[([A-Z_]+)\] (\S+):
    (.*)$`;
  - codes: `MESH_INCONSISTENT_ORIENTATION`, `MESH_INSIDE_OUT`, `MESH_DUPLICATE_VERTICES` and
    `REGION_NOT_CLOSED` (doc 04 §6.5);
  - the run continues. No test in the default suite (`test_configs/default.toml`, with
    `mcell_tests_private`) triggers any of them.
- **A failed OBJ write** prints "Error: Could not write geometry to <prefix>.obj and <prefix>.mtl."
  and the run continues. VTK printed its own error and also continued.
- **Initialization is faster** with many counted objects (§7). The build no longer contains VTK.
- **On ARM64 (Apple silicon), results now match x86-64** where the macOS check found them
  differing: four exported volumes that were 1 ULP off, and two tests that failed (§8).

## 6. How it was checked

- **The spike** (2026-10-27/28) was a VTK-free prototype compared with VTK:
  - number formatting matched `vtkNumberToString` on 4M values;
  - the triangle-triangle test had 0 errors against exact arithmetic on 4M pairs;
  - containment agreed with double-precision VTK on the special cases, 2,000 random sphere pairs
    and all 2,415 pairs of the neuropil spines. With float, VTK reported a 1e-9 gap as an
    intersection. It also reversed the nesting of the big spheres (§1).
- **The predicates port.** The port, built with MCell's flags, against the original C file:
  - 9 input families of 1M cases each: 0 bit mismatches;
  - 2M integer-grid cases: 0 sign errors against exact 128-bit arithmetic;
  - control: the same file built with contraction gave 28,480 wrong signs, which is why the
    no-contraction flag is required.
- **The module** (`ga_test`) was compared with double-precision VTK, the spike and an exact
  rational reference:
  - cleaning, closedness, volume and area were bit-identical in every case;
  - the OBJ output was byte-identical to test 3100's references;
  - triangle-triangle had 0 mismatches on 4M pairs, and on 60,000 pairs checked in exact fractions,
    most of them near-degenerate;
  - containment agreed with VTK everywhere except VTK's reversed nesting.
- **The A/B check** (phase 1c) ran VTK and the module side by side at all five call sites, over the
  whole test suite: 27,434 comparisons, 0 differences. They covered:
  - counted volumes: 2,349 objects and 19,063 pairs;
  - compartments: 1,019 objects and 484 pairs;
  - 155 BNGL export volumes and 166 `data_model_to_pymcell` volumes and areas, bit for bit;
  - 11 OBJ exports, byte for byte.
- **The gates.** After phases 1c, 2, 2b, 3 and 4 and the fix, the full `default.toml` suite ran
  (2,840 tests, with `mcell_tests_private`).
  - Each run was compared with the previous one, per test status and per output file. Viz frames
    were compared frame by frame where the last frame differed.
  - No test changed status, except 3120 through its approved reference update.
  - The last run had 2,277 passed and 376 failed, all of the failures predating the project.
- **Platforms.**
  - `predicates.cpp` cross-compiles for arm64 macOS and for x64 and ARM64 Windows (clang-cl), with 0
    fused multiply-adds under the chosen flags. Without them, arm64 has 267.
  - **macOS** (2026-11-05: Apple M1 Pro, macOS 27.2, AppleClang 21), on a copy of the build root:
    - the new code builds without warnings of its own, and its two floating-point objects have no
      fused multiply-adds;
    - all 34 warning and error models print what Linux prints;
    - in a first tier of 277 tests, every difference from Linux came from contraction in older
      code (§8). After the two fixes, the statuses and all 192 exported BNGL files match Linux.
  - **Windows:** part of phase 5.

## 7. The numbers

| | With VTK | Without |
|---|---|---|
| `mcell.so` | 21,358,264 bytes (VTK: 13.5 MB) | 6,379,352 after phase 3; 6,400,056 with the mesh checks and the fix |
| VTK build in `mcell_tools` | about 8 min at `-j32`; 914 MB build directory, 632 MB clone | none |
| MCell configure / compile, clean, `-j32` | 6.4 s / 141.6 s | 3.2 s / 133.0 s |
| Containment of the 70 counted neuropil spines (2,415 pairs) | 16.1–16.3 s | 0.033–0.038 s |
| Container + 100 counted vesicles (spike) | 40.2 s | 0.051 s |
| Two nested spheres, 328k triangles each (spike) | 3.6 s, nesting reversed | 0.32 s |
| Mesh checks of the whole neuropil model (897k walls, 223 objects) | — | 0.60–0.66 s |

The mesh-check time was measured in MCell (`mdl/3640`). For comparison, `init_counted_volumes` of the
same model, with its waypoints, takes 5.3 s.

## 8. Found along the way

### Counted volumes of intersecting objects with a parent

While checking the phase-4 warning texts, an older bug (2020) showed up and was confirmed with a
small model.

- **The bug.** A counted object that intersects another counted object, but has a parent that
  intersects nothing, got the parent's volume as its outside index. Its inside index was
  `INTERSECTS`. A molecule entering it then got an invalid counted-volume index, and counting
  crashed (`mol_or_rxn_count_event.cpp`). This happened in the Python API and in MDL on the MCell4
  engine; MCell3 ran the model. The front-hit branch of
  `update_counted_volume_id_when_crossing_wall` also tested the outside index while assigning the
  inside one.
- **The fix** (`c3d77618`). An object that intersects another counted object gets `INTERSECTS` on
  both sides, even when it has a parent, so a crossing molecule's counted volume comes from its
  position. The branch now tests the index it assigns.
- **The test.** `mcell_tests` `tests/pymcell4_positive/2610_count_in_intersecting_objects_w_parent`
  (`ba0ae972`) checks the counts against the molecules' positions after every iteration. It fails
  without the fix.

### Found by the macOS check: contraction in two older expressions

Apple Clang fuses a multiply and an add into one instruction by default on ARM64; x86-64 has no fused
multiply-add without `-mfma`. The macOS session isolated the two expressions that made its results
differ from Linux. It bisected objects, functions and call sites. The build from before the removal
showed both.
- **`src4/bngl_exporter.cpp:213`** (2021): a compartment's exported volume is its volume times
  `length_unit^3` minus its children's volumes, one multiply-subtract. Four exported volumes were 1
  ULP off, e.g. `VOLUME_L4 0.12499999999999999` instead of `0.125`. Fixed in `cd6cc248` by
  compiling the file without contraction, which completes the removal's platform-independent
  volumes.
- **`len3_squared` (`src4/defines.h:677`, 2019), through `distance3` at `src4/grid_utils.inl:993`.**
  The tile-neighbor search across a shared edge places tiles from the edge's length and compares the
  positions exactly, so one bit changes which tiles are searched for a reaction partner. Tests
  `data_model/2800` and `data_model_pymcell4/0820` failed on Apple silicon. This is a separate fix,
  outside the removal: `229cbbfc` compiles `diffuse_react_event.cpp` without contraction.

On Linux x86-64 both changes leave `mcell.so` byte-identical. Other MCell files still contract on
ARM64. A build of the whole of MCell without contraction also matched Linux in those 277 tests, and
is the next step if the full suite finds more differences there.

## 9. What is left, and where the records are

- **Next:** phase 5 (these docs; the macOS and Windows checks) and phase 6 (the merge into
  `mcell4_nanobind`). In phase 6 the `mcell4_nanobind` root switches its `mcell_tools` to branch
  `mcell4_nanobind`, which has no VTK step. `mcell4_dev` keeps building VTK for the `mcell4_dev`
  root until the removal reaches it.
- **Bow-tie and pinched vertices** (a vertex shared by otherwise separate parts) are not checked;
  neither VTK nor MCell3 checked them. They are planned as check E.
- **Edge connectivity by vertex index.** MCell4 links walls, and the edge checks match edges, by
  exact coordinates. A later project moves both to the vertex indices in new MCell4 code, and
  revises the duplicate-vertex warning with them. It will settle whether MDL models keep MCell3's
  links, and how meshes with duplicate vertices are treated.
- **`bngl_export.toml`** (BioNetGen on the exported models, compared exactly with references) was
  not run: there were no BioNetGen binaries. Its references may need updating for the last digits of
  volumes.
- **376 test failures predate the project** and were not investigated. Gates therefore compared
  output files as well as statuses.
- **Containment cycles** are possible in principle with objects of several components ("any
  component inside"). This is not addressed.
- **The working records**, in the `mcell4_nanobind_novtk` build root (outside any repository):
  - `PLAN_vtk_removal.md`: every decision with its wording, the phase table, results and open items;
  - the handovers;
  - `vtk_removal_spike/`: the spike, the module's unit tests and the experiments behind each
    decision;
  - `vtk_removal_runs/`: every gate's log, outputs and comparisons;
  - `platform_checks/`: the macOS and Windows kit and its results.
