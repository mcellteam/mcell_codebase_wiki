# TODO — Open Issues in MCell (collected 2026-11-24)

> **Status: open.** The user ended the vertex-index project (doc 08) on 2026-11-24, after step 12b,
> and asked for every issue that is not part of the VTK removal (doc 07) or of the vertex-index work
> itself to be collected here, so that later projects can pick them up. The project's planned steps
> 13-17 moved here with them (§1, §2, §3, §4.1). Each item says whether it was **planned** (agreed with
> the user as later work), **proposed** (asked, never decided) or only **found**.
>
> - **Code:** `mcell` `1bb0f809` (branch `mcell4_nanobind_vertexidx`, merged into `mcell4_nanobind` on
>   2026-11-24), `libbng` `b340b8a1`. Citations are into that code and into MCell3's code, which neither
>   project changed. "Since before the project"
>   means that the build of `0652d19b`, where the vertex-index project started, behaves the same.
> - **Who sees it:** each item names the molecules (volume, surface, or both) and the front end:
>   - Python API models (`MCell4Converter`);
>   - MDL files on MCell3's engine (`mcell file.mdl`), or on MCell4's MDL path (`mcell -mcell4`, also
>     `-mdl2datamodel4`, through `MCell3WorldConverter`). Both paths run MCell3's parser and
>     initialization first;
>   - data models (`data_model_to_pymcell`).
> - **Evidence:** doc 04 §3.7 and §6.12 (known issues, with citations), and the working records in the
>   `mcell4_nanobind_vertexidx` build root, outside any repository: `PLAN_vertex_index.md` (the
>   decisions in the user's words) and `vertex_index_spike/experiments/<folder>/` (models, runs, a
>   README each). The VTK removal's records are in the `mcell4_nanobind_novtk` root.
> - **Not here:** the paused coupled checkpoint/resume work keeps its own list of known code issues,
>   in the handover of the `mcell4_nanobind` root (`NEXT_SESSION_PROMPT.md` §9).

**At a glance:**
- **Wrong results without any message:** §1.2 (Python API only), §1.3, §1.4, §1.5, §1.6, §1.7,
  §3 (shuffle bias), §5.3.
