# 03 — nfsimCInterface (the C API wrapper around NFsim)

> Keystone document linking **MCell** to **NFsim**. `nfsimCInterface` is the thin
> C-callable boundary that lets MCell's C core drive the C++ network-free
> simulator NFsim.
>
> Project root (`$ROOT`): the build root, the parent of this wiki folder (see `CLAUDE.md`).
> Target: `$ROOT/nfsimCInterface`

---

## 1. Purpose & Scope

MCell's legacy simulation core (`mcell/src/*.c`) is **C** (compiled with a C
compiler, or as C-linkage translation units). NFsim, in contrast, is a **C++**
engine (`System`, `Molecule`, `ReactionClass`, `Complex`, `Compartment`, STL
containers, `shared_ptr`, etc.). C code cannot directly call C++ methods, pass
C++ objects, or include C++ headers.

`nfsimCInterface` solves this by providing a **stable, `extern "C"` boundary**:

- Every public function is declared inside an `extern "C"` block, so it gets
  C linkage and is callable from MCell's `.c` files.
- The implementation files are C++ (`.cpp`), so internally they freely call the
  C++ `NFapi::*` functions and manipulate STL objects.
- All C++ objects that must cross the boundary are hidden behind **`void*`
  opaque handles** (e.g. `std::map`, `std::vector<...>` are created, queried,
  and destroyed only through accessor functions — MCell never sees their type).
- Results that *are* materialized into C live in flat POD structs with raw
  `char*`/`double*`/`int*` arrays (`malloc`/`free` ownership) so C can read them.

The layer sits directly on top of NFsim's own C++ facade, `NFapi`
(`nfsim/include/NFapi.hh`), which already collapses NFsim internals into a few
high-level entry points. `nfsimCInterface` is essentially a 1:1 C re-export of
`NFapi` plus a handful of MCell-specific helpers (incremental seeding, observable
logging, `.gdat` output).

---

## 2. Directory & Layout

```
nfsimCInterface/
├── CMakeLists.txt              # builds nfsim_c_static, nfsim_c (shared), NFsimC::NFsimC alias
├── example.c                   # standalone usage demo (not built by MCell)
├── nfsim_python.py             # Python/ctypes binding demo
├── README.md
├── LICENSE.txt
└── src/
    ├── nfsim_c.h               # PUBLIC C API: ~30 functions + POD result structs
    ├── nfsim_c.cpp             # implementation: wraps NFapi::* calls
    ├── nfsim_c_structs.h       # PUBLIC C API for opaque STL container handles
    └── nfsim_c_structs.cpp     # implementation: create/get/size/delete on void* maps & vectors
```

Only four source files compose the library:

| File | Role |
|---|---|
| `src/nfsim_c.h` | Declares the high-level C API and the C result structs (`queryOptions`, `reactantQueryResults`, `compartmentStruct`, `observableResults`, …). |
| `src/nfsim_c.cpp` | Implements those functions by forwarding to `NFapi::*` and translating C++ ⇄ C. |
| `src/nfsim_c_structs.h` | Declares the opaque-handle accessor API (`map_*`, `mapvector_*`, `mapvectormap_*`). |
| `src/nfsim_c_structs.cpp` | Implements those accessors as `reinterpret_cast` over `std::map`/`std::vector`. |

The internal C++ typedefs used to interpret the opaque handles
(`nfsim_c_structs.cpp:9-11`, mirrored in `nfsim_c.cpp:11-13`):

```cpp
typedef std::map<std::string, std::string>            Map;          // one query result row
typedef std::vector<Map*>                             MapVector;    // list of result rows (e.g. reactions on a complex)
typedef std::map<std::string, MapVector*>             MapVectorMap; // reactant-key -> its reactions
```

---

## 3. The C API Surface (MOST IMPORTANT)

### 3.1 Opaque handle model

