# libbng — BioNetGen Language Library (developer reference)

`namespace BNG::`  ·  located at `$ROOT/libbng` (`$ROOT` = the build root; see `CLAUDE.md`)

libbng is a self-contained C++17 library bundled inside MCell that implements the
data model and runtime machinery for the **BioNetGen Language (BNGL)**: parsing
BNGL text, representing rule-based species and complexes as graphs, applying
reaction rules to generate products on-the-fly, and canonicalizing complexes for
fast species matching. MCell links it as `BNG::bng` and uses it as the
authoritative store of molecule types, species, reaction rules and reaction
classes during simulation.

---

## 1. Purpose & scope

BNGL is a rule-based modeling language for reaction networks. Instead of
enumerating every species, the modeler declares **molecule types** with
**components** (binding sites) that may carry **states**, and **reaction rules**
that act on *patterns* of complexes. The full species network is generated
on demand by applying rules.

libbng provides:

| Capability | Where |
|---|---|
| BNGL text parsing (flex/bison) + semantic analysis | `bngl_scanner.l`, `bngl_parser.y`, `ast.*`, `semantic_analyzer.*`, `parser.*` |
| Complex / species representation as molecule graphs | `cplx.*`, `elem_mol.*`, `elem_mol_type.*`, `species.*` |
| Reaction rule application (graph rewriting → products) | `rxn_rule.*` |
| Reaction-class generation on the fly (unimol/bimol) | `rxn_container.*`, `rxn_class.*` |
| Graph isomorphism / subgraph matching | `graph.*` (Boost Graph Library) |
| Canonical species naming (canonical label) | `cplx.cpp` (nauty/Traces) |
| Owning engine + shared data | `bng_engine.*`, `bng_data.*`, `bng_config.*` |

Two matching modes underpin everything (`bng/cplx.h:36`): a `Cplx` is used either
as a **pattern** (partially specified — used in rules/observables) or as a **fully
qualified species** (all components present with states set). Simple complexes
(single molecule, no components — `SPECIES_CPLX_FLAG_ONE_MOL_NO_COMPONENTS`) are
fast-pathed; everything else uses graph algorithms.

---

## 2. Directory & module layout

```
libbng/
├── CMakeLists.txt              # top-level: add_subdirectory(bng) + test/api
├── README.md  LICENSE
├── bng/                        # THE library source (namespace BNG::)
│   ├── CMakeLists.txt          # builds target `libbng` (alias BNG::bng) + parser_tester_libbng
│   ├── bng.h                   # single umbrella public header
│   ├── bng_defines.h           # id types, sentinels, constants, flags
│   ├── shared_defines.{h,cpp}  # small_vector, uint_set, base typedefs
│   ├── base_flag.{h,cpp}       # BaseFlag / BaseSpeciesCplxMolFlag bitflag base classes
│   │
│   ├── bng_engine.{h,cpp}      # BNGEngine — owns species, rxns, data
│   ├── bng_data.{h,cpp}        # BNGData — parsed model (mol types, components, compartments…)
│   ├── bng_config.{h,cpp}      # BNGConfig — units/config (MCell subclasses this)
│   │
│   ├── cplx.{h,cpp}            # Cplx — complex/pattern; canonicalization (nauty) lives here
│   ├── elem_mol.{h,cpp}        # ElemMol (molecule instance) + Component
│   ├── elem_mol_type.{h,cpp}   # ElemMolType (molecule type) + ComponentType
│   ├── species.{h,cpp}         # Species : Cplx (adds id, D, time/space step, flags)
│   ├── species_container.{h,cpp}# SpeciesContainer — owns/dedups Species
│   │
│   ├── rxn_rule.{h,cpp}        # RxnRule (82 KB cpp) — rule application / product generation
│   ├── rxn_class.{h,cpp}       # RxnClass — applicable rxns for a (set of) reactant(s)
│   ├── rxn_container.{h,cpp}   # RxnContainer — owns RxnRules, builds RxnClasses on demand
│   ├── rxn_compartment_utils.{h,cpp} # compartment/orientation processing for rules
│   │
│   ├── graph.{h,cpp}           # Boost adjacency_list graph + subgraph isomorphism
│   ├── ast.{h,cpp}             # parse tree node types
│   ├── semantic_analyzer.{h,cpp}# AST → BNGData conversion + checks (41 KB)
│   ├── parser.{h,cpp}          # parse_bngl_file / parse_single_cplx_string entry points
│   ├── parser_utils.{h,cpp}    # ParserContext, string pool, numeric conversion
│   ├── bngl_scanner.l          # flex lexer
│   ├── bngl_parser.y           # bison grammar
│   ├── bngl_names.h            # BNGL keyword string constants
│   ├── filesystem_utils.{h,cpp}# path helpers (also used directly by MCell)
│   ├── debug_config.h          # DEBUG_* compile switches
│   ├── test.cpp                # dev-only main() for parser_tester_libbng
│   └── utils/                  # misc helpers
│
├── libs/                       # bundled third-party (header/SYSTEM include)
│   ├── nauty/                  # graph canonical labeling (C, compiled as C++)
│   │   └── CMakeLists.txt      # builds STATIC target `nauty` (alias Nauty::nauty)
│   ├── sparsehash/src/         # Google sparsehash (dense_hash_map etc.)
│   └── boost/                  # vendored Boost (graph, pool, dynamic_bitset, …)
│
└── test/
    ├── api/                    # per-dir C++ executables exercising the engine (CMake)
    ├── parser/                 # python-driven parser tests (parser_tester_libbng)
    └── boost_subgraph/
```