- **Crashes and aborts:** §1.2 and §1.4 (an abort in libbng), §1.7 (an abort in the converter), §3
  (`Model.apply_vertex_moves()` with no moves), §6.1 (MCell3's engine), §7.2 (a parity redo).

---

## 1. Surface classes (found 2026-11-23; planned step 13)

**How it was found:**
- **The models:** 48 small models (`vertex_index_spike/experiments/step13_questions/`, README and
  `results.md` there). Each is an MDL file and the same Python API model, run through MCell3's engine,
  MCell4's MDL path and the Python API.
  - Geometry: one 0.2 um cube (vertices at ±0.1 um), with surface classes on `ALL` and on regions.
  - Molecules: either 1000 volume molecules V released at the center, or 200 surface molecules S'
    on the bottom face.
  - Counts at iterations 10/30/100 of 1 us, seed 1.
- **The baseline:** both MCell4 paths were also run with the build of `0652d19b`. 95 of the 96
  results are identical; the other one is step 6a's fix. So everything in this section predates the
  project.

### 1.1 How it works today (read in the code)

- **MCell3's initialization** checks surface classes for MDL files, on both paths:
  - **Within one class** (`check_for_conflicts_in_surface_class`, `src/init.c:4329`, called at `:568`):
    - two different properties (TRANSPARENT, REFLECTIVE, ABSORPTIVE, CLAMP) for one molecule on
      overlapping sides, that is equal marks or either mark `;`. This covers named molecules and
      `ALL_MOLECULES`, `ALL_VOLUME_MOLECULES` and `ALL_SURFACE_MOLECULES`;
    - one property given through `ALL_MOLECULES` and also through `ALL_VOLUME_MOLECULES` or
      `ALL_SURFACE_MOLECULES` (`:4371-4444`);
    - for volume molecules only, TRANSPARENT, ABSORPTIVE or CLAMP together with a reaction of the
      class (`:4931-5197`).

    Surface molecules are left out of the last check on purpose (the comment at `:4983`). REFLECTIVE
    with a reaction isn't checked at all.
  - **Between the classes of one wall** (`check_for_conflicting_surface_classes`, `src/init.c:5209-6623`):
    two different properties for one molecule on overlapping sides.
    - It is called at `:2420` for each wall with more than one class; a wall lists each class once
      (`:2316-2341`).
    - A property of one class beside a reaction of another class is never checked.
  - MCell3 gives each property a reaction of its own (`split_reaction`, `src/mcell_reactions.c:3255-3300`).
  - **A check before MCell3's initialization could see everything it needs:** MCell3's parser already
    stores each region's walls (`mdl_normalize_elements`, `src/mdlparse_util.c:3925`) and its class
    (`:6202`).
- **MCell4,** for both converters:
  - **Rules:** each property becomes a rule between the molecule and the class, at rate `DBL_GIGANTIC`
    (`libmcell/api/mcell4_converter.cpp:528-588`; on MCell4's MDL path
    `src4/mcell3_world_converter.cpp:1296-1347`).
  - **Reaction classes:** all rules of one molecule with one class go into one reaction class, which
    takes the type of the first rule added. The other rules' types are checked only by an `assert`,
    which Release builds compile out (`libbng/bng/rxn_class.cpp:404-408`).
  - **A wall's classes** are those of its regions, met once per region (`src4/rxn_utils.inl:196-258`). So a
    class that two regions bring to a wall counts twice.
  - **A volume molecule hitting a wall** (`src4/diffuse_react_event.cpp:1025-1085`):
    - any TRANSPARENT class lets it pass. The `assert` that it is the only class (`:1041`) is compiled
      out;
    - otherwise the probabilities of all matching classes are summed (`test_many_intersect`,
      `src4/rxn_utils.inl:655`). A REFLECTIVE or ABSORPTIVE rule at `DBL_GIGANTIC` swamps any reaction.
  - **A surface molecule at a region border:** the first matching class that reflects or absorbs
    decides (`src4/diffusion_utils.inl:597-627`).
  - **A surface molecule's reaction with its wall's class:** one matching class is picked, and the
    reaction time is drawn from that class's rate alone (`src4/diffuse_react_event.cpp:1749-1776`). MCell3's
    engine does the same (`test_many_unimol`, `src/react_trig.c:1148`, `src/react_cond.c:384`;
    `timeof_unimolecular`, `src/react_cond.c:33`).
  - **The Python API** checks each surface property on its own (`libmcell/api/surface_property.h:37-62`),
    never a class or a wall.

### 1.2 What MCell3 refuses runs through the Python API, and the listing order decides

**Who:** Python API models. MDL files with these conflicts stop on both paths with MCell3's error,
exit 1.

- **Volume molecules, two classes on every wall:**
  - with TRANSPARENT in either class, the molecules pass, as with TRANSPARENT alone (`x1`, `x3`, `x5`,
    `x6a`, `x6b`);
  - REFLECTIVE with ABSORPTIVE gives a mix (`x2`: 945/455/17 V left; `x2r` 946/437/7). Pure
    absorption leaves 907/301/1, pure reflection 1000.
  - a CLAMP with REFLECTIVE gives the same mix (`x4`), with no clamp emission (§1.7).
- **Volume molecules, one class:**
  - two properties: the one listed first wins (`w1` reflects, `w1r` absorbs);
  - a property with a reaction: the reaction never happens (`w3`, `w4`);
  - ABSORPTIVE through `ALL_MOLECULES` and `ALL_VOLUME_MOLECULES`: absorbs (`w2`).
- **Surface molecules:**
  - REFLECTIVE and ABSORPTIVE through two regions on the same walls: the region listed last decides
    (`s1a` absorbs, `s1b` reflects);
  - REFLECTIVE and ABSORPTIVE in one class: REFLECTIVE first reflects (`s2a`). ABSORPTIVE first
    aborts Python (`s2b`: exit 134, the `release_assert` at `libbng/bng/rxn_class.cpp:69`).
- **Planned** (4A of 2026-11-17, round 5): MCell4's own checks, for both converters. MCell4's only
  check of this kind is `CLAMP_ON_WALL_TWICE` (step 6b), which stays an error (1A of 2026-11-17,
  round 5).

### 1.3 REFLECTIVE with a reaction of a volume molecule: MCell4 loses the reaction

**Who:** volume molecules, on both MCell4 front ends. MCell3's engine runs these correctly: 358 W at
iteration 100, as without REFLECTIVE (`x9a`).

- **One class** (`x12a`), with no message on either MCell4 path:
  - MCell4's MDL path absorbs every V (907/301/1, 0 W);
  - the Python API reflects every V (0 W).

  The two rules share one reaction class, typed by whichever rule comes first (§1.1).
- **Two classes on the same walls** (`x12b`), and CellBlender's class `reflect`
  (`REFLECTIVE = ALL_MOLECULES;`) beside a reaction class (`x12c`): 0 W on both MCell4 front ends.
  The REFLECTIVE rule at `DBL_GIGANTIC` swamps the reaction in the summed probabilities.
- **Why it matters:** CellBlender puts `reflect` on compartments. A model with `reflect` and a
  reaction class on the same walls loses the reaction on MCell4, with no message. No suite model
  has this (§1.10).
- **Proposed** (2026-11-23, not decided): REFLECTIVE rules take no part in a volume molecule's
  wall-hit draw, since walls reflect anyway; the one-class form is refused.

### 1.4 A border property with a reaction of a surface molecule, in one class: wrong on MCell4

**Who:** surface molecules. MCell3's engine allows this on purpose (`src/init.c:4983`) and runs it:
27 T at iteration 100 with ABSORPTIVE (`s3a`), 119 with REFLECTIVE (`s3b`), 64 with TRANSPARENT
(`s3c`).

- **MCell4's MDL path:** all 200 S are gone by iteration 10, with 0 T.
- **The Python API:** aborts with ABSORPTIVE (exit 134 at `libbng/bng/rxn_class.cpp:69`), and never reacts
  with REFLECTIVE or TRANSPARENT.
- **What works on every path:** the reaction in a second class on the same walls (`s4a`: 25 T
  against 27 on MCell3's engine; `s4b`: 128 against 119; within noise).

### 1.5 A class that reaches a wall through two regions acts twice on MCell4

**Who:** both MCell4 front ends. MCell3's engine lists a class once per wall.

- **Volume molecules:** a reaction class on `ALL` and on a region of every wall reacts at twice the
  rate: 528 W against 358 (`x9b`). Two different reaction classes give 528 on every path (`x11`).
- **Any class:** each hit of a volume molecule draws one more random number per extra region, so
  trajectories change. With CellBlender's `reflect` on `ALL` and also on a region of every wall
  (`x13b`), MCell4 still has the 1000 V of `x13a` but at other positions; MCell3's engine gives
  identical positions.
- **Surface molecules:** no doubling, since one class is picked, but other random draws (`x9s_d`:
  127 T against 135 in `x9s_c`).
- **Clamps:** a clamp class through two regions already stops the run (`CLAMP_ON_WALL_TWICE`).
- **In the suite, 8 models** (§1.10):
  - `nutmeg4_pymcell4/0720`, the `CLAMP_ON_WALL_TWICE` error test;
  - 7 CellBlender data models whose `reflect` is on a compartment's `ALL` and on its membrane
    region:
    - `data_model_bngl/0000`-`0003`, which MCell4's sets skip;
    - `data_model_bngl/0200` and `data_model_pymcell4/1600`, which fail on MCell4 since before the
      project (§8.1);
    - `data_model_pymcell4/1500_fceri_it_100`, which passes. With `reflect` once per wall, its
      frames 1 and 100 change, but only through the random stream (`step13_questions/dm1500/`).

### 1.6 A property of one class with a reaction of another class on the same wall

**Who:** volume molecules. All three paths agree, and MCell3 doesn't check this.

- **ABSORPTIVE in one class, a reaction in another** (`x10`): absorbed, 0 W.
- **TRANSPARENT in one class, a reaction in another** (`x14`): the molecules pass, 0 W.
- **Effect:** the reaction can never happen, and the user gets no message.
- **Proposed** (2026-11-23): an error, for both converters.

### 1.7 Clamps in a Python API class's `properties` list emit nothing

**Who:** volume molecules, Python API models. Data models reach this too: `data_model_to_pymcell`
writes a class with several properties as such a list
(`utils/data_model_to_pymcell/python_generator.cpp:560-581`).

- **Why:** `convert_clamps` reads only the class's own `type` (`libmcell/api/mcell4_converter.cpp:817-848`).
- **A clamp given in `properties`** gets no clamp event, and the wall only absorbs: 907/301/1 V,
  against about 48,000 on both MDL paths (`c0_clamp_only`, `c1_clamp_and_refl_W_props`).
- **A class with a clamp type and a `properties` list** aborts: exit 134 at
  `release_assert(sc->properties.empty() && "TODO")` (`:824`; `c1_clamp_type_and_props`).
- **What works:** the clamp as the class's own type. It gives what MCell4's MDL path gives
  (`c1_clamp_py_type`).

### 1.8 Two reaction classes of one surface molecule on one wall act as one (both engines)

**Who:** surface molecules, on all three paths.

- **The run** (`x11s`): surface molecules that don't diffuse, and two reaction classes on the same walls,
  each with `S' @ sc' -> T'` at 1e4 s^-1. The result is 20/53/127 T on every path. One reaction predicts
  19/52/126; two predict 36/90/173.
- **Why:** both engines pick one class and draw the reaction time from its rate alone (§1.1).
- Volume molecules aren't affected: two reaction classes add up on every path (`x11`).

### 1.9 The marks `S'` and `S,` for surface molecules

- **Today:** for a surface molecule, a rule's mark selects the molecule's own orientation, not the
  side it comes from, on both engines (doc 04 §6.12).
- **Planned:** the user will redesign surface classes on surface molecules (2026-11-17). Until then
  this stays.

### 1.10 The scan of the suite, and the questions left open

- **The scan** (`step13_questions/`, its `tools/`): 666 tests of the sets MCell4 runs, 610 models read.
  - No model has conflicting classes on a wall or in a class, a property with a reaction on one
    wall, or a clamp in a `properties` list.
  - 8 models have a class that reaches a wall through two regions (§1.5).
  - So these remedies would change no suite outcome. The exception is `1500`'s frames in its two
    MCell4 sets.
- **Questions asked on 2026-11-23,** left open when the project ended (each with the recommendation
  made then):
  1. **What MCell4 refuses,** for both converters:
     - MCell3's rules;
     - for volume molecules, a TRANSPARENT, ABSORPTIVE or CLAMP of one class beside a reaction of
       another (§1.6);
     - for surface molecules, a border property and a reaction in one class (§1.4), with advice to
       give the reaction a class of its own.
  2. **REFLECTIVE with a reaction** of a volume molecule (§1.3): fix it, and refuse the one-class form.
  3. **Where MCell4's MDL path checks:** before MCell3's initialization, from the parsed data, through
     the call that step 12b added in `src/mcell.c`. One MCell4 function would serve both converters,
     as `create_clamp_release_events` does; Python API models would be checked at `initialize()`.
  4. **A class through two regions** (§1.5): each class acts once per wall, as on MCell3's engine.
     `1500` gets new frames in its two MCell4 sets.
  5. **Clamps in `properties`** (§1.7): support them, one `ClampSpec` per clamp property.
  6. **Two reaction classes of a surface molecule** (§1.8): a known issue of both engines, for later.
  7. **The lines:** the code `SURFACE_CLASS_CONFLICT`, one line per conflict, then one stop (a
     `ValueError` in Python, exit 1 on MCell4's MDL path). For example:
     - `Error: [SURFACE_CLASS_CONFLICT] Scene.box: 12 walls give the volume molecule V both TRANSPARENT (surface class sc1 through the region ALL) and ABSORPTIVE (surface class sc2 through the region whole)`;
     - the stop: `Error: 2 surface-class conflicts: Scene.box, surface class sc`.
  8. **Tests:** `nutmeg4_pymcell4` and `nutmeg4_mdl` models with one conflict each; `mdl` tests with
     references for both engines for 2 and 4; a `pymcell4_positive` test for 5. Three sub-steps, each
     with a full-suite gate.

---

## 2. Clamps (volume molecules)

### 2.1 An exported data model lists a clamp class once per clamp event (planned step 14)

- **Details:** doc 04 §6.12.
- **Planned** (1A of 2026-11-17, round 6): export the clamps from the model's clamp definitions (the
  `ClampSpec` list), one entry per surface class with all its properties.
- Then no clamp event is needed for a class on no wall. Today such an event exists only so that the
  export keeps the class.

### 2.2 A clamp's volume molecule can leave a watertight box (planned step 16)

- **Details:** doc 04 §3.7. Since before the project.
- **Reproducer:** test `pymcell4_positive/1700` with `cclamp_left` at 1e-4 M, visualization every 1,000
  iterations, and `cclamp_right` renamed `aclamp_right` so that it sorts first. At seed 32, molecule
  527344 reaches (−1.003533, −0.363464, −0.548987); seeds 71 and 104 do the same with the classes as
  named.
- **Planned** (2A of 2026-11-18, round 2): in a scratch build, stop when a molecule is first outside
  the box, and record its start, its first jump and the walls on its path; then fix it.

### 2.3 Barriers lower a clamp's concentration (planned, part of step 15)

- **Details:** doc 04 §3.7, "Barriers change trajectories".
- **The effect:** with a barrier of any kind every iteration, a clamp molecule's first jump is cut
  to half a step. Test 1700's bins then come out 2.5-5% below the clamps' target; with barriers every
  1,000 iterations they are within 2%.
- **Planned** (3A of 2026-11-18, round 2): find which timing the clamp's emission rate assumes, and
  compare with MCell3's engine, before changing anything.

---

## 3. The order within an iteration, and the shuffles (planned step 15)

**Details:** doc 04 §3.7, "The order within an iteration". These concern volume and surface molecules
on MCell4. The decisions are 1A and 2A of 2026-11-18, round 1.

- **New volume molecules** (from clamps, releases and reaction products) diffuse after the existing
  ones, in the order they were made. So the clamp that runs first gets the first chance at shared
  partners in every time step. **Planned:** look at it as a whole before choosing a remedy.
- **Clamps run in a fixed order:** by surface class name, then molecule, then side.
  - **Planned:** a seeded shuffle at each iteration, under five conditions that keep it
    deterministic (the plan, step 15):
    1. draw from `world->rng`, which checkpoints save;
    2. MCell's own loop, not `std::shuffle`;
    3. each swap partner drawn from 0..i;
    4. the sorted order as the start;
    5. no draw with fewer than two clamps.
  - **Form:** one event that holds every clamp, shuffles them and runs them.
- **The molecule-order shuffle is biased:** it picks each swap partner from 0..n−2 instead of 0..i. So
  two molecules are always swapped, and the last molecule never stays last.
- **`randomize_vector` has the same loop,** and reads far out of bounds for an empty list.
  - So `Model.apply_vertex_moves()` with no moves added, and the default `randomize_order=True`,
    ends Python with a segmentation fault.
  - **Planned:** both loops pick from 0..i, and `randomize_vector` returns on fewer than two items,
    with a Python API test.
- **Effect of the plan:** every MCell4 model with two or more acting clamps, 10,000 or more
  iterations, or randomized vertex moves gets new results. Report them before replacing reference
  data.

---

## 4. Vertex moves (Python API models)

### 4.1 The wall–wall hit reported for a vertex move depends on the vertex numbering (planned step 17)

**Details:** doc 04 §6.12; the Decisions of 2026-11-18 on step 7, with the pyramid runs in
`vertex_index_spike/experiments/step7_questions/`.

- **The check** visits a moved vertex's 1-ring in ascending index order and stops at the first edge
  that crosses another object's wall or needs a redo (`src4/partition.cpp:644`).
  - The order decides which hit is reported, whether a redo cancels the move without one, and the
    draws from `aux_rng`.
  - Whether the move is cancelled doesn't depend on the order.
- **The ascending order comes from a sort** (`src4/partition.cpp:592`). It was kept in step 7 only to
  preserve the order of the code before the MeshTable. The user, 2026-11-18: "later we do not want
  to sort anything and test will eventually be changed to be insensitive to the ordering."
- **A cancelled move:** the vertices checked after it see the vertex at its old position
  (`partition.cpp:575-577`, `:621`).
- **Planned** (6A of 2026-11-18, on step 7):
  - every edge is tested, and every crossing is reported;
  - nothing is sorted;
  - two things are settled: which of the edge's two walls is reported, and how a cancelled move is
    seen by the vertices checked after it.

  The expected hits change once: ask before replacing reference data.

### 4.2 Found, to check whether intended

- **A moving face pushes every volume molecule in its way** to its new position, whatever the surface
  class, TRANSPARENT included (`collect_volume_molecules_moved_due_to_moving_wall`,
  `src4/dyn_vertex_utils.inl:191`).
  - Found on 2026-11-20 while writing `pymcell4_positive/1580`: 197 molecules ended up at the new face,
    and none were left in the added part.
- **A release at time k·dt runs in iteration k+1,** after a vertex move applied after iteration k
  (`vertex_index_spike/experiments/step11_questions/`, `release_timing/`).

---

## 5. MDL files on MCell4's MDL path (`MCell3WorldConverter`)

### 5.1 MCell4's MDL path without MCell3's initialization (planned work)

- **Details:** doc 04 §6.12, "Planned work".
- **The idea:** for `-mcell4`, the parsed structures would become an `API::Model`, initialized as a
  Python API model is. That would retire `mcell3_world_converter.cpp`.
- **The cost:** everything MCell3's initialization works out would have to come from parse-time
  data.
- **Memory** (the user, 2026-11-20, 1A: decide later): MCell4's MDL path keeps MCell3's model for the
  whole run.
  - That is about 4.4 GB of the full neuropil's 10.4 GB peak.
  - MCell4's `Wall` is 520 bytes; MCell3's `struct wall` is 248.
  - The options asked then: free MCell3's model after the conversion (in `src/mcell.c`), or measure
    the neuropil through a data model and the Python API.

### 5.2 Some values still come from MCell3's initialization

- **What:** the partition's placement and the random-number state (doc 04 §6.12).
- These go with §5.1.

### 5.3 A region other than `ALL` in a release of volume molecules becomes the whole object (7C of 2026-11-19)

- **Details:** doc 04 §6.12, with the run `two_shells_*` (`step8_questions/`).
- **The effect:** a release into one watertight shell of a two-shell object places volume molecules
  in both shells.
- **Counts:** a count of volume molecules inside such a region stops with a developer message.
- **The Python API** refuses both.
- **Possible remedies:** a clear error in the converter, or support for regions made of whole
  watertight shells on both MCell4 front ends.

### 5.4 A spherical release by a negative concentration (volume molecules)

- **Details:** doc 04 §6.12; the user, 2026-11-19: "we may change today's behavior".
- **Today:** MCell3's engine releases nothing. MCell4's MDL path stops with the misleading "tries to
  release more than INT_MAX" message. The Python API refuses such a site when it is made.
- **Possible remedies:** the Python API's clear error on MCell4's MDL path too, or removal inside the
  sphere on every path.

### 5.5 Dynamic geometry from MDL files and data models

- **Since step 8 it is refused** (volume and surface molecules):
  - an MDL file with `DYNAMIC_GEOMETRY` stops on MCell4's MDL path, with a message that points to
    MCell3's engine and to `Model.add_vertex_move`;
  - `data_model_to_pymcell` stops on an object with `"dynamic": true`.

  Before step 8, both ran with fixed geometry, with no message.
- **Possible later work:** support both through MCell4's vertex moves.

### 5.6 Visualization output: one block with a constant period from iteration 0

- **MCell4's MDL path** accepts one `VIZ_OUTPUT` block whose iterations start at 0 with a constant
  period (`src4/mcell3_world_converter.cpp:1702-1726`).
  - A list such as `[0, 1, 100]` stops with a developer message (`Conversion error: Expected ... is
    false.`).
  - MCell3's engine accepts any list.
- **The Python API** has only `every_n_timesteps` (`libmcell/api/mcell4_converter.cpp:1775`).

---

## 6. MCell3's engine (`mcell file.mdl`)

MCell3's code stays unchanged by rule, so each item here needs that rule lifted first.

### 6.1 A vertex index equal to the vertex count, or negative, crashes MCell3's engine

- **Details:** doc 04 §6.12; `vertex_index_spike/experiments/index_range/` and `step11_questions/idx/`.
- **Why:** the range test compares with `>` instead of `>=` and doesn't reject a negative index
  (`src/init.c:2092-2095`).
- **MCell4's MDL path** has stopped such a file since step 12b, before MCell3's initialization
  (`MESH_INVALID_INDICES`).

### 6.2 MCell3's engine links walls by coordinates

- **How:** `surface_net` keys each edge by its end points' coordinates (`src/wall_util.c:287-293`).
  `edge_hash` hashes the raw bytes of the coordinates (`src/edge_util.c:69-73`), but `edge_equals`
  compares them within `EPS_C` (`src/edge_util.c:34-51`).
- **So the hash and the equality disagree:**
  - copies closer than the tolerance are compared only when they land in the same bucket;
  - a cube whose walls share no vertex index links, but with vertex copies 1e-16 apart it fails
    with "BARE EDGE".
- **Pairs:** walls on one edge are linked in pairs (`src/wall_util.c:302-346`), so an even number of
  consistently oriented walls on an edge passes `is_manifold`.
- **Source:** the VTK removal's plan (`mcell4_nanobind_novtk`, "Open items",
  `vtk_removal_spike/experiments/mcell3_edges/`).
- MCell4 no longer uses any of this (doc 08).

### 6.3 MCell3's transparency setup assumes that `ALL` is an object's first region

- **The assumption:** `src/dyngeom.c:1598-1620` and `:1729-1758`; the comment at `:1751-1752` says
  "Could be unsafe".
- **It doesn't always hold:** every instantiation copies an object's regions in reverse
  (`mdl_copy_object_regions`, `src/mdlparse_util.c:3133-3150`). So `ALL` comes first only after an
  odd number of copies (`vertex_index_spike/experiments/region_order/`).
- **The effect:** MCell3 then uses the first region's volume to decide which way a transparent class
  lets volume molecules through (`insert_volume_molecule_encl_mesh`, `src/dyngeom.c:565-620`). That is
  wrong for such objects, but it matters only in MCell3's dynamic geometry.
- MCell4 finds `ALL` by name and doesn't depend on the order of regions.

---

## 7. Geometry, apart from connectivity

- **7.1 Walls that cross at an angle aren't detected** (13A of 2026-11-11). `check_for_overlapped_walls`
  finds only overlaps in one plane (doc 04 §6.12). **Planned:** a check for walls that cross.
- **7.2 A parity redo aborts the run:** if the parity ray grazes an edge closely enough to need a redo,
  a `release_assert` ends the run (doc 04 §6.12).
- **7.3 A very short shared edge** in a wall with nonzero area stops the run with "Detected malformed
  geometry object" (doc 04 §6.12).
- **7.4 Containment cycles are possible in principle:** with objects of several components, the
  "any component inside" rule could make A contain B, B contain C and C contain A, and the parent
  search would recurse without end. Not tried (doc 07 §9).
- **7.5 The order of a compartment's children** in its release region follows heap addresses, so
  data-model outputs can differ between runs (doc 04 §6.12; seen in `pymcell4/4200`, `4220` and
  `pymcell4_positive/4210`).
- **7.6 Parallel mesh checks,** if ever needed (planned work, doc 04 §6.12): split by object, and keep
  sums in a fixed order.

---

## 8. Tests, test models and tools

### 8.1 376 test runs fail since before both projects

- **What:** 87 test sources, in 18 MCell4 sets (Appendix A). All of them are `FAILED_DIFF`: the run
  ends, and an output differs from its reference.
- **Since when:** the same 376 failed at `0652d19b`; the `mcell4_nanobind` root's baseline of
  2026-10-21 had 379.
- **Not investigated.** Both projects therefore compared output files as well as statuses at every
  gate.

### 8.2 `bngl_export.toml` has never been run in these projects

- **What:** its 10 public tests run BioNetGen on the exported BNGL and compare the results exactly.
- **Why not run:** the build roots have no BioNetGen binaries.
- **Risk:** the VTK removal changed exported volumes in their 7th to 16th digit, so the references
  may need updating (doc 07 §9).

### 8.3 Test models

- **`mdl/2810_wall_overlap_2boxes_inside_one`** (todo): its note blames MCell3's initialization, but its
  object `Outer` has walls 10-18 reversed (doc 04 §6.12).
- **The full neuropil** (private `mdl/3650`, skip; benchmark B4110; `models/salk/neuropil_recon`):
  - two objects stop it on MCell4's MDL path: `d010p_sps373` (`MESH_NOT_MANIFOLD`, vertex 123) and
    `d022` (`MESH_ORPHANED_VERTICES`, vertex 17566);
  - `d000_er` has 12 shells that face inward (doc 04 §6.12);
  - a copy repaired in both places gives the outputs of `0652d19b` byte for byte
    (`vertex_index_spike/experiments/step11_neuropil/`).

### 8.4 The test runner and the suite

- **Two tests pass without any comparison:** `pymcell4/2150_default_viz_and_count_outputs_seed_2` and
  private `pymcell4/1600_presyn_molecule_escape`.
  - They keep their references under `seed_00002` and `seed_00624`.
  - The tester looks in the folder of the seed it ran (`mcell_tests/scripts/tester_base.py:227`), and
    requires no reference.
- **Known run-to-run differences:**
  - `0000_seeds_w_fail` (`log.tmp`);
  - `nutmeg_negative/0074` (`XXX`);
  - `nutmeg4_pymcell4/2760`-`2785` (checkpoint timing);
  - the child order of §7.5.
- **Time:**
  - one MCell3 run, `mdl/4202_lotka_voltera_reaction_limited_it_400_long` (315 s), sets the suite's wall
    time;
  - test 2500 writes 50,000 visualization files, which timed out on NFS;
  - 32 test sources (59 runs, 0.78 GB) write visualization output that no reference compares, 8 of
    them todo tests (`vertex_index_runs/viz_survey/`).
- **The wiki's `wiki_check.py`** indexes the whole build root and, on NFS with full-suite work folders
  in it, doesn't finish in minutes. Its audit reports 4 missing files that predate both projects:
  doc 06's pybind11 paths, and the design note's script.

---

## 9. Platforms (from the VTK removal's macOS and Windows checks, doc 07 §6)

Every difference listed here predates the VTK removal.

- **macOS:** one CellBlender example's converter script computes a vertex with Python's `math.sin`,
  which differs in the last digit.
- **Windows:**
  - OBJ files are written with CRLF line endings, against LF references;
  - one data-model path is longer than Windows' 260 characters;
  - MCell3's log buffering splices merged stdout and stderr;
  - some tests need `bash`.

---

## 10. Differences kept on purpose

These were decided; they aren't to be "fixed" without a new decision.

- **Region borders differ between the engines** (step 6). MCell4 gives `ALL`, and a region of every
  wall, the boundary edges of a non-watertight mesh; MCell3's engine gives them none. Example:
  `mdl/2775` with surface molecules (doc 04 §6.12).
- **Barriers change trajectories,** on both engines, and the engines have different barriers (doc 04
  §3.7). Turning visualization off is no remedy: it removes barriers.
- **`CLAMP_ON_WALL_TWICE` is an error** (1A of 2026-11-17, round 5).
- **Clamps run in a fixed order:** by surface class name, then molecule, then side (1A of 2026-11-18),
  until §3's shuffle.

---

## 11. Follow-ups of the two projects

- **The VTK removal and the vertex-index work into `mcell4_dev`** (pybind11), possibly (the user,
  2026-11-05). Today both live only on the `mcell4_nanobind` line.

---

## Appendix A. The 376 failing runs (gate `step12b2`, `mcell` `1bb0f809`)

**All are `FAILED_DIFF`, all in MCell4 sets:**
- `bngl_pymcell4`, `bngl_pymcell4_export`: 39 each;
- `bngl_data_model_pymcell4`, `bngl_data_model_pymcell4_w_bngl`: 38 each;
- `bngl_data_model_pymcell4_chkpt`, `bngl_data_model_pymcell4_w_bngl_chkpt`: 30 each;
- `mdl`, `mdl_data_model_pymcell4`, `mdl_data_model_pymcell4_w_bngl`, `data_model_converter`: 25 each;
- `pymcell4`: 17;
- `mdl_data_model_pymcell4_chkpt`, `mdl_data_model_pymcell4_w_bngl_chkpt`: 13 each;
- `data_model_pymcell4`, `data_model_pymcell4_w_bngl`: 6 each;
- `data_model`: 3;
- `data_model_pymcell4_chkpt`, `data_model_pymcell4_w_bngl_chkpt`: 2 each.

Each set's name ends in `_mcell4`. The full list is in `vertex_index_runs/step12b2.log`.

**The 87 test sources:**
- **`bngl/` (32):**
  - `0010`, `0011`, `0012`, `0014`, `0015`;
  - `0100`, `0200`, `0300`, `0320`, `0321`, `0600`;
  - `1010`, `1011`, `1020`, `1021`, `1022`, `1030`, `1040`, `1050`, `1060`, `1070`, `1075`,
    `1090`, `1110`, `1120`;
  - `1300`, `1310`, `1311`, `1320`;
  - `2050`, `2060`, `2500`.
- **`mdl/` (22):**
  - `0320`, `0321`, `0322`, `0323`, `0350`, `0435`;
  - `0500`, `0501`, `0505`;
  - `1450`, `2042`, `2500`, `2540`;
  - `2620_conc_clamp_react_it_50`, `2670_flux_clamp_react_it_50`;
  - `3012`, `4005`;
  - `4200`, `4201`, `4202`, `4210`, `4211`.
- **`pymcell4/` (13):**
  - `1250`, `1720`, `2000`, `2100`;
  - `2410`, `2420`;
  - `2500`, `2505`, `2507`;
  - `2700`, `2701`;
  - `4200`, `4220`.
- **`pymcell4_positive/`:** `1802`, `4210`.
- **`tutorial/`:** `0060`, `0070`.
- **`data_model/`:** `0200`, `0210`, `2810`.
- **`data_model_bngl/`:** `0200`.
- **`data_model_pymcell4/`:** `1600`.
- **`mcell_tests_private`:**
  - `bngl/`: `3000`, `3001`, `3005`, `3010`, `3210`, `3215`, `3220`;
  - `data_model_bngl/`: `3055`;
  - `mdl/`: `1500`, `1510`, `3050`.
