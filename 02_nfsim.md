# NFsim (Network-Free Simulator)

> Developer reference for the bundled **nfsim** dependency of MCell.
> Source root: `$ROOT/nfsim` (`$ROOT` = the build root, the parent of this wiki folder; see `CLAUDE.md`)

---

## 1. Purpose & Scope

**NFsim** is the **N**etwork-**F**ree **sim**ulator: a general-purpose stochastic
simulator for *rule-based* models of biochemical reaction kinetics. It tracks
individual molecule instances and the complexes (bonded aggregates) they form,
and fires reaction *rules* on them using a Gillespie-style stochastic algorithm.

### What "network-free" means

Traditional rule-based modeling (e.g. BioNetGen's `bionetgen` network generation)
**enumerates** every possible chemical species and reaction implied by a set of
rules into a full *reaction network*, then integrates that network with ODEs or
the Gillespie SSA. For combinatorially complex systems (multi-site proteins,
polymers, scaffolds) this network is astronomically large or infinite, so it
cannot be generated.

NFsim avoids ever building the network. Instead it:

- Represents each molecule as an explicit object (`Molecule`) with component
  states and bonds.
- Represents each rule as a `ReactionClass` whose left-hand side is a set of
  **patterns** (`TemplateMolecule`, "regular expressions for molecules").
- Maintains, for every rule and reactant position, a live list of the molecules
  currently matching that pattern (`ReactantList` of `MappingSet`s).
- At each step, picks a rule with probability proportional to its propensity
  (number of matching reactant tuples x rate), selects matching reactant
  instances at random, and applies the rule's transformations
  (`TransformationSet`) directly to those molecule objects.

This makes the simulation cost scale with the number of molecules and rules,
**not** with the size of the (never-generated) reaction network.

| Approach | What is enumerated | Engine |
|---|---|---|
| ODE / network-based | All species + reactions, up front | Deterministic ODE integration |
| SSA / network-based | All species + reactions, up front | Gillespie over fixed network |
| **Network-free (NFsim)** | Nothing — patterns matched on the fly | Gillespie over rules + particles |

### Origin

NFsim is the upstream open-source project by **Michael W. Sneddon, Justin S.
Hogg, James R. Faeder, and Thierry Emonet** (Yale University / University of
Pittsburgh, NSF-funded). This tree is **v1.11**
(`nfsim/README.txt:18`, `nfsim/NFsim_manual_v1.11.pdf`). It is part of the
**BioNetGen / RuleBender** ecosystem: models are written in BNGL, compiled by
BioNetGen to **BNG-XML**, and that XML is read by NFsim's `NFinput` module.
Inside this MCell build it is a **bundled, unmodified vendored dependency**
(the CMake file deliberately suppresses NFsim's warnings rather than fixing the
code — see `nfsim/CMakeLists.txt:40-48`).

---

## 2. Directory & Module Layout

```
nfsim/
├── CMakeLists.txt              # build: NFsim_static / _fpic / NFsim (shared) / NFsim_EXE
├── README.txt                  # upstream v1.11 readme + authors
├── NFsim_manual_v1.11.pdf      # upstream manual
├── include/                    # all public headers (PUBLIC include dirs)
│   ├── NFcore/                 # the simulation core
│   │   ├── NFcore.hh           # *** master header: System, Molecule(Type), Complex, ReactionClass, Observable
│   │   ├── observable.hh, templateMolecule.hh
│   │   ├── moleculeLists/      # moleculeList.hh
│   │   └── reactionSelector/   # reactionSelector.hh (DirectSelector)
│   ├── NFreactions/            # rule firing machinery
│   │   ├── mappings/           # mapping.hh, mappingSet.hh, mappingGenerator.hh
│   │   ├── reactions/          # reaction.hh (BasicRxnClass, FunctionalRxnClass, MMRxnClass, DOR...)
│   │   ├── transformations/    # transformation.hh, transformationSet.hh, molecule/speciesCreator.hh
│   │   └── reactantLists/      # reactantContainer.hh, reactantList.hh, reactantTree.hh
│   ├── NFinput/                # BNG-XML reader; TinyXML/ vendored XML parser
│   ├── NFfunction/             # global/local/composite rate functions; muParser/
│   ├── NFoutput/               # observable output (.gdat)
│   ├── NFscheduler/            # Scheduler, NFstream
│   ├── NFproperty/             # diffusion classes (Einstein-Stokes, Saffman-Delbruck)
│   ├── NFutil/                 # RNG (MTrand/), string ops, conversions
│   └── nauty24/                # vendored Nauty (graph canonical labeling)
├── src/                        # mirrors include/ one-to-one
│   ├── NFapi.cpp / NFapiAux.cpp / NFaux.cpp   # *** C++ API used by the C wrapper
│   ├── NFsim.cpp               # standalone-executable main()
│   ├── NFcore/  NFreactions/  NFinput/  NF.../  nauty24/
└── test/  tools/  validate/  models/  bin/  doc/
```

### Core subsystems

| Subsystem | Dir | Responsibility |
|---|---|---|
| **NFcore** | `src/NFcore` | `System`, `MoleculeType`/`Molecule`, `ReactionClass`, `Complex`/`ComplexList`, `Observable`, `TemplateMolecule` |
| **NFreactions** | `src/NFreactions` | Pattern→molecule mapping (`MappingSet`), rule application (`TransformationSet`), reactant bookkeeping (`ReactantList`/`ReactantTree`) |
| **NFinput** | `src/NFinput` | BNG-XML parsing (TinyXML + **Boost** string/range), builds the `System` |
| **reactionSelector** | `src/NFcore/reactionSelector` | Picks next reaction class from propensities (`DirectSelector`) |
| **NFfunction** | `src/NFfunction` | Rate-law functions (muParser), local/global/composite functions, DOR support |
| **NFoutput** | `src/NFoutput` | `.gdat`/observable count output |
| **NFutil** | `src/NFutil` | Mersenne-Twister RNG, string/number conversion |
| **nauty24** | `src/nauty24` | Canonical graph labeling of complexes (species identity for queries) |
| **NFapi** | `src/NFapi.cpp` | Thin C++ facade over `System` consumed by `nfsimCInterface` |

---

## 3. Key Classes & Headers

Nearly all core classes live in **namespace `NFcore`**, declared in the single
master header `nfsim/include/NFcore/NFcore.hh` (~1478 lines; impl split across
`src/NFcore/*.cpp`).

### `System` — `NFcore.hh:291`, impl `src/NFcore/system.cpp`
The top-level simulation object. Owns:
- `vector<MoleculeType*> allMoleculeTypes` (`NFcore.hh:530`)
- `vector<ReactionClass*> allReactions` (`NFcore.hh:531`)
- `ComplexList allComplexes` (`NFcore.hh:537`)
- `CompartmentList allCompartments` (`NFcore.hh:539`)
- observables, functions, parameters, the `ReactionSelector* selector` (`:594`)
- running state: `a_tot` (total propensity), `current_time`, `nextReaction` (`:559-561`)

Lifecycle: add MoleculeTypes/reactions → `prepareForSimulation()` (`:380`) → call
`sim()` / `stepTo()` / `singleStep()`. Inherits `HierarchicalNode` (`:147`) which
gives every BNG object a property bag.

### `MoleculeType` — `NFcore.hh:629`, impl `src/NFcore/moleculeType.cpp`
The *type* of a molecule (e.g. `L`, `R`). Knows its component names, default and
possible component states, equivalency/symmetric-site classes, which reactions
the type can participate in, and which observables it affects. Holds **all
`Molecule` instances** of that type and can populate the system with defaults.

### `Molecule` — `NFcore.hh:~897`, impl `src/NFcore/molecule.cpp`
An individual molecule instance. Stores component state array (`int *component`),
bond array (`Molecule **bond`, `int *indexOfBond`), its `ID_complex`,
`ID_unique`, and `rxnListMappingId2` (`NFcore.hh:1118`) — the set of reaction
slots this molecule currently matches. Key methods:
- `static bind / unbind` (`:991-994`), `setBondTo` (`:986`)
- `setComponentState` (`:933`)
- `traverseBondedNeighborhood` / `breadthFirstSearch` / `depthFirstSearch`
  (`:999-1001`) — walk a complex
- `updateRxnMembership()` (`:1009`) — re-test which rules this molecule matches
  after a change; `addToObservables`/`removeFromObservables` (`:1010-1011`)
- `getComplex()` (`:917`) resolves `ID_complex` via the System's `ComplexList`

### `Complex` — `NFcore.hh:1330`, impl `src/NFcore/complex.cpp`
A connected set of bonded molecules (`list<Molecule*> complexMembers`, `:1382`).
Membership changes dynamically as bonds form/break (`updateComplexMembership`,
`refactorToNewComplex`, `mergeWithList`, `:1346-1352`). Produces a **canonical
label** via Nauty (`getCanonicalLabel`, `:1373`; `generateCanonicalLabel`,
`:1395`) — this string is the species identity used by the query API.
`ComplexList` (`NFcore.hh:227`) is the container, with a free-list
(`nextAvailableComplex`) for recycling complex IDs.

### `ReactionClass` — `NFcore.hh:1168`, impl `src/NFcore/reactionClass.cpp`
Abstract base for a **reaction rule**. Holds the rate law, propensity `a`, and a
`TransformationSet`. Subtypes (declared in
`include/NFreactions/reactions/reaction.hh`):

| Subtype | Header | Purpose |
|---|---|---|
| `BasicRxnClass` | `reaction.hh:15` | Standard rule; flat `ReactantList` per reactant |
| `FunctionalRxnClass` | `reaction.hh:45` | Rate from a global/composite function |
| `MMRxnClass` | `reaction.hh:61` | Michaelis-Menten (kcat/Km) |
| `DORRxnClass` | `reaction.hh:77` | "Distribution of rates" — rate depends on local context; uses a weighted `ReactantTree` |
| `DOR2RxnClass` | `reaction.hh:148` | Two-reactant DOR |

Rule-type constants `BASIC_RXN`, `DOR_RXN`, … at `NFcore.hh:1177-1181`.
Core virtuals: `init()`, `prepareForSimulation()`, `tryToAdd()`, `remove()`,
`update_a()`, `fire(random)` (`NFcore.hh:1201,1212-1218`).

### `TemplateMolecule` — `include/NFcore/templateMolecule.hh:35`, impl `templateMolecule.cpp`
"Regular-expression-like" pattern object that defines a rule's reactant
(`templateMolecule.hh:17-23`). Specifies required component states and bonds; an
individual `Molecule` either matches it or not via `compare()`. Holds
`MapGenerator`s that, on a successful match, record which molecule fills each
slot of the resulting `MappingSet`.

### `Observable` — `include/NFcore/observable.hh:31`, impl `observable.cpp`
Counts things of interest. `add()`/`subtract()` adjust a running `count`
(`observable.hh:80-90`); types `MOLECULES = 1` / `SPECIES = 2`
(`observable.hh:118-119`) with subclasses `MoleculesObservable` (`:148`) and
`SpeciesObservable` (`:171`). These are what get written to `.gdat` output and
returned by the query API.

### `ReactionSelector` — `include/NFcore/reactionSelector/reactionSelector.hh:27`
Abstract strategy for choosing the next firing rule given propensities.
`DirectSelector` (`reactionSelector.hh:45`) implements the direct method:
`getNextReactionClass(rc)` returns the chosen `ReactionClass` and the random
remainder used to pick the specific reactant tuple (`refactorPropensities`,
`update`, `getAtot`, `:35-40`).

### Class relationships (text diagram)

```
System
 ├─ allMoleculeTypes : MoleculeType*   ──owns──>  Molecule*  (instances)
 │                         │                          │ bonds
 │                         │ knows rules of            ▼
 │                         └────────────►  Complex (bonded aggregate, Nauty-labeled)
 ├─ allReactions    : ReactionClass*
 │      ├─ TemplateMolecule (LHS patterns) ──match──► Molecule
 │      ├─ ReactantList / ReactantTree  of  MappingSet  (live matches)
 │      └─ TransformationSet  ── applies ──► state changes / bind / unbind / add / delete
 ├─ allComplexes    : ComplexList
 ├─ obsToOutput     : Observable*
 └─ selector        : ReactionSelector (DirectSelector)
```

---

## 4. NFreactions: How a rule maps and fires

(See `include/NFreactions/...`; details below cite the headers.)

1. **Patterns → set-up.** A rule's reactants are `TemplateMolecule`s. From them a
   `TransformationSet` (`transformations/transformationSet.hh:41`) is built and
   the rule's edits registered: `addStateChangeTransform`, `addBindingTransform`,
   `addUnbindingTransform`, `addDeleteMolecule`, `addAddMolecule`, … (`:77-168`).
   `finalize()` (`:225`) freezes it and attaches a `MapGenerator`
   (`mappings/mappingGenerator.hh:28`) to each template component so the template
   knows which slot of a future `MappingSet` it fills.

2. **Mapping (`mappings/`).** A `Mapping` (`mapping.hh:25`) = (transform type,
   component index, one `Molecule*`). A `MappingSet` (`mappingSet.hh:25`) is the
   array of `Mapping`s for one matched reactant — i.e. one concrete reactant
   instance. When `TemplateMolecule::compare()` succeeds, `MapGenerator::map()`
   calls `MappingSet::set(index, molecule)` to point each slot at the matched
   molecule.

3. **Reactant bookkeeping (`reactantLists/`).** Each rule keeps, per reactant
   position, a `ReactantList` (`reactantList.hh:34`) of currently-matching
   `MappingSet`s (or a rate-weighted `ReactantTree`, `reactantTree.hh:31`, for
   DOR rules). Both implement the `ReactantContainer` interface
   (`reactantContainer.hh:22`): O(1) `pushNextAvailableMappingSet`,
   `removeMappingSet`, and `pickRandom`. The rule's propensity is computed from
   these list sizes x rate.

4. **Firing (`transformations/`).** When a rule fires, the `ReactionClass`
   gathers one `MappingSet` per reactant and calls
   `TransformationSet::transform(MappingSet**)` (`transformationSet.hh:176`).
   That applies any *add* transforms first, then loops over each reactant's
   `Transformation`s, calling `apply()` per the type constants in
   `transformation.hh:130-177`:
   `STATE_CHANGE`, `BINDING`, `UNBINDING`, `REMOVE`, `ADD`, `INCREMENT/DECREMENT_STATE`,
   `INCREMENT/DECREMENT_POPULATION`, `CHANGE_COMPARTMENT`, `LOCAL_FUNCTION_REFERENCE`.
   E.g. `BindingTransform::apply` (`src/.../transformation.cpp:100`) calls
   `Molecule::bind`; `RemoveMoleculeTransform::apply` marks molecules for deletion.
   Affected molecules are then collected and their rule-membership and
   observables are updated.

---

## 5. The Simulation Loop (Gillespie / Direct Method)

The main loop is `System::sim(duration, sampleTimes, verbose)`
(`src/NFcore/system.cpp:668`). It is a textbook Gillespie **direct method**:

```
recompute_A_tot();                              // sum of all rule propensities  (system.cpp:692)
while (current_time < end_time) {
    if (a_tot > ATOT_TOLERANCE)
        delta_t = -log(RANDOM_OPEN()) / a_tot;   // exponential waiting time      (system.cpp:710)
    else break;                                  // nothing can react
    ... output observables up to each sample time ...
    randElement = getNextRxn();                  // pick the rule (DirectSelector) (system.cpp:741)
    current_time += delta_t;
    nextReaction->fire(randElement);             // apply transforms, update lists (system.cpp:760)
}
```

- **Propensity selection.** `System::getNextRxn()` (`system.cpp:623`) delegates to
  `selector->getNextReactionClass(nextReaction)`; `DirectSelector` walks the
  cumulative propensities until the running sum >= `r * a_tot` and returns both
  the chosen `ReactionClass*` and the leftover random fraction used to pick the
  specific reactant tuple from that rule's `ReactantList`.
- **Firing.** `ReactionClass::fire()` selects matching `MappingSet`s, runs the
  `TransformationSet`, and incrementally updates each touched molecule's rule
  membership and observable counts. Propensities are **not** globally recomputed
  every step — they are updated incrementally as molecules change (the loop only
  calls `recompute_A_tot()` at sample boundaries, `system.cpp:732`).
- **Variants.** `stepTo(stoppingTime)` (`system.cpp:795`) advances without output
  and stops just before overshooting; `singleStep()` (`system.cpp:845`) fires
  exactly one reaction; `equilibrate()` runs then resets the clock. These are the
  entry points the API layer uses for MCell's hybrid coupling.
- Null events (rejected intra-complex bindings) are counted in
  `NULL_EVENT_COUNTER` (`NFcore.hh:501`).

---

## 6. Input Handling — NFinput & Boost

Header `include/NFinput/NFinput.hh` (`namespace NFinput`, `using namespace NFcore`).
Implementation in `src/NFinput/`:

| File | Role |
|---|---|
| `NFinput.cpp` | Reads **BNG-XML** (via vendored TinyXML, `include/NFinput/TinyXML/`) and builds the `System`: molecule types, species, reaction rules, observables, functions, compartments |
| `parseFuncXML.cpp` | Parses functional rate-law expressions |
| `parseSymRxns.cpp` | Symmetric/equivalent-site reaction parsing |
| `commandLineParser.cpp` | CLI flags for the standalone executable |
| `rnfRunner.cpp`, `walk.cpp` | `.rnf` script running; molecule-graph walking |

**Boost usage** (`src/NFinput/NFinput.cpp:2-4`):
```cpp
#include <boost/algorithm/string.hpp>
#include <boost/range.hpp>
```
Used to tokenize pattern/label strings, e.g.
`boost::split(strs, label, boost::is_from_range(',', ','))` (`NFinput.cpp:896`),
and similarly to split on `!` (bonds), `@` (compartment), `_` (identifiers)
(`NFinput.cpp:913,951,953,2847`). This is the **only** place NFsim pulls in
Boost. The Boost headers come from the bundled copy in `libbng/libs` — MCell's
build wires them in explicitly because there is no system Boost on macOS
(`mcell/CMakeLists.txt:598-610`).

The C++ API layer keeps the parsed XML in `NFinput::XMLStructures` /
`NFinput::XMLFlags` statics so the System can be re-initialized cheaply
(`NFapi.cpp`).

---

## 7. Build System (`nfsim/CMakeLists.txt`)

A self-contained CMake project (`project(NFsim CXX)`, `CMakeLists.txt:3`). It
globs all `src/**/*.cpp` and `*.c` (`:36-38`) and builds four targets from the
same sources:

| Target | Type | CMake line | Used by |
|---|---|---|---|
| `NFsim_static` | static lib | `:52` | linked into the **mcell** executable |
| `NFsim_static_fpic` | static lib, PIC | `:53,55` | linked into **shared objects** (the C interface / `mcell.so`) |
| `NFsim` | **shared** lib | `:54,55` | shared consumers |
| `NFsim_EXE` (output `NFsim`) | executable | `:67-68` | standalone command-line simulator (non-Windows only) |

Namespaced aliases (`:60-61`):
```cmake
add_library(NFsim::NFsim     ALIAS NFsim_static)        # plain static
add_library(NFsim::NFsim_pic ALIAS NFsim_static_fpic)   # PIC static
```

Per-target config (`:72-82`):
- **PUBLIC** include dirs `NFSIM_INCLUDE_DIRS` (`:7-34`) — every `include/`
  subdirectory — so consumers inherit all NFsim headers transitively.
- `target_compile_features(... PUBLIC cxx_std_11)` — NFsim requires **C++11**.
- A long list of `-Wno-*` warning suppressions (`:42-48`): the tree is vendored
  and intentionally **not modified**, so its warnings are silenced rather than
  fixed. C++-only flags (`-Wno-delete-non-virtual-dtor`, `-Wno-reorder`) are
  applied via generator expressions; MSVC gets `/EHsc`.
- The standalone `NFsim_EXE` is Unix-only because MSVC breaks the `.ilk` file
  (`:65-66`).

**Boost** is *not* found by NFsim itself; it is injected by the parent MCell
build (`mcell/CMakeLists.txt:604-610`) as a SYSTEM PRIVATE include of
`libbng/libs`. Note the build guard: **gcc-8.3.0 segfaults NFsim**, so MCell's
top-level CMake hard-errors on that exact compiler version
(`mcell/CMakeLists.txt:21-28`).

---

## 8. How NFsim Is Consumed (the MCell boundary)

MCell does **not** call NFsim's C++ classes directly. The layering is:

```
   mcell  (C)
     │  includes ONLY  nfsim_c.h + nfsim_c_structs.h   (extern "C")
     │  links  NFsimC::NFsimC  and  NFsim::NFsim
     ▼
 nfsimCInterface  (C++ wrapper, builds nfsim_c / nfsim_c_static)
     │  #include "NFapi.hh";  links  NFsim::NFsim_pic
     ▼
 NFsim::NFapi  (namespace NFapi, src/NFapi.cpp) — facade over a single global System*
     ▼
 NFcore::System / Molecule / Complex / ReactionClass / Observable
```

### The NFapi facade — `nfsim/include/NFapi.hh`, `src/NFapi.cpp`

NFapi keeps three statics — `System* system`, `NFinput::XMLStructures*`,
`NFinput::XMLFlags` (`NFapi.hh:105-107`) — and exposes a small set of functions
that wrap the heavy NFcore classes:

| NFapi function | NFcore touched |
|---|---|
| `setupNFSim` (`NFapi.cpp:75`) | parse BNG-XML → XMLStructures |
| `resetSystem` / `deleteSystem` (`:89,99`) | (re)build / delete the `System*` |
| `initSystemXML` / `initSystemNauty` (`:106,118`) | `NFinput::initStartSpecies…`, `System::prepareForSimulation` |
| `stepSimulation()` / `stepSimulation(rxn)` (`:372,352`) | `System::singleStep`, `ReactionClass::fire` |
| `queryByNumReactant` (`:130`) | `Complex::getCanonicalLabel`, `ReactionClass` rate/name |
| `querySystemStatus` (`:296`) | `System::getAllComplexes`, `Complex::isAlive/getCanonicalLabel/getCompartment` |
| `queryObservables` (`:345`) | `System::getAllObservableCounts` |
| `getCompartmentInformation` (`:17`) | `CompartmentList::getCompartment` |
| `extractSpeciesCompartmentFromNauty` (`:203`) | `NFinput::transformComplexString`, `Compartment` |

### The C wrapper — `nfsimCInterface/src/nfsim_c.{h,cpp}`

Every `extern "C"` function in `nfsim_c.h` calls the matching `NFapi::` function
and owns MCell-specific bookkeeping (gdat output, `preInitMap`,
`currentReactions`). Examples: `setupNFSim_c` → `NFapi::setupNFSim`;
`initSystemNauty_c` → `NFapi::initSystemNauty`; `stepSimulation_c` →
`NFapi::stepSimulation`; `queryObservables_c`, `initAndQueryByNumReactant_c`,
`initAndQuerySystemStatus_c`, `extractSpeciesCompartmentFromNauty_c`, etc.
`nfsim_c_structs.{h,cpp}` provides opaque C accessors
(`map_get`, `mapvector_get`, `mapvectormap_get…`) over the `std::map`/`std::vector`
results returned through `void*`, so MCell never sees a C++ type.

**Which NFsim classes the C interface wraps (transitively, via NFapi):**
`System` (lifecycle + stepping + observables + complexes), `Complex` (canonical
label, alive flag, compartment), `MoleculeType`/`Molecule` (rule membership for
the reactant queries), `ReactionClass` (name/rate/firing), `Observable`
(counts), and `Compartment`. MCell sees **none** of these directly.

### MCell side — `mcell/CMakeLists.txt`

- Both bundled siblings are added as subdirectories
  (`add_subdirectory(.../nfsim …)` `:354`; `nfsimCInterface` `:355`).
- The MCell executable links `NFsimC::NFsimC NFsim::NFsim`
  (`mcell/CMakeLists.txt:543`), and adds `nfsimCInterface/src` to its include path
  (`:470`) — MCell's NFsim glue is its own C files `src/nfsim_func.c`,
  `src/react_*_nfsim.c`, `src/react_util_nfsim.c` (`:400-409`).
- MCell's call sites use only the `_c` API: `setupNFSim_c` in `argparse.c`,
  `logNFSimObservables_c`/`deleteNFSimSystem_c` in `mcell_run.c`,
  `constructNauty_c` in `vol_util.c`/`react_outc_nfsim.c`,
  `initAndQueryByNumReactant_c`/`initAndQuerySystemStatus_c` in the
  `react_*_nfsim.c` trigger/output paths.

**The clean cut:** NFsim's C++ classes are fully encapsulated inside
`nfsimCInterface` (which alone includes `NFapi.hh` and the NFsim headers and
links `NFsim::NFsim_pic`). MCell crosses the boundary purely through the
`extern "C"` `_c` functions plus the opaque-pointer STL accessors.

---

## 9. Quick Reference — file map

| Want | Look at |
|---|---|
| All core class declarations | `nfsim/include/NFcore/NFcore.hh` |
| Simulation loop | `nfsim/src/NFcore/system.cpp:668` (`sim`), `:795` (`stepTo`), `:845` (`singleStep`) |
| Rule-firing machinery | `nfsim/include/NFreactions/transformations/transformationSet.hh`, `transformation.hh` |
| Pattern matching | `nfsim/include/NFcore/templateMolecule.hh`, `NFreactions/mappings/` |
| BNG-XML + Boost | `nfsim/src/NFinput/NFinput.cpp` |
| C++ API facade | `nfsim/include/NFapi.hh`, `nfsim/src/NFapi.cpp` |
| C wrapper (MCell boundary) | `nfsimCInterface/src/nfsim_c.{h,cpp}`, `nfsim_c_structs.{h,cpp}` |
| Build targets / aliases | `nfsim/CMakeLists.txt:52-70` |
| MCell linkage / gcc-8.3.0 guard | `mcell/CMakeLists.txt:21-28,354-355,534,589-601` |
