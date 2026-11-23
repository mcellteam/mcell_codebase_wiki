# MCell Core and Dependencies

> Developer reference for the MCell simulator core: the two engine generations
> (`src/` = MCell3, `src4/` = MCell4), the MCell4 main loop and core objects, and
> how the C++ engine consumes its two external dependencies (libbng, NFsim), plus the geometry
> code: the mesh checks, connectivity from vertex indices, and the in-tree geometry analysis that
> replaced VTK (§6).
>
> Companion documents (written by other agents):
> - `01_libbng.md` — internals of the BNG library (`BNG::` namespace).
> - `02_nfsim.md` / `03_nfsimCInterface.md` — internals of NFsim and its C wrapper.
> - The Python API layer (`libmcell/`, code-generated bindings) is documented separately.
>
> This document describes the dependencies **from the MCell side** — how `src4` and
> `src` *call into* them — and does not re-derive their internals.

---

## 1. Two-Engine Architecture

MCell ("Monte Carlo Cell") is a 3D spatial, particle-based reaction-diffusion
simulator. The tree contains **two generations of engine** that coexist and are
linked together:

| | **MCell3 (legacy)** | **MCell4 (current)** |
|---|---|---|
| Directory | `mcell/src/` | `mcell/src4/` |
| Language | C, compiled as C++ (`-std=gnu++17`) | Modern C++17 (the geometry `.cpp` files of §6 are C++23) |
| Driven by | MDL text files (flex/bison parser) | Python API / BNGL (via libbng) |
| Master state | `struct volume` (god-struct, `src/mcell_structs.h:1052`) | `class World` (`src4/world.h:58`) |
| Species/rxn storage | hand-rolled hash tables in `volume` | delegated to `BNG::BNGEngine` |
| Network-free rxns | NFsim (`src/*_nfsim.c`) | **not used** |
| Build artifact | the `mcell` executable | the `mcell.so` Python module (`mcell4_so`) and also the executable's run engine |

### 1.1 How they relate

The two engines are **not** independent simulators that you pick between at the
top. In the current design:

- **MCell3 is the MDL front-end.** When you run the `mcell` executable on an `.mdl`
  file, MCell3 parses it (flex/bison) and fully initializes its `struct volume`.
- **MCell4 is the execution engine.** The fully-built `volume` struct is then
  *converted* into an MCell4 `World` and the MCell4 engine runs the actual
  simulation.

The bridge between the two lives in `src4/`:

| File | Role |
|---|---|
| `src4/mcell3_world_converter.h/.cpp` | `class MCell3WorldConverter` — reads the MCell3 `volume` struct and builds an equivalent MCell4 `World` object graph (`mcell3_world_converter.h:39`). |
| `src4/mcell4_iface_for_mcell3.h/.cpp` | Thin **C-callable shim** so the legacy C code in `src/mcell.c` can hand off to the C++ MCell4 engine. Holds the single global converter `g_converter` (`mcell4_iface_for_mcell3.cpp:17`). The header only includes `mcell_structs_shared.h` so it is includable from C. |

`mcell4_iface_for_mcell3.h` exposes five functions (`mcell4_iface_for_mcell3.h:20-24`):

```c
void mcell4_check_vertex_indices(struct volume* s);                  // before MCell3's initialization (§6.2)
bool mcell4_convert_mcell3_volume(struct volume* s);                 // volume -> World
bool mcell4_run_simulation(bool dump_initial_state, bool dump_with_geometry=false);
void mcell4_convert_to_data_model(bool only_for_viz);                // World -> data_model.json
void mcell4_delete_world();
```

### 1.2 The MCell3WorldConverter pipeline

MCell4's MDL path is an MDL file run with `mcell -mcell4` (also `-mdl2datamodel4`). On it,
`main` (`src/mcell.c`) runs MCell3's parser (`:68`); then MCell4's check of every wall's vertex
indices (`mcell4_check_vertex_indices`, `:73-75`, which calls
`MCell3WorldConverter::check_vertex_indices_before_mcell3_init`, `mcell3_world_converter.cpp:528`;
§6.2); then MCell3's initialization (`src/mcell.c:83`); and then the converter (`:99`). MCell3's
engine (an MDL file run without `-mcell4`) runs the parser and the initialization, and none of
MCell4's code.

`MCell3WorldConverter::convert(volume* s)` (`mcell3_world_converter.cpp:103`) runs an
ordered pipeline that reads MCell3 structs and creates MCell4 objects:

| Step | Method | What it creates or checks |
|---|---|---|
| 0 | the `DYNAMIC_GEOMETRY` stop (`:108-112`) | stops when MCell3's parser scheduled dynamic geometry, which MCell4 doesn't support (§6.6) |
| 1 | `convert_simulation_setup` (`:213`) | fills `SimulationConfig` (units, RNG seed, partition bounds) |
| 2 | `convert_species` (`:941`) | BNG species via `World::get_all_species()` |
| 3 | `create_initial_surface_region_release_event` (`:119`) | initial surface molecule release |
| 4 | `convert_rxns` (`:1370`) | BNG reaction rules; it collects the clamps (`add_clamp`, `:1150`) |
| 5 | `add_partition` (`:126`) | the single initial `Partition` (`PARTITION_ID_INITIAL`) |
| 6 | `convert_geometry_objects` (`:561`) | first the mesh checks of every object (`check_polygonal_objects`, §6.2); then `GeometryObject`, `Wall`, `Region`, linked from each object's `MeshTable` (§6.4) |
| 7 | `convert_release_events` (`:1476`) | `ReleaseEvent`s, with their walls, areas and boxes computed by MCell4 (§6.6) |
| 8 | `convert_viz_output_events` (`:1665`) | `VizOutputEvent` |
| 9 | `convert_mol_or_rxn_count_events` (`:1875`) | `MolOrRxnCountEvent` |
| 10 | `create_clamp_events` (`:2055`) | the `ClampReleaseEvent`s, built by the function the Python API's converter uses too (§6.6) |
| 11 | `MeshUtils::check_usage` (`:143-146`) | the usage errors (§6.3) |

**What MCell4 takes from MCell3.**
- **From MCell3's parser:** on MCell4's MDL path, MCell3's parser and initialization run before the
  converter (`src/mcell.c:68`, `:83`; §8.1). The converter reads the parsed model: each object's
  walls as parsed, with one MCell4 vertex per MCell3 vertex, and the regions, reactions, releases
  and counts.
- **Computed by MCell4,** as for Python API models (§6): the mesh checks, the wall constants and
  links, the region borders, the walls, areas and boxes of releases, and the walls of clamps.
- **Still from MCell3's initialization:** the partition's placement and the random-number state
  (§6.12).
- **Where MCell3's parser or initialization stops a model** on MCell4's MDL path, MCell4 never sees
  it (§6.2). MCell4's check of the vertex indices runs before MCell3's initialization, which
  would read the vertices through them.

The converter keeps **ID-mapping tables** (`mcell3_world_converter.h:106-158`) — MCell3
species-id ↔ MCell4 species-id, `vector3*` ↔ partition vertex index, `wall*` ↔ wall
index, `region*` ↔ region index — so cross-references between the two object models
stay consistent during translation.

### 1.3 Which engine each artifact uses

- The **`mcell` executable** (`add_executable(mcell ...)`, `mcell/CMakeLists.txt:518-524`) is
  built from the MCell3 C sources + the MDL bison/flex outputs, and links the
  shared `MCell::mcell4` static engine library. Default MDL runs flow
  MCell3-parse → convert → MCell4-run.
- The **`mcell4_so` Python module** (`nanobind_add_module(mcell4_so ...)`,
  `mcell/CMakeLists.txt:497`, entry `libmcell/api/exports.cpp`, import name `mcell`) is
  the pure MCell4 path: the Python API builds an MCell4 `World` directly via
  `libmcell`, with **no** MCell3/MDL involved.
- Both link the common static engine library **`mcell4`**
  (`add_library(${PROJECT_NAME} STATIC ...)`, `src4/CMakeLists.txt:118-121`, aliased
  `MCell::mcell4`).

---

## 2. Directory Trees

### 2.1 `src4/` (MCell4 — modern C++17 engine)

```
src4/
├── world.{h,cpp}                  # central World object; init + main loop
├── partition.{h,cpp}              # spatial subdivision (partition -> subparts)
├── scheduler.{h,cpp}              # Calendar/Bucket time-ordered event queue
├── base_event.{h,cpp}             # BaseEvent abstract class + event_type_index ordering
│   ── Events ──
├── diffuse_react_event.{h,cpp}    # THE per-timestep diffusion + reaction event
├── release_event.{h,cpp}          # molecule release sites
├── clamp_release_event.{h,cpp}    # concentration clamp release
├── viz_output_event.{h,cpp}       # visualization output (CellBlender/.dat)
├── mol_or_rxn_count_event.{h,cpp} # observables (COUNT) output
├── defragmentation_event.{h,cpp}  # compact molecule arrays
├── mol_order_shuffle_event.{h,cpp}# shuffle molecule order (reproducibility)
├── sort_mols_by_subpart_event.{h,cpp}
├── rxn_class_cleanup_event.{h,cpp}, species_cleanup_event.{h,cpp}
├── run_n_iterations_end_event.h, custom_function_call_event.h
│   ── Core data ──
├── molecule.{h,cpp}               # Molecule (vol or surf, tagged union)
├── geometry.{h,cpp}               # GeometryObject, Wall, Region, Edge, Grid
├── wall.{h,cpp}, wall_overlap.{h,cpp}, region_expr.{h,cpp}, region_utils.{h,cpp}
├── grid_position.{h,cpp}          # surface grid tile geometry
├── collision_structs.{h,cpp}      # Collision result records
├── count_buffer.{h,cpp}           # buffered .dat/.gdat output
├── simulation_config.{h,cpp}      # SimulationConfig : public BNG::BNGConfig
├── simulation_stats.{h,cpp}, memory_limit_checker.{h,cpp}
│   ── Hot-path header-included code (see §3.4) ──
├── collision_utils.inl, collision_utils_subparts.inl
├── diffusion_utils.inl, exact_disk_utils.inl
├── geometry_utils.inl, wall_utils.inl, grid_utils.inl, dyn_vertex_utils.inl
├── rxn_utils.inl
│   ── Geometry (§6; the .cpp files are C++23) ──
├── predicates.{hpp,cpp}           # Shewchuk's exact orientation predicates
├── geometry_analysis.{hpp,cpp}    # mesh algorithms: volume, facing, containment, OBJ writer
├── mesh_table.{hpp,cpp}           # MeshTable: connectivity from vertex indices; the mesh checks
├── mesh_utils.{hpp,cpp}           # geometry report, usage errors, meshes of geometry objects
├── counted_volume_utils.{h,cpp}   # counted volumes, containment mapping, OBJ export
│   ── Dependency glue / export ──
├── bng_data_to_datamodel_converter.{h,cpp}, bngl_exporter.{h,cpp}
│   ── MCell3<->MCell4 bridge ──
├── mcell3_world_converter.{h,cpp}
└── mcell4_iface_for_mcell3.{h,cpp}
```

### 2.2 `src/` (MCell3 — legacy C-as-C++ engine)

```
src/
├── mcell.c                        # executable main()
├── argparse.{c,h}                 # CLI parsing into the volume struct
│   ── MDL parser (flex/bison) ──
├── mdlparse.y, mdllex.l           # MDL grammar + lexer
├── mdlparse_util.{c,h}, mdlparse_aux.h
├── dyngeom_parse.y, dyngeom_lex.l # dynamic-geometry-only MDL subset parser
│   ── Engine ──
├── init.c                         # build runnable state from parsed model
├── diffuse.c, diffuse_trimol.c, diffuse_util.{c,h}, diffuse.h  # per-timestep movement
├── react_cond.c, react_outc.c, react_outc_trimol.c, react_output.c  # reactions
├── grid_util.{c,h}, edge_util.{c,h}, wall_util.c  # geometry helpers
├── count_util.{c,h}, chkpt.{c,h}  # counting / checkpointing
│   ── Public C API helpers ──
├── mcell_init.{c,h}, mcell_run.{c,h}, mcell_reactions.{c,h}
├── mcell_objects.{c,h}, mcell_release.{c,h}, mcell_species.{c,h}
├── mcell_viz.{c,h}, mcell_react_out.{c,h}, mcell_surfclass.{c,h}
├── mcell_dyngeom.{c,h}
│   ── NFsim integration (MCell3 only) ──
├── nfsim_func.{c,h}, react_nfsim.h
├── react_trig_nfsim.c, react_outc_nfsim.c, react_util_nfsim.c
│   ── Central structs / shared ──
├── mcell_structs.h                # struct volume + all engine structs
├── mcell_structs_shared.h         # small subset shared with src4 (C++)
└── ...                            # mem_util, sym_table, vector, rng, isaac64, etc.
```

---

## 3. MCell4 Engine Core (`src4/`)

### 3.1 `World` — the central object

`class World` (`world.h:58`) owns the entire simulation state. Key members:

| Member | Type | Purpose |
|---|---|---|
| `config` | `SimulationConfig` | all configuration constants (`world.h:241`) |
| `bng_engine` | `BNG::BNGEngine` | **owns all species + reactions** (delegated to libbng, `world.h:243`) |
| `stats` | `SimulationStats` | iteration counter + counters (`world.h:245`) |
| `scheduler` | `Scheduler` | time-ordered event calendar (`world.h:250`) |
| `partitions` | `PartitionVector` | spatial partitions (`world.h:259`) |
| `count_buffers` | `CountBufferVector` | buffered observable output (`world.h:261`) |
| `rng` | `rng_state` | single RNG state for the whole simulation (`world.h:256`) |
| `callbacks` | `API::Callbacks&` | Python callback hooks (`world.h:248`) |

Two run entry points:

- **MDL mode:** `init_and_run_simulation()` (`world.cpp:623`) → `init_simulation()` then
  `run_n_iterations(total_iterations, true)` (`world.cpp:627-633`). Called by the
  executable via `mcell4_run_simulation`.
- **Python mode:** `init_simulation(start_time)` (`world.cpp:242`) is called once, then
  `run_n_iterations(num_iterations, ...)` (`world.cpp:379`) may be called repeatedly,
  finally `end_simulation()` (`world.cpp` ~`:582`).