---

## 3. Public API surface

`bng/bng.h` is the only header a consumer must include (`bng/bng.h:18-26`). It
pulls in `species_container.h`, `bng_engine.h`, `rxn_rule.h`, `rxn_class.h`,
`rxn_container.h`, `rxn_compartment_utils.h`, `species.h`, `parser.h`,
`bngl_names.h`.

### Key public classes

| Class | Header | Role |
|---|---|---|
| `BNGEngine` | `bng_engine.h:32` | Top-level owner. Holds `SpeciesContainer all_species`, `RxnContainer all_rxns`, `BNGData data`, and a `const BNGConfig&`. Accessors `get_all_species()`, `get_all_rxns()`, `get_data()`, `get_config()`; pattern matching and BNGL export. |
| `BNGData` | `bng_data.h:116` | Parsed, mostly-immutable model: state names, `ComponentType`s, `ElemMolType`s, `Compartment`s, `RxnRule`s, parameters, seed species, observables. `find_or_add_*` / `find_*_id` / `get_*` API. |
| `BNGConfig` | `bng_config.h:87` | Units & runtime config. `use_bng_units` flag (`bng_config.h:120`). **MCell's `SimulationConfig` derives from this.** |
| `SpeciesContainer` | `species_container.h:29` | Owns `Species*`; dedups via canonical-name map (`find_or_add`, `find_or_add_delete_if_exist`, `get`, `find`, `remove`, `get_species_vector`). Holds the ALL_MOLECULES/VOLUME/SURFACE superspecies ids. |
| `Species` | `species.h:25` | `: public Cplx, public ElemMolTypeSpeciesCommonData`. Adds `species_id_t id`, diffusion constant `D`, `space_step`/`time_step`, rxn-capability flags, instantiation counting, reactant-class id. |
| `Cplx` | `cplx.h:36` | `: public BaseSpeciesCplxMolFlag`. A complex or pattern: `ElemMolVector elem_mols`, orientation, canonical `name`, a Boost `Graph`. Matching: `matches_pattern`, `matches_fully`, `canonicalize`. |
| `ElemMol` | `elem_mol.h:86` | One elementary-molecule instance/pattern: `elem_mol_type_id`, `compartment_id`, `components`. |
| `Component` | `elem_mol.h:30` | A binding site instance: `component_type_id`, `state_id`, `bond_value`. |
| `ElemMolType` | `elem_mol_type.h:88` | Molecule type definition: name, allowed `component_type_ids`, diffusion/step. |
| `ComponentType` | `elem_mol_type.h:22` | Component definition: name + `allowed_state_ids`. |
| `Compartment` | `bng_data.h:53` | Named 2D/3D compartment with parent/children hierarchy, volume/area. |
| `RxnRule` | `rxn_rule.h:180` | Unidirectional rule (reversible → two rules). `reactants`/`products` are `CplxVector` patterns; product generation via reactant/product graphs; rate handling. |
| `RxnClass` | `rxn_class.h:31` | All applicable reactions for a specific reactant (or reactant pair); holds `RxnClassPathwayVector pathways`. Created on the fly by `RxnContainer`. |
| `RxnContainer` | `rxn_container.h:85` | Owns `RxnRule`s; lazily builds unimol/bimol `RxnClass`es (`get_unimol_rxn_class`, `get_bimol_rxn_class`, `get_bimol_rxns_for_reactant`), reactant-class machinery (`get_reacting_classes`, `get_reactant_class`), `find_rxn_rule_by_name`, `get_rxn_rules_vector`. |
| `ReactantClass` | `rxn_container.h:39` | Equivalence class of species sharing the same reaction-id bitsets (optimizes bimol lookup). |
| free fns `parse_bngl_file`, `parse_single_cplx_string` | `parser.h:27,41` | BNGL parsing entry points. |
| `process_compartments_and_set_orientations` etc. | `rxn_compartment_utils.h` | post-parse rule fixups. |

