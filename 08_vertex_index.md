# 08 — Edge Connectivity by Vertex Index (record, 2026)

> **What happened.** MCell4 takes each wall's neighbours only from the vertex indices of the
> model: MDL `ELEMENT_CONNECTIONS` and the Python API's `wall_list`. Before, walls were joined where
> the coordinates of their edges' ends matched: in Python API models by MCell4's copy of MCell3's
> `surface_net`, and on MCell4's MDL path by copying the links that MCell3's initialization had
> made the same way. MCell4 now checks every geometry object's mesh at initialization, writes a
> geometry report, stops on errors and never repairs a mesh. MCell3's engine is unchanged, and so
> is MCell3's code, but for one call that MCell4's MDL path makes in `src/mcell.c`. The work is on
> branch `mcell4_nanobind_vertexidx` of `mcell` and `mcell_tests` (November 2026), on top of the
> VTK removal (doc 07), and was merged into `mcell4_nanobind` on 2026-11-24.
>
> An MDL file takes one of two paths: **MCell3's engine** (`mcell file.mdl`) or **MCell4's MDL
> path** (`mcell -mcell4 file.mdl`, also `-mdl2datamodel4`). Both run MCell3's parser and MCell3's
> initialization; on MCell4's MDL path `MCell3WorldConverter` then builds MCell4's model.
>
> **This document** is the record: why, what was decided, what changed for users of each front
> end, the issues fixed along the way, how it was checked, the numbers, and what is left. **Doc 04
> §6** is the reference for the code as it is now (`mcell` `1bb0f809`), with the known issues in
> §6.12 and §3.7. "Before" means `mcell` `0652d19b`, the VTK removal's last commit. The working
> records (the plan, the handovers, the experiments and every gate run) stay in the
> `mcell4_nanobind_vertexidx` build root, outside any repository (§9).

---

## 1. Why

- **The model's connectivity wasn't used.** A model lists each wall's three vertex indices, but
  MCell joined walls by coordinates.
  - In Python API models, `surface_net`, MCell4's copy of MCell3's linking, keyed each edge by the
    coordinates of its ends and paired the walls whose edges matched (in `src4/geometry.cpp`,
    removed in `7a5b6565`).
  - On MCell4's MDL path, `MCell3WorldConverter` copied the links, edges, wall constants and
    region borders of MCell3's initialization, which keys edges by coordinates too
    (`src/wall_util.c:287-293`, `src/edge_util.c:34-51`).

  So walls that the model keeps apart were joined wherever their vertices had equal coordinates,
  and nothing checked that the indices made a sound mesh. The VTK removal kept this and named the
  move to vertex indices as the next project (doc 07 §9).
- **Broken meshes went unnoticed.** Four findings of 2026-11-09
  (`MCELL4_GEOMETRY_VALIDATION_FINDINGS.md` in the build root), checked against `master` as of its
  last commit of 2025-05-02, which still had VTK, on a CellBlender cube run through the Python API:
  1. counting volume molecules inside a non-watertight object gave no error, and the count stayed
     at 100 while molecules left through the hole; releasing volume molecules into it gave no error
     either;
  2. a vertex index out of range in `wall_list` wasn't checked: the constructor's range test sits
     after a `throw`, and the out-of-bounds read surfaced as a misleading overlapped-walls error;
  3. a wall that faces against its neighbours wasn't detected. Crossings of that wall are counted
     the wrong way round, and a release of volume molecules by concentration takes its number from
     a signed volume that the wall reduces (6.67 instead of 8 um^3);
  4. the lists were checked only by the constructor, so lists changed through setters before
     `initialize()` reached the converter unchecked.

  The VTK removal had already made counting inside a non-watertight object an error for Python API
  models, and added warnings for a release into a non-watertight object and for an inconsistently
  oriented or inside-out counted object. It also removed `Region::is_manifold()`, which finding 3
  showed to be always true (`81b63cdf`).
- **Crashes.** On MCell4's MDL path, MCell4's converter crashed (segmentation fault) where MCell3
  had dropped a wall (a wall of `REMOVE_ELEMENTS` in its parser, a zero-area wall in its
  initialization) or left an edge unlinked (walls that run along an edge in the same direction, as
  in test `mdl/2810`; two walls on the same three vertices). And a vertex index equal to the vertex
  count, or below 0, crashed MCell3's initialization on both paths, before the converter.
- **Warnings let broken uses run.** Since the VTK removal the mesh checks were warnings, so a run
  went on with, for example, an inside-out counted object, whose crossings are counted the wrong way
  round. A transparent cube with 2 um edges and 1,000 volume molecules released at its center, an
  MDL file on MCell4's MDL path (`vertex_index_spike/experiments/cube_cases/transparent/` in the
  build root):

  | Iteration | Inside, by position | Counted, walls facing out | Counted, walls facing in |
  |---|---|---|---|
  | 250 | 620 | 620 | 766 |
  | 1,000 | 150 | 150 | 850 |
  | 3,000 | 52 | 52 | 948 |

- **No test had to change.** A read-only scan of every test mesh (605 mesh files, 8,090 objects;
  §6) found that linking by index changes no test's outcome. It found faults in three models:
  `mdl/2810` (a todo test) is inconsistently oriented, `d000_er` of the neuropil models has 12
  shells that face inward, and the full neuropil has one non-manifold vertex.

## 2. Decisions (all by the user)