### 3.2 `Partition` — spatial subdivision

`class Partition` (`partition.h`) is the spatial container. The simulation volume is
divided into one or more **partitions** (an axis-aligned lattice; in MDL mode there is
typically a single initial partition `PARTITION_ID_INITIAL`). Each partition is further
divided into a 3D grid of **subparts** (subpartitions): `num_subparts ==
num_subparts_per_partition_edge^3` (`simulation_config.h:70-72`).

Subparts are the spatial-hashing unit used to find reaction partners cheaply:

- `SubpartReactantsSet` (`partition.h:44`) — per-subpart sets of molecule IDs indexed by
  species, so bimolecular collision search only looks at molecules in nearby subparts.
- `WallsInSubpart` (`partition.h:38`) — `uint_set<wall_index_t>`, the walls overlapping
  each subpart, for fast ray–wall collision.

The partition also owns the molecule array, the geometry objects (`GeometryObject`,
`Wall`, `Region`), and the surface grids. `World::get_partition_index(pos)`
(`world.h:102`) finds which partition contains a point; `add_partition()` (`world.h:114`)
appends a new lattice partition.

### 3.3 The Scheduler and the event system

The scheduler (`scheduler.h`) is a **time-ordered calendar**:

- `Calendar` (`scheduler.h:50`) holds a `std::deque<Bucket>` (`scheduler.h:47`). Each
  `Bucket` (`scheduler.h:32`) covers `BUCKET_TIME_INTERVAL == 1` internal time unit and
  holds a `std::list<BaseEvent*>` (`scheduler.h:43`). Bucketing keeps inserts/pops
  cheap while preserving floating-point time precision.
- `Scheduler::handle_next_event()` (`scheduler.cpp:294`) pops the next event, optionally
  computes its barrier-limited time step, calls `event->step()`, then either reschedules
  it (`update_event_time_for_next_scheduled_time()`) or deletes it.
- Events can also be scheduled **asynchronously** (`schedule_event_asynchronously`,
  `scheduler.h:134`) from callbacks, guarded by a mutex; they are merged in before any
  query (`schedule_events_from_async_queue`, `scheduler.h:179`).

All events derive from `BaseEvent` (`base_event.h:63`). The abstract interface:
`step()` (the work), `update_event_time_for_next_scheduled_time()` (periodic
rescheduling via `periodicity_interval`), and the **barrier protocol**
(`is_barrier()` / `may_be_blocked_by_barrier_and_needs_set_time_step()`).

**Event ordering within a single time** is given by `type_index` constants
(`base_event.h:30-56`) — lower runs first. The deliberate holes leave room for
external events:

| `type_index` | Event | Constant |
|---|---|---|
| 0 | start-iteration checkpoint check | `EVENT_TYPE_INDEX_CALL_START_ITERATION_CHECKPOINT` |
| 200 | release | `EVENT_TYPE_INDEX_RELEASE` |
| 290 | mol/rxn count (observables) | `EVENT_TYPE_INDEX_MOL_OR_RXN_COUNT` |
| 300 | viz output | `EVENT_TYPE_INDEX_VIZ_OUTPUT` |
| 301 | simulation-end check | `EVENT_TYPE_INDEX_SIMULATION_END_CHECK` |
| 400 | rxn class cleanup | `EVENT_TYPE_INDEX_RXN_CLASS_CLEANUP` |
| 410 | species cleanup | `EVENT_TYPE_INDEX_SPECIES_CLEANUP` |
| 420 | sort mols by subpart | `EVENT_TYPE_INDEX_SORT_MOLS_BY_SUBPART` |
| 490 | clamp release | `EVENT_TYPE_INDEX_CLAMP_RELEASE` |
| **500** | **diffuse & react** (spans the whole timestep) | `EVENT_TYPE_INDEX_DIFFUSE_REACT` |
| 900 | defragmentation | `EVENT_TYPE_INDEX_DEFRAGMENTATION` |
| 910 | mol shuffle | `EVENT_TYPE_INDEX_MOL_SHUFFLE` |
| 980 | barrier | `EVENT_TYPE_INDEX_BARRIER` |
| 990 | end-iteration call | `EVENT_TYPE_INDEX_CALL_END_ITERATION` |

The critical ordering rule: **counts and viz output (290/300) run *before* diffuse&react
(500)** within an iteration, for MCell3 compatibility — observables reflect the state at
the *start* of the timestep. `DiffuseReactEvent` is the only event that "spans" a time
step; it is barrier-limited so it cannot diffuse past the next count/viz/release event.

The concrete event classes:

| Event class | File | What it does |
|---|---|---|
| `DiffuseReactEvent` | `diffuse_react_event.{h,cpp}` | the heart of the simulation (§3.5) |
| `ReleaseEvent` | `release_event.{h,cpp}` | inject molecules at release sites |
| `ClampReleaseEvent` | `clamp_release_event.{h,cpp}` | concentration-clamp surface release |
| `VizOutputEvent` | `viz_output_event.{h,cpp}` | write CellBlender/.dat viz frames |
| `MolOrRxnCountEvent` | `mol_or_rxn_count_event.{h,cpp}` | evaluate COUNT observable expressions |
| `DefragmentationEvent` | `defragmentation_event.{h,cpp}` | compact molecule storage |
| `MolOrderShuffleEvent` | `mol_order_shuffle_event.{h,cpp}` | randomize molecule iteration order |
| `SortMolsBySubpartEvent` | `sort_mols_by_subpart_event.{h,cpp}` | cache locality optimization |
| `RxnClassCleanupEvent` / `SpeciesCleanupEvent` | `rxn_class_cleanup_event.*` / `species_cleanup_event.*` | reclaim unused BNG rxn classes/species (network-free growth control) |
| `RunNIterationsEndEvent` | `run_n_iterations_end_event.h` | marker/barrier ending a `run_n_iterations` call |
| `CustomFunctionCallEvent` | `custom_function_call_event.h` | Python callback at a scheduled time |

### 3.4 `Molecule`, geometry, `SimulationConfig`

- **`Molecule`** (`molecule.h:52`) is a single struct for both kinds of particle. A
  `molecule_flag_t` (`molecule.h:30`) bit distinguishes volume vs surface; `is_vol()`
  / `is_surf()` (`molecule.h:212/220`) gate access to a tagged union (`molecule.h:146/159`)
  holding either a 3D position (volume mol) or a wall+grid-tile location (surface mol).
- **Geometry** (`geometry.h`): `GeometryObject` (a mesh), `Wall` (a triangle),
  `Region` (a named subset of walls), `Edge`, `Grid` (surface tessellation). Region set
  logic is expressed by `RegionExpr` (`region_expr.h`).
- **`SimulationConfig`** (`simulation_config.h:29`) — `class SimulationConfig : public
  BNG::BNGConfig`. It *inherits* libbng's configuration and adds MCell-specific spatial
  parameters: partition/subpart geometry, radial diffusion-step lookup tables
  (`radial_2d_step`/`radial_3d_step`, `simulation_config.h:77-78`), cleanup
  periodicities, memory limit, etc. `init()` (`simulation_config.h:110`) chains to
  `BNGConfig::init()` then builds the subpart edge length and radial step tables.

### 3.5 The simulation main loop

`World::run_n_iterations()` (`world.cpp:379`) is the loop:

```
do {
  check_checkpointing_signal();                       // world.cpp:406
  double time = scheduler.get_next_event_time();      // world.cpp:420
  current_iteration = time_to_iteration(time);        // world.cpp:423
  if (current_iteration >= first + num_iterations) break;
  ... periodic count-buffer flush / progress report ...
  EventExecutionInfo info = scheduler.handle_next_event();  // world.cpp:456  <-- runs one event
  ... report progress ...
  if (terminate && info.type_index == SIMULATION_END_CHECK) break;  // world.cpp:490
} while (true);
```

So the loop is purely **event-driven**: it does not itself "do" diffusion. It just keeps
pulling the next-earliest event off the scheduler and calling `step()`. An "iteration" is
one internal time unit; within it the ordered events fire (release → count → viz →
end-check → ... → diffuse&react → cleanup).

**Inside `DiffuseReactEvent::step()`** (`diffuse_react_event.cpp`, ~114 KB — the largest
file in `src4`) the actual physics happens for that timestep:

1. **Diffusion** — each molecule gets a Brownian displacement (3D for volume mols, 2D
   along the surface for surface mols), sampled from the radial step lookup tables.
2. **Ray tracing** — the displacement is traced through subparts and against walls,
   handling subpart crossings, wall reflection/transparency/absorption, and periodic
   boundaries.
3. **Reaction** — along the path, candidate bimolecular partners are gathered from the
   nearby `SubpartReactantsSet`, the reaction probability is computed (exact-disk
   method), and reactions fire; unimolecular reactions are scheduled by lifetime.
   New molecules created mid-timestep are handled via `DiffuseAction`
   (`diffuse_react_event.h:72`) with their remaining timestep.

### 3.6 The `.inl` files (header-included hot-path code)

The diffusion/reaction inner loops are split into `.inl` files that are
`#include`d *directly into* the `.cpp` translation units that need them (not compiled
separately). This lets the compiler **inline** these hot functions across what would
otherwise be a translation-unit boundary, with no link-time call overhead — important
because these run millions of times per simulation.

| `.inl` file | Contents | Included by (examples) |
|---|---|---|
| `collision_utils.inl` (56 KB) | molecule/wall/subpart collision detection | `diffuse_react_event.cpp:41`, `geometry.cpp:30`, `partition.cpp:22`, `wall.cpp:30`, `release_event.cpp:31` |
| `collision_utils_subparts.inl` | subpart-crossing collision helpers | (via collision_utils) |
| `diffusion_utils.inl` | Brownian step sampling, displacement | `diffuse_react_event.cpp:43` |
| `exact_disk_utils.inl` (35 KB) | exact-disk bimolecular reaction probability | `diffuse_react_event.cpp:42` |
| `geometry_utils.inl` | wall bounding boxes, point/wall geometry | many (`geometry.cpp:29`, `release_event.cpp:30`, `viz_output_event.cpp:31`, ...) |
| `grid_utils.inl` (77 KB) | surface-grid tile lookups / neighbor walks | `diffuse_react_event.cpp:45`, `grid_position.cpp:18`, `release_event.cpp:32` |
| `wall_utils.inl` | wall neighbor / surface-mol helpers | `diffuse_react_event.cpp:46`, `partition.cpp:23`, `region_utils.cpp:21` |
| `rxn_utils.inl` | reaction-class lookup / outcome helpers | `diffuse_react_event.cpp:44`, `region_utils.cpp:20` |
| `dyn_vertex_utils.inl` | dynamic-geometry vertex moves | `partition.cpp:21` |

The `.h` companions (`collision_utils.h`? no — note these are `.inl` with matching
small `geometry_utils.h`, `wall_utils.h` declaring the namespaced free functions)
declare the function signatures; the `.inl` carries the bodies.

### 3.7 Known issues of the engine

The known issues of the geometry code are in §6.12. These concern the engine as a whole; each was
found during the vertex-index project (doc 08) and predates it.

- **Barriers change trajectories** (by design).
  - **What's a barrier** (`is_barrier`): counts (`mol_or_rxn_count_event.h:203`), visualization
    output (`viz_output_event.h:42`), releases (`release_event.h:142`), and the end of each
    `run_iterations` call (`run_n_iterations_end_event.h:38-40`).
  - **How a jump ends:** the scheduler asks only when the next barrier is, whatever its kind
    (`Calendar::get_time_up_to_next_barrier`, `scheduler.cpp:212`). A volume or surface
    molecule's jump ends there (`diffuse_react_event.cpp:172`).
    - So only the set of barrier times matters: an output at times that already have a barrier
      changes nothing.
    - Turning visualization off is no remedy: it removes barriers, and so changes results.
  - **Which molecules change:** on MCell4, only volume and surface molecules whose step would
    cross the barrier: those that start off the iteration grid (reaction products, clamp
    molecules) or have a time step longer than one iteration. MCell4 always takes single steps,
    since its multi-step diffusion is switched off (`ENABLE_SAFE_DIFFUSION_STEP` commented out,
    `diffusion_utils.inl:40`).
  - **MCell3's engine has other barriers:** the next release, volume output and visualization
    output (`src/mcell_run.c:541-555`). Counts are no barrier there.
    - Every volume and surface molecule's step ends at the barrier.
    - A volume molecule may also take several time steps in one jump when the barrier is more
      than two time steps away (`safe_diffusion_step`, `src/diffuse.c:5062-5067`).
    - So on MCell3, fewer visualization frames change trajectories even when counts run every
      iteration.
  - **Clamps:** a clamp's volume molecules start at the half step
    (`clamp_release_event.cpp:230`). With a barrier every iteration their first jump is cut to
    half a step, and the clamp's concentration comes out lower.
    - In test `pymcell4_positive/1700` with its left clamp at 1e-4 M, bins 0-3 of the five lie
      2.5-5% below the linear profile between the clamps with a barrier every iteration, and
      within 2% of it with barriers every 1,000 iterations.
    - Planned: find which timing the clamp's emission rate assumes, and compare with MCell3,
      before changing anything.
- **The order within an iteration.**
  - **New volume molecules** diffuse after the existing ones, in the order they were made
    (`diffuse_react_event.cpp:76-122`). This covers clamps, releases and reaction products. So when
    the volume molecules of two clamps compete in the same time step for the same partners, the
    clamp that runs first gets the first chance every time step.
  - **The clamps' order:** clamps run by surface class name, then molecule name, then side (§6.6).
  - **The molecule-order shuffle** of volume and surface molecules
    (`Partition::shuffle_schedulable_molecule_ids`, `partition.cpp:941`, every 10,000 iterations
    by default) picks each swap partner from 0..n−2 (`(n-1) * rand`, `:954`) instead of 0..i.
    - So its orders aren't equally likely: two molecules are always swapped, and the last molecule
      never stays last.
  - **`randomize_vector`** (`partition.cpp:141`), which orders vertex moves (`:169`, `:206`), has
    the same loop.
    - With an empty list, `n-1` wraps around (unsigned) and it reads far out of bounds.
    - So `Model.apply_vertex_moves()` with no moves added, and the default `randomize_order=True`
      (`libmcell/definition/model.yaml:241`), ends Python with a segmentation fault.
  - Planned:
    - at each iteration, a seeded shuffle of the clamps from `world->rng`, MCell's own loop, the
      sorted order as the start, no draw with fewer than two clamps;
    - both shuffle loops picking from 0..i, with `randomize_vector` returning on fewer than two
      items;
    - a review of the order of new volume molecules.

    Every MCell4 model with two or more acting clamps, 10,000 or more iterations, or randomized
    vertex moves then gets new results.