### Identifier types & sentinels (`bng/bng_defines.h`, `shared_defines.h`)

`species_id_t`, `elem_mol_type_id_t`, `component_type_id_t`, `state_id_t`,
`compartment_id_t`, `rxn_rule_id_t`, `rxn_class_pathway_index_t`,
`reactant_class_id_t`, `bond_value_t`, `orientation_t` — all integer ids with
`*_INVALID` / `*_NONE` sentinels (e.g. `SPECIES_ID_INVALID`,
`COMPARTMENT_ID_NONE`, `STATE_ID_DONT_CARE`, `BOND_VALUE_BOUND/UNBOUND/ANY`).
Containers: `small_vector<T>` and `uint_set<T>` from `shared_defines.h`.

---

## 4. The BNGL parser

Flex/bison driven, **not reentrant** (uses globals — note the warning at
`parser.h:26`).

- **Lexer** `bng/bngl_scanner.l`: prefix `bngl`, `%option noyywrap yylineno`.
  Tokenizes BNGL section keywords (`begin`/`end`, `parameters`, `molecule types`,
  `compartments`, `reaction rules`, `seed species`, `observables`, `actions`),
  identifiers, ints/doubles, strings, arrows (`->`, `<->`), `=>`, and punctuation
  `~ , ! ? . ( ) / + - * ^ { } = ; @ :`. The pseudo-keyword `!CPLX`
  (`bngl_scanner.l:81`) switches the grammar into single-complex parse mode.
- **Grammar** `bng/bngl_parser.y` (`%require "3.0"`, prefix `bngl`): builds an AST
  (`%code requires #include "bng/ast.h"`). Tokens at `bngl_parser.y:100-125`;
  top rule `model_sections` at `:195`; molecule-type and `cplx` productions follow.
  Provides `create_parser_context` / `get_parser_context` / `delete_parser_context`
  (`bngl_parser.y` `%code provides`).
- **AST → data** `bng/semantic_analyzer.{h,cpp}` (`SemanticAnalyzer`): evaluates
  parameter expressions, resolves component/state names, builds `ElemMolType`s,
  `Compartment`s, `RxnRule`s, seed species and observables into `BNGData`, and runs
  consistency checks.
- **Drivers** `bng/parser.cpp`:
  - `parse_bngl_file(file, BNGData&, parameter_overrides)` — clears `bng_data`,
    runs `bnglparse()`, then `SemanticAnalyzer::check_and_convert_parsed_file`,
    returns error count (`parser.cpp:28`).
  - `parse_single_cplx_string(str, BNGData&, Cplx& out)` — prepends `!CPLX`, scans
    the in-memory string, fills one `Cplx` (`parser.cpp:72`). Does *not* clear
    `bng_data`, so callers can add complexes incrementally.

CMake generates the scanner/parser via `BISON_TARGET`/`FLEX_TARGET` into
`${BINARY_DIR}/deps/` (`bng/CMakeLists.txt:32-48`); `YY_NO_UNISTD_H` is defined
for the build.

---

## 5. nauty integration (graph canonicalization)

Two graph libraries are used for different jobs:

- **Boost Graph Library** (`bng/graph.{h,cpp}`): the working representation. Each
  `Cplx` owns a `Graph` = `boost::adjacency_list<vecS, vecS, undirectedS, …>`
  whose vertices are `Node`s tagged as either a molecule (`ElemMol*`) or a
  component (`Component*`), edges are bonds (`graph.h:54-197`). Pattern matching
  uses `get_subgraph_isomorphism_mappings(pattern, cplx, only_first_match, res)`
  (`graph.h:211`) — this is how rule reactant patterns are matched against species
  and how observables count matches.