NFsim query results are nested C++ containers. Rather than deep-copying them
into C, MCell receives a `void*` and walks it through accessor functions defined
in `nfsim_c_structs.cpp`. The three handle shapes:

- **`Map` (`void*`)** — a single result "row": string→string fields such as
  `"label"`, `"name"`, `"rate"`, `"diffusion_function"`, `"reactionDimensionality"`,
  `"resample"`, `"compartment"`.
- **`MapVector` (`void*`)** — an ordered list of `Map*` (e.g. all reactions a
  complex can fire, or all complexes in the system).
- **`MapVectorMap` (`void*`)** — keyed by reactant nauty-string → its `MapVector`
  of reactions.

### 3.2 Opaque-handle accessor API (`nfsim_c_structs.h` / `.cpp`)

| C function | Returns | Purpose | Impl |
|---|---|---|---|
| `void* map_create()` | `Map*` | allocate empty result row | `nfsim_c_structs.cpp:27` |
| `const char* map_get(void* map, const char* key)` | field value or `NULL` | read one field of a result row | `nfsim_c_structs.cpp:17` |
| `void* mapvector_create()` | `MapVector*` | allocate result list (MCell passes it into a query) | `nfsim_c_structs.cpp:31` |
| `int mapvector_size(void*)` | count | number of rows | `nfsim_c_structs.cpp:98` |
| `void* mapvector_get(void* vec, int i)` | `Map*` | i-th row | `nfsim_c_structs.cpp:108` |
| `void mapvector_delete(void*)` | — | free list + rows | `nfsim_c_structs.cpp:55` |
| `void* mapvectormap_create()` | `MapVectorMap*` | allocate reactant→reactions container | `nfsim_c_structs.cpp:35` |
| `int mapvectormap_size(void*)` | count | number of reactant keys | `nfsim_c_structs.cpp:103` |
| `void* mapvectormap_get(void* c, char* reactant)` | `MapVector*` | reactions for a reactant key | `nfsim_c_structs.cpp:67` |
| `char** mapvectormap_getKeys(void* c)` | `char**` (malloc'd) | list of reactant keys | `nfsim_c_structs.cpp:76` |
| `void mapvectormap_delete(void*)` | — | deep-free the whole container | `nfsim_c_structs.cpp:39` |

### 3.3 POD result structs (`nfsim_c.h`)

- `struct queryOptions` (`nfsim_c.h:59-68`) — the **input** to a query: parallel
  arrays of reactant nauty strings (`initKeys`) + seed counts (`initValues`),
  plus option key/value pairs (`optionKeys`/`optionValues`, e.g.
  `"numReactants"="2"`, `"onlyActive"="0"`). Maps to `NFapi::numReactantQueryIndex`.
- `struct compartmentStruct` (`nfsim_c.h:70-75`) — `name`, `spatialDimensions`,
  `size`, `outside`. Output of `getCompartmentInformation_c`.
- `struct observableResults` (`nfsim_c.h:34-38`) — parallel `char** names` /
  `double* values` + count.
- `struct reactantQueryResults` / `reactionResult` (`nfsim_c.h:29-57`) — a fully
  materialized C copy of a `MapVectorMap` (built by `map2ReactantQueryResults`,
  `nfsim_c.cpp:273`). MCell largely uses the opaque-handle path instead and only
  calls `delete_reactantQueryResults` defensively.

### 3.4 High-level C API (`nfsim_c.h` / `nfsim_c.cpp`)

| C function (signature) | What it does | Wraps | Impl |
|---|---|---|---|
| `int setupNFSim_c(const char* filename, int seed, int verbose)` | Load the BNGL→XML model file into NFsim, seed the RNG. Returns 0/-1. | `NFapi::setupNFSim` + `NFutil::SEED_RANDOM` | `nfsim_c.cpp:38` |
| `int resetSystem_c()` | Reset the system to a clean pre-seed state (before re-seeding molecules). | `NFapi::resetSystem` | `nfsim_c.cpp:55` |
| `int deleteNFSimSystem_c()` | Tear down the C++ `System` (destructors). | `NFapi::deleteSystem` | `nfsim_c.cpp:61` |
| `int initSystemXML_c(const char* xml)` | Seed the system from an XML init string. | `NFapi::initSystemXML` | `nfsim_c.cpp:66` |
| `int initSystemNauty_c(const char** nautyStrings, const int* counts, int n)` | Seed from `n` (nauty-string, count) pairs. | `NFapi::initSystemNauty(map)` | `nfsim_c.cpp:73` |
| `int constructNauty_c(const char* nautyString, const int count)` | **Incrementally** add/subtract `count` copies of a species in a static seeding table (`preInitMap`), clamped ≥0. MCell-specific. | (no NFapi call) | `nfsim_c.cpp:91` |
| `int initFromConstruct_c()` | Seed the system from the accumulated `preInitMap`. | `NFapi::initSystemNauty(preInitMap)` | `nfsim_c.cpp:83` |
| `void queryByNumReactant_c(const int n, void* results)` | Fill a `MapVectorMap` with complexes that participate in reactions taking `n` reactants. | `NFapi::queryByNumReactant` | `nfsim_c.cpp:342` |
| `void initAndQueryByNumReactant_c(queryOptions opts, void* results)` | **Reset + seed (per opts) + query reactions by reactant count**, memoized. The core reaction-discovery call. Fills a `MapVectorMap`. | `NFapi::initAndQueryByNumReactant` | `nfsim_c.cpp:408` |
| `void querySystemStatus_c(const char* option, void* results)` | Fill a `MapVector` with all complexes (option `"complex"`) currently in the system. | `NFapi::querySystemStatus` | `nfsim_c.cpp:249` |
| `void initAndQuerySystemStatus_c(queryOptions opts, void* results)` | Reset + seed + step + query full system status into a `MapVector`. Also registers `"reaction"` option keys in the reaction log. | `NFapi::initAndQuerySystemStatus` | `nfsim_c.cpp:376` |
| `observableResults queryObservables_c()` | Snapshot current observable name/value pairs. | `NFapi::queryObservables` | `nfsim_c.cpp:356` |
| `int stepSimulation_c()` | Advance the simulation one step (fire one reaction). | `NFapi::stepSimulation()` | `nfsim_c.cpp:440` |
| `int stepSimulationRxn_c(const char* rxn)` | Advance exactly one step by firing the named reaction. | `NFapi::stepSimulation(rxn)` | `nfsim_c.cpp:447` |
| `compartmentStruct getCompartmentInformation_c(const char* name)` | Look up a BNGL compartment's dimensionality/size/outside. `""` ⇒ default 3-D vol. | `NFapi::getCompartmentInformation` (returns `shared_ptr<Compartment>`) | `nfsim_c.cpp:454` |
| `void freeCompartmentInformation_c(compartmentStruct*)` | Free `name`/`outside` of a struct. | — | `nfsim_c.cpp:479` |
| `int delete_compartmentStructs(compartmentStruct)` | Same, by value. | — | `nfsim_c.cpp:334` |
| `const char* extractSpeciesCompartmentFromNauty_c(const char* nauty)` | Parse the compartment name out of a species nauty/graph string (`strdup`'d). | `NFapi::extractSpeciesCompartmentFromNauty` | `nfsim_c.cpp:325` |
| `int logNFSimObservables_c(double time, int seed)` | Memoized observable snapshot appended to `observableLog`; auto-flushes to `.gdat` at 10000 rows. MCell-specific. | `NFapi::queryObservables` (via reset/init) | `nfsim_c.cpp:127` |
| `int logNFSimReactions_c(const char* name)` | Increment a fired-reaction counter. MCell-specific. | — | `nfsim_c.cpp:165` |
| `int outputNFSimObservables_c(int seed)` | Write `<input>.seed_NNNNN.gdat` + reactions `.gdat`. | — | `nfsim_c.cpp:171` |
| `int outputNFSimObservablesF_c(const char* file)` | Stream `observableLog` to a named `.gdat`. | — | `nfsim_c.cpp:191` |
| `int outputNFSimReactionsF_c(const char* file)` | Stream fired-reaction counts to a file. | — | `nfsim_c.cpp:234` |
| `int delete_reactantQueryResults(reactantQueryResults)` | Free a materialized C result tree. | — | `nfsim_c.cpp:307` |

---

## 4. How it wraps NFsim

The wrapper never touches NFsim's deep internals directly; it goes through the
`NFapi` namespace facade (`nfsim/include/NFapi.hh`), which owns the global
`System* system`, `XMLStructures`, and memoization dictionaries
(`NFapi.hh:102-107`). Translation pattern in `nfsim_c.cpp`:

1. **Marshal C → C++.** Convert `const char*` → `std::string`, build
   `std::map<string,int>` from parallel C arrays, populate
   `NFapi::numReactantQueryIndex` from a `queryOptions`
   (`nfsim_c.cpp:376-405`, `408-437`).
2. **Reinterpret opaque handles.** The `void* results` MCell passes is
   `reinterpret_cast` back to the real STL type, e.g.
   `MapVectorMap* q = reinterpret_cast<MapVectorMap*>(results);`
   (`nfsim_c.cpp:344`, `398`, `423`), then handed to `NFapi` by reference so
   NFsim fills it in place.
3. **Call NFapi.** e.g. `NFapi::initAndQueryByNumReactant(options, *q)`
   (`nfsim_c.cpp:424`). `NFapi` internally builds/reuses the `System`, computes
   reaction membership (`calculateRxnMembership` over `Complex`/`ReactionClass`,
   `NFapi.hh:80`), and serializes each reaction into a `map<string,string>`
   (name/rate/dimensionality/resample).
4. **Marshal C++ → C** only where a flat struct is returned — e.g.
   `getCompartmentInformation_c` reads `shared_ptr<Compartment>` getters and
   `malloc`+`memcpy` strings into `compartmentStruct` (`nfsim_c.cpp:454-477`);
   `queryObservables_c` flattens a `map<string,double>` into parallel arrays.

Bool→int convention throughout: `NFapi` returns `bool`; the `_c` functions
return `0` on success / `-1` (or `1` for step) on failure.

---

## 5. Build System (`nfsimCInterface/CMakeLists.txt`)

```cmake
project(nfsim_c CXX)                                  # C++ project (impl is .cpp)

add_library(nfsim_c_static STATIC ${SRC_FILES})      # static variant (no link deps)
add_library(nfsim_c        SHARED ${SRC_FILES})       # shared variant, links PIC nfsim
add_library(NFsimC::NFsimC ALIAS nfsim_c_static)      # <-- the alias MCell links
target_link_libraries(nfsim_c PUBLIC NFsim::NFsim_pic)
```

Targets produced (`CMakeLists.txt:13-16`):

- **`nfsim_c_static`** — STATIC library, exposed as **`NFsimC::NFsimC`** (the
  name MCell consumes). Carries no link dependency itself; the final MCell exe
  links NFsim separately.
- **`nfsim_c`** — SHARED library, links `NFsim::NFsim_pic` (the
  position-independent NFsim build).
- **`NFsimC::NFsimC`** — ALIAS → `nfsim_c_static`.

Include dirs (`CMakeLists.txt:18-30`, applied to both targets):

- **PUBLIC** `src/` — so consumers (MCell) can `#include "nfsim_c.h"` /
  `"nfsim_c_structs.h"`.
- **PRIVATE** the source root + `$<TARGET_PROPERTY:NFsim::NFsim,INTERFACE_INCLUDE_DIRECTORIES>`
  — pulls NFsim's own header search paths (for `NFapi.hh`, `NFsim.hh`, etc.)
  straight off the NFsim target.
- `cxx_std_11`, `POSITION_INDEPENDENT_CODE ON`, and a few `-Wno-*` relaxations.

MCell wires it up in `mcell/CMakeLists.txt`:

- `add_subdirectory(.../nfsimCInterface ...)` — `mcell/CMakeLists.txt:365`
- include dir `.../nfsimCInterface/src` — `mcell/CMakeLists.txt:471`
- links `NFsimC::NFsimC NFsim::NFsim` into the `mcell` target —
  `mcell/CMakeLists.txt:544`

---

## 6. Integration with MCell (CRITICAL)

MCell's C core includes `nfsim_c.h` (via `react_nfsim.h`) and a local opaque-map
helper `map_c.h`. Calls cluster in five files. Below is exactly which interface
functions each calls and why.

### 6.1 Reaction discovery / triggering — `mcell/src/react_trig_nfsim.c`

This is the hot path: deciding whether two (or one) molecules can react, and
building the MCell `struct rxn` from NFsim's answer.

- **Query option builders** `initializeNFSimQueryForBimolecularReactions`
  (`react_trig_nfsim.c:45`) and `...Unimolecular...` (`:110`) pack each
  molecule's `graph_data->graph_pattern` (a **nauty species string**) into a
  `queryOptions` with `numReactants` and `onlyActive`.
- `trigger_bimolecular_preliminary_nfsim` (`:128`): `mapvectormap_create()`
  (`:158`) → `initAndQueryByNumReactant_c(options, results)` (`:159`) →
  `mapvectormap_size(results)` (`:163`) to cheaply test "do these two species
  react at all?", caching the boolean by graph-hash. `mapvectormap_delete`
  (`:166`,`:173`).
- `trigger_bimolecular_nfsim` (`:183`): same query (`:232`); if reactions exist
  (`mapvectormap_size > 0`, `:237`) it calls `new_reaction()` +
  `initializeNFSimReaction(...)` (`:239`) to fill the `struct rxn`, then
  `mapvectormap_delete` (`:256`).
- `initializeNFSimReaction` (`:300`): walks the opaque result —
  `mapvectormap_getKeys` (`:305`), `mapvectormap_get` (`:306`),
  `mapvectormap_size` (`:309`), `mapvector_size` (`:311`),
  then per pathway `mapvector_get` (`:360`) + `map_get` to extract
  `"rate"` → `cum_probs`, `"name"` → `external_reaction_data.reaction_name`,
  `"resample"` → resample flag (`:361-365`).
- Unimolecular variant around `:445-478`: `initAndQueryByNumReactant_c` (`:460`),
  `mapvectormap_size` (`:465`), `delete_reactantQueryResults` (`:478`),
  `mapvectormap_delete` (`:476`).

### 6.2 Reaction outcome / product generation — `mcell/src/react_outc_nfsim.c`

After a reaction fires, MCell must learn the **products** and re-seed NFsim.

- `calculate_nfsim_diffusion_derived_data` (`:614`): uses
  `extractSpeciesCompartmentFromNauty_c` (`:620`) +
  `getCompartmentInformation_c` (`:622`) to get a species' spatial dimensions
  and compute space/time steps.
- `calculate_nfsim_reactivity` (`:654`): `mapvectormap_create` (`:659`) →
  `initAndQueryByNumReactant_c` (`:660`) → keys/size walk to set reactivity flags.
- Product handling: `initAndQuerySystemStatus_c` (`:599`,`:726`,`:786`,`:788`)
  to enumerate the resulting complexes (a `MapVector`), then `mapvector_get` /
  `map_get` (`:678-680`,`:729-730`) to read each product's label and compartment;
  `getCompartmentInformation_c` (`:352`,`:354`,`:446`,`:649`) for geometry;
  `delete_compartmentStructs` (`:475`).
- **Re-seeding the simulator after a reaction**: `constructNauty_c` is called to
  decrement consumed reactants and increment created products in the seeding
  table (`:601`,`:805`,`:811`,`:813`).

### 6.3 Orientation / compartment utilities — `mcell/src/react_util_nfsim.c`

`calculate_reactant_orientation` (`:7`) determines surface vs. volume geometry:
`extractSpeciesCompartmentFromNauty_c` for both reactants (`:16`,`:18`),
`getCompartmentInformation_c` (`:34`,`:35`) to compare each compartment's
`outside` against the other's `name`, then `freeCompartmentInformation_c`
(`:82`,`:83`).

### 6.4 Initial population seeding — `mcell/src/vol_util.c`

On molecule release, `constructNauty_c(graph_pattern, +delta / -delta)`
(`:1365`,`:1379`) registers initial populations into NFsim's seeding table so the
observable system reflects released counts.

### 6.5 Run-loop hooks — `mcell/src/mcell_run.c`, `react_outc.c`

- Per output interval: `logNFSimObservables_c(time, seed)` —
  `mcell_run.c:92`.
- On run end (`nfsim_flag` set): `outputNFSimObservables_c(seed_seq)` then
  `deleteNFSimSystem_c()` — `mcell_run.c:403-404`.
- Each fired reaction is recorded: `logNFSimReactions_c(name)` —
  `react_outc.c:1151`,`:1286`.
- Model load: `setupNFSim_c(...)` — `argparse.c:410`.

### 6.6 Data flow across the boundary (summary)

```
                MCell (C)                         nfsimCInterface (extern "C")          NFsim (C++, via NFapi)
  ─────────────────────────────────       ───────────────────────────────────     ───────────────────────────
  graph_data->graph_pattern  ──────────►  queryOptions {initKeys, optionKeys}
  (nauty species string)                          │
  void* = mapvectormap_create()  ─────────────────┤
                                                  ▼
                                          initAndQueryByNumReactant_c  ──────────►  NFapi::initAndQueryByNumReactant
                                          (reinterpret void* → MapVectorMap)         (reset → initSystemNauty →
                                                                                      compute ReactionClass membership)
  walk result with                       ◄── fills MapVectorMap in place ──────────
  mapvectormap_getKeys / _get,
  mapvector_get, map_get("rate"/"name")
        │
        ▼
  build struct rxn (cum_probs,
  external_reaction_data[])

  on release / on product:
  constructNauty_c(pattern, ±count)  ───►  preInitMap[pattern] += count
                                                  │ (initFromConstruct_c → NFapi::initSystemNauty)
  per interval: logNFSimObservables_c  ──►  NFapi::queryObservables → observableLog
  on end: outputNFSimObservables_c     ──►  write .gdat ; deleteNFSimSystem_c → NFapi::deleteSystem
```

Key invariant: **the only payload that crosses C→C++ is the nauty/graph-pattern
string** (MCell's canonical species identity) plus integer counts and option
flags. **The only payload that crosses C++→C is flattened text** (string→string
result rows read field-by-field via `map_get`, or POD structs). No C++ object is
ever named in MCell code — everything is `void*` plus the accessor functions.

---

## 7. Cross-references

- NFsim C++ facade: `nfsim/include/NFapi.hh`, `nfsim/src/NFapi.cpp`
- MCell callers: `mcell/src/react_trig_nfsim.c`, `react_outc_nfsim.c`,
  `react_util_nfsim.c`, `nfsim_func.c`, `vol_util.c`, `mcell_run.c`,
  `react_outc.c`, `argparse.c`; declarations in `mcell/src/react_nfsim.h`.
- MCell's local opaque-map helper header used alongside: `mcell/src/map_c.h`.
- Build wiring: `mcell/CMakeLists.txt:365` (add_subdirectory), `:471` (include),
  `:544` (link `NFsimC::NFsimC NFsim::NFsim`).