- **A clamp's volume molecule can leave a watertight box.**
  - **Seen in** test `pymcell4_positive/1700` with its left clamp at 1e-4 M and visualization every
    1,000 iterations: 3 of 480 runs stop with "Molecule … escaped the simulation area".
  - **Reproducible:** seed 32 gives the same molecule at the same position every time. The build
    from before the project gives the identical escape.
  - **The mechanism isn't known.**
    - A clamp's first jump has a part along the wall's plane (`pick_clamped_displacement`,
      `diffusion_utils.inl:336`).
    - So a clamp's volume molecule released near an edge of the clamped face is carried toward the
      adjacent face at once.
  - **MCell3's engine** allows unlimited space, so a leak there would go unreported.
  - Planned: find where and how it leaves the box, then fix it.

---

## 4. How MCell4 Uses libbng

MCell4 **delegates all species and reaction storage and chemistry to libbng**. The
integration boundary is clean: `World` owns one `BNG::BNGEngine`, and
`SimulationConfig` *is a* `BNG::BNGConfig`.

| MCell-side construct | libbng type | Where |
|---|---|---|
| `World::bng_engine` | `BNG::BNGEngine` | `world.h:243` |
| `SimulationConfig` | `: public BNG::BNGConfig` | `simulation_config.h:29` |
| `World::get_all_species()` | `BNG::SpeciesContainer&` | `world.h:168` → `bng_engine.get_all_species()` |
| `World::get_all_rxns()` | `BNG::RxnContainer&` | `world.h:171` → `bng_engine.get_all_rxns()` |
| BNGL/compartment data | `bng_engine.get_data()` → `BNG::BNGData` | `partition.cpp:896` |

The call surface MCell4 actually uses:

- **`SpeciesContainer`** — `get_all_species().get(species_id)` returns a `BNG::Species`
  (`world.cpp:705`); `.get_count()` (`world.cpp:263`) and `.dump()` (`world.cpp:656`).
  Used everywhere the engine needs a species' diffusion constant, mass, flags, etc.
- **`RxnContainer`** — `get_all_rxns().get(rxn_rule_id)` returns a `BNG::RxnRule`
  (`world.cpp:687`); `.dump()` (`world.cpp:657`). The reaction-utility `.inl`
  (`rxn_utils.inl`) consults BNG rxn classes during diffusion to decide outcomes.
- **`BNGData`** — `bng_engine.get_data().get_compartment(id)` returns a
  `BNG::Compartment` (`partition.cpp:896,902`), used for compartment hierarchy /
  region containment. `.dump()` (`world.cpp:655`).
- **`BNGConfig`** — `SimulationConfig::init()` calls `BNGConfig::init()`
  (`simulation_config.h:111`); final warnings via
  `bng_engine.get_config().print_final_warnings()` (`world.cpp:609`).
- **BNGL parsing** (`parse_bngl_file`) and BNGL export (`bngl_exporter.cpp`,
  `bng_data_to_datamodel_converter.cpp`) are driven from MCell through the BNG engine
  on the Python/BNGL path.

The MCell3 converter also funnels through BNG: `convert_species`/`convert_rxns`
(`mcell3_world_converter.cpp:941/1370`) create the BNG species and rxn rules from the
legacy `volume` data, so by the time MCell4 runs, *all* chemistry lives in the BNG
engine regardless of whether the model came from MDL or Python.

---

## 5. How MCell Uses NFsim

**NFsim (network-free simulator) is an MCell3-only feature.** It provides *network-free
reaction handling*: instead of enumerating a fixed reaction network at init time,
molecules carry a graph/pattern (a Nauty-canonicalized string), and the reactions a
molecule can undergo are **looked up on demand** by querying the external NFsim engine.
This is what makes rule-based models with combinatorially huge state spaces tractable.

### 5.1 MCell4 (`src4`) does **not** use NFsim

Confirmed by search: the only `nfsim` occurrences in `src4/` are (a) commented-out
field references in the converter (`mcell3_world_converter.cpp:1205-1207`), (b) the
nfsimCInterface include path in `src4/CMakeLists.txt:133` (carried for the shared link),
and (c) a `nfsim_export` *naming* flag in the BNGL exporter (`bngl_exporter.cpp:43`) that
only affects how rates are *labeled* when exporting BNGL for an external NF run. The
MCell4 engine itself performs no NFsim queries. Network-free behavior in MCell4 is
instead handled by libbng's own on-the-fly species/rxn generation.

### 5.2 MCell3 NFsim integration

NFsim is activated by the `-rules <file>` CLI flag (`argparse.c:286`, sets `nfsim_flag`),
which calls `setupNFSim_c` at the end of arg parsing (`argparse.c:410`). The integration
uses the **nfsimCInterface C API** (header `nfsim_c.h`, included at `argparse.c:32`),
not C++ NFsim directly.

| File | Role | NFsim C-API calls |
|---|---|---|
| `nfsim_func.c/.h` | Function-pointer dispatch so molecules pull spatial params (diffusion const, space/time step, flags) from *either* standard species fields *or* NFsim graph data, based on the `EXTERNAL_SPECIES` flag. `initialize_diffusion_function` (`:22`), `initialize_rxn_diffusion_functions` (`:43`). Plus a graph-pattern→`graph_data` hashmap (`:147-155`). | — |
| `react_trig_nfsim.c` | Reaction **triggering** — decides which reactions can fire by querying NFsim. `trigger_bimolecular_nfsim` (`:183`), `pick_unimolecular_reaction_nfsim` (`:431`), `adjust_rates_nfsim` (`:265`). Heavy use of `mapvector`/`queryOptions` result structs. | `initAndQueryByNumReactant_c` (`:159,232,460`) |
| `react_outc_nfsim.c` | Reaction **outcomes/firing** — applies results, creates products, derives diffusion data. `prepare_reaction_nfsim` (`:203`), `outcome_nfsim` (`:757`), `calculate_nfsim_diffusion_derived_data` (`:614`). | `constructNauty_c`, `initAndQuerySystemStatus_c`, `initAndQueryByNumReactant_c`, `extractSpeciesCompartmentFromNauty_c`, `getCompartmentInformation_c` |
| `react_util_nfsim.c` | `calculate_reactant_orientation` (`:7`) — derives orientation from NFsim compartment info. | `extractSpeciesCompartmentFromNauty_c`, `getCompartmentInformation_c`, `freeCompartmentInformation_c` |
| `react_nfsim.h` | Shared prototypes; pulls `queryOptions`/`queryResults`/`compartmentStruct` types from `nfsim_c.h`. | — |

**NFsim C-API functions consumed by MCell3:** `setupNFSim_c` (init),
`initAndQueryByNumReactant_c`, `initAndQuerySystemStatus_c`, `constructNauty_c`,
`extractSpeciesCompartmentFromNauty_c`, `getCompartmentInformation_c`,
`freeCompartmentInformation_c` (all from `nfsim_c.h`). The deeper internals of these are
documented in `02_nfsim.md` / `03_nfsimCInterface.md`.

---

## 6. Geometry: Mesh Checks, Connectivity, Counted Volumes, Compartments, OBJ Export (self-contained reference)

> This section stands alone as MCell's geometry reference: how MCell4 checks the mesh of each
> geometry object, links its walls, keeps what it learned for the run, and uses it. It describes
> the code at `mcell` `1bb0f809`; the line citations are to that commit.
>
> - **The VTK removal** (doc 07) replaced VTK with the in-tree geometry analysis of §6.7.
> - **The vertex-index project** (doc 08; branch `mcell4_nanobind_vertexidx`) made the file's
>   connectivity authoritative: MDL `ELEMENT_CONNECTIONS` and the Python API's `wall_list`.
>   - MCell4 takes each wall's neighbours only from the walls' vertex indices, never from their
>     coordinates.
>   - It never repairs a mesh: nothing is merged, reordered or reoriented. It only reports:
>     errors stop the run, warnings don't.
>   - MCell3's engine is unchanged, and so is MCell3's code, but for one call that MCell4's MDL
>     path makes in `src/mcell.c` (§1.2).
>
> An MDL file takes one of two paths: **MCell3's engine** (`mcell file.mdl`) or **MCell4's MDL
> path** (`mcell -mcell4 file.mdl`, also `-mdl2datamodel4`). Both run MCell3's parser and MCell3's
> initialization; on MCell4's MDL path `MCell3WorldConverter` then builds MCell4's model.

### 6.1 Where the code is

Five file pairs in `src4/` hold the geometry code. The rest of MCell calls them.

| Files | Namespace | Role |
|---|---|---|
| `src4/predicates.{hpp,cpp}` | `MCell::Predicates` | Shewchuk's adaptive exact `orient2d` and `orient3d` (`src4/predicates.hpp:24`, `:30`), ported from his public-domain C implementation. The sign is always exact: zero only for exactly collinear or coplanar points. `exactinit()` runs once, when the library is loaded (`src4/predicates.cpp:377`). |
| `src4/geometry_analysis.{hpp,cpp}` | `MCell::GeometryAnalysis` | The algorithms on plain meshes (§6.7): volume and area, the parts that face the wrong way, containment, the OBJ writer. Points are `std::array<double, 3>` and triangles are index triples (`geometry_analysis.hpp:38-39`). It includes no MCell header. |
| `src4/mesh_table.{hpp,cpp}` | `MCell::MeshUtils` | The `MeshTable`, an object's connectivity built from its walls' vertex indices, and every check of one mesh (§6.2). Also each object's block of the geometry report, and its terminal lines (§6.3). It includes only `geometry_analysis.hpp`. |
| `src4/mesh_utils.{hpp,cpp}` | `MCell::MeshUtils` | MCell's side of it: the report file (`GeometryReport`, `mesh_utils.hpp:50`), the usage errors (`UsageErrors`, `check_usage`, `check_release_usage`, `:94-134`), and the meshes of `GeometryObject`s for `GeometryAnalysis` (`get_mesh`, `get_unmerged_mesh`, `:74`, `:80`). |
| `src4/counted_volume_utils.{h,cpp}` | `MCell::CountedVolumeUtils` | Counted volumes, the containment mapping (shared with compartments), the parent search, the BNGL export volume and the OBJ export. Called `vtk_utils` / `VtkUtils` before the VTK removal. |

The engine code that uses them:
- **`src4/geometry.{h,cpp}`:**
  - `GeometryObject` keeps its mesh check for the run, and with it the coordinate facts and the
    per-vertex lists (§6.5). `link_walls` links the walls (§6.4).
  - `Region::init_border_edges` gives a region its borders.
  - `Geometry::update_moved_walls` applies vertex moves.
- **The converters**, which run the checks and stop the run:
  - `MCell4Converter` for Python API models (`libmcell/api/mcell4_converter.cpp`);
  - `MCell3WorldConverter` for MCell4's MDL path (`src4/mcell3_world_converter.cpp`), with its
    check of the vertex indices before MCell3's initialization (§6.2).
- **`src4/release_event.cpp`, `src4/clamp_release_event.{h,cpp}`:** the walls, areas and boxes of
  releases and clamps (§6.6).
- **`libmcell/api/introspection.cpp`, `libmcell/api/model.cpp`:** the Python API's getters, its
  releases after initialization and the OBJ export.
- **`utils/data_model_to_pymcell/`:** the same checks on a data model's lists (§6.6).
- **`utils/mesh_utils_test/`:** the test program of the checks (§6.11).

`predicates.cpp`, `geometry_analysis.cpp` and `mesh_table.cpp` form the static library
`MCell::geometry` (§6.10). It depends on nothing else in MCell, so `data_model_to_pymcell` and
`mesh_utils_test` can link it on their own.

### 6.2 The mesh checks

**When.** Every geometry object is checked once, at initialization, before any wall is linked
and before anything uses the mesh.

| Front end | Where | Input |
|---|---|---|
| Python API | `Model.initialize()` → `MCell4Converter::convert_before_init` (`mcell4_converter.cpp:64`) → `check_geometry_objects` (`:71`, `:889`), after `convert_simulation_setup`, which sets the seed and the length unit | `MeshUtils::check_lists` (`mesh_table.hpp:365`) on `vertex_list`, `wall_list` and the surface regions' wall indices, as they are at `initialize()`. Changes made through setters after the constructor are checked too. |
| MCell4's MDL path: an MDL file run with `mcell -mcell4` (also `-mdl2datamodel4`) | MCell3's parser runs first (`src/mcell.c:68`). Then MCell4 checks every wall's vertex indices (`check_vertex_indices_before_mcell3_init`, `mcell3_world_converter.cpp:528`, called at `src/mcell.c:74`; below), and MCell3's initialization runs (`:83`). Then `MCell3WorldConverter::convert_geometry_objects` (`mcell3_world_converter.cpp:561`) starts with `check_polygonal_objects` (`:482`, `:567`). | Before MCell3's initialization, pass 0 on the walls MCell3's parser stored (`polygon_object::element`). Then `MeshUtils::check_parsed_walls` (`mesh_table.hpp:375`) on the same walls; the converter adds one MCell4 vertex per MCell3 vertex, in order (`mcell3_world_converter.cpp:862`). |
| `data_model_to_pymcell` | the volume of a BNGL compartment (§6.6) | `MeshUtils::check_mesh` (`data_model_geometry.cpp:51`) on the data model's lists |

MCell3's engine (an MDL file run without `-mcell4`) runs none of these checks.

**The `MeshTable`** (`mesh_table.hpp:105`) holds one object's connectivity as arrays.
- **Ids** are 0-based positions in the object's lists, as in the file. Entry `i = 3w + k` is
  side `k` of wall `w`: the walk from the wall's vertex `k` to its vertex `(k + 1) % 3`.
  `Wall::nb_walls[k]` lies across that side (`mesh_table.hpp:22-26`).
