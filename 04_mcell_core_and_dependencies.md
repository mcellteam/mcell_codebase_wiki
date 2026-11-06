# MCell Core and Dependencies

> Developer reference for the MCell simulator core: the two engine generations
> (`src/` = MCell3, `src4/` = MCell4), the MCell4 main loop and core objects, and
> how the C++ engine consumes its two external dependencies (libbng, NFsim), plus the in-tree
> geometry analysis that replaced VTK (§6).
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
| `src4/mcell3_world_converter.h/.cpp` | `class MCell3WorldConverter` — reads the MCell3 `volume` struct and builds an equivalent MCell4 `World` object graph (`mcell3_world_converter.h:37`). |
| `src4/mcell4_iface_for_mcell3.h/.cpp` | Thin **C-callable shim** so the legacy C code in `src/mcell.c` can hand off to the C++ MCell4 engine. Holds the single global converter `g_converter` (`mcell4_iface_for_mcell3.cpp:17`). The header only includes `mcell_structs_shared.h` so it is includable from C. |

`mcell4_iface_for_mcell3.h` exposes four functions (`mcell4_iface_for_mcell3.h:17-20`):

```c
bool mcell4_convert_mcell3_volume(struct volume* s);                 // volume -> World
bool mcell4_run_simulation(bool dump_initial_state, bool dump_with_geometry=false);
void mcell4_convert_to_data_model(bool only_for_viz);                // World -> data_model.json
void mcell4_delete_world();
```

### 1.2 The MCell3WorldConverter pipeline

`MCell3WorldConverter::convert(volume* s)` (`mcell3_world_converter.cpp:101`) runs an
ordered pipeline that reads MCell3 structs and creates MCell4 objects:

| Step | Method | MCell4 object created |
|---|---|---|
| 1 | `convert_simulation_setup` (`:195`) | fills `SimulationConfig` (units, RNG seed, partition bounds) |
| 2 | `convert_species` (`:826`) | BNG species via `World::get_all_species()` |
| 3 | `create_initial_surface_region_release_event` | initial surface molecule release |
| 4 | `convert_rxns` (`:1257`) | BNG reaction rules |
| 5 | `add_partition` (`:111-116`) | the single initial `Partition` (`PARTITION_ID_INITIAL`) |
| 6 | `convert_geometry_objects` (`:415`) | `GeometryObject`, `Wall`, `Region` |
| 7 | `convert_release_events` (`:1363`) | `ReleaseEvent`, `ClampReleaseEvent` |
| 8 | `convert_viz_output_events` (`:1551`) | `VizOutputEvent` |
| 9 | `convert_mol_or_rxn_count_events` (`:1761`) | `MolOrRxnCountEvent` |