| Date | Decision |
|---|---|
| 2026-11-11 | **The mesh as given.** The model's connectivity is authoritative. MCell4 takes each wall's neighbours only from the vertex indices; nothing is matched by coordinates, and nothing of MCell3's linking is copied or ported. MCell4 never repairs a mesh (no merging, no reorienting); it reports. Errors stop the run, warnings don't. MCell3's engine and code stay unchanged. |
| 2026-11-11 | **The checks of every object, in order:** the indices first (out of range or negative, a wall that isn't three indices or uses one vertex twice, two walls on the same three vertices, wall indices of a surface region out of range); zero-area walls, by MCell's own area test, an error; equal coordinates a warning; then manifold mesh (an error), consistently oriented (an error) and watertight (a warning), each run only if the one before passes. The words are "watertight" and "non-watertight", never "closed" or "open". |
| 2026-11-11 | **One report and one stop.** A geometry report with one block per object is always written. Each problem gets one terminal line, `Error: [CODE] <object>: <text>`. Every object is checked before the run stops once, naming every object with errors. The checks get no Python bindings. |
| 2026-11-11 | **Inside-out and non-watertight objects become errors where a use needs a watertight object that faces outward:** an inside-out counted object, compartment, or object of a release by concentration; a non-watertight object as the region of a release of volume molecules. Both were warnings. |
| 2026-11-11 | **One code for both front ends.** On MCell4's MDL path the walls are linked by index with the same code as for Python API models, and MCell4 computes the wall and edge constants itself. Each object keeps its table and its facts for the run. The OBJ export stops merging equal coordinates. Getters for an object's area and volume. Walls that cross each other stay a known issue. |
| 2026-11-11 | **Tests:** model tests, and a C++ test program for the checks whose expected values come from the Python prototype `coboundary.py`, which stays out of the repositories. No existing test should change. |
| 2026-11-12 | The report is `reports/geometry_report_<seed>.txt`, named and headed like MCell's other reports, with each object's shells and area, and the signed volume of a watertight object. Equal coordinates means exactly equal (0.0 equals −0.0), within one object. On MCell4's MDL path, zero-area walls that MCell3's initialization dropped are reported as zero-area walls. |
| 2026-11-12 | **The checks run serially.** They take 0.7 s of the full neuropil's 83 s of startup; GCC's parallel algorithms are parallel only with TBB present at build time; GNU OpenMP isn't fork-safe; and users who run one seed per core would get too many threads. |
| 2026-11-12 | **Tests are self-contained and deterministic:** they never run external Python. Reference tools run once, outside the repositories, and their results are stored next to the test meshes. |
| 2026-11-13 | **Non-manifold vertices go in one list, unclassified**, in the report and on the terminal. Every list of ids gives its count and says what the ids are. On MCell4's MDL path, the walls are checked as MCell3's parser stored them, without the walls of `REMOVE_ELEMENTS`. |
| 2026-11-16 | The terminal texts, singular and plural. `REMOVE_ELEMENTS` runs on MCell4's MDL path: the removed walls are left out, and the report keeps the file's wall ids. The stop is a `ValueError` for the Python API and exit code 1 for MDL files, `-mdl2datamodel4` included. The Python API's malformed lists and the wall indices of its surface regions join the index checks. Each gate also compares every terminal line and report with the step-1 scan. MCell4 ignores MCell3's region bounding boxes. |
| 2026-11-17 | **On MCell4's MDL path, a model must behave exactly like the same model through the Python API.** MCell4 computes everything from what the file says, with MCell3 only as its reader, and nothing in an MCell4 run depends on MCell3's engine. |
| 2026-11-17 | **One region-border rule for every region, `ALL` included** (surface molecules): a side of a wall in the region is a border when its edge has exactly one wall in the region. |
| 2026-11-17 | On MCell4's MDL path, MCell4 computes the walls and areas of releases onto regions (surface molecules), and the boxes of releases into regions and the walls of clamps (volume molecules), itself. TRANSPARENT lets surface molecules pass a region border, as MCell4's documentation says. |
| 2026-11-17 | **A wall clamped more than once** for the same volume molecule and side is an error. One MCell4 function builds every clamp event, one per clamp, also when its class is on no wall. The meaning of the marks `S'` and `S,` for surface molecules stays as it is for now. |
| 2026-11-18 | Clamp events run in the order of surface class name, molecule name and side. Test 1700 gets a ten times steeper gradient, not a weaker check. Effects of the order within an iteration, and the clamp's volume molecule that leaves a box, are later steps. |
| 2026-11-18 | **Only the model's indices count:** an object whose walls meet only at equal coordinates is non-watertight. `data_model_to_pymcell` uses the same checks. Step 7 first compares the kept facts with the old values in a temporary build. The vertex-move check keeps its ascending order for now through a sort; later nothing is sorted. The suite runs from the machine's local disk. |
| 2026-11-19 | The BNGL tests write visualization output at the first and last iteration only, and their references keep only those frames. Visualization output is a barrier, but these tests count every iteration, which already puts a barrier at each one, so their counts and kept frames don't change; the three tests without observables get new last frames. |
| 2026-11-19 | Releases after a vertex move use the moved geometry, for volume and surface molecules; in models with intersecting counted objects, the counted volumes after a move come from the current walls. |
| 2026-11-19 | **The usage errors** (volume molecules): a shell that faces the wrong way, for counted objects, compartments and releases by concentration into one object; non-watertight by its indices, for every object of a release into a region (by number, by concentration, removals). One line per object and use, one MCell4 function at the end of both converters, one stop. |
| 2026-11-19 | On MCell4's MDL path `DYNAMIC_GEOMETRY` stops the run, and `data_model_to_pymcell` stops on a dynamic object; both were dropped silently. A negative concentration (volume molecules) or density (surface molecules) removes molecules, as on MCell3's engine. On MCell4's MDL path, regions other than `ALL` in releases of volume molecules stay a known issue. |
| 2026-11-19 | The OBJ export writes each mesh as given: every vertex in the model's order, and the walls with the model's indices; its files name MCell. Nothing in MCell merges coordinates. |
| 2026-11-20 | `Model.get_object_area` and `get_object_volume`, the volume signed, and an error for a non-watertight object. No code computes a signed volume for a non-watertight mesh or shell. `get_vertex` gets the argument check of the other getters, and the API docs' errors are fixed. |
| 2026-11-20 | **Orphaned vertices, which no wall uses, are an error**, replacing their acceptance of 2026-11-19: then no export has to decide whether to drop them, and every vertex has a wall for its normal. No separate check of orphaned edges: every edge comes from a side of a wall. |
| 2026-11-20 | Model tests for every line, stop and exit code; the full neuropil checked and benchmarked against the build from before the project. The memory of MCell3's model, which MCell4's MDL path keeps through the run, is recorded, to be decided later. The docs: doc 04 §6 rewritten as the reference, with the known issues, and this record. |
| 2026-11-23 | **No vertex index out of range, and no negative index, from any input of MCell4.** MCell4's MDL path checks the vertex indices of every wall, also of the walls that `REMOVE_ELEMENTS` removes, right after MCell3's parser and before MCell3's initialization, which would read the vertices through them; the run stops with `MESH_INVALID_INDICES`. The converter's own check refuses a removed wall's index out of range too. MCell3's engine path doesn't change. No input names an edge: MCell4 makes every edge from a side of a wall. |
| 2026-11-23 | Python API models: `Model.add_vertex_move` raises for an index out of range, as the getters do, instead of ending Python; the getters' message for a wall index names the wall list; the constructor's range test, which never ran, goes. Every statement about an MDL file names its path: MCell3's engine, or MCell4's MDL path. |
| 2026-11-24 | **The project ends after step 12b.** Its planned steps 13-17 aren't about connectivity by vertex index. They, and every other open issue found during the VTK removal and this project, go into a TODO note for later projects, `TODO_open_issues.md`. |

## 3. Steps and commits

All in `mcell` on `mcell4_nanobind_vertexidx` unless noted; `mcell_tests` has a branch of the same
name. Each code step was gated by a full test-suite run (§6).

| Step | Content | Commits |
|---|---|---|
| 1 | A read-only scan of every test mesh with every planned check, cross-checked against the prototypes. | – |
| 2, 3 | The plan; a baseline run of the full suite on `0652d19b`. | – |
| 4 | The `MeshTable`, the checks and the report writer in `src4/mesh_table.{hpp,cpp}`, called by nothing yet; the test program `mesh_utils_test`. | `b014e9f4`; `mcell_tests` `4d8e1488` |
| 5 | The checks run on every object in both converters; the geometry report, the terminal lines and the one stop. `REMOVE_ELEMENTS` runs on MCell4's MDL path. | `e530244c`; `mcell_tests` `ebf2fb39` |
| 6 | Walls linked and region borders made from each object's table in both converters, with one border rule; `surface_net` and the copying of MCell3's links removed. | `7a5b6565`; `mcell_tests` `e7f51681` |
| 6a | TRANSPARENT lets surface molecules pass a region border. | `97b2f5c5`; `mcell_tests` `1f7478f4` |
| 6b | On MCell4's MDL path, MCell4 computes the walls and areas of releases onto regions, the boxes of releases into regions and the walls of clamps; one function builds every clamp event; `CLAMP_ON_WALL_TWICE`. | `72cb4d9c`; `mcell_tests` `a38dceba` |
| 7 | Each object keeps its check for the run, and every reader uses it; only the OBJ export still merges coordinates. | `6ef31aaf` |
| 7v | Tests only: the BNGL tests write visualization at the first and last iteration, and their references keep only those frames. | `mcell_tests` `562b5ed5`; `mcell_tests_private` `10e886d` |
| 7a | Releases after vertex moves use the moved geometry. | `84765c8e`; `mcell_tests` `3b5e6177`; `mcell_tests_private` `d15b60a` |
| 8 | The usage errors; on MCell4's MDL path `DYNAMIC_GEOMETRY` stops the run, and `data_model_to_pymcell` stops on a dynamic object; removals by a negative concentration or density. | `33ff673d`; `mcell_tests` `10a574b5` |
| 9 | The OBJ export writes each mesh as given; nothing in MCell merges coordinates. | `620451de`; `mcell_tests` `a4e0e7e6` |
| 10 | `Model.get_object_area` and `get_object_volume`; no signed volume for a non-watertight mesh; `get_vertex`'s argument check; doc fixes. | `46afdfd1`; `mcell_tests` `d0561af3` |
| 10a | Orphaned vertices are an error. | `e62b9af8`; `mcell_tests` `b43fdd8f` |
| 11 | Model tests for every line; the full neuropil checked and benchmarked. | `mcell_tests` `53b720b1` |
| 12, 12a | The wiki: doc 04 §6 and §3.7, and the docs that point to them; this record. | wiki `d65cbcf` (branch `vertex_index`); this record |
| 12b | MCell4's MDL path checks every wall's vertex indices before MCell3's initialization; the converter's check refuses a removed wall's index out of range; the index arguments of the Python API; three comments. | `1bb0f809`; `mcell_tests` `2272e059` |

The `mcell` and `mcell_tests` commits are pushed, and since 2026-11-24 they are on `mcell4_nanobind`
too. The two `mcell_tests_private` commits are on its branch of the same name, pushed on 2026-11-24
(§9).

## 4. What replaced matching by coordinates

Doc 04 §6 has the full reference. In short:
- **The `MeshTable`** (`src4/mesh_table.hpp:105`): one object's connectivity, built from its walls'
  vertex indices in a few passes over flat arrays: per wall side, its edge and direction; per edge,
  its ends and its walls; per vertex, its numbers of walls and edges. It is the C++ form of the
  sparse matrices of the Python prototype `coboundary.py`, which stays outside the repositories
  (§6).
- **The checks** (`check_mesh`, `mesh_table.cpp:603`), in a fixed order, on the lists as the model
  gives them: Python API models at `Model.initialize()`, MCell4's MDL path in
  `MCell3WorldConverter` on the walls MCell3's parser stored, and BNGL compartments in
  `data_model_to_pymcell` (doc 04 §6.2). On MCell4's MDL path the vertex indices of every wall are
  checked first, right after MCell3's parser and before MCell3's initialization
  (`MCell3WorldConverter::check_vertex_indices_before_mcell3_init`, `mcell3_world_converter.cpp:528`,
  called from `src/mcell.c:74`).
- **The geometry report and the one stop** (`GeometryReport`, `src4/mesh_utils.hpp:50`): every run
  writes `reports/geometry_report_<seed>.txt`, every problem gets one terminal line, and a model
  with errors stops once, after every object is checked (doc 04 §6.3).
- **Wall links and region borders** come from the table, by the same code for both converters
  (`GeometryObject::link_walls`, `geometry.cpp:60`; `Region::init_border_edges`, `:589`; doc 04
  §6.4).
- **What each object keeps** (`GeometryObject::mesh_check`, `geometry.h:136`): the table, the
  flags, the shells, the area and the signed volume (recomputed after vertex moves), and per-vertex
  lists. Counted volumes, compartments, releases, the BNGL export, the getters, the OBJ export,
  vertex moves, vertex normals, the corner-tile search and region-border crossings read it (doc 04
  §6.5).
- **The usage errors** (`MeshUtils::check_usage`, `mesh_utils.cpp:248`), at the end of both
  converters, and **one function for every clamp event** (`create_clamp_release_events`,
  `clamp_release_event.cpp:268`; doc 04 §6.3, §6.6).
- **Gone:**
  - `surface_net`, with `compatible_edges`, `refine_edge_pairs` and `have_common_region`;
    `initialize_neighboring_walls_and_their_edges` and `init_surface_region_edges`; and in
    `MCell3WorldConverter`, the copying of MCell3's wall constants, edges, neighbours and region
    borders (step 6, `7a5b6565`);
  - the copying of MCell3's release walls, areas and boxes and of its clamp walls, with
    `convert_concentration_clamp_release` and `update_and_schedule_concentration_clamps` (step 6b,
    `72cb4d9c`);
  - `get_connected_vertices` and the partition's lists of walls per vertex,
    `find_shared_edge_index_of_neighbor_wall`, `compute_geometry_object_area`,
    `add_or_find_geometry_vertex`, and `GeometryAnalysis::is_closed`, `is_watertight`,
    `has_non_manifold_edge`, `count_inconsistent_edges` and `count_duplicate_points` (step 7,
    `6ef31aaf`);
  - `Region`'s volume (step 7a, `84765c8e`);
  - `MeshUtils::check_meshes`, `check_release_regions` and the mesh-check warnings of the VTK
    removal (step 8, `33ff673d`);
  - `GeometryAnalysis::clean_mesh` and `MeshUtils::get_clean_mesh` (step 9, `620451de`). Nothing in
    MCell merges coordinates any more;
  - the Python constructor's range test of `wall_list`, which never ran (step 12b, `1bb0f809`).
- **Conventions:**
  - ids are 0-based positions in the model's lists; side k of a wall runs from its vertex k to its
    vertex (k + 1) % 3;
  - two walls are neighbours only when a side of each uses the same two vertex indices;
  - "watertight" means that every edge has exactly two walls;
  - a signed volume is computed only for a watertight mesh or shell;
  - no input index below 0, or not below its list's length, is accepted;
  - errors stop the run, warnings don't, and no mesh is repaired.

## 5. What users see differently

"Before" is `mcell` `0652d19b`, "now" `1bb0f809`. The cases of the two tables were run on the build
from before the project and on this project's builds (§6); the others come from the runs of the
step that changed them.

**Every model that MCell4 converts** (Python API models, and MDL files on MCell4's MDL path):
- **Walls are joined only through shared vertex indices.** Where a model's indices and coordinates
  agree, the links are as before, bit for bit (§6). Where they don't, the mesh is different: a cube
  whose faces each have their own four vertices was joined by coordinates; now it is non-watertight
  (24 boundary edges), with a warning for 8 groups of equal coordinates.
- **Every run writes a geometry report**, `reports/geometry_report_<seed>.txt`.
- **Mesh errors stop the run** once every object is checked: a `ValueError` for the Python API, exit
  code 1 on MCell4's MDL path. Warnings don't stop it. The messages are in the tables below.
- **Region borders of surface molecules follow one rule,** `ALL` included: a side of a wall of the
  region is a border when its edge has exactly one wall in the region.
  - So `ALL`, or a region of every wall, of a non-watertight object now has the mesh's boundary
    edges as borders, where an ABSORPTIVE rule for a surface molecule absorbs. Before, the Python
    API gave such a region its boundary edges but not `ALL`, and MCell4's MDL path had MCell3's
    borders, which leave out both.
  - Test `mdl/2775`: 1,000 surface molecules on a box without its top face, with the class on
    `ALL`, drop to 704 on MCell4, through either front end, and stay at 1,000 on MCell3's engine. No
    test from before the project has the case.
- **TRANSPARENT lets surface molecules pass a region border**; it absorbed them (§8).
- **Clamps of volume molecules** run in the order of surface class name, molecule name and side, for
  both front ends. A Python API model with two or more acting clamps therefore draws other random
  numbers than before (test 1700, below). A wall clamped twice for the same volume molecule and side
  stops the run.
- **A negative concentration (volume molecules) or density (surface molecules) removes molecules,**
  truncated toward zero, as on MCell3's engine; it stopped MCell4 (§8).

**The mesh checks** (no molecules involved; doc 04 §6.9 has every text):

| Case | Before | Now |
|---|---|---|
| A vertex index out of range, a negative wall index in a surface region, or lists changed through setters (Python API) | Not checked. The run went on (an index equal to the vertex count, a negative wall index in a surface region), or stopped with an unrelated error such as "Non-manifold mesh is not allowed" or "Detected malformed geometry object". | error `MESH_INVALID_INDICES` |
| A vertex index equal to the vertex count, or below 0, in an MDL file | Both paths: MCell3's initialization crashed (segmentation fault). An index above the count got MCell3's message "object Scene.box has elements with out of bounds vertex indices". A wall that `REMOVE_ELEMENTS` removes was never checked. | MCell4's MDL path: error `MESH_INVALID_INDICES` for any wall, before MCell3's initialization. MCell3's engine: as before |
| A wall that uses one vertex twice | Python API: no message. MCell4's MDL path: MCell3's initialization drops the wall as degenerate, then MCell4's converter crashed. | error `MESH_INVALID_INDICES` |
| Two walls on the same three vertices | Python API: "Cannot allow overlapped wall ...". MCell4's MDL path: crash. | error `MESH_DUPLICATE_WALLS` |
| A zero-area wall | Python API: "Detected malformed geometry object" when it shares an edge. MCell4's MDL path: crash. | error `MESH_ZERO_AREA_WALLS` |
| An edge of three or more walls | Python API: `ValueError: Non-manifold mesh is not allowed, error for <name>.` MCell4's MDL path: no message. | error `MESH_NOT_MANIFOLD` |
| Groups of walls that touch only at a vertex | No message. | error `MESH_NOT_MANIFOLD` |
| Two walls that run along their shared edge in the same direction | Python API: a warning for counted objects, compartments and the objects of releases by concentration; no message for the others. MCell4's MDL path: crash. | error `MESH_INCONSISTENT_ORIENTATION`, for every object |
| A vertex that no wall uses | No message. | error `MESH_ORPHANED_VERTICES` |
| Vertices with equal coordinates | Warning: "MCell joins walls by coordinates, so it treats such vertices as one vertex." | warning `MESH_DUPLICATE_VERTICES`; the walls aren't joined |
| A non-watertight object | No message. | warning `MESH_NOT_WATERTIGHT` |

For MDL files, MCell3's parser runs first on both paths, as before. It stops malformed lists, and
wall indices of a surface region or of `REMOVE_ELEMENTS` out of range, negative ones too, which its
message shows as 4294967295. On MCell4's MDL path, MCell3's initialization runs after MCell4's
check of the vertex indices; where an object is also counted inside or released into, it may stop
first with its own message (doc 04 §6.2). On MCell3's engine, a vertex index equal to the vertex
count, or below 0, still crashes MCell3's initialization (doc 04 §6.12).

**The uses of volume molecules** (the usage errors; doc 04 §6.3):

| Case | Before | Now |
|---|---|---|
| Counting volume molecules inside a non-watertight object | Python API: "Counting inside an open surface is not possible, error for <name>." and "Processing of counted volumes failed, terminating."; the Python process ended with exit code 1. MCell4's MDL path: MCell3's initialization stops first. | error `REGION_NOT_WATERTIGHT` |
| Counting inside an inside-out object, or an inside-out compartment | Warning `MESH_INSIDE_OUT`, and the run went on; a counted object's crossings were counted the wrong way round (§1). | error `MESH_INSIDE_OUT` |
| A non-watertight compartment (Python API) | `RuntimeError: Compartment object must be closed, error for <name>.`, for the first one. | error `REGION_NOT_WATERTIGHT` for each, then one stop |
| Releasing volume molecules into a non-watertight object | Python API: warning `REGION_NOT_CLOSED`. By number, the molecules were placed by ray parity; by concentration, the number came from a signed volume that such an object doesn't have (401 volume molecules in a cube with 2 um edges without its two top walls, where the whole cube gets 482). MCell4's MDL path: MCell3's initialization stops first. | error `REGION_NOT_WATERTIGHT`, also in `Model.release_molecules` |
| A release of volume molecules by concentration into an inside-out object | Warning `MESH_INSIDE_OUT`, then "tries to release more than INT_MAX" when the release runs. | error `MESH_INSIDE_OUT`, at initialization |
| An object watertight only by coordinates (each face with its own vertices), counted or released into | Used as watertight, as on MCell3's engine: 482 volume molecules at 1e-4 M in a cube with 0.2 um edges. | error `REGION_NOT_WATERTIGHT` |

**Python API models** (`MCell4Converter`; also the models that `data_model_to_pymcell` generates):
- The checks run at `Model.initialize()`, on the lists as they are then. The constructor's range test
  of `wall_list`, which sat after its `throw` and never ran, is gone (step 12b); its other checks are
  unchanged.
- `Model.add_vertex_move` raises `RuntimeError: Vertex index -1 is out of range for vertex_list of
  box.` for an index below 0 or not below the object's vertex count, as the getters do; it ended
  Python (§8).
- The getters' message for a wall index out of range names the wall list: `Wall index -1 is out of
  range for wall_list of box.` (§8).
- Every error raises a `ValueError` once all its lines are printed. No error ends the Python process
  any more.
- `Model.release_molecules` checks its release as the converter does; on an error it raises and
  releases nothing.
- New getters: `Model.get_object_area(object)` in um^2 and `Model.get_object_volume(object)` in um^3,
  signed, at the walls' current positions. The volume raises a `ValueError` for a non-watertight
  object.
- `Model.export_geometry` writes each object as given: every vertex of `vertex_list` in order, at its
  current position, and the walls of `wall_list` as the faces. The first line of each file names
  MCell. Before, the vertices were renumbered in the order the walls first use them, and equal
  coordinates were merged.
- After a vertex move, releases use the moved geometry (volume and surface molecules), and in models
  with intersecting counted objects the counted volumes come from the current walls (§8).
- `Model.get_vertex` checks its argument like the other getters (§8).
- A clamp class on two regions that share no wall clamps each wall once, as before, with other
  random draws, now the same as the MDL file's on MCell4's MDL path. On two regions that share
  walls, the run stops.

**MCell4's MDL path** (`mcell -mcell4`, also `-mdl2datamodel4`; `MCell3WorldConverter`):
- MCell3's parser runs first. Then MCell4 checks every wall's vertex indices, also those of the
  walls that `REMOVE_ELEMENTS` removes, and stops on one out of range with `MESH_INVALID_INDICES`
  and a report of the objects with an index problem. Then MCell3's initialization runs. Where
  MCell3's parser or initialization stops a file, the user sees MCell3's message and gets no report
  (doc 04 §6.2). The converter's checks run on the walls that MCell3's parser stored, with one
  MCell4 vertex per MCell3 vertex.
- No more crashes on a vertex index out of range, degenerate walls, `REMOVE_ELEMENTS` or edges that
  MCell3 left unlinked (§8).
  With `REMOVE_ELEMENTS` the model runs as on MCell3's engine, and the report keeps the file's wall
  ids.
- MCell4 links the walls and computes the wall constants, the region borders, the walls and areas of
  releases onto regions, the boxes of releases into regions and the walls of clamps itself. On every
  model of the suite the neighbours, shared edges, wall constants, release walls, areas and boxes,
  and clamp walls came out bit for bit as MCell3's (§6); the region borders differ by design
  (above).
- `DYNAMIC_GEOMETRY` stops the run; MCell4 ignored it before.
- `-mdl2datamodel4` runs the same checks, with the usage errors, and writes no data model when they
  fail. It also stops on `DYNAMIC_GEOMETRY`; before, it exported every object as not dynamic.
- Still from MCell3's initialization: the partition's placement and the random-number state. A region
  other than `ALL` in a release of volume molecules still becomes the whole object (doc 04 §6.12).

**`data_model_to_pymcell`:**
- It computes a BNGL compartment's volume after `check_mesh` on the data model's lists; a wall that
  uses one vertex twice now gives "Geometry object <name> has invalid vertex indices, could not
  compute volume." (§8).
- A data model with a dynamic object stops before any file is written; it ran with fixed geometry
  (§8).

**MCell3's engine** (`mcell file.mdl`) is unchanged, and none of MCell4's code runs there: a vertex
index equal to the vertex count, or below 0, still crashes it (doc 04 §6.12). Where its results
differ from MCell4's by design, doc 04 §6.12 says so, as for the region borders above.

**Existing tests whose stored outputs changed** (each change was shown before the stored files were
replaced):
- `pymcell4_positive/3100_export_geometry_obj` and `3120_dyn_geom_export_geometry_obj`: the OBJ
  export as given (step 9).
- `pymcell4/3200_sphere_collision_against_each_other`,
  `3205_sphere_collision_against_each_other_rand_order` and private
  `hebrew_univ_jerusalem/pymcell4/5120_move_sphere_w_elasticity`: their releases at time 0 run after
  the first vertex move, and now draw positions from the moved box. The volume molecules' positions
  moved by 0.01 um; their numbers didn't change (step 7a).
- `pymcell4_positive/1700_linear_conc_gradient_w_conc_clamp` (volume molecules): the new clamp order
  changed its random draws, and its check that each bin holds more than the next failed in one round
  of ten. The check was fragile: neighbouring bins were about 25 molecules apart, with a noise of
  about 10. The test got a ten times steeper gradient and visualization every 1,000 iterations
  (step 6b).
- The BNGL tests (step 7v, tests only): 578 reference frames deleted, and new last frames for
  `bngl/0960`, `0970` and `0980`, which have no observables.
- The test program's report mesh `removed_walls` (step 12b): its wall 13, a removed wall with a
  vertex index out of range, went; such a wall is now reported, and the new report mesh
  `removed_wall_out_of_range` tests it.

## 6. How it was checked

- **Prototypes first.** Python prototypes in the build root (`possible_implementation/`):
  `coboundary.py` builds a mesh's sparse operators with SciPy (edge–vertex, wall–edge and
  wall–vertex incidence) and classifies its edges and vertices, and two companions check its
  orientation; `edge_table_proto.py` writes the table's C++ design (`edge_table_cpp.md`) as plain
  loops; `check_edge_table_proto.py` compares the two on 131 meshes. They never enter a repository.
- **The step-1 scan** (`vertex_index_spike/step1_scan/`): a standalone C++ copy of every planned
  check ran on every mesh of the suite, the benchmarks and the private models, extracted as MCell4
  receives them: 605 mesh files, 8,090 objects, 18,339,737 walls. It agreed with `coboundary.py` and
  NumPy on every quantity, and showed that no test changes outcome.
- **The test program** `mesh_utils_test` (doc 04 §6.11): 137 connectivity meshes with the
  prototype's results, computed once outside the repositories and stored next to the meshes, and 30
  report meshes with hand-checked report blocks and terminal lines; 3,534 checks. Changed expected
  values fail it. On real meshes, `check_mesh` agreed with the scan on all 8,090 objects.
- **Comparison builds before each switch.** Temporary code computed the new values next to the old
  ones at every use, over the whole suite, and wrote every difference to a file.
  - Step 6, the links: 11,018 object conversions. The neighbours, the shared edges and their
    constants, and (on MCell4's MDL path) MCell4's wall constants against MCell3's: the same, bit for bit.
    715 regions gained borders; each holds every wall of a non-watertight object, and no test reads
    them.
  - Step 6b, MCell4's MDL path: 1,110 releases onto regions (walls, their order, cumulative
    areas), 748 releases into regions (boxes) and 87 clamps (walls): the same, bit for bit.
  - Step 7, the kept facts: 8,357 object conversions with 11.85 million walls and 35.5 million linked
    sides; every counted object, compartment, release check, release by concentration, BNGL export
    and data model; the lookups of vertex moves (496,824 of each kind) and 1,648,774 region-border
    crossings of surface molecules: no difference.
- **The gates.** After every code step the full suite (`test_configs/default.toml`, with
  `mcell_tests_private`) ran, and was compared with the previous run per test status, per output
  file and, from step 5, per terminal line and geometry report against what the step-1 scan
  predicted. Every difference was explained before the next step.
  - Apart from an NFS timeout of test 2500, three changes of status came up. Each was resolved by
    the test changes listed in §5: 1700 in step 6b; `bngl/0960`, `0970` and `0980` in step 7v; 3200,
    3205 and 5120 in step 7a.
  - From the baseline to the last gate (`step12b2`) no test changed status (§7).
- **The suite moved to the local disk** in step 7. On NFS it took 17 to 40 minutes, and test 2500,
  which writes 50,000 visualization files, timed out; from a copy on the machine's local disk it
  takes about 5.5 minutes.
- **The new tests fail when their expectations change.** Each test of step 11 was changed once, and
  all 14 runs failed at the changed check. Test 1580 also fails with the behaviour from before its
  fix (the partition's count of vertex moves set back to 0 in gdb). The same holds for step 12b's
  tests 0270 and 1360 and the test program's new and changed report meshes; 1360 also aborts on
  the build from before the project.
- **Small models for every decision,** run on MCell3's engine, on MCell4's MDL path and through the
  Python API (`vertex_index_spike/experiments/`, a README in each folder). For this record, the
  mesh cases and the usage cases of those experiments also ran on the build from before the
  project (`vertex_index_spike/experiments/step12a_docs/`).
- **The full neuropil** (private `mdl/3650`, skipped in the suite) stops on its two faults (doc 04
  §6.12). With them repaired in a copy, its count file and the position of every volume and surface
  molecule at iterations 0 and 50, about 3 million each time, are byte for byte those of the build
  from before the project.
- **The docs:** every `file:line` citation of doc 04 §6, §3.7 and this record was read against its
  line; with step 12b, every citation into a file the project changed, in every doc of the wiki,
  was checked against its line at `1bb0f809`, also those in code samples and in backtick spans
  that cross a line. Every quoted message was found in the source.

## 7. The numbers

| | Before (`0652d19b`) | Now (`1bb0f809`) |
|---|---|---|
| **Full suite** (`default.toml`, with `mcell_tests_private`) | | |
| Passed, failed | 2,277, 376 | 2,371, 376: no test changed status; 94 new test runs, from 36 new tests |
| Wall time | 17 min 21 s, on NFS | 5 min 27 s, from the local disk |
| Visualization output of a run | 28.5 GB in 411,109 files | 15.9 GB in 222,723 files (step 7v) |
| **Code** | | |
| `mcell.so` | 6,400,024 bytes | 6,584,888 bytes (+2.9%) |
| `mcell`, `0652d19b..1bb0f809` | | 65 files, 4,064 lines added and 1,434 removed; `mesh_table.{hpp,cpp}` 1,471 lines, `mesh_utils_test.cpp` 715 |
| `mesh_utils_test` | | 137 + 30 meshes, 3,534 checks, 0.06 s |
| `check_mesh` alone: the 8,090 objects of the step-1 scan; the full neuropil | | 8.0 s; 1.5 s |
| **Full neuropil** on MCell4's MDL path (1,763 objects, 3,990,190 walls, 50 iterations; the repaired copy; CPU seconds per phase; measured at `e62b9af8`, before step 12b's one pass over the walls' indices) | | |
| MCell3's parser | 6.52 | 6.49 |
| MCell3's initialization | 38.59 | 39.03 |
| MCell4's conversion | 27.88, of which the releases 0.08 | 29.06, of which the checks 1.69 and the releases, with MCell4's own walls and areas, 0.52 |
| MCell4's initialization | 12.54, of which the old mesh-check warnings 1.36 | 11.05 |
| The 50 iterations | 113.54 | 114.07 |
| Whole run, median of 3: wall, user, peak memory | 198.7 s, 190.1 s, 10,334 MB | 201.3 s, 192.5 s, 10,464 MB (+1.3%) |
| Memory held after the conversion | | 156 MB more, about 39 bytes per wall (the kept checks and MCell4's own release walls), against about 50 bytes per wall estimated when it was decided |

The suite's wall time is set by one MCell3 run, `mdl/4202_lotka_voltera_reaction_limited_it_400_long`
(315 s), so less visualization output saved CPU time (4%) but not wall time. The neuropil's 10.4 GB
peak predates the project: MCell4's MDL path keeps MCell3's model, about 4.4 GB of it, through the
whole run (§9).

## 8. Fixed along the way

Twenty-five issues, all from before the project, grouped by who saw them. Each says what was
wrong, the step and commit that fixed it, and the test that covers it.

### MCell4's MDL path (`MCell3WorldConverter`)

1. **A degenerate wall crashed MCell4.** MCell3's initialization drops a zero-area wall, or a wall
   that uses one vertex twice, and leaves a null wall pointer, which the converter followed
   (segmentation fault). Step 5 (`e530244c`): such a wall is reported, `MESH_ZERO_AREA_WALLS` or
   `MESH_INVALID_INDICES`. Test: `nutmeg4_mdl/0240_mesh_errors_error`.
2. **`REMOVE_ELEMENTS` crashed MCell4** in the same place. Step 5: the removed walls are left out, and
   the model runs as on MCell3's engine. Tests:
   `mdl/1340_remove_elements_surf_mols_on_region_vol_mols_from_point_it_100`,
   `nutmeg4_mdl/0230_orphaned_vertices_error`.
3. **An edge that MCell3 left unlinked crashed MCell4.** MCell3's initialization leaves an edge
   without its record where two walls run along it in the same direction (`mdl/2810`, a cube or a
   plane with one wall reversed), and where two walls have the same three vertices; the converter
   followed the null pointer. Step 5: `MESH_INCONSISTENT_ORIENTATION` or `MESH_DUPLICATE_WALLS` stop
   the run first, and since step 6 MCell4 links the walls itself. Tests: `nutmeg4_mdl/0240`;
   `mdl/2810` (todo) now stops with the orientation error.
4. **A vertex index equal to the vertex count, or below 0, crashed MCell4's MDL path** in MCell3's
   initialization, before the converter: `instance_polygon_object` stops only on an index above the
   count ("Error: object Scene.box has elements with out of bounds vertex indices") and then reads
   each wall's vertices through its indices. The indices of a wall that `REMOVE_ELEMENTS` removes weren't
   checked at all. Step 12b (`1bb0f809`): right after MCell3's parser, before MCell3's
   initialization, MCell4 checks the vertex indices of every wall, removed or not, and stops with
   `MESH_INVALID_INDICES`, also with `-mdl2datamodel4`; `check_parsed_walls` reports a removed wall
   with an index out of range too. MCell3's engine still crashes on such a file. Tests:
   `nutmeg4_mdl/0270_vertex_index_out_of_range_error`; the test program's report mesh
   `removed_wall_out_of_range`.
5. **Valid models were rejected when a region carried MCell3's bounding box** without holding every
   wall: `Conversion error: Expected 'r->region_has_all_elements || r->bbox == nullptr' is false.`,
   and then the run crashed (item 6). MCell3's initialization gives a box to the first region of
   every object, which is `ALL` only after an odd number of copies (each instantiation copies an
   object's regions in reverse), and to the regions of releases and counts of volume molecules; and
   with `REMOVE_ELEMENTS` even `ALL` doesn't hold every wall of the file. MCell4 never used the box.
   Step 5: the check is gone, and such models run with MCell3's counts. No test of its own.
6. **A failed conversion of a meta object was ignored**, and the run crashed later in the check of
   overlapped walls. Step 5: both calls are checked, and such errors stop with their
   `Conversion error:` lines and exit code 1. No test of its own: no test model fails that conversion.
7. **`DYNAMIC_GEOMETRY` was dropped silently.** The converter never read MCell3's list of geometry
   changes, so the run kept its first geometry and exited with 0. Step 8 (`33ff673d`): the run stops,
   also with `-mdl2datamodel4`. Test: `nutmeg4_mdl/0220_dynamic_geometry_error`.

### Python API models (`MCell4Converter`, `Model`)

8. **Indices weren't checked** (findings 2 and 4). The constructor's range test never runs, negative
   wall indices of a surface region pass it, and setters bypass the constructor. An index equal to
   the vertex count, a negative wall index in a surface region or a wall that uses one vertex twice
   then ran without a message; other cases gave an unrelated error (non-manifold, overlapped walls,
   malformed geometry). Step 5: the index checks at `initialize()`, on the lists as they are then;
   the constructor's dead range test went in step 12b. Test: `nutmeg4_pymcell4/0780_mesh_errors_error`
   (with setters).
9. **`get_geometry_bounding_box` read vertices without three coordinates out of bounds**; it runs
   before the checks. Step 5: it skips them, and the checks report them. Test: `nutmeg4_pymcell4/0780`.
10. **A wall was clamped twice** (volume molecules) when its clamp class reached it through two
    regions, since the converter made one clamp event per region. In the case checked, the model held
    about a sixth more volume molecules than the same MDL file on MCell4's MDL path (21,979 against
    18,991). Step 6b (`72cb4d9c`): one event per clamp, each wall once, and a wall clamped twice
    stops the run. Tests:
    `mdl/2690_conc_clamp_two_separate_regions`, `2691_flux_clamp_two_separate_regions`,
    `nutmeg4_pymcell4/0720_clamp_on_wall_twice_error`, `nutmeg4_mdl/0200_clamp_on_wall_twice_error`.
11. **Releases after vertex moves used the geometry from before the move.**
    - Volume molecules: a region's volume was computed at its first use and never again, so a release
      by concentration after a move that doubled a cube gave 602 or 1,204 molecules, depending on an
      earlier release; the boxes and the inside test's data went stale, so the molecules landed only
      where the object had been.
    - Surface molecules: the walls' cumulative areas were computed once, at conversion; by density, 400
      instead of 800 molecules landed on the stretched sides.
    - In models with intersecting counted objects, a volume molecule's counted volume after a move came
      from the waypoints of initialization.

    Step 7a (`84765c8e`). Tests: `pymcell4_positive/1560_release_into_moved_object_vol_mols`,
    `1570_release_onto_moved_region_surf_mols`,
    `1580_count_in_intersecting_objects_after_vertex_move`.
12. **`Model.get_vertex` crashed Python** (segmentation fault) before `initialize()` or with `None`: it
    skipped the argument check of the other getters. Step 10 (`46afdfd1`). Test:
    `pymcell4_positive/1340_get_vertex`.
13. **`Model.add_vertex_move` ended Python** (exit 134) for -1 on the model's first object and for an
    index past the model's last vertex: a `release_assert` came before the argument check. Step 12b:
    the getters' check, `RuntimeError: Vertex index -1 is out of range for vertex_list of box.` Test:
    `pymcell4_positive/1360_index_argument_errors`.
14. **The getters' message for a wall index named a vertex:** `get_wall`, `get_wall_unit_normal`,
    `get_wall_color`, `set_wall_color` and a checkpointed surface molecule's `wall_index` said
    "Vertex index -1 is out of range for vertex_list of box." Step 12b: "Wall index -1 is out of
    range for wall_list of box." Test: `pymcell4_positive/1360`.
15. **The API docs had errors.** `get_molecule` was said to return `None` for a molecule that doesn't
    exist (it raises a `RuntimeError`); `get_vertex`'s `vertex_index` was said to index `wall_list`;
    three methods called their `wall_index` the index of a vertex; `set_wall_color` described its
    object wrongly; and the dynamic-geometry guide said that `get_vertex` returns a `Vec3` (it returns
    a list). Step 10.
16. **The release error "is not closed" never fired**: the function whose result it tested always
    returned true. Step 8: a release of volume molecules into a non-watertight object stops with
    `REGION_NOT_WATERTIGHT`, and the dead error is an assertion. Test:
    `nutmeg4_pymcell4/0730_usage_errors_inside_out_and_non_watertight_error`.
17. **`refine_edge_pairs`**, which chose which two walls to link on an edge of three or more walls, read
    the normals of the wrong walls: the partition's walls at the object's own wall numbers. It went
    with `surface_net` in step 6 (`7a5b6565`); such edges are errors now.

### Both converters (MCell4's engine)

18. **TRANSPARENT absorbed surface molecules at region borders.** MCell4 took every surface-class rule
    without products as absorptive at a border, and a TRANSPARENT rule has none. On a box without its
    top face, with the class on its bottom face, 1,000 surface molecules went to 84, as with
    ABSORPTIVE; MCell3's engine keeps 1,000. Step 6a (`97b2f5c5`):
    `RxnUtils::is_absorptive_at_region_border` checks the rule's type first (`rxn_utils.inl:48`).
    Tests: `mdl/2785_surf_transparent_open_box_bottom`, `2786_surf_transparent_open_box_all`,
    `2787_surf_transparent_products_across_border`.
19. **A negative concentration stopped MCell4** (volume molecules) with "tries to release more than
    INT_MAX molecules"; MCell3's engine removes them. Step 8. Test:
    `mdl/0390_negative_vol_conc_release_it_10`.
20. **A negative density stopped MCell4 the same way** (surface molecules). Step 8. Test:
    `mdl/1330_surf_mol_on_cube_negative_density_release_it_10`.
21. **A release of volume molecules by concentration into an inside-out object** stopped when it ran,
    with the INT_MAX message: its signed volume is negative. Step 8: `MESH_INSIDE_OUT` at
    initialization. Tests: `nutmeg4_mdl/0210_usage_errors_inside_out_and_non_watertight_error`,
    `nutmeg4_pymcell4/0730`, `0750_release_molecules_usage_error`.
22. **A vertex that no wall uses was accepted,** and `Model.get_vertex_unit_normal` then raised
    "Internal error: there are no walls that use vertex ..." for it. Step 10a (`e62b9af8`):
    `MESH_ORPHANED_VERTICES`. Tests: `nutmeg4_mdl/0230_orphaned_vertices_error`,
    `nutmeg4_pymcell4/0770_orphaned_vertices_error`.
23. **`Partition::add_or_find_geometry_vertex` was unused.** Removed in step 7 (`6ef31aaf`).

### `data_model_to_pymcell`

24. **Dynamic objects ran with fixed geometry:** the converter never read `"dynamic"`. Step 8: it stops
    before writing any file. Test: `nutmeg4_pymcell4/0760_data_model_dynamic_object_error`.
25. **A wall that uses one vertex twice was dropped silently** when a BNGL compartment's volume was
    computed. Step 7 (`6ef31aaf`): "Geometry object <name> has invalid vertex indices, could not
    compute volume." Test: `nutmeg4_pymcell4/0800_data_model_compartment_mesh_error`.

## 9. What is left, and where the records are

- **The project ended after step 12b** (the user, 2026-11-24). The planned steps 13-17 aren't about
  connectivity by vertex index, so they left the project. With every other open issue found along the
  way, they are listed for later projects in `TODO_open_issues.md` (step 13's investigation of
  surface classes in its §1). The steps, numbered as in the plan (doc 04 says "Planned" in words):
  - **13:** MCell4's own checks for conflicting surface classes on one wall, for both converters (doc
    04 §6.12, "Conflicting surface classes on one wall").
  - **14:** clamps in exported data models from the model's clamp definitions, one entry per class,
    and then no clamp event without walls (§6.12, "An exported data model lists a clamp class once per
    clamp event").
  - **15:** effects within an iteration: a seeded shuffle of the clamps, the order of new volume
    molecules, the two shuffle loops (with the crash of `Model.apply_vertex_moves()` with no moves),
    and the barriers' effect on clamp concentrations (§3.7, "Barriers change trajectories" and "The
    order within an iteration").
  - **16:** the clamp's volume molecule that leaves a watertight box (§3.7).
  - **17:** the wall–wall check of vertex moves independent of the order in which it visits a vertex's
    neighbours: every edge tested, every crossing reported, nothing sorted (§6.12, "The wall–wall hit
    reported for a vertex move depends on the vertex numbering").
- **Planned work that isn't an issue** (doc 04 §6.12):
  - MCell4's MDL path without MCell3's initialization. This would also free the memory of MCell3's
    model, which MCell4's MDL path keeps through the whole run: about 4.4 GB of the full neuropil's
    10.4 GB peak. MCell4's `Wall` is 520 bytes, MCell3's `struct wall` 248.
  - A check for walls that cross at an angle.
  - Parallel checks, if ever needed: split by object, and keep sums in a fixed order.
  - The meaning of the marks `S'` and `S,` for surface molecules, to be redesigned.
- **Known issues:** doc 04 §6.12 (geometry, and MDL files on either path) and §3.7 (the engine).
  - MCell3's engine still crashes on a vertex index equal to the vertex count, or below 0, as before
    the project: MCell3's code stays unchanged, and MCell4's check runs only on MCell4's MDL path
    (§8, item 4).
  - 376 test failures predate the project and weren't investigated, as in the VTK removal; the
    gates therefore compared output files as well as statuses.
- **The merge** (2026-11-24), as the VTK removal did:
  - `mcell` (`1bb0f809`) and `mcell_tests` (`2272e059`) fast-forwarded `mcell4_nanobind`, and the
    `mcell4_nanobind` build root pulled them;
  - the two `mcell_tests_private` commits (`10e886d`, `d15b60a`) went to GitHub on that repository's
    branch `mcell4_nanobind_vertexidx`, which the `mcell4_nanobind` root now uses: its `master` lacks
    them, and lacks the 17 commits of `mcell4_dev` that both projects ran with;
  - these docs went into the `mcell4_nanobind` root's wiki;
  - the branches `mcell4_nanobind_vertexidx` stay on GitHub as a record.
- **The working records,** in the `mcell4_nanobind_vertexidx` build root, outside any repository:
  - `PLAN_vertex_index.md`: every decision in its wording, the steps, the results, the known issues
    and the TODOs;
  - `NEXT_SESSION_PROMPT.md`: the handover;
  - `MCELL4_GEOMETRY_VALIDATION_FINDINGS.md`: the four findings of §1;
  - `possible_implementation/`: the prototypes and the table's design;
  - `vertex_index_spike/`: the step-1 scan, the generator of `mesh_utils_test`'s data, and the
    experiments behind each decision, with this record's runs of the build from before the project;
  - `vertex_index_runs/`: every gate's log and comparisons, and the comparison scripts. Of the gates'
    work folders only the last one, `step12b2`'s, is kept, on the local disk of the machine that ran
    it; the others were deleted when the project ended.