- **An edge** is a pair of vertices `lo < hi`. Edges are numbered in the order the entries first
  use them.
- **Arrays:**
  - per entry: `edge_of` and `sign_of`, which is +1 when the entry runs from `lo` to `hi`;
  - per edge: `lo`, `hi`, `wall_count`, `entry0`, `entry1`;
  - per vertex: `walls_per_vertex`, `edges_per_vertex`.
- **`neighbor(i)`** (`:132`) is the entry across entry `i` when its edge has exactly two walls.
- **`build_mesh_table`** (`mesh_table.cpp:185`) builds it:
  - it counts the walls per vertex;
  - pass 1 puts each entry in the bucket of its edge's end with fewer walls (ties: `lo`), then
    takes prefix sums (`:198-212`);
  - pass 2 numbers the edges in the order of first appearance and fills the rest (`:215-251`).
- **Origin:** the table is the C++ form of the sparse matrices `d0`, `d1` and `A20` of the
  prototype `coboundary.py` (`mesh_table.hpp:28-30`). The prototype is kept outside the
  repositories; its results are the test program's expected values (§6.11).

**The checks of one object, in order** (`check_mesh`, `mesh_table.cpp:603`):

| # | Check | Finds | Kind | If it fails |
|---|---|---|---|---|
| 1 | Equal coordinates (`find_equal_coordinates`, `mesh_table.cpp:545`) | groups of vertices with exactly equal coordinates (0.0 equals -0.0; NaN equals nothing), on the coordinates as the model gives them; skipped when a vertex doesn't have three coordinates | warning | nothing is skipped: connectivity comes from the indices |
| 2 | Pass 0 (`check_indices`, `mesh_table.cpp:170`) | a wall with an index below 0 or not below the vertex count; a wall that uses one vertex twice; too many vertices or walls for 32-bit ids. `check_lists` adds vertices without three coordinates, walls without three indices, and wall indices of a surface region out of range. | error | without valid indices there is no table, so 3-9 don't run. A bad region index doesn't stop them. |
| 3 | Zero-area walls (`find_zero_area_walls`, `is_zero_area`, `mesh_table.cpp:504`) | walls whose area is at most 1e-12 internal units squared, or NaN: the test of `Wall::initialize_wall_constants` (`src4/wall.cpp:306`) | error | nothing is skipped |
| 4 | Orphaned vertices (`find_orphaned_vertices`, `mesh_table.cpp:535`) | vertices that no checked wall uses | error | nothing is skipped |
| 5 | Walls with the same three vertices (`find_duplicate_walls`, `mesh_table.cpp:276`) | groups of such walls, in any order of the vertices | error | nothing is skipped |
| 6 | 1.1 Manifold mesh (`find_non_manifold`, `mesh_table.cpp:332`) | edges with three or more walls, and the vertices where the walls don't form one fan | error | 7-9 don't run |
| 7 | 1.2 Consistently oriented (`find_inconsistent_edges`, `mesh_table.cpp:395`) | edges of two walls that both run along the edge in the same direction | error | 8-9 don't run |
| 8 | 1.3 Watertight (`find_boundary_edges`, `mesh_table.cpp:405`) | boundary edges, which have only one wall | warning: a non-watertight mesh is fine to simulate; only some uses need a watertight object (§6.3) | nothing is skipped |
| 9 | Shells (`find_shells`, `mesh_table.cpp:415`, `GeometryAnalysis::judge_parts`) | the shells, their areas, the signed volumes of the watertight ones, and the shells that face the wrong way | listed in the report; an error only through a usage error (§6.3) | – |

- **The non-manifold vertices.** The code keeps three lists in `NonManifold`
  (`mesh_table.hpp:162-174`), each vertex under the first that applies:
  - `edge_vertices`: the ends of the edges with three or more walls;
  - `bowtie_vertices` (`:165`): two or more open fans, that is `edges_per_vertex` minus
    `walls_per_vertex` is at least 2;
  - `pinched_vertices` (`:166`): a closed fan and two or more fans in all. Closed fans come from a
    union-find over the walls' corners, joined at each edge of two walls (`mesh_table.cpp:346-381`).

  The lists are kept apart only so that the test program can compare each with the prototype. The
  report and the terminal give every non-manifold vertex once, in one list, unclassified
  (`NonManifold::vertices`, `mesh_table.hpp:173`).
- **The object's area** is always computed. Its signed volume, summed over its shells, is computed
  only when the object is watertight, and each shell's only when that shell is watertight
  (`mesh_table.cpp:671-697`; §6.8).
- **The checks run serially.** On the full neuropil (1,763 objects, 3,990,190 walls) the MDL
  converter's checks take 1.7 s of CPU.

**MDL `REMOVE_ELEMENTS`.** MCell3 marks the walls it removed in `polygon_object::side_removed`
(`mcell3_world_converter.cpp:454`): its parser the walls of `REMOVE_ELEMENTS`, its initialization
the zero-area walls it dropped; only the area tells them apart.
- `check_parsed_walls` (`mesh_table.cpp:752`) checks a removed wall with zero area, so that it is
  reported, and a removed wall with a vertex index out of range, which pass 0 reports (`:772`). It
  leaves out the others as `REMOVE_ELEMENTS` walls. On MCell4's MDL path the check before MCell3's
  initialization already stops a file with such an index (below).
- The report keeps the file's wall ids and adds the line `Removed by REMOVE_ELEMENTS: 1 (wall 0)`.
  A vertex that only removed walls use is orphaned (check 4).
- MCell4 simulates only the kept walls. The converter checks that table wall `j` is the file's
  wall `file_wall_ids[j]` (`mcell3_world_converter.cpp:898`).
- Accepted cost: a wall that the file removes and that also has zero area stops the run as a
  zero-area wall. MCell3's engine never measures such walls.