- **nauty / Traces** (`bng/cplx.cpp:24-25`, includes `nauty/traces.h`,
  `nauty/nausparse.h`): used **only for canonical labeling** of a complex so that
  two structurally identical complexes get the *same* canonical string `name`
  regardless of how molecules/components were ordered in the input. This is what
  makes species dedup in `SpeciesContainer` an O(1) name-map lookup.
  - `Cplx::canonicalize()` builds a sparse graph, assigns colors (a color encodes
    molecule type, and for components the component type + state — see
    `cplx.cpp:482-486`), then calls `Traces(&sg1, labels, ptn, orbits, &options,
    &stats, &cg1)` (`cplx.cpp:557`) with `DEFAULTOPTIONS_TRACES` and
    `getcanon = TRUE`. The returned `labels` permutation reorders molecules and
    components into canonical order, then the `Cplx` is rebuilt and `name` is set.
  - `Traces` is chosen over `sparsenauty`/`nauty` because it leaks less; the code
    calls `nausparse_freedyn()` afterward (`cplx.cpp:566`).
  - **Gotcha:** nauty uses many globals; the comment at `cplx.cpp:550` warns the
    canonicalization path is **not thread-safe**.

nauty's C sources are compiled **as C++** and forced **STATIC**
(`libs/nauty/CMakeLists.txt`): a comment there records that a *shared* nauty fails
to link into `mcell.so` with `undefined reference to updatecan`.

---

## 6. Build system

### Targets

| Target | Defined in | Notes |
|---|---|---|
| `libbng` (static/normal lib) + alias **`BNG::bng`** | `bng/CMakeLists.txt:61-62` | The library MCell links. |
| **`Nauty::nauty`** (alias of static `nauty`) | `libs/nauty/CMakeLists.txt` | Forced `STATIC`; compiles 8 nauty `.c` files as C++. |
| `parser_tester_libbng` | `bng/CMakeLists.txt:92` | Dev-only executable. Recompiles all libbng sources **plus** `test.cpp` (it does **not** link libbng, to avoid duplicate symbols), so its include dirs/defs are spelled out explicitly. |
| `api` test executables | `test/api/*/CMakeLists.txt` | One `add_executable` per subdir (`0000_unimol_rxn`, `0010_dump_complex_graph`, `0100/0110/0120_generate_network*`). |

> Note: there is **no `bng_analyzer` target** in this tree. The development tool is
> `parser_tester_libbng`; analysis-style usage is exercised by the `test/api`
> executables.

### Usage requirements (propagated to anything linking `BNG::bng`)

From `bng/CMakeLists.txt:64-87`:
- `target_compile_features(... PUBLIC cxx_std_17)` — **C++17 required**.
- PUBLIC include dirs: `libbng/bng` and `libbng/` (so consumers write
  `#include "bng/..."`).
- PUBLIC **SYSTEM** include dirs: `libbng/libs` and `libbng/libs/sparsehash/src`
  (third-party warnings suppressed).
- PRIVATE: generated flex/bison headers in `deps/`; definition `YY_NO_UNISTD_H`.
- Links `PRIVATE Nauty::nauty` (and `stdc++fs` on non-Win/non-Apple).

nauty's PUBLIC include dir means linking it gives its headers for free
(`libs/nauty/CMakeLists.txt`).

---

## 7. Integration points with MCell

MCell adds the library and links it in **`mcell/CMakeLists.txt`**:

| Line | Statement |
|---|---|
| `mcell/CMakeLists.txt:361` | `add_subdirectory(${CMAKE_SOURCE_DIR}/../libbng/ ${CMAKE_CURRENT_BINARY_DIR}/libs/libbng)` |
| `mcell/CMakeLists.txt:471-473` | `include_directories(... ../libbng ../libbng/libs ../libbng/libs/sparsehash/src)` |
| `mcell/CMakeLists.txt:543` | `mcell` links `... BNG::bng NFsimC::NFsimC NFsim::NFsim JsonCpp::JsonCpp Nauty::nauty ...` |
| `mcell/CMakeLists.txt:562` | `add_dependencies(mcell4_so libmcell mcell4 libbng ...)` |
| `mcell/CMakeLists.txt:565` | `mcell4_so` (the `.so`) links `... BNG::bng JsonCpp::JsonCpp Nauty::nauty ...` |

`libmcell` itself does **not** directly link `BNG::bng`; BNG symbols are resolved
transitively at the executable / `.so` link step (note also lines 57/100 comments
about not turning bundled static libs like nauty/libbng into shared objects).

### How MCell consumes the API

- **Header usage:** `bng/bng.h` is included by ~19 MCell files
  (`src4/world.h`, `molecule.h`, `collision_structs.h`, converters, API objects…);
  `bng/bng_defines.h` by ~13 (`src4/defines.h`, `callbacks.h`, generators…);
  `bng/filesystem_utils.h` by ~5; `bng/bngl_names.h` by ~4. A few files include
  specific headers (`bng/species.h`, `bng/rxn_container.h`, `bng/cplx.h`,
  `bng/shared_defines.h`).