The converter keeps **ID-mapping tables** (`mcell3_world_converter.h:94-145`) — MCell3
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
  (`add_library(${PROJECT_NAME} STATIC ...)`, `src4/CMakeLists.txt:116-119`, aliased
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
│   ── Geometry analysis (§6; the .cpp files are C++23) ──
├── predicates.{hpp,cpp}           # Shewchuk's exact orientation predicates
├── geometry_analysis.{hpp,cpp}    # mesh algorithms: closedness, volume, containment, OBJ writer
├── mesh_utils.{hpp,cpp}           # meshes of geometry objects; mesh-check warnings
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
| `partitions` | `PartitionVector` | spatial partitions (`world.h:263`) |
| `count_buffers` | `CountBufferVector` | buffered observable output (`world.h:265`) |
| `rng` | `rng_state` | single RNG state for the whole simulation (`world.h:256`) |
| `callbacks` | `API::Callbacks&` | Python callback hooks (`world.h:248`) |

Two run entry points:

- **MDL mode:** `init_and_run_simulation()` (`world.cpp:624`) → `init_simulation()` then
  `run_n_iterations(total_iterations, true)` (`world.cpp:628-634`). Called by the
  executable via `mcell4_run_simulation`.
- **Python mode:** `init_simulation(start_time)` (`world.cpp:242`) is called once, then
  `run_n_iterations(num_iterations, ...)` (`world.cpp:380`) may be called repeatedly,
  finally `end_simulation()` (`world.cpp` ~`:583`).

### 3.2 `Partition` — spatial subdivision

`class Partition` (`partition.h`) is the spatial container. The simulation volume is
divided into one or more **partitions** (an axis-aligned lattice; in MDL mode there is
typically a single initial partition `PARTITION_ID_INITIAL`). Each partition is further
divided into a 3D grid of **subparts** (subpartitions): `num_subparts ==
num_subparts_per_partition_edge^3` (`simulation_config.h:69-71`).

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
- **`SimulationConfig`** (`simulation_config.h:28`) — `class SimulationConfig : public
  BNG::BNGConfig`. It *inherits* libbng's configuration and adds MCell-specific spatial
  parameters: partition/subpart geometry, radial diffusion-step lookup tables
  (`radial_2d_step`/`radial_3d_step`, `simulation_config.h:76-77`), cleanup
  periodicities, memory limit, etc. `init()` (`simulation_config.h:109`) chains to
  `BNGConfig::init()` then builds the subpart edge length and radial step tables.

### 3.5 The simulation main loop

`World::run_n_iterations()` (`world.cpp:380`) is the loop:

```
do {
  check_checkpointing_signal();                       // world.cpp:407
  double time = scheduler.get_next_event_time();      // world.cpp:421
  current_iteration = time_to_iteration(time);        // world.cpp:424
  if (current_iteration >= first + num_iterations) break;
  ... periodic count-buffer flush / progress report ...
  EventExecutionInfo info = scheduler.handle_next_event();  // world.cpp:457  <-- runs one event
  ... report progress ...
  if (terminate && info.type_index == SIMULATION_END_CHECK) break;  // world.cpp:491
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

---

## 4. How MCell4 Uses libbng

MCell4 **delegates all species and reaction storage and chemistry to libbng**. The
integration boundary is clean: `World` owns one `BNG::BNGEngine`, and
`SimulationConfig` *is a* `BNG::BNGConfig`.

| MCell-side construct | libbng type | Where |
|---|---|---|
| `World::bng_engine` | `BNG::BNGEngine` | `world.h:243` |
| `SimulationConfig` | `: public BNG::BNGConfig` | `simulation_config.h:28` |
| `World::get_all_species()` | `BNG::SpeciesContainer&` | `world.h:168` → `bng_engine.get_all_species()` |
| `World::get_all_rxns()` | `BNG::RxnContainer&` | `world.h:171` → `bng_engine.get_all_rxns()` |
| BNGL/compartment data | `bng_engine.get_data()` → `BNG::BNGData` | `partition.cpp:889` |

The call surface MCell4 actually uses:

- **`SpeciesContainer`** — `get_all_species().get(species_id)` returns a `BNG::Species`
  (`world.cpp:706`); `.get_count()` (`world.cpp:263`) and `.dump()` (`world.cpp:657`).
  Used everywhere the engine needs a species' diffusion constant, mass, flags, etc.
- **`RxnContainer`** — `get_all_rxns().get(rxn_rule_id)` returns a `BNG::RxnRule`
  (`world.cpp:688`); `.dump()` (`world.cpp:658`). The reaction-utility `.inl`
  (`rxn_utils.inl`) consults BNG rxn classes during diffusion to decide outcomes.
- **`BNGData`** — `bng_engine.get_data().get_compartment(id)` returns a
  `BNG::Compartment` (`partition.cpp:889,895`), used for compartment hierarchy /
  region containment. `.dump()` (`world.cpp:656`).
- **`BNGConfig`** — `SimulationConfig::init()` calls `BNGConfig::init()`
  (`simulation_config.h:110`); final warnings via
  `bng_engine.get_config().print_final_warnings()` (`world.cpp:610`).
- **BNGL parsing** (`parse_bngl_file`) and BNGL export (`bngl_exporter.cpp`,
  `bng_data_to_datamodel_converter.cpp`) are driven from MCell through the BNG engine
  on the Python/BNGL path.

The MCell3 converter also funnels through BNG: `convert_species`/`convert_rxns`
(`mcell3_world_converter.cpp:826/1257`) create the BNG species and rxn rules from the
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
field references in the converter (`mcell3_world_converter.cpp:1092-1094`), (b) the
nfsimCInterface include path in `src4/CMakeLists.txt:131` (carried for the shared link),
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

## 6. Geometry Analysis: Counted Volumes, Compartments, Mesh Checks, OBJ Export (self-contained reference)

> This section stands alone as MCell's geometry-analysis reference. Until 2026, MCell used VTK for
> this work. The `mcell4_nanobind_novtk` branch replaced VTK with the in-tree code below, which has
> no external dependencies. The reasons, the decisions and how the replacement was checked are in
> doc 07.

### 6.1 Where the code is

Four file pairs in `src4/` hold all of it. The rest of MCell calls them.

| Files | Namespace | Role |
|---|---|---|
| `src4/predicates.{hpp,cpp}` | `MCell::Predicates` | Shewchuk's adaptive exact `orient2d` and `orient3d` (`predicates.hpp:24`, `:30`), ported from his public-domain C implementation. The sign is always exact: zero only for exactly collinear or coplanar points. `exactinit()` runs once, when the library is loaded (`predicates.cpp:377`). |
| `src4/geometry_analysis.{hpp,cpp}` | `MCell::GeometryAnalysis` | The algorithms on plain meshes: points are `std::array<double, 3>` and triangles are index triples (`geometry_analysis.hpp:37-44`). It includes no MCell header. |
| `src4/mesh_utils.{hpp,cpp}` | `MCell::MeshUtils` | MCell's side. It builds the mesh of a `GeometryObject` (`get_mesh`, `get_clean_mesh`, `mesh_utils.hpp:42`, `:47`) and runs the mesh checks, which print parsable warnings (§6.5). |
| `src4/counted_volume_utils.{h,cpp}` | `MCell::CountedVolumeUtils` | Counted volumes, the containment mapping (shared with compartments), the parent search, the BNGL export volume and the OBJ export. Called `vtk_utils` / `VtkUtils` before the removal. |

`predicates.cpp` and `geometry_analysis.cpp` form the static library `MCell::geometry` (§6.6). It
depends on nothing else in MCell, so `data_model_to_pymcell` can link it on its own.

### 6.2 What MCell does with it

1. **Counted-volume hierarchy.** Counting molecules inside objects needs, for each counted object,
   the counted volume a molecule enters and the one it returns to. `World::init_counted_volumes`
   (`src4/world.cpp:146`) calls `CountedVolumeUtils::initialize_counted_volumes`
   (`src4/counted_volume_utils.cpp:320`). That function:
   - checks the cleaned mesh of each counted object and makes a `Solid` of it
     (`prepare_counted_object_solids`, `:50`; errors in §6.5);
   - classifies every pair of objects with `GeometryAnalysis::containment`
     (`compute_containement_mapping`, `:87`);
   - sets each object's `counted_volume_index_inside` and `_outside` (`define_counted_volumes`, `:264`):
     - an object that intersects another counted object gets `COUNTED_VOLUME_INDEX_INTERSECTS` on
       both sides (`:280`, `:295`), so the counted volume of a molecule that crosses one of its
       walls is recomputed from the molecule's position;
     - any other object takes its inside index from its parents. Its outside index comes from its
       direct parent: the parent on the longest containment path (`get_direct_parent_info`, `:163`),
       with the path lengths cached in a `PathLengthCache`.

   If any step fails, the run stops with "Processing of counted volumes failed, terminating."
   (`world.cpp:151`).
2. **Compartments** (BNGL compartments in the Python API). `MCell4Converter` calls
   `set_parent_and_children_compartments` (`libmcell/api/mcell4_converter.cpp:1076`,
   `libmcell/api/compartment_utils.cpp:85`). It checks each compartment's mesh, converted to the
   internal units the simulation will use (`:31`), and finds each compartment's parent with the
   same `compute_containement_mapping` and `get_direct_parent_info`.
3. **Volumes and areas.** One function, `GeometryAnalysis::volume_and_area` (§6.4), serves:
   - `Region::initialize_volume_info_if_needed` (`src4/geometry.cpp:835`, call at `:867`), which
     gives the volume that a release by concentration uses;
   - the BNGL export, through `CountedVolumeUtils::get_geometry_object_volume`
     (`counted_volume_utils.cpp:361`, called at `src4/bngl_exporter.cpp:99` and `:208`). It uses the
     same walls in the same order as the object's region, so the exported volume is the one
     releases use;
   - `data_model_to_pymcell`, on the data model's mesh in µm
     (`utils/data_model_to_pymcell/data_model_geometry.cpp:51-62`).
4. **Mesh checks** (§6.5).
   - `MeshUtils::check_meshes` (`src4/mesh_utils.cpp:283`) runs from `World::init_simulation`,
     right after the counted volumes are set up (`world.cpp:275`). So the Python API and MDL models
     on the MCell4 engine share it.
   - `check_release_regions` (`mesh_utils.cpp:277`) covers a release made after the initialization
     (`libmcell/api/model.cpp:234`).
   - `World::checked_meshes` (`src4/world.h:260`) records what was already reported, so each
     warning is printed once.
5. **Non-manifold check of every Python API object.** `MCell4Converter::convert_geometry_objects`
   raises `ValueError` before it links the walls (`mcell4_converter.cpp:939-941`). MDL models are
   not checked there.
6. **OBJ export.** `World::export_geometry_to_obj` (`world.cpp:722`) calls
   `export_geometry_objects_to_obj` (`counted_volume_utils.cpp:375`), which calls
   `GeometryAnalysis::write_obj`. The output has one named object per geometry object, in µm, each
   with a white material of opacity 0.25.

### 6.3 `GeometryAnalysis`: function → what it does → used by

| Function (`geometry_analysis.hpp`) | What it does | Used by |
|---|---|---|
| `clean_mesh` (`:50`) | Merges exactly equal points (0.0 equals −0.0). Numbers the points in the order the triangles first use them. Drops unused points and triangles that use a point twice. | the counted volumes, the compartments, the checks, `data_model_to_pymcell` |
| `is_closed` (`:54`) | Every edge belongs to exactly two triangles. | the BNGL export volume |
| `is_watertight` (`:58`) | No edge belongs to only one triangle. | the counted volumes, the compartments, check D, `data_model_to_pymcell` |
| `has_non_manifold_edge` (`:62`) | Some edge belongs to three or more triangles. | the counted volumes, the compartments, the API check, `data_model_to_pymcell` |
| `count_inconsistent_edges` (`:67`) | Counts the edges whose two triangles both run along the edge in the same direction. | check A |
| `count_duplicate_points` (`:71`) | Counts the points equal to an earlier point. | check C |
| `find_inside_out_parts` (`:86`) | Finds the connected components that face the wrong way for their nesting (a cavity must face inward). | check B1 |
| `volume_and_area` (`:100`) | Signed volume and area, computed as the simulation computes them (§6.4). | `Region`, the BNGL export, `data_model_to_pymcell`, check B2 |
| `triangles_intersect` (`:103`) | Tells exactly whether two closed triangles share a point. It uses Guigue–Devillers on the predicates (`geometry_analysis.cpp:371`). | `containment` |
| `Solid` (`:125`) | A mesh prepared for containment tests: a bounding-volume hierarchy of its triangles, plus its connected components. | the counted volumes, the compartments |
| `point_in_solid` (`:115`) | Returns `Outside`, `Inside`, `OnSurface` or `Undecided`. An exact on-surface test comes first; then the parity of the crossings along a ray decides. A ray that grazes an edge or a vertex is replaced by another direction (`geometry_analysis.cpp:872`). | `containment` |
| `containment` (`:122`) | Returns `Disjoint`, `Identical`, `Intersect`, `FirstInSecond` or `SecondInFirst` (`geometry_analysis.cpp:1234`; rule in §6.4). | the counted volumes, the compartments |
| `number_to_string` (`:144`) | The shortest text that reads back as the same double (`std::to_chars`), in `vtkNumberToString`'s notation. | the OBJ export |
| `write_obj` (`:154`) | Writes `<prefix>.obj` and `<prefix>.mtl` with the bytes MCell's patched `vtkOBJExporter` wrote, except for the coordinates, which are written as doubles. | the OBJ export |

### 6.4 Definitions that matter

- **Precision.** Computation and output use double everywhere. Every decision (closed, touching,
  inside) takes its signs from the exact predicates. Decisions are therefore exact as long as no
  intermediate product underflows or overflows: coordinate differences between about 1e-80 and
  1e80, or zero (`geometry_analysis.hpp:15-23`).
- **Edges are matched by exact coordinates** after `clean_mesh`; vertex indices play no part. MCell
  links walls the same way: `compatible_edges` accepts the same vertex or exactly equal coordinates
  (`src4/geometry.cpp:51`, `:93-96`). So two walls that use different copies of the same vertex are
  joined there, and check C warns about such copies.
- **Closed** means that every edge belongs to exactly two triangles. That one definition holds for
  counted objects, compartments, the BNGL export and `data_model_to_pymcell`.
  - A *non-manifold edge* (three or more triangles) is reported before an *open* one (one
    triangle).
  - An open surface (a manifold with boundary) is fine in a simulation. Only counting inside it, or
    releasing into it, has no meaning.
- **Volume and area.**
  - The volume is `Region`'s formula: the sum of the signed volumes of the tetrahedra that join
    each triangle to the center of the triangles' bounding box. The triangles are used as given and
    in order, without cleaning or reorientation.
  - The area is the sum of `0.5 |(b − a) × (c − a)|` over the triangles, as in
    `Wall::precompute_wall_constants` (`src4/wall.cpp:304`).
  - An inside-out mesh has a negative volume. VTK reported the absolute value.
- **Containment rule** (the same as the former VTK code):
  - surfaces that touch or cross are `Identical` when their point lists are equal, and `Intersect`
    otherwise;
  - otherwise the first solid is inside the second when any of its connected components is inside.
    One point decides for each component, since the surfaces don't touch;
  - failing that, the same test runs the other way round; if neither holds, the solids are
    `Disjoint`.
- **No floating-point contraction** in `predicates.cpp` and `geometry_analysis.cpp` (§6.6). Fused
  multiply-add breaks the predicates' error-free arithmetic, and it would change the volume's last
  bits from one platform to another. All of MCell is now compiled without contraction (§6.6), so
  exported volumes, and simulations, are the same on ARM64 as on x86-64.

### 6.5 Errors and warnings that users see

**Errors**, which stop the run (or raise an exception in Python):

| Case | Message | Where it can occur |
|---|---|---|
| a counted object (`counted_volume_utils.cpp:60`, `:64`) | "Non-manifold mesh is not allowed, error for <name>.", otherwise "Counting inside an open surface is not possible, error for <name>.", then "Processing of counted volumes failed, terminating." | Python API. MDL on the MCell4 engine too, but there MCell3's check runs first (`src/count_util.c:1421`) and stops open counted objects, so MDL gets only the non-manifold error, for an edge with an even number of walls. |
| two identical counted objects (`counted_volume_utils.cpp:123`) | "Identical counted objects are not supported yet, error for <A> and <B>." (printed as a warning), then the same stop | every front end |
| any Python API object (`mcell4_converter.cpp:940`) | `ValueError`: "Non-manifold mesh is not allowed, error for <name>." | Python API, including data model → Python |
| a BNGL compartment (`compartment_utils.cpp:59`, `:62`, `:108`, `:130`) | `RuntimeError`: the non-manifold message, otherwise "Compartment object must be closed, error for <name>."; also intersecting compartments, and a compartment with several parents | Python API with BNGL compartments |

Other messages:
- The BNGL export returns "Compartment object <name> is not watertight and its volume cannot be
  computed." (`bngl_exporter.cpp:101`, `:210`).
- `data_model_to_pymcell` prints "Warning: could not compute volume of a geometry object: …" with the
  non-manifold or not-watertight message, and writes a fallback value
  (`data_model_geometry.cpp:53`, `:56`; `bngl_generator.cpp:239`).
- A failed OBJ write prints "Could not write geometry to <prefix>.obj and <prefix>.mtl.", and the run
  continues (`counted_volume_utils.cpp:386`).

**Warnings**, after which the run continues, come from `MeshUtils` (`src4/mesh_utils.cpp:14-41`, texts
at `:113-275`).
- Each problem gives one line through `mcell_warn` (stderr): `Warning: [CODE] name: text`.
- Scripts can parse it with `^Warning: \[([A-Z_]+)\] (\S+): (.*)$`.
- The name is the geometry object's name as MCell has it (`Scene.box` for MDL).

| Check | Code | Applies to | Fires when |
|---|---|---|---|
| A | `MESH_INCONSISTENT_ORIENTATION` | counted objects, compartments, objects whose volume a release by concentration uses | two walls run along their shared edge in the same direction |
| B1 | `MESH_INSIDE_OUT` | counted objects, compartments | a closed part faces the wrong way for its nesting. The text gives the signed volume. |
| B2 | `MESH_INSIDE_OUT` | a release by concentration into one object | the object's signed volume is negative. The run then stops with the release's INT_MAX error. |
| C | `MESH_DUPLICATE_VERTICES` | every object | two vertices used by walls have exactly the same coordinates |
| D | `REGION_NOT_CLOSED` | releases of volume molecules into a region | an object of the region has an edge of only one wall |

For MDL models, MCell3 stops open or inconsistently oriented counted objects and release regions
before the conversion (`src/count_util.c:1421`, `src/init.c:3595`, `:3625`), so only B and C can
appear there. Bow-tie and pinched vertices are not checked yet (doc 07).

### 6.6 Build

- **The library** (`src4/CMakeLists.txt:42-120`). `GEOMETRY_SOURCE_FILES`
  (`src4/CMakeLists.txt:44-47`: `predicates.cpp`, `geometry_analysis.cpp`) make up the static
  library `mcell_geometry`, alias `MCell::geometry` (`src4/CMakeLists.txt:110-114`). `mcell4`
  links it publicly (`:120`). `libmcell` (`libmcell/CMakeLists.txt:157`) and `data_model_to_pymcell`
  (`utils/data_model_to_pymcell/CMakeLists.txt:28`) link it privately.
- **Per-file flags**, all in `src4/CMakeLists.txt`:
  - The new `.cpp` files are compiled as C++23 (`MCELL_CXX23`, `src4/CMakeLists.txt:54-58`):
    `-std=gnu++2b`, or `/std:c++latest` under an MSVC-style front end. The rest of MCell is C++17,
    and the new headers stay valid C++17.
  - `predicates.cpp` and `geometry_analysis.cpp` are compiled without floating-point contraction
    (`MCELL_NO_FP_CONTRACTION`, `src4/CMakeLists.txt:67-75`): `-ffp-contract=off`; `/fp:precise`
    for MSVC; `/clang:-ffp-contract=off` for clang-cl, which contracts even with `/fp:precise`.
  - `mesh_utils.cpp` includes glm through MCell's headers, so glm's deprecated-volatile warning is
    switched off for it (`MCELL_CXX23_GLM`, `src4/CMakeLists.txt:81-89`).
  - **All of MCell** is compiled without contraction since `0652d19b`: `mcell_project_options`
    passes `-ffp-contract=off` (`/clang:-ffp-contract=off` for clang-cl) to every first-party and
    bundled target (`mcell/CMakeLists.txt:132-140`). On x86-64 the code doesn't change. On ARM64 it
    makes MCell compute as on x86-64: the macOS full-suite check found 30 test runs that failed
    only there because of contraction (doc 07 §8).
  - Before that, two engine files were switched off on their own, and still are
    (`src4/CMakeLists.txt:91-108`). The macOS check found both (doc 07 §8):
    - `bngl_exporter.cpp`: a compartment's exported volume is its volume times `length_unit^3`
      minus its children's volumes, one multiply-subtract;
    - `diffuse_react_event.cpp`: the tile-neighbor search across a shared edge
      (`grid_utils.inl`) places tiles from the edge's length and compares the positions exactly.
- **Platform floor:** GCC 11+, MSVC 2022, or Xcode 15+ on macOS 13.3+. This comes from
  `std::to_chars` for double, `std::ranges` and `std::bit_cast`. MCell sets no
  `CMAKE_OSX_DEPLOYMENT_TARGET`.
- **No VTK anywhere.** There is no `find_package(VTK)`, no `${VTK_LIBRARIES}` and no
  `vtk_module_autoinit`. `mcell_tools` (branch `mcell4_nanobind`) has no VTK build step (doc 05).

### 6.7 Tests that exercise it

- `mcell_tests` `tests/pymcell4_positive/3100_export_geometry_obj` and
  `3120_dyn_geom_export_geometry_obj` compare the exported OBJ files byte for byte with their
  references.
- `tests/pymcell4_positive/2610_count_in_intersecting_objects_w_parent` counts inside counted
  objects that intersect and have a parent. After every iteration it compares the counts with the
  counts from the molecules' positions. It needs no reference data.
- The BNGL export tests (`test_configs/bngl_to_pymcell4_export.toml`) write each volume into
  `exported.bngl` with 17 digits, then run the exported model again.
- Every test that counts inside objects, releases by concentration, or uses BNGL compartments goes
  through §6.2.

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
4. `mcell_init_simulation` (`:77`), checkpoint reads, `mcell_init_output` (`:85`).
5. **Engine branch** (`:92`):
   ```c
   if (state->use_mcell4 || state->mdl2datamodel4) {
       mcell4_convert_mcell3_volume(state);          // :93  -> build MCell4 World
       if (mdl2datamodel4) mcell4_convert_to_data_model(...);  // :98  export JSON & exit
       else mcell4_run_simulation(...);              // :103 -> run MCell4 engine
       mcell4_delete_world();                        // :106
   } else {
       mcell_run_simulation(state);                  // :109 -> legacy MCell3 engine
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
(the same engine the executable uses, with `MCell::geometry`, §6.6) plus BNG/JsonCpp/Nauty (`:563-569`). The binding details
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