**MDL files on MCell4's MDL path: what runs before the converter.** MCell3's parser, MCell4's
check of the vertex indices, then MCell3's initialization. Where MCell3's parser or
initialization stops a file, the user sees MCell3's message and gets no geometry report.
- **Pass 0:**
  - MCell3's parser refuses vertices without three coordinates, walls without three indices, and
    wall indices of a surface region or of `REMOVE_ELEMENTS` out of range. A negative one too: it
    reads them as unsigned, so its message shows -1 as 4294967295 ("Region element specifier ...
    refers to sides 4294967295...4294967295").
  - **MCell4's check before MCell3's initialization** (`check_vertex_indices_before_mcell3_init`,
    `mcell3_world_converter.cpp:528`) runs pass 0 on every wall of every instantiated polygonal
    object as the parser stored it, also on the walls that `REMOVE_ELEMENTS` removes, with the
    file's ids. MCell3's initialization would read each wall's vertices through its indices
    (`src/init.c:2098`).
    - If an index is below 0 or not below the object's vertex count
      (`mcell3_world_converter.cpp:537-539`), it writes the geometry report with a block for each
      object with an index problem (a wall that uses a vertex twice included), prints their
      `MESH_INVALID_INDICES` lines and stops once (`:557`). The other objects get no block: the run
      ends before their checks.
    - Otherwise it writes nothing, and the converter checks every object.
  - A wall that uses one vertex twice is then the only pass-0 case left for the converter.
    MCell3's initialization drops it as degenerate ("Degenerate polygon found and automatically
    removed"), and `check_parsed_walls` reports it.
  - On MCell3's engine none of MCell4's checks run: MCell3's initialization stops an index above
    the vertex count (`src/init.c:2092-2095`) and crashes on one equal to the count or below 0
    (§6.12).
- **Uses of volume molecules:** MCell3's initialization, which MCell4's MDL path runs too, links
  walls by coordinates and stops counted objects and release regions that it finds
  non-watertight by its own rules:
  - "Cannot count molecules or events inside non-manifold object region" (`src/count_util.c:1438`);
  - "Cannot release a 3D molecule inside the unclosed region" (`src/init.c:3599`, `:3629`).

  So on MCell4's MDL path a counted object or release region reaches MCell4's usage errors only
  when MCell3's initialization finds it watertight: for example a cube whose faces each have their
  own four vertices, watertight by its coordinates and non-watertight by its vertex indices.

### 6.3 The geometry report, the terminal lines and the one stop

**The report**, `reports/geometry_report_<seed in 5 digits>.txt`
(`SimulationConfig::get_geometry_report_file_name`, `simulation_config.cpp:45`):
- **Header:** `Geometry report, <date>`, written by `BNG::initialize_report_file`
  (`libbng/bng/bng_config.cpp:134`), as for MCell's other reports.
- **Blocks:** it is always written, with one block per object in the order the converter checks
  them (`write_report_block`, `mesh_table.cpp:789`).
- **Ids:** every list of ids gives its count and says what the ids are. Walls have the file's ids
  (`REMOVE_ELEMENTS`).
- **Units and the usage errors:** areas are in um^2 and volumes in um^3; coordinates are written
  as given. The usage errors (below) add nothing to the report.

The stored report of `nutmeg4_mdl/0240_mesh_errors_error`, an MDL file with one object per error,
on MCell4's MDL path:

```
Geometry report, 2026-11-20 13:52:53

Object Scene.repeated: 8 vertices, 12 walls
  Walls that use a vertex twice: 1 (wall 5)
  Manifold mesh: not checked
  Consistently oriented: not checked
  Watertight: not checked
Object Scene.duplicate: 3 vertices, 2 walls, 3 edges
  Walls with the same three vertices: 1 group
    walls 0 1
  Manifold mesh: True
  Consistently oriented: True
  Watertight: True
  Shells: 1, area 1 um^2, signed volume 0 um^3
Object Scene.zero: 5 vertices, 3 walls, 7 edges
  Zero-area walls: 1 (wall 2)
  Manifold mesh: True
  Consistently oriented: True
  Watertight: False (5 boundary edges)
  Shells: 1, area 4 um^2
Object Scene.book: 5 vertices, 3 walls, 7 edges
  Manifold mesh: False
    edges with 3 or more walls: 1
      edge 0-1: walls 0 1 2
    non-manifold vertices: 2 (vertices 0 1)
  Consistently oriented: not checked
  Watertight: not checked
Object Scene.bowtie: 5 vertices, 2 walls, 6 edges
  Manifold mesh: False
    non-manifold vertices: 1 (vertex 0)
  Consistently oriented: not checked
  Watertight: not checked
Object Scene.flipped: 8 vertices, 12 walls, 18 edges
  Manifold mesh: True
  Consistently oriented: False
    walls 0 and 4, edge 0-2
    walls 1 and 4, edge 2-6
    walls 4 and 10, edge 0-6
  Watertight: not checked
```

Blocks of the other lines, from `nutmeg4_mdl/0250_mesh_warnings` and
`0210_usage_errors_inside_out_and_non_watertight_error`:

```
Object Scene.touching: 8 vertices, 8 walls, 12 edges
  Vertices with equal coordinates: 1 group
    vertices 0 4 at (5, 0, 0)
  Manifold mesh: True
  Consistently oriented: True
  Watertight: True
  Shells: 2, area 4.732050808 um^2, signed volume 0.3333333333 um^3
Object Scene.inside_out: 8 vertices, 12 walls, 18 edges
  Manifold mesh: True
  Consistently oriented: True
  Watertight: True
  Shells: 1, area 24 um^2, signed volume -8 um^3
  Shells facing the wrong way: 1
    shell 0: 12 walls from wall 0, signed volume -8 um^3
```

The other report lines are `Removed by REMOVE_ELEMENTS: ...`, `Orphaned vertices (used by no wall):
...`, `Too many vertices or walls to check`, `Vertices that don't have three coordinates: ...`,
`Walls that don't have three vertex indices: ...`, `Walls with a vertex index out of range: ...` and
`Surface regions with a wall index out of range: ...` with one line per region
(`mesh_table.cpp:801-853`). `mcell_tests/tests/utils_tests/mesh_utils/reports/` has a block for each.

**The terminal lines** (`problem_lines`, `mesh_table.cpp:947`; `format_problem_line`, `:1028`):
- **Format:** one line per object and problem, printed through `mcell_error_raw` to MCell's error
  stream. Every error line comes first: the objects in report order, each object's problems in the
  block's order. Then every warning (`GeometryReport::finish`, `mesh_utils.cpp:83-95`).
- **Parsing:** scripts can parse the lines with `^(Error|Warning): \[([A-Z_]+)\] (\S+): (.*)$`.
- **Object names:** the name is the object's name as MCell has it: `Scene.box` for MDL files, the
  object's own name for Python API models.
- **Shells that face the wrong way** get no terminal line. The report lists them.

The lines of `0240` above, with the one stop:

```
Error: [MESH_INVALID_INDICES] Scene.repeated: 1 wall that uses a vertex twice; see reports/geometry_report_00001.txt
Error: [MESH_DUPLICATE_WALLS] Scene.duplicate: 1 group of walls with the same three vertices; see reports/geometry_report_00001.txt
Error: [MESH_ZERO_AREA_WALLS] Scene.zero: 1 wall with zero area; see reports/geometry_report_00001.txt
Error: [MESH_NOT_MANIFOLD] Scene.book: 1 edge with 3 or more walls, 2 non-manifold vertices; see reports/geometry_report_00001.txt
Error: [MESH_NOT_MANIFOLD] Scene.bowtie: 1 non-manifold vertex, where groups of walls touch only at a vertex; see reports/geometry_report_00001.txt
Error: [MESH_INCONSISTENT_ORIENTATION] Scene.flipped: 3 pairs of walls run along their shared edge in the same direction; see reports/geometry_report_00001.txt
Warning: [MESH_NOT_WATERTIGHT] Scene.zero: 5 boundary edges; see reports/geometry_report_00001.txt
Error: 6 geometry objects have errors: Scene.repeated, Scene.duplicate, Scene.zero, Scene.book, Scene.bowtie, Scene.flipped; see reports/geometry_report_00001.txt
```

**The one stop.** Once every object is checked, a model with errors stops once, naming every
object with errors (`stop_message`, `mesh_table.cpp:1047`). Warnings don't stop the run.
- **Python API:** a `ValueError` with that text (`mcell4_converter.cpp:918`).
- **MCell4's MDL path, also `-mdl2datamodel4`:** `mcell_error` prints `Error: ...` and the run
  ends with exit code 1 (`mcell3_world_converter.cpp:489`). `-mdl2datamodel4` then writes no data
  model.
- **The check of the vertex indices before MCell3's initialization** (§6.2) stops the same way
  (`:557`), with the same lines and stop. Its report holds only the objects with an index problem.

**The usage errors** concern volume molecules only. Surface molecules are released onto walls and
counted on walls, which needs no inside.
- **When:** once the releases and counts are converted, `MeshUtils::check_usage`
  (`mesh_utils.cpp:248`) checks how the model uses each object. It runs at the end of both
  converters (`mcell4_converter.cpp:118`, `mcell3_world_converter.cpp:143`).
- **Format:** each use gives one line, `Error: [CODE] <object>: <use>, but <problem>; see <report>`,
  where `<report>` is the report's path. After every line comes one stop:
  `2 geometry objects have usage errors: ...` (`usage_stop_message`, `mesh_table.cpp:1051`), as a
  `ValueError` or with exit code 1. The objects come in the partition's order; for each, its counts,
  its compartment, then its releases in the scheduler's order.
- **The two codes:**
  - **`REGION_NOT_WATERTIGHT`:** a non-watertight object is counted inside, is a compartment, or is
    an object of the region of a release of volume molecules. The release can be by number, by
    concentration, or a removal by a negative number.
  - **`MESH_INSIDE_OUT`:** an object has a shell that faces the wrong way, and the model counts
    inside it, or it is a compartment, or a release of volume molecules by concentration goes into
    that object alone. Such a release takes its number from the object's signed volume. A volume
    molecule that crosses a wall of a counted object takes the side the wall faces, so every
    crossing of an inside-out object is counted the wrong way round.
- **Compartments** (Python API) are checked for watertightness first, in
  `MCell4Converter::check_compartments_watertight` (`mcell4_converter.cpp:1088`, called at `:1129`).
  The lines are the same and the stop is their own; the compartment tree needs watertight meshes.
- **`Model.release_molecules`** checks its release the same way (`check_release_usage`,
  `model.cpp:235`). It raises a `ValueError` and releases nothing.

The lines of `nutmeg4_mdl/0210`: an inside-out cube that the model counts inside and releases
into by concentration, and a cube whose faces each have their own four vertices, with three
releases and a count. The two warnings come from the mesh checks, which print before the usage
errors:

```
Warning: [MESH_DUPLICATE_VERTICES] Scene.unshared: 8 groups of vertices with equal coordinates; see reports/geometry_report_00001.txt
Warning: [MESH_NOT_WATERTIGHT] Scene.unshared: 24 boundary edges; see reports/geometry_report_00001.txt
Error: [MESH_INSIDE_OUT] Scene.inside_out: the model counts inside this object, but 1 shell faces the wrong way; see reports/geometry_report_00001.txt
Error: [MESH_INSIDE_OUT] Scene.inside_out: release site Scene.rel_conc releases volume molecules into this object by concentration, but 1 shell faces the wrong way; see reports/geometry_report_00001.txt
Error: [REGION_NOT_WATERTIGHT] Scene.unshared: the model counts inside this object, but it is non-watertight (24 boundary edges); see reports/geometry_report_00001.txt
Error: [REGION_NOT_WATERTIGHT] Scene.unshared: release site Scene.rel_number releases volume molecules into this object, but it is non-watertight (24 boundary edges); see reports/geometry_report_00001.txt
Error: [REGION_NOT_WATERTIGHT] Scene.unshared: release site Scene.rel_remove removes volume molecules from this object, but it is non-watertight (24 boundary edges); see reports/geometry_report_00001.txt
Error: 2 geometry objects have usage errors: Scene.inside_out, Scene.unshared; see reports/geometry_report_00001.txt
```

### 6.4 Wall links and region borders

**Wall links.** Both converters call `GeometryObject::link_walls(p, table)` (`geometry.cpp:60`)
once an object's walls exist (`mcell4_converter.cpp:968`, `mcell3_world_converter.cpp:904`).
- **An edge with two walls** links the wall of its first entry, the forward wall, with the wall of
  its second entry, the backward wall:
  - each wall's `nb_walls` names the other;
  - both get one `Edge`, with `edge_num_used_for_init` set to the first entry's side and its
    constants from `Edge::reinit_edge_constants`.

  Entries follow the order of the model's walls, so the first wall that uses an edge is its forward
  wall. That is also the order in which walls were once linked by coordinates.
- **A boundary edge** keeps no neighbour and an uninitialized edge.
  `nutmeg4_pymcell4/2860_validate_volumetric_mesh` depends on this ("Wall side 10: neighbor wall
  with index 0 not found.", `geometry.cpp:281`).
- **No edge has three or more walls** here: such meshes stop at check 6.
- **Wall order:** an object's walls are created one after another, so table wall `j` is
  `wall_indices[j]` (`geometry.cpp:63`).
- **Wall constants:** MCell4 computes every wall's constants with
  `Wall::initialize_wall_constants`, on MCell4's MDL path too (`mcell3_world_converter.cpp:662`).

**Region borders.** `Region::init_border_edges(obj, table)` (`geometry.cpp:589`) applies one rule
to every region, `ALL` included, for both converters.
- Side `k` of a wall of the region is a border when its edge has exactly one wall in the region:
  a boundary edge of the mesh, or an edge whose other wall is outside the region.
- So a region has no border only when none of its edges has a wall outside it: the whole of a
  watertight mesh, or whole watertight shells of an object.
- MCell3's engine gives no borders to `ALL`, or to a region that holds every wall (§6.12).

**Who reads the borders: surface molecules only.** A volume molecule meets a surface class only when
it hits a wall.
- **The crossing:** `WallUtils::is_wall_edge_region_border` (`wall_utils.inl:126`) is called when a
  surface molecule of a species with surface-class rules crosses a wall's side
  (`diffusion_utils.inl:659`, `:698`). `GeometryObject::get_neighbor_side` (`geometry.cpp:49`) gives
  the side of the wall across, from the table (`diffusion_utils.inl:695`).
- **The grid's tile checks** use region membership, not borders.
- **What a surface class does at a border:**
  - REFLECTIVE reflects;
  - ABSORPTIVE absorbs;
  - TRANSPARENT lets the surface molecule pass.

  `RxnUtils::is_absorptive_at_region_border` (`rxn_utils.inl:48`) decides. A surface-class rule
  absorbs at a border only when it is neither TRANSPARENT nor REFLECTIVE and has the absorptive
  shape. It is used at `diffusion_utils.inl:621`, `wall_utils.inl:375` and `region_utils.cpp:32`.
- **A rule's orientation mark** (`S'`, `S,`) selects the surface molecule's own orientation, not
  the side it comes from.

### 6.5 What each object keeps for the run

**The kept check.** Each `GeometryObject` keeps the converter's `MeshCheck` as `mesh_check`
(`geometry.h:136`), with `first_vertex_index` (`:140`), the partition's index of its vertex 0.
- **Partition indices:** object vertex `v` is partition vertex `first_vertex_index + v`, and table
  wall `j` is `wall_indices[j]`.
- **When it's set:** the converters set it with `set_mesh_check` (`geometry.cpp:116`) once the
  walls are linked and the regions have their borders (`mcell4_converter.cpp:1028`,
  `mcell3_world_converter.cpp:910`).
- **Index facts never change** during a run, since vertex moves change only coordinates: the table,
  the flags `manifold()`, `consistently_oriented()` and `watertight()` (`mesh_table.hpp:335-343`),
  the boundary edges and the shells.

**Coordinate facts.** `get_area` and `get_signed_volume` (`geometry.cpp:176`, `:184`) return the
check's values until a vertex of the object moves.
- **After a vertex move:** `Geometry::update_moved_walls` marks the facts out of date
  (`geometry.cpp:1000`).
- **The next call recomputes them** as `check_mesh` computed them (`update_coordinate_facts`,
  `:157`):
  - every vertex of the object in order, at its current position, and the walls in order, with
    their vertices from the table (`get_mesh_as_given`, `:135`);
  - then `volume_and_area` for a watertight object and `surface_area` for any other.
- **The signed volume** is a `std::optional<double>`, empty for a non-watertight object. No code
  computes a signed volume for a non-watertight mesh or shell.

**Per-vertex lists**, built from the table on first use (`geometry.cpp:192`, `:202`):
- `get_walls_of_vertex`: the walls that use a vertex, ascending;
- `get_neighbors_of_vertex`: its 1-ring, the other ends of its edges in edge order.

`get_wall_for_vertex_pair` (`:98`) returns the first wall, in wall order, that uses both vertices.

**Who reads the kept facts:**

| Reader | What it reads |
|---|---|
| Counted volumes, compartments (§6.6) | the flags, asserted: the usage errors stopped anything else (`counted_volume_utils.cpp:58`, `compartment_utils.cpp:46-47`) |
| Usage errors (§6.3) | the flags, the boundary edges, the shells that face the wrong way |
| Releases of volume molecules by concentration into one object | the signed volume (`release_event.cpp:866`) |
| BNGL export | the volume (`get_geometry_object_volume`, `counted_volume_utils.cpp:347`) and the area (`bngl_exporter.cpp:99-105`, `:208-217`) |
| `Model.get_object_area`, `get_object_volume` | the area, the flags and the signed volume (`introspection.cpp:225`, `:238-251`) |
| OBJ export | `get_mesh_as_given` (`counted_volume_utils.cpp:371`) |
| Vertex moves | the walls of the moved vertex (`partition.cpp:244`, `:500`), its 1-ring (`:589`) and the wall of a vertex pair (`:627`) |
| `Model.get_vertex_unit_normal` | the walls of the vertex (`introspection.cpp:187`) |
| The corner-tile search of surface molecules | the walls of each shared vertex (`wall_utils.inl:91`) |
| Region-border crossings of surface molecules | the side across (`diffusion_utils.inl:695`) |

**Memory.** On the full neuropil (3,990,190 walls), the converted model holds 156 MB more than
before the project, about 39 bytes per wall. That covers the kept checks and the release walls that
MCell4 computes itself. The run peaks at 10.46 GB, against 10.33 GB before the project.

### 6.6 What MCell does with it

1. **Counted-volume hierarchy** (volume molecules).
   - **The entry point:** counting volume molecules inside objects needs, for each counted object,
     the counted volume that a volume molecule enters and the one it returns to.
     `World::init_counted_volumes` (`src4/world.cpp:146`) calls
     `CountedVolumeUtils::initialize_counted_volumes` (`counted_volume_utils.cpp:308`).
   - **The steps:**
     - it makes a `Solid` of each counted object from `get_unmerged_mesh`: the walls as given, with
       their points numbered in the order the walls first use them and nothing merged
       (`prepare_counted_object_solids`, `:49`);
     - it classifies every pair of objects with `GeometryAnalysis::containment`
       (`compute_containement_mapping`, `:75`);
     - it sets each object's `counted_volume_index_inside` and `_outside`
       (`define_counted_volumes`, `:252`):
       - an object that intersects another counted object gets `COUNTED_VOLUME_INDEX_INTERSECTS` on
         both sides, so the counted volume of a volume molecule that crosses one of its walls is
         computed from the molecule's position;
       - any other object takes its inside index from its parents. Its outside index comes from its
         direct parent: the parent on the longest containment path (`get_direct_parent_info`,
         `:151`), with the path lengths cached in a `PathLengthCache`.
   - **Intersecting counted objects:** in such models, a volume molecule released inside an object
     copies its counted volume from the nearest waypoint when no counted wall lies between.
     - Waypoints are computed at initialization. After the first vertex move, the counted volume is
       always computed from the walls' current positions (`collision_utils.inl:1565-1569`).
   - **Identical counted objects** give "Identical counted objects are not supported yet, error for
     <A> and <B>." (`counted_volume_utils.cpp:111-112`, printed as a warning), then the run stops
     with "Processing of counted volumes failed, terminating." (`world.cpp:151`).
   - **Non-watertight and inside-out counted objects** never get this far: the usage errors stop them.
2. **Compartments** (BNGL compartments in the Python API, volume and surface molecules).
   - `MCell4Converter::convert_compartments` (`mcell4_converter.cpp:1114`) first runs
     `check_compartments_watertight` (§6.3).
   - Then `set_parent_and_children_compartments` (`compartment_utils.cpp:85`) finds each
     compartment's parent with the same containment mapping and parent search. It works on the
     meshes in the internal units the simulation will use.
   - Compartments that intersect, or a compartment with several parents, raise a `RuntimeError`.
3. **Releases into and onto regions.** Both converters compute each release's walls, areas and box
   themselves: Python API models at `mcell4_converter.cpp:1335` and `:1460`, MCell4's MDL path at
   `mcell3_world_converter.cpp:1545` and `:1597`.
   - **Volume molecules** released into a region are placed by parity, with the inside test of each
     object (`collision_utils.inl:1484`), within the box of the region expression.
     - By concentration into one object, the number comes from the object's signed volume
       (`num_vol_mols_from_conc`, `release_event.cpp:857`).
     - Into an expression, the number comes from the box's volume.
     - A negative number from a concentration removes volume molecules, truncated toward zero as on
       MCell3's engine (`check_max_release_count`, `:523`).
   - **Surface molecules** released onto a region use its walls and their cumulative areas
     (`initialize_walls_for_release`, `:457`).
     - By density, the number comes from the total area.
     - A negative density removes surface molecules, also truncated toward zero.
   - **After vertex moves** (Python API models only):
     - `Geometry::update_moved_walls` increments the partition's count of vertex moves
       (`Partition::get_vertex_moves_count`, `partition.h:919`; `geometry.cpp:1007`).
     - A release into or onto a region recomputes its box and its walls' cumulative areas when the
       count has changed since it last computed them (`update_region_data_after_vertex_moves`,
       `release_event.cpp:497`, called at `:1301`).
     - The regions of moved objects drop their box, walls per subpartition and waypoints
       (`geometry.cpp:1011-1016`, `Region::set_coordinates_changed`, `geometry.h:374`), and
       rebuild them at their next use.
     - A release at time k·dt runs in iteration k+1, after a vertex move applied after iteration k.
   - **MCell4's MDL path** takes the whole object for a region other than `ALL` in a release of
     volume molecules (§6.12).
4. **Clamps** (volume molecules). One function builds the clamp events for both converters:
   `create_clamp_release_events` (`clamp_release_event.cpp:268`; contract at
   `clamp_release_event.h:86-98`), called at `mcell4_converter.cpp:839` and
   `mcell3_world_converter.cpp:2056`.
   - **The events:** one per clamp (surface class, volume molecule, side), over every wall that
     carries the class through one of its regions. Each wall is used once, in wall order.
   - **Their order:** by class name, then molecule name, then side, whatever order the converter
     gives.
   - **A wall clamped more than once** for the same molecule and side stops the run, as a
     `ValueError` or with exit code 1. The text: `[CLAMP_ON_WALL_TWICE] Scene.box1: 2 walls are
     clamped more than once for the volume molecule V (surface class clamp_sc through the regions
     ALL and top)`.
5. **The BNGL export** writes each compartment's volume and area from the kept facts
   (`bngl_exporter.cpp:99-105`, `:208-217`).
   - A non-watertight compartment gives "Compartment object <name> is not watertight and its volume
     cannot be computed." (`:101`, `:210`).
6. **`data_model_to_pymcell`.**
   - **BNGL compartments:** for each one, `compute_volume_and_area` (`data_model_geometry.cpp:28`,
     called at `bngl_generator.cpp:236`) checks the data model's lists with `check_mesh`.
     - It refuses a mesh with one of these messages:
       - "Triangle point index N is out of range.";
       - "Geometry object X has invalid vertex indices, could not compute volume.";
       - "Non-manifold mesh is not allowed, error for X.";
       - "Geometry object X is not watertight, could not compute volume."
         (`data_model_geometry.cpp:56-70`).

       The orientation isn't checked there.
     - The converter then prints "Warning: could not compute volume of a geometry object: ...". It
       writes the message as a comment in `model.bngl` and uses volume 1
       (`bngl_generator.cpp:239-243`).
   - **Dynamic objects:** before writing any file, `check_dynamic_objects`
     (`mcell4_generator.cpp:58`) stops on an object with `"dynamic": true`. The text: "Error: geometry
     object Cube is dynamic, which MCell4 doesn't support; run this model on MCell3's engine, or move
     vertices through the Python API (Model.add_vertex_move)."
7. **The getters** (Python API).
   - **What they return:** `Model.get_object_area(object)` and `Model.get_object_volume(object)`
     (`introspection.cpp:217`, `:229`) return um^2 and um^3, at the walls' current positions.
   - **The volume is signed:** it is negative for an inside-out object, and a cavity is subtracted.
   - **A non-watertight object:** the volume getter raises `ValueError: Geometry object
     <name> is non-watertight (4 boundary edges), so it has no volume; see <report>`. The area is
     defined for every object.
   - **Before `initialize()`, or with `None`,** both getters raise the errors of the other geometry
     getters.
8. **The OBJ export** (Python API).
   - **The path:** `Model.export_geometry` (`model.cpp:174`) calls `World::export_geometry_to_obj`
     (`world.cpp:721`), which calls `export_geometry_objects_to_obj` (`counted_volume_utils.cpp:359`).
   - **What it writes:** each object as given, every vertex of its `vertex_list` in that order, at
     its current position, in um. The faces are its walls in `wall_list` order, with the model's
     indices.
   - **The files:** `GeometryAnalysis::write_obj` writes one named object per geometry object, each
     with a white material of opacity 0.25. The first line of each file is `# Wavefront OBJ file
     written by MCell` or `# Wavefront MTL file written by MCell`.
   - **A failed write** prints "Could not write geometry to <prefix>.obj and <prefix>.mtl.", and the
     run continues (`:381`).
9. **Vertex moves exist only in Python API models** (`Model.add_vertex_move`).
   - **MCell4's MDL path:** `MCell3WorldConverter` stops an MDL file with `DYNAMIC_GEOMETRY`, also for
     `-mdl2datamodel4` (`mcell3_world_converter.cpp:108-112`). The text: "MCell4 doesn't support
     DYNAMIC_GEOMETRY; run this model on MCell3's engine (without -mcell4), or move vertices through
     the Python API (Model.add_vertex_move)."
   - **Wall-wall collisions of a move** are checked from the per-vertex lists (§6.5, §6.12).

### 6.7 `GeometryAnalysis`: function → what it does → used by

| Function | What it does | Used by |
|---|---|---|
| `unmerged_mesh` (`geometry_analysis.hpp:50`) | The triangles as given, their points numbered in the order the triangles first use them. Nothing is merged or dropped; unused points are left out. | counted volumes and compartments, through `MeshUtils::get_unmerged_mesh` |
| `find_inside_out_parts` (`geometry_analysis.hpp:65`) | The connected components that face the wrong way for their nesting. | the test program's cross-check only |
| `judge_parts` (`geometry_analysis.hpp:81`) | Which way each given part faces, by the same rule: a part must face outward when an even number of the other watertight parts enclose it, inward (a cavity) when an odd number does. Only watertight, consistently oriented parts whose sign is certain are judged. | `check_mesh` (the shells) |
| `volume_and_area` (`geometry_analysis.hpp:99`) | Signed volume and area, computed as the simulation computes them (§6.8). For watertight triangles only. | `check_mesh`, `GeometryObject::update_coordinate_facts`, `data_model_to_pymcell` |
| `surface_area` (`geometry_analysis.hpp:103`) | The area of `volume_and_area` alone, the same terms in the same order, so the same bits. | `check_mesh` and `update_coordinate_facts` for non-watertight meshes and shells |
| `triangles_intersect` (`geometry_analysis.hpp:106`) | Tells exactly whether two triangles share a point. It uses Guigue–Devillers on the predicates (`geometry_analysis.cpp:369`). | `containment` |
| `Solid` (`geometry_analysis.hpp:128`) | A mesh prepared for containment tests: a bounding-volume hierarchy of its triangles, plus its connected components. | counted volumes, compartments, `judge_parts` |
| `point_in_solid` (`geometry_analysis.hpp:118`) | Returns `Outside`, `Inside`, `OnSurface` or `Undecided`. An exact on-surface test comes first; then the parity of the crossings along a ray decides. A ray that grazes an edge or a vertex is replaced by one in another direction (`geometry_analysis.cpp:826-828`). | `judge_parts`; `containment` through the same code |
| `containment` (`geometry_analysis.hpp:125`) | Returns `Disjoint`, `Identical`, `Intersect`, `FirstInSecond` or `SecondInFirst` (`geometry_analysis.cpp:1176`; the rule is in §6.8). | counted volumes, compartments |
| `number_to_string` (`geometry_analysis.hpp:147`) | The shortest text that reads back as the same double (`std::to_chars`), in `vtkNumberToString`'s notation. | the OBJ export, the report's coordinates |
| `write_obj` (`geometry_analysis.hpp:161`) | Writes `<prefix>.obj` and `<prefix>.mtl` in the layout of MCell's former patched `vtkOBJExporter`. The coordinates are written as doubles, and the first line of each file names MCell. | the OBJ export |

### 6.8 Definitions that matter

- **Connectivity comes from the vertex indices only.** Two walls are neighbours when they use the
  same two vertex indices for a side; equal coordinates never join walls. A cube whose faces each
  have their own four vertices is therefore non-watertight (24 boundary edges), with a warning for
  8 groups of equal coordinates.
- **Manifold mesh:** no edge has three or more walls, and at every vertex the walls form a single
  fan, which goes all the way around the vertex or, at the mesh's boundary, doesn't.
- **Consistently oriented:** the two walls of every edge run along it in opposite directions.
- **Watertight:** every edge has exactly two walls; with check 6 passed, no edge has only one.
  - The words are "watertight" and "non-watertight", never "closed" or "open", in MCell4's
    messages, the report and these docs.
  - A non-watertight surface is fine to simulate. Only counting inside it, using it as a
    compartment, or releasing volume molecules into it has no meaning.
- **Shells** are walls joined through edges of two walls. Once check 6 passes, they are exactly the
  connected pieces.
- **Facing.** Walls follow the right-hand rule, so their normals point outward and a watertight
  shell that faces outward has a positive signed volume.
  - A shell must face outward when an even number of the object's other shells enclose it, and
    inward (a cavity) when an odd number does.
  - An inside-out object has a negative signed volume.
- **Volume and area.**
  - **The volume:** the sum of the signed volumes of the tetrahedra that join each wall to a
    reference point d, the center of the walls' bounding box (`geometry_analysis.cpp:1106-1119`).
    The term is (a − d)·((b − d)×(c − d))/6, the determinant written out in six terms after
    O'Rourke (`:1127-1135`). The walls are used as given and in order. It is the 3D form of the
    shoelace formula.
  - **For a watertight mesh** the tetrahedra cover the enclosed solid once and cancel everywhere
    else, so the sum doesn't depend on d. For a non-watertight mesh it does, so MCell never
    computes it there.
  - **The area:** the sum of `0.5 |(b − a) × (c − a)|` over the walls, the expression of
    `Wall::initialize_wall_constants` (`src4/wall.cpp:304`).
  - **One function for every use:** `volume_and_area` serves the checks, the kept facts,
    releases, the BNGL export, the getters and `data_model_to_pymcell`.
- **Zero area:** at most 1e-12 internal units squared (`EPS`, `libbng/bng/shared_defines.h:87`),
  or NaN. This is the test of `Wall::initialize_wall_constants` (`src4/wall.cpp:306`); MCell3's
  test is the same.
- **Precision.** Computation and output use double everywhere.
  - Every decision (touching, inside) takes its signs from the exact predicates.
  - Decisions are therefore exact as long as no intermediate product underflows or overflows:
    coordinate differences between about 1e-80 and 1e80, or zero (`geometry_analysis.hpp:22-24`).
- **Containment rule** (the same as the former VTK code):
  - Surfaces that touch or cross are `Identical` when their point lists are equal, and `Intersect`
    otherwise (`geometry_analysis.cpp:1183`).
    - The point lists come from `unmerged_mesh`, so two copies count as identical only when they
      list the same points in the same order of first use.
  - Otherwise the first solid is inside the second when any of its connected components is inside.
    One point decides for each component, since the surfaces don't touch.
  - Failing that, the same test runs the other way round. If neither holds, the solids are
    `Disjoint`.
- **No floating-point contraction** in the geometry library (§6.10).
  - Fused multiply-add breaks the predicates' error-free arithmetic, and it would change the
    volume's last bits from one platform to another.
  - All of MCell is compiled without contraction, so exported volumes, and simulations, are the
    same on ARM64 as on x86-64.

### 6.9 Errors and warnings that users see

**The checks of each mesh** (§6.2, §6.3) apply to:
- Python API models, at `initialize()`, including data model → Python;
- MDL files on MCell4's MDL path (`mcell -mcell4`, `-mdl2datamodel4`): the vertex indices before
  MCell3's initialization, the rest for whatever MCell3's parser and initialization let through
  (§6.2).

Errors stop the run once all objects are checked; warnings don't.

| Code | Kind | Text (singular form) | Notes |
|---|---|---|---|
| `MESH_INVALID_INDICES` | error | `1 wall with a vertex index out of range`, `1 wall that uses a vertex twice`, `1 vertex that doesn't have three coordinates`, `1 wall that doesn't have three vertex indices`, `1 surface region with a wall index out of range`, joined with ", " in one line; or `too many vertices or walls to check` | MCell4's MDL path: a vertex index out of range, in any wall, stops the run before MCell3's initialization; a wall that uses a vertex twice is reported by the converter (§6.2) |
| `MESH_ORPHANED_VERTICES` | error | `1 orphaned vertex, used by no wall` | with `REMOVE_ELEMENTS`, also a vertex that only removed walls use |
| `MESH_DUPLICATE_WALLS` | error | `1 group of walls with the same three vertices` | |
| `MESH_ZERO_AREA_WALLS` | error | `1 wall with zero area` | MCell4's MDL path: also the walls MCell3's initialization dropped as degenerate |
| `MESH_NOT_MANIFOLD` | error | `1 edge with 3 or more walls, 2 non-manifold vertices`; with vertices only, `1 non-manifold vertex, where groups of walls touch only at a vertex` | |
| `MESH_INCONSISTENT_ORIENTATION` | error | `1 pair of walls runs along its shared edge in the same direction` | |
| `MESH_DUPLICATE_VERTICES` | warning | `1 group of vertices with equal coordinates` | |
| `MESH_NOT_WATERTIGHT` | warning | `1 boundary edge` | |
| (the stop) | error | `2 geometry objects have errors: Scene.fin, Scene.flipped; see <report>` | `ValueError` in Python; exit code 1 on MCell4's MDL path |

The plural forms are `2 walls with ...`, `2 pairs of walls run along their shared edge in the same
direction`, `4 boundary edges`, and so on (`mesh_table.cpp:947-1026`).

**Usage errors** (§6.3), for volume molecules, apply to Python API models and to MDL files on
MCell4's MDL path. `Model.release_molecules` checks its own release.

| Code | Text |
|---|---|
| `REGION_NOT_WATERTIGHT` | `<object>: the model counts inside this object, but it is non-watertight (4 boundary edges)`; `this object is a compartment, but ...`; `release site <name> releases volume molecules into this object, but ...`; `... removes volume molecules from this object, but ...` |
| `MESH_INSIDE_OUT` | `<object>: the model counts inside this object, but 1 shell faces the wrong way`; `this object is a compartment, but 2 shells face the wrong way`; `release site <name> releases volume molecules into this object by concentration, but ...` |
| (the stop) | `2 geometry objects have usage errors: Scene.box, CP; see <report>` |

**Other errors:**

| Case | Message | Front end |
|---|---|---|
| a wall clamped twice (§6.6) | `[CLAMP_ON_WALL_TWICE] ...` | both |
| MDL `DYNAMIC_GEOMETRY` | `Error: MCell4 doesn't support DYNAMIC_GEOMETRY; ...` (§6.6) | MCell4's MDL path (`-mcell4`, `-mdl2datamodel4`) |
| a dynamic object in a data model | `Error: geometry object Cube is dynamic, which MCell4 doesn't support; ...` | `data_model_to_pymcell` |
| `Model.get_object_volume` of a non-watertight object | `ValueError: Geometry object <name> is non-watertight (4 boundary edges), so it has no volume; see ...` | Python API |
| two identical counted objects | "Identical counted objects are not supported yet, error for <A> and <B>." (printed as a warning), then "Processing of counted volumes failed, terminating." | both |
| BNGL compartments that intersect, or with several parents | `RuntimeError` (`compartment_utils.cpp:100`, `:109`, `:131`) | Python API |
| two walls whose shared edge is extremely short | "Detected malformed geometry object" (`src4/wall.cpp:115-117`, `:178`, `:200`); also after a vertex move | both |
| a release or count of volume molecules on a surface region | "Cannot release volume molecule onto a surface region ." / "Cannot count volume molecules or reactions on a surface region ." (`mcell4_converter.cpp:1254`, `:1257`) | Python API |
| a vertex or wall index below 0, or not below the object's count, passed to `get_vertex`, `get_vertex_unit_normal`, `add_vertex_move`, `get_wall`, `get_wall_unit_normal`, `get_wall_color` or `set_wall_color` | `RuntimeError: Vertex index -1 is out of range for vertex_list of box.`; `RuntimeError: Wall index -1 is out of range for wall_list of box.` (`libmcell/api/geometry_object.h:79-91`) | Python API |

**Other messages** (the run goes on):
- The BNGL export: "Compartment object <name> is not watertight and its volume cannot be computed."
  (§6.6).
- `data_model_to_pymcell`, for a BNGL compartment whose mesh it refuses: "Warning: could not compute
  volume of a geometry object: ..." (§6.6).
- A failed OBJ write: "Could not write geometry to <prefix>.obj and <prefix>.mtl."
- On MCell4's MDL path, MCell3's own messages come first wherever MCell3's parser or
  initialization stops a file (§6.2).

### 6.10 Build

- **The library** (`src4/CMakeLists.txt:42-122`).
  - `GEOMETRY_SOURCE_FILES` (`src4/CMakeLists.txt:45-49`: `predicates.cpp`,
    `geometry_analysis.cpp`, `mesh_table.cpp`) make up the static library `mcell_geometry`, alias
    `MCell::geometry` (`:112-116`).
  - `mcell4` links it publicly (`:122`).
  - Linked privately by:
    - `libmcell` (`libmcell/CMakeLists.txt:157`);
    - `data_model_to_pymcell` (`utils/data_model_to_pymcell/CMakeLists.txt:28`);
    - the test program `mesh_utils_test` (`utils/mesh_utils_test/CMakeLists.txt:23-25`), added by
      `utils/CMakeLists.txt:6`.
- **Per-file flags:**
  - **C++23:** the geometry library's files are compiled as C++23 (`MCELL_CXX23`,
    `src4/CMakeLists.txt:56-60`): `-std=gnu++2b`, or `/std:c++latest` under an MSVC-style front
    end.
    - The rest of MCell is C++17, and the new headers (`.hpp`) stay valid C++17.
    - `mesh_utils_test.cpp` gets the same flag in its own `CMakeLists.txt`.
  - **No contraction:** the geometry library's files are compiled without floating-point
    contraction (`MCELL_NO_FP_CONTRACTION`, `src4/CMakeLists.txt:69-77`): `-ffp-contract=off`;
    `/fp:precise` for MSVC; `/clang:-ffp-contract=off` for clang-cl, which contracts even with
    `/fp:precise`.
  - **glm's warning:** `mesh_utils.cpp` is C++23 and includes glm through MCell's headers, so glm's
    deprecated-volatile warning is switched off for it (`MCELL_CXX23_GLM`,
    `src4/CMakeLists.txt:83-91`).
  - **All of MCell** is compiled without contraction since `0652d19b`.
    - `mcell_project_options` passes `-ffp-contract=off` (`/clang:-ffp-contract=off` for
      clang-cl) to every first-party and bundled target (`mcell/CMakeLists.txt:132-140`).
    - On x86-64 the code doesn't change. On ARM64 it makes MCell compute as on x86-64: the macOS
      full-suite check found 30 test runs that failed only there because of contraction
      (doc 07 §8).
  - **Two engine files** were switched off on their own before that, and still are
    (`src4/CMakeLists.txt:93-110`). The macOS check found both (doc 07 §8):
    - `bngl_exporter.cpp`: a compartment's exported volume is its volume times `length_unit^3`
      minus its children's volumes, one multiply-subtract;
    - `diffuse_react_event.cpp`: the tile-neighbor search across a shared edge (`grid_utils.inl`)
      places tiles from the edge's length and compares the positions exactly.
- **Platform floor:** GCC 11+, MSVC 2022, or Xcode 15+ on macOS 13.3+. This comes from
  `std::to_chars` for double, `std::ranges` and `std::bit_cast`. MCell sets no
  `CMAKE_OSX_DEPLOYMENT_TARGET`.
- **No VTK anywhere.** There is no `find_package(VTK)`, no `${VTK_LIBRARIES}` and no
  `vtk_module_autoinit`. `mcell_tools` (branch `mcell4_nanobind`) has no VTK build step (doc 05).

### 6.11 Tests that exercise it

**The test program** `bin/mesh_utils_test` (source `mcell/utils/mesh_utils_test/`) checks the code
of `mesh_table.hpp`. Its data is in `mcell_tests/tests/utils_tests/mesh_utils/` (README there).
- **How it runs:** the set `mesh_utils` of `mcell_tests/test_configs/external.toml` runs it
  (`TesterExternal`).
- **Size:** 137 connectivity meshes, 30 report meshes and 3,534 checks, in 0.06 s.
- **The connectivity meshes** carry the expected results of the prototype `coboundary.py`, of
  SciPy and of NumPy. They were computed once, outside the repositories; the test never runs
  Python.
- **The report meshes** each have a hand-checked report block and terminal lines, one per error or
  warning. Six of them combine non-manifold edges and vertices.
- **Cross-checks:**
  - every area against the sum of the walls' areas, bit for bit;
  - the wrong-facing shells against `find_inside_out_parts` on `unmerged_mesh`;
  - the orphaned vertices against the vertices that no wall of the file uses;
  - the singular and plural form of every line, and the stops.

**Model tests** (`mcell_tests/tests/`):

| What | Tests | Molecules |
|---|---|---|
| Mesh errors and warnings: every line, the stop, the exit code, the report lines | `nutmeg4_mdl/0240_mesh_errors_error`, `0250_mesh_warnings`, `0260_mdl2datamodel4_mesh_error`; `nutmeg4_pymcell4/0780_mesh_errors_error` (with setters), `0790_mesh_warnings` | none |
| Orphaned vertices (with `REMOVE_ELEMENTS` in the MDL test) | `nutmeg4_mdl/0230_orphaned_vertices_error`, `nutmeg4_pymcell4/0770_orphaned_vertices_error` | none |
| Vertex indices out of range on MCell4's MDL path, stopped before MCell3's initialization, also in a wall that `REMOVE_ELEMENTS` removes | `nutmeg4_mdl/0270_vertex_index_out_of_range_error` | none |
| Usage errors | `nutmeg4_mdl/0210_usage_errors_inside_out_and_non_watertight_error`; `nutmeg4_pymcell4/0730_usage_errors_inside_out_and_non_watertight_error`, `0740_compartment_not_watertight_error`, `0750_release_molecules_usage_error` | volume |
| Dynamic geometry stops | `nutmeg4_mdl/0220_dynamic_geometry_error`, `nutmeg4_pymcell4/0760_data_model_dynamic_object_error` | volume |
| A data model's broken compartment mesh, BNGL mode | `nutmeg4_pymcell4/0800_data_model_compartment_mesh_error` | volume and surface |
| `REMOVE_ELEMENTS`, a region after the removed walls | `mdl/1340_remove_elements_surf_mols_on_region_vol_mols_from_point_it_100` | surface, and volume from a point source |
| Region borders on non-watertight meshes | `mdl/2775_surf_absorptive_borders_non_watertight`, `2776`-`2779` | surface |
| TRANSPARENT at region borders | `mdl/2785`-`2787` | surface |
| Clamps | `mdl/2690_conc_clamp_two_separate_regions`, `2691_flux_clamp_two_separate_regions`; `nutmeg4_mdl/0200_clamp_on_wall_twice_error`, `nutmeg4_pymcell4/0720_clamp_on_wall_twice_error` | volume |
| Removals by a negative concentration or density | `mdl/0390_negative_vol_conc_release_it_10` (volume), `mdl/1330_surf_mol_on_cube_negative_density_release_it_10` (surface) | volume; surface |
| Releases after vertex moves | `pymcell4_positive/1560_release_into_moved_object_vol_mols`, `1570_release_onto_moved_region_surf_mols` | volume; surface |
| Counts after a move, intersecting counted objects | `pymcell4_positive/1580_count_in_intersecting_objects_after_vertex_move` | volume |
| Counts in intersecting objects with a parent | `pymcell4_positive/2610_count_in_intersecting_objects_w_parent` | volume |
| The getters | `pymcell4_positive/1350_get_object_area_volume`; `1340_get_vertex` (argument errors) | none |
| Vertex and wall indices out of range in the getters, `set_wall_color` and `add_vertex_move` | `pymcell4_positive/1360_index_argument_errors` | none |
| OBJ export, byte for byte | `pymcell4_positive/3100_export_geometry_obj`, `3120_dyn_geom_export_geometry_obj` | none |
| A boundary edge keeps no neighbour | `nutmeg4_pymcell4/2860_validate_volumetric_mesh` | none |
| Errors after vertex moves (`src4/wall.cpp`) | `nutmeg4_pymcell4/1600_message_dyn_geom_obj_degenerate`, `3150_dyn_vert_wall_area_0` | none |

- **The BNGL export tests** (`test_configs/bngl_to_pymcell4_export.toml`) write each volume into
  `exported.bngl` with 17 digits, then run the exported model again.
- **Every model that MCell4 converts** writes a geometry report, so the suite's other MCell4 tests
  run the checks too.
- **`mdl/2810_wall_overlap_2boxes_inside_one`** (todo) stops with `MESH_INCONSISTENT_ORIENTATION`
  (§6.12).

### 6.12 Known issues

Issues of the engine that aren't about geometry, such as barriers and the order within an
iteration, are in §3.7.

**MDL files** (each item names the path: MCell3's engine, or MCell4's MDL path through
`MCell3WorldConverter`):
- **On MCell3's engine, a vertex index equal to the vertex count, or negative, still crashes.**
  - MCell3's initialization compares with `>` instead of `>=` and doesn't reject a negative index
    (`src/init.c:2092-2095`). `init_tri_wall` then reads past the vertex array, or `vertices[-1]`
    (`src/init.c:2098`).
  - MCell3's code stays unchanged. On MCell4's MDL path, MCell4's check before MCell3's
    initialization stops such a file with `MESH_INVALID_INDICES` (§6.2).
- **On MCell4's MDL path, a region other than `ALL` in a release of volume molecules becomes the
  whole object.**
  - Where: `mcell3_world_converter.cpp:1425-1427`, `:1444-1446`.
  - Example: a release into `Scene.box[r]`, with `r` one watertight shell of a two-shell object,
    places volume molecules in both shells. MCell3's engine places them only in `r`. A release by
    concentration takes the whole object's volume.
  - A count of volume molecules inside such a region stops with a developer message
    (`Conversion error: Expected 'ends_with(reg_name, ",ALL")' is false.`, `:1970`).
  - The Python API refuses both.
  - No suite test has either case.
  - Planned: a clear error in the converter, as the Python API gives, or support for regions made
    of whole watertight shells on MCell4's MDL path and for Python API models alike.
- **A spherical release by a negative concentration** (volume molecules):
  - MCell3's engine releases nothing;
  - on MCell4's MDL path, MCell4 stops at that release with the misleading "tries to release more
    than INT_MAX" message
    (`check_max_release_count`, `release_event.cpp:523`);
  - the Python API refuses such a site when it's made ("Negative release value of concentration
    may be set only when region is set.", `libmcell/api/release_site.h:101`).

  Possible changes: the Python API's clear error on MCell4's MDL path too, or removal inside the
  sphere for both.
- **On MCell4's MDL path, some values still come from MCell3's initialization:** the partition's
  placement (`mcell3_world_converter.cpp:229-329`) and the random-number state (`:225`). Planned:
  MCell4's MDL path without MCell3's initialization (below).
- **The two engines' region borders differ** (by design; §6.4).
  - MCell3's engine gives no borders to `ALL`, or to a region that holds every wall
    (`src/init.c:2353-2354`, `:2373-2374`). MCell4 gives them the boundary edges of a
    non-watertight mesh.
  - So an ABSORPTIVE rule for a surface molecule on such a region gives different results:
    `mdl/2775` keeps 1000 surface molecules on MCell3's engine and 704 on MCell4's MDL path.
- **Conflicting surface classes on one wall** are checked only by MCell3's initialization
  (`check_for_conflicting_surface_classes`, `src/init.c:5209`), which both paths for MDL files run.
  An example is a CLAMP with a TRANSPARENT, REFLECTIVE or ABSORPTIVE rule for the same molecule. So
  an MDL file gets MCell3's errors on both paths, and Python API models get none. Planned: MCell4's
  own checks for both converters (a wall clamped twice already stops, §6.6).

**Geometry and its uses:**
- **Walls that cross at an angle aren't detected.** `check_for_overlapped_walls`
  (`wall_overlap.cpp:326`) finds only overlaps of walls in the same plane. Planned: a check for walls that cross.
- **If the parity ray grazes an edge closely enough to need a redo, the run aborts.** Where:
  `release_assert` at `collision_utils.inl:1539`; the comment at `:1524` calls it a known issue.
- **A wall with nonzero area but an extremely short shared edge** stops the run with "Detected
  malformed geometry object" (`src4/wall.cpp:178`, `:200`).
- **The wall–wall hit reported for a vertex move depends on the vertex numbering** (Python API).
  - The check visits a moved vertex's 1-ring in ascending index order: the table's list, sorted at
    `partition.cpp:592` only to keep the order of the code before the table.
  - It stops at the first edge that crosses another object's wall or needs a redo
    (`partition.cpp:644`). So the numbering decides which hit is reported, whether a redo cancels
    the move without one, and the draws from `aux_rng`.
  - Whether the move is cancelled doesn't depend on the order.
  - Planned: every edge tested, every crossing reported, nothing sorted. It will also settle which
    of the edge's two walls is reported, and how a cancelled move is seen by the vertices checked
    after it.
- **The order of a compartment's children** in its release region follows heap addresses
  (`libmcell/api/geometry_object.h:26`, `instantiation.cpp:209-211`). So the data-model output can
  differ between runs; seen in `pymcell4/4200`, `pymcell4/4220` and `pymcell4_positive/4210`.
- **An exported data model lists a clamp class once per clamp event**
  (`ClampReleaseEvent::to_data_model`, `clamp_release_event.cpp:48`).
  - The exporter of reaction rules skips clamp rules (`bng_data_to_datamodel_converter.cpp:387`).
  - So a class with two clamp rules is exported as two entries of the same name.
  - A clamp whose class is on no wall still gets an event, only so that the export keeps the class.
  - Planned: the clamps exported from the model's clamp definitions, one entry per class, and no
    event without walls.

**In test and benchmark models:**
- **The full neuropil stops on MCell4's MDL path.** Private `mdl/3650` (skip), benchmark B4110 and
  `models/salk/neuropil_recon` share its geometry. Two objects stop it:
  - `MESH_NOT_MANIFOLD` for `d010p_sps373` (vertex 123);
  - `MESH_ORPHANED_VERTICES` for `d022` (vertex 17566).

  A copy repaired in both places gives, byte for byte, the outputs of MCell from before the
  project (`0652d19b`).
- **`mdl/2810_wall_overlap_2boxes_inside_one` (todo, skipped on MCell3's engine) gives the wrong
  cause.** Its `todo` note blames MCell3's initialization for the converter's crash on MCell4's MDL
  path, but its object `Outer` has walls 10-18 reversed (7 inconsistent edges). On MCell4's MDL
  path MCell4 now stops it with `MESH_INCONSISTENT_ORIENTATION`.
- **`d000_er` in the neuropil models** has 12 shells that face inward and lie inside no other shell
  (11 tetrahedra and one 22-wall piece).
  - The report lists them.
  - There is no usage error: `d000_er` isn't counted and isn't a compartment, and its only volume
    use is a release by concentration into an expression, which takes its number from the box.
  - Its oriented surface molecules face reversed on those pieces, as on MCell3's engine.

**Planned work that isn't an issue:**
- **MCell4's MDL path without MCell3's initialization.**
  - Parsing and initialization are separate calls (`src/mcell.c:68`, `:83`). For `-mcell4`, the
    parsed structures would become an `API::Model`, initialized as a Python API model is, which
    would retire `mcell3_world_converter.cpp`.
  - The cost: everything MCell3's initialization works out would have to come from parse-time
    data.
  - Memory: MCell4's MDL path keeps MCell3's model through the whole run, since nothing frees it
    before `mcell4_run_simulation` (`src/mcell.c:98-112`). That is about 4.4 GB of the full
    neuropil's 10.4 GB peak. MCell4's `Wall` is 520 bytes, MCell3's `struct wall` 248.
- **Parallel checks**, if ever needed: split by object, and keep sums in a fixed order.
- **Surface classes on surface molecules:** the interpretation of `S'` and `S,` stays (the mark
  selects the surface molecule's own orientation) until it is redesigned.

---

## 7. The MCell3 Legacy Path (`src/`)

### 7.1 MDL parser (flex/bison)

**MDL = Model Description Language**, MCell3's declarative text input. An MDL file defines
species, reactions, geometry (meshes/objects), surface classes, release sites, viz
output, reaction-data output (COUNT/TRIGGER), and run control (iterations, time step,
partitions, checkpointing).

- `mdlparse.y` (bison, ~2793 lines) — a `%pure-parser` with name-prefix `mdl`, a ~50-member
  `%union` (`mdlparse.y:72-137`), ~390 tokens, start symbol `mdl_format`
  (`mdlparse.y:617`). Top-level entry `mdlparse_init(struct volume *vol)`
  (`mdlparse.y:2707`); per-file driver `mdlparse_file()` (`mdlparse.y:2627`, handles
  `INCLUDE_FILE` recursion).
- `mdllex.l` (flex, ~390 lines) — reentrant lexer (`%option reentrant prefix="mdl"`),
  hundreds of keyword→token rules with aliases (e.g. `D_3D`/`DIFFUSION_CONSTANT_3D`).
- `dyngeom_parse.y` / `dyngeom_lex.l` — a **stripped-down geometry-only** MDL parser
  (prefix `dg`, ~16 tokens) used to re-read mesh geometry at runtime for *dynamic
  geometry*; it understands only objects/vertices/regions/transforms/partitions/includes.

### 7.2 `init.c` and `diffuse.c`

- **`init.c`** (~6805 lines) turns the parsed model into runnable state: reaction
  network, species, geometry (vertices/walls/regions), partitions, molecule placement,
  counters, viz/output, checkpoints. The *orchestrating driver* is
  `mcell_init_simulation()` (`mcell_init.c:185`), which calls init.c building blocks via
  `CHECKED_CALL`: `init_species` (`init.c:543`), `init_vertices_walls` (`:617`),
  `init_regions` (`:709`), `init_partitions` (`:1394`), `instance_obj` (`:1617`,
  recursive object instantiation), `init_surf_mols` (`:2526`), `init_releases` (`:3832`),
  `check_for_overlapped_walls` (`:6733`).
- **`diffuse.c`** (~5321 lines) is the MCell3 per-timestep movement engine. The
  workhorse is `run_timestep()` (`diffuse.c:3784`): it pulls molecules from
  `schedule_next(local->timer)` and dispatches volume mols → `diffuse_3D`
  (`diffuse.c:2967`) and surface mols → `diffuse_2D` (`:3374`) +
  `react_2D_all_neighbors` (`:3553`). Support: `pick_displacement` (`:322`), `ray_trace`
  (`:963`), `exact_disk` (`:1365`, bimolecular probability), the
  `collide_and_react_with_*` family (`:4215`+), `reflect_or_periodic_bc` (`:4689`).
  (Note: this engine only runs when MCell4 is *not* selected.)

### 7.3 The `mcell_*.c` public API helpers

These provide a structured C API over the parser/engine (mostly returning
`MCELL_STATUS`, 0 = success). They are what the MDL parser actions and the legacy
SWIG/pymcell layer call.

| File | Purpose | Key functions |
|---|---|---|
| `mcell_init.c` | state bring-up / config | `mcell_create`, `mcell_init_state` (`:156`), `mcell_init_simulation` (`:185`), `mcell_init_output` (`:396`), `mcell_set_partition`, `mcell_set_iterations`, `mcell_set_time_step` |
| `mcell_run.c` | run loop | `mcell_run_simulation` (`:339`), `mcell_run_iteration` (`:442`), `mcell_flush_data`, `mcell_print_final_statistics` |
| `mcell_reactions.c` | reactions | `mcell_add_reaction` (`:565`), `mcell_add_surface_reaction` (`:870`), `mcell_add_clamp`, `mcell_change_reaction_rate`, `mcell_create_reaction_rates` |
| `mcell_objects.c` | geometry | `mcell_create_poly_object` (`:172`), `mcell_create_instance_object`, `mcell_create_region` (`:1174`), vertex/connection/region list builders |
| `mcell_release.c` | release sites | `mcell_create_geometrical_release_site` (`:161`), `mcell_create_region_release`, `mcell_new_release_pattern` |
| `mcell_species.c` | species | `mcell_create_species` (`:47`), `mcell_add_to_species_list` |
| `mcell_viz.c` | viz output | `mcell_create_viz_output` (`:47`), `mcell_set_molecule_viz_state` |
| `mcell_react_out.c` | COUNT/TRIGGER output | `mcell_create_count` (`:200`), `mcell_new_output_request`, `mcell_get_counter_value` |
| `mcell_surfclass.c` | surface classes | `mcell_create_surf_class` (`:63`), `mcell_assign_surf_class_to_region` |

### 7.4 Central data structures

- `mcell_structs_shared.h` (~74 lines) — the dependency-light subset shared with `src4`
  C++ code: constants, direction bit flags, reaction-outcome sentinels
  (`RX_DESTROY`=0/`RX_A_OK`=1, `MAX_MATCHING_RXNS`=64), `enum viz_mode_t`. No struct
  definitions.
- `mcell_structs.h` (~86 KB) — the central god-header. Highlights:
  - **`struct volume`** (`:1052`) — master global state (partitions/subvolumes,
    geometry, reaction & species hashes, schedulers + per-partition storage, symbol
    tables, object trees `root_object`/`root_instance`, RNG, CLI args incl. the
    `use_mcell4`/`mdl2datamodel4`/`dump_mcell4` interop flags at `:1313-1377`).
  - **Molecules** (C-as-OOP shared layout): `abstract_molecule` (`:708`),
    `volume_molecule` (`:731`), `surface_molecule` (`:760`), `species` (`:549`).
  - **Reactions:** `rxn` (`:586`), `pathway` (`:645`), `product` (`:665`).
  - **Geometry:** `wall` (`:827`), `edge` (`:812`), `surface_grid` (`:885`),
    `geom_object` (`:1862`), `region` (`:1824`), `waypoint` (`:916`).
  - **Releases:** `release_site_obj` (`:1461`), `release_pattern` (`:1493`).
  - **Counting:** `output_block` (`:1644`), `output_set` (`:1667`), `counter` (`:1011`).

### 7.5 How MCell3 results feed MCell4

Already covered in §1.2: the parsed/initialized `volume` struct is passed to
`mcell4_convert_mcell3_volume` → `MCell3WorldConverter::convert` → MCell4 `World`, which
then runs via `mcell4_run_simulation` → `World::init_and_run_simulation`. MCell3 is the
*front-end*; MCell4 is the *engine*.

---

## 8. Entry Points

### 8.1 The `mcell` executable — `src/mcell.c`

`main()` (`mcell.c:33`) is short and linear:

1. `mcell_create()` (`:37`) — allocate the `MCELL_STATE` (= `struct volume`).
2. `mcell_argparse(argc, argv, state)` (`:41`) — parse CLI (wraps `argparse_init`).
3. `mcell_init_state` (`:56`), then `parse_input(state)` (`:68`) — **run the MDL parser**.
   On MCell4's MDL path (`-mcell4`, `-mdl2datamodel4`), `mcell4_check_vertex_indices(state)`
   (`:73-75`) then checks every wall's vertex indices and stops on one out of range (§6.2).
4. `mcell_init_simulation` (`:83`), checkpoint reads, `mcell_init_output` (`:91`).
5. **Engine branch** (`:98`):
   ```c
   if (state->use_mcell4 || state->mdl2datamodel4) {
       mcell4_convert_mcell3_volume(state);          // :99  -> build MCell4 World
       if (mdl2datamodel4) mcell4_convert_to_data_model(...);  // :104  export JSON & exit
       else mcell4_run_simulation(...);              // :109 -> run MCell4 engine
       mcell4_delete_world();                        // :112
   } else {
       mcell_run_simulation(state);                  // :115 -> legacy MCell3 engine
       mcell_print_stats();
   }
   ```

So **the executable chooses the engine purely from flags on the `volume` struct**:
default → MCell3 (`mcell_run_simulation`); `-mcell4`/`-n` → MCell4 converter + run;
`-mdl2datamodel4[viz]`/`-u`/`-a` → MCell4 converter that only exports a JSON data model
and exits.

### 8.2 `argparse.c`

`argparse_init(int argc, char *const argv[], struct volume *vol)` (`argparse.c:142`)
parses with `getopt_long_only` over `long_options[]` (`:50-72`), writing results directly
into the `volume` struct. Engine-selecting flags: `-mcell4`/`-n` → `vol->use_mcell4=1`
(`:347`); `-mdl2datamodel4`/`-u` and `-mdl2datamodel4viz`/`-a` → `vol->mdl2datamodel4`
(`:363/:368`); `-dump_mcell3`/`-dump_mcell4`/`-dump_mcell4_with_geometry` (`:351/:355/:359`);
`-rules`/`-r` → NFsim mode (`:286`, later `setupNFSim_c` at `:410`). The leftover
positional argument is the MDL filename (`:381-406`).

### 8.3 The `mcell4_so` Python module

`NB_MODULE(mcell, m)` is defined in `libmcell/api/bindings.cpp:176`; the module's
import name is **`mcell`**. The CMake target is `mcell4_so`
(`nanobind_add_module(mcell4_so ... libmcell/api/exports.cpp)`,
`mcell/CMakeLists.txt:497`), with its output name forced to `mcell` and suffix `.so`/`.pyd`
(`:573-579`). This module links `MCell::libmcell` + `nanobind-static` + `MCell::mcell4`
(the same engine the executable uses, with `MCell::geometry`, §6.10) plus BNG/JsonCpp/Nauty (`:563-569`). The binding details
(`define_pybinding_*`, code-generated wrappers) are covered by the libmcell document — for
this reference it is enough to note that `mcell4_so` is the Python entry point and drives
the *same* MCell4 `World` engine, just constructed directly from the Python API instead of
from an MDL file.

### 8.4 Summary: executable vs module

```
                ┌─────────────────────────────┐
  mcell.mdl ──► │ mcell EXECUTABLE             │
                │  MDL parse (MCell3, src/)    │
                │  -> struct volume            │
                │  -> mcell3_world_converter   │──┐
                └─────────────────────────────┘  │
                                                  ▼
                                       ┌──────────────────────┐
                                       │  MCell4 World engine │
                                       │  (src4/, MCell::mcell4)│
                                       │  scheduler + events  │
                                       │  + BNG::BNGEngine    │
                                       └──────────────────────┘
                                                  ▲
                ┌─────────────────────────────┐  │
  model.py ───► │ mcell.so  (mcell4_so)       │──┘
                │  Python API (libmcell/)     │
                │  builds World directly      │
                └─────────────────────────────┘
```

Both front-ends converge on the single shared MCell4 engine library `MCell::mcell4`. The
executable additionally carries the MCell3 MDL machinery and NFsim; the Python module
does not run NFsim and never touches MDL.