- **The one subclass:** `src4/simulation_config.h:28`
  `class SimulationConfig : public BNG::BNGConfig` — MCell extends `BNGConfig`
  with partitioning, RNG, cleanup-periodicity and memory-limit settings. This is
  the sole inheritance from a BNG class.

- **Engine ownership & accessors:** `src4/world.h:243` holds
  `BNG::BNGEngine bng_engine;`. `World::get_all_species()` and
  `World::get_all_rxns()` (`world.h:168-172`) forward to the engine. Partitions
  and events query through these.

- **Most-used BNG symbols** (approx. counts across MCell):

  | Symbol | uses | Symbol | uses |
  |---|---|---|---|
  | `BNG::Species` | ~62 | `BNG::compartment_id_t` | ~42 |
  | `BNG::BNGData` | ~36 | `BNG::COMPARTMENT_ID_NONE` | ~33 |
  | `BNG::RxnRule` | ~31 | `BNG::rxn_rule_id_t` | ~25 |
  | `BNG::Cplx` | ~23 | `BNG::RxnClass` | ~22 |
  | `BNG::Compartment` | ~20 | `BNG::RXN_RULE_ID_INVALID` | ~16 |
  | `BNG::ElemMolType` | ~12 | `BNG::RxnContainer` | ~10 |
  | `BNG::RxnType` | ~10 | `BNG::ElemMol` | ~9 |
  | `BNG::species_id_t` | ~9 | `BNG::SpeciesContainer` | ~8 |
  | `BNG::BNGEngine` | ~7 | `BNG::BNGConfig` | ~5 |

- **Key call surface:**
  - **Parsing:** `BNG::parse_bngl_file(file, bng_data, parameter_overrides)` —
    called from `libmcell/api/{subsystem,observables,instantiation,model,
    bngl_utils}.cpp`.
  - **Species lookups (hot path):** `world->get_all_species().get(species_id)`
    (~28 sites in diffusion/release/count events),
    `.get_species_vector()`, `.find_or_add(species)`.
  - **Reaction lookups:** `world->get_all_rxns()` →
    `get_rxn_rules_vector()`, `get_reactant_classes()`,
    `get_reacting_classes(species)`, `get_reactant_class(id)`,
    `find_rxn_rule_by_name(name)` (used by `mcell3_world_converter.cpp`),
    plus the unimol/bimol `RxnClass` getters used during reaction execution.
  - **Model construction / conversion** (`libmcell/api/bng_converter.cpp`,
    `mcell4_converter.cpp`): `bng_data.find_or_add_state_name`,
    `find_or_add_component_type`, `find_or_add_elem_mol_type`, `add_compartment`,
    `find_compartment`/`find_compartment_id`, `get_compartments`, `get_parameters`,
    `get_observables`, `get_seed_species`, `get_rxn_rules`, `get_elem_mol_types`;
    `BNG::process_compartments_and_set_orientations(...)`.

### Notable coupling / gotchas

1. **C++17 is mandatory** (propagated by `BNG::bng`); MCell must build at ≥ C++17.
2. **nauty must stay STATIC and compiled as C++** — a shared nauty breaks the
   `mcell.so` link (`undefined reference to updatecan`). Don't "modernize" this.
3. **Canonicalization is not thread-safe** (nauty globals, `cplx.cpp:550`). Any
   parallelization of species creation must serialize `Cplx::canonicalize()`.
   The header comment at `bng_engine.h:24-31` anticipates per-partition engines
   that would need synchronization.
4. **The parser is non-reentrant** (global `g_ctx`, `bnglparse()` globals). Only
   one BNGL parse at a time.
5. **MCell subclasses `BNGConfig`** — changes to `BNGConfig`'s ctor/members ripple
   into `SimulationConfig` (`src4/simulation_config.h`).
6. **Bundled Boost & sparsehash** live under `libbng/libs` and are added to MCell's
   include path; MCell relies on libbng's vendored Boost (graph/pool/
   dynamic_bitset). Watch for header collisions if MCell ever adds its own Boost.
7. **`BNGData` is the parse result and the conversion target** — both the BNGL
   path and MCell's programmatic API funnel into the same `find_or_add_*` methods,
   so id assignment/dedup semantics there are load-bearing.
8. Reaction classes and product species are generated **lazily**; reaction maps are
   updated on demand (`rxn_container.h:117-201`), and `RxnContainer` must be told
   when new species appear. `BNGEngine::initialize()` must be called if libbng is
   driven directly from `BNGData` output (`bng_engine.h:50-52`).
