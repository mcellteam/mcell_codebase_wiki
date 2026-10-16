# MCell Python API generation & build system (developer reference)

`module: mcell` (built target `mcell4_so`, `OUTPUT_NAME "mcell"`, `.so`/`.pyd`)
· project root: `$ROOT` (the build root, parent of this wiki folder; see `CLAUDE.md`)

This is the keystone document for **how MCell's Python API and the `mcell` binary
are generated and built**. MCell exposes a **nanobind** module `mcell.so` (migrated
from pybind11 — see `06_nanobind_migration_feasibility.md`). The bulk of
that API — C++ class declarations, their nanobind bindings, a `mcell.pyi` type
stub, and the RST reference docs — is **code-generated from YAML** by a standalone
Python generator. Hand-written C++ subclasses extend the generated base classes
where custom behavior is needed. The whole thing is compiled by CMake, which is
itself driven by orchestration scripts in `mcell_tools/`, against a
statically-built CPython 3.13 with a `uv` venv in `mcell_python/`.

The pipeline at a glance:

```
definition/*.yaml  ──gen.py──▶  generated/gen_*.{h,cpp}   (C++ Gen<Class> + nanobind)
                              ├▶ generated/gen_constants.{h,cpp}, gen_names.h
                              ├▶ generated/gen_vectors_{make_opaque.h,bind.cpp}
                              ├▶ generated/mcell.pyi        (Python type stub)
                              └▶ documentation/generated/*.rst (doc.py)

api/<class>.{h,cpp}  (hand-written  class X : public GenX  subclasses)
        +  api/exports.cpp  (NB_MODULE entry)
        ▼
CMake (mcell/CMakeLists.txt)  ──nanobind_add_module──▶  lib/mcell.so   (target mcell4_so)
                              ──add_executable──────▶  mcell           (CLI binary)

mcell_tools/run.py ▶ build.py ▶ {build VTK, cmake+make mcell, bundle}
```

---

# PART A — The YAML → C++/Python code generator

Everything in this part lives under
`$ROOT/mcell/libmcell/definition/`.

| File | Role |
|---|---|
| `gen.py` (2334 lines) | The generator. Reads all YAMLs, emits every generated file. |
| `constants.py` | All naming/string constants, the category list, type maps, file paths. |
| `doc.py` | RST documentation generator (called at the end of `gen.py`). |
| `*.yaml` | The API definition (schema described below). |

## 1. The YAML schema

### 1.1 Categories (input files & ordering)

`constants.py:14` lists the input files in `CATEGORIES`, which also fixes
documentation ordering:

```python
CATEGORIES = [ 'constants', 'model', 'simulation_setup', 'subsystem',
               'geometry', 'instantiation', 'observables', 'callbacks',
               'introspection', 'checkpointing', 'submodules' ]
```

`gen.py:2295` (`load_and_generate_data_classes`) loads `<cat>.yaml` for each
category, tags every class with its originating file (`value[KEY_CATEGORY] = cat`,
used by `doc.py` for grouping), and merges everything into one big
`data_classes` dict that is then handed to `generate_data_classes`.

What each file defines:

| File | Top-level entries |
|---|---|
| `model.yaml` | `Model` (`superclasses: [Subsystem, Instantiation, Observables, Introspection]`) |
| `subsystem.yaml` | `ComponentType`, `Component`, `ElementaryMoleculeType`, `ElementaryMolecule`, `Complex`, `Species`, `SurfaceProperty`, `SurfaceClass`, `ReactionRule`, `Subsystem` |
| `geometry.yaml` | `Color`, `Region`, `SurfaceRegion`, `GeometryObject` |
| `instantiation.yaml` | `ReleasePattern`, `MoleculeReleaseInfo`, `ReleaseSite`, `InitialSurfaceRelease`, `Instantiation` |
| `observables.yaml` | `VizOutput`, `CountTerm`, `Count`, `Observables` |
| `introspection.yaml` | `Molecule`, `Wall`, `WallWallHitInfo`, `Introspection` |
| `simulation_setup.yaml` | `Config`, `Notifications`, `Warnings` |
| `constants.yaml` | `constants:` (~25 entries) + `enums:` (14 enums: `Orientation`, `Notification`, `WarningLevel`, `VizMode`, `Shape`, `SurfacePropertyType`, `ExprNodeType`, `RegionNodeType`, `ReactionType`, `MoleculeType`, `BNGSimulationMethod`, `CountOutputFormat`, …) |
| `checkpointing.yaml` | `BaseChkptMol`, `ChkptVolMol`, `ChkptSurfMol`, `RngState` |
| `callbacks.yaml` | `MolWallHitInfo`, `ReactionInfo` (no superclass — POD-ish callback args) |
| `submodules.yaml` | `geometry_utils`, `bngl_utils`, `run_utils`, `data_utils` (`type: submodule`) |

### 1.2 A class definition

Top-level keys are class names. Under each (recognized in `gen.py` /
`constants.py:49-76`):

- `type:` — `class` (default) or `submodule`. Submodules become a Python
  module namespace of free functions, not a class.
- `superclass:` — single base class string: one of `BaseDataClass`,
  `BaseIntrospectionClass`, or another API class (e.g. `Species` → `Complex`,
  `SurfaceClass` → `SurfaceProperty`, `GeometryObject` → `Region`).
- `superclasses:` — a YAML list, for multiple inheritance. Only `Model` uses it.
- `doc:` — block-scalar documentation (literal `:` inside text is escaped `\:`).
- `examples:` — whitespace/newline-separated relative paths to example `.py`
  files (linked into the RST docs via `EXAMPLES_BASE_URL`, `constants.py:42`).
- `items:` — list of attributes (§1.3).
- `methods:` — list of methods (§1.4).
- Developer-only annotations (`internal`, `todo`, `TODO`, `old_doc`) are ignored
  by the generator.

The four "container" classes `Model`, `Subsystem`, `Instantiation`,
`Observables` are special-cased throughout (`is_container_class`, `gen.py:337`):
they derive from `BaseExportClass` rather than `BaseDataClass`.

### 1.3 An item (attribute) — `name`, `type`, `default`, `doc`, `examples`

The **type syntax** is interpreted by `yaml_type_to_cpp_type` (`gen.py:148`),
`yaml_type_to_py_type` (`gen.py:216`) and the predicates around them:

| YAML type | C++ type | Python (.pyi) |
|---|---|---|
| `float` | `double` | `float` |
| `str` | `std::string` | `str` |
| `int` | `int` | `int` |
| `uint32`/`uint64` | `uint`/`uint64_t` | `int` |
| `bool` | `bool` | `bool` |
| `Vec2`/`Vec3`/`IVec3` | `Vec2`/`Vec3`/`IVec3` | `Vec3`/… |
| `py::object` | `py::object` | `Any` |
| `ClassName*` (pointer) | `std::shared_ptr<ClassName>` | `ClassName` |
| `List[X]` | `std::vector<X>` | `List[X]` |
| `Dict[K,V]` | `std::map<K,V>` | `Dict[K,V]` |
| `std::function<…>` | (verbatim) | `Callable` |
| an enum name | the enum class | the enum |

`default:` semantics (`get_default_or_unset_value`, `gen.py:264`):
- **absent** → the attribute is a *required* constructor argument
  (`check_semantics` will throw if it is not set, `gen.py:874`).
- `default: unset` → the per-type sentinel: `FLT_UNSET`, `INT_UNSET`,
  `STR_UNSET`, `nullptr`, `Orientation::NOT_SET`, etc. (`constants.py:112-123`).
- `default: empty` → empty container (`std::vector<…>()` / `std::map<…>()`).
- a literal → numeric/bool/string/enum-member/ctor-call/named-constant, emitted
  with `.`→`::` rewriting for enums (`Orientation.DEFAULT` → `Orientation::DEFAULT`).

### 1.4 A method — under `methods:`

`name`, optional `return_type` (void if absent), optional `is_const: true`,
optional `params:` (each param has `name`/`type`/`default`/`doc`), and `doc`.
**Overloads** are expressed as multiple list entries with the same `name` but
different param types; the generator detects them (`is_overloaded`, `gen.py:1355`)
and emits `py::overload_cast<…>` casts in the nanobind binding (`py::` is an alias
for `nanobind`, so this resolves to `nb::overload_cast`).

### 1.5 Enums and constants (`constants.yaml`)

`enums:` — list of `{name, doc?, values:[{name, value:int, doc?}]}`.
`constants:` — list of `{name, type, value, doc?}` where `value` can be a literal,
a C macro (`INT32_MAX`, `FLT_MAX`), or a named C++ global (e.g. `AllMolecules`,
defined in `api/globals.cpp` — see §3).

### 1.6 Annotated example — `Species` (subsystem.yaml:298)

```yaml
Species:
  superclass: Complex                       # GenSpecies : public Complex
  doc: |                                     # → nanobind docstring + species.rst
    There are three ways how to use this class\: ...
  examples: tests/pymcell4_positive/0040_to_bngl_str/model.py
  items:
  - name: name                              # ctor arg, .def_property, .pyi field
    type: str
    default: unset                          # → STR_UNSET sentinel
    doc: | Name of the species in the BNGL format. ...
  - name: diffusion_constant_2d
    type: float
    default: unset                          # → FLT_UNSET
  - name: target_only
    type: bool
    default: False                          # → false
  methods:
  - name: inst                              # → .def("inst", &Species::inst, ...)
    return_type: Complex*                   # → std::shared_ptr<Complex>
    params:
    - name: orientation
      type: Orientation
      default: Orientation.DEFAULT          # → Orientation::DEFAULT
    - name: compartment_name
      type: str
      default: unset
```

This one class produces the files listed in §1.7. Because `Species` has
`superclass: Complex`, `GenSpecies` inherits `Complex`'s items (`name`,
`elementary_molecules`, `orientation`, `compartment_name`) — those are marked
`inherited` internally (`inherit_from_superclasses`, `gen.py:1651`) and passed to
the `Complex` base constructor (see the generated `SPECIES_CTOR()` macro below).

### 1.7 What each YAML class generates

For class `X` (`generate_class_files`, `gen.py:1693`):

| Output | Generated by | Contents |
|---|---|---|
| `generated/gen_x.h` | `generate_class_header` (`gen.py:789`) | `#define X_CTOR()` macro, `class GenX { … }` (attributes + virtual getters/setters + pure-virtual methods + `copy`/`deepcopy`/`__eq__`/`to_str` decls), forward decl `class X;`, and the `void define_pybinding_X(py::module_&)` declaration. |
| `generated/gen_x.cpp` | `generate_class_implementation_and_bindings` (`gen.py:1602`) | `GenX::check_semantics`, `set_initialized`, `set_all_attributes_as_default_or_unset`, `copy_x`/`deepcopy_x`, `__eq__`/`eq_nonarray_attributes`, `to_str`, the `define_pybinding_X` **nanobind binding** body, and `export_to_python` (for Python-script export). |
| `work/x.h` (template) | `generate_class_template` (`gen.py:843`) | A *starter* `class X : public GenX` for new classes — copied by hand into `api/` the first time, never overwritten there. |

The nanobind binding body (`write_pybind11_bindings`, `gen.py:1450` — the
generator function keeps its historical name) emits, for a data class:
`py::class_<X, Superclass>(m,"X", doc)` (the `py::` namespace is an alias for
`nanobind`; **no `shared_ptr<X>` holder** — nanobind infers it from
`stl/shared_ptr.h`) with a `py::init<…>()` matching the ctor args (names +
defaults), then `__copy__`/`__deepcopy__`/`__str__`/`__eq__`/`dump`/`check_semantics`,
then one `.def(...)` per declared method, then `.def_prop_rw("attr", &X::get_attr,
&X::set_attr)` per attribute (with `py::rv_policy::reference` for list attributes
so Python can mutate them in place). The function returns `void`
(`write_define_binding_decl`, `gen.py:704`) — nanobind's `class_<X, Base>` does
not slice to `class_<X>` the way pybind11's did, and the return value is unused.

nanobind-specific emission (all in `write_pybind11_bindings` / its helpers; details
in `06_*.md` §11.5):
- **`.none()`** is appended to every nullable (object/`py::object`) `py::arg`
  (`arg_none_suffix`, `gen.py:1404`) — nanobind rejects `None` for such args otherwise.
- Classes that derive (transitively) from a `std::enable_shared_from_this` base are
  constructed via a **`py::new_` make-shared factory** instead of `py::init`
  (`use_factory`, the `g_esft_classes` closure computed by `set_global_esft_classes`,
  `gen.py:1476-1505`) — otherwise `shared_from_this()` throws `bad_weak_ptr`. The
  closure is `Region`/`CountTerm`/`ComponentType`/`ElementaryMoleculeType` **plus**
  their subclasses `GeometryObject`/`SurfaceRegion`.
- enums are bound `py::enum_<E>(m,"E", py::is_arithmetic())` with **no
  `.export_values()`** (nanobind has none) — each enumerator is re-exported with an
  explicit `m.attr("NAME") = E::NAME` (`write_enum_binding`).
- bool attributes export to Python as `True`/`False` (not `1`/`0`) in
  `export_to_python` (nanobind rejects `int` for a `bool` arg).

### 1.8 Aggregate (non-per-class) outputs

| Output | Generated by | Contents |
|---|---|---|
| `generated/gen_constants.h` / `.cpp` | `generate_constants_and_enums` (`gen.py:1886`) | `const` definitions and `enum class` definitions (+ `operator<<`), and `define_pybinding_constants` / `define_pybinding_enums`. |
| `generated/gen_names.h` | `generate_names_header` (`gen.py:2111`) | `const char* const NAME_…`, `CLASS_…`, `ENUM_…`, `EV_…`, `CV_…` string constants for every class/attr/enum/value/constant — used by the hand-written converters. |
| `generated/gen_vectors_make_opaque.h` | `generate_vector_bindings` (`gen.py:1927`) | `NB_MAKE_OPAQUE(std::vector<…>)` for every `List[…]` type used (nanobind's analogue of `PYBIND11_MAKE_OPAQUE`; still required so `bind_vector` wins over the `stl/vector.h` caster). Included from `api_common.h:37` *before* any `std::vector` use so the containers are opaque (mutable from Python). |
| `generated/gen_vectors_bind.cpp` | same | `gen_vectors_bind(py::module_&)`: `py::bind_vector<…>(m,"VectorX")`. (nanobind's `bind_vector` accepts any Python sequence directly, so the pybind11 `implicitly_convertible` from `list`/`tuple`/numpy is no longer emitted; numpy interop is a remaining follow-up.) |
| `generated/mcell.pyi` | `generate_pyi_file` (`gen.py:2230`) | The **Python type stub**: forward `class X(): pass` decls, `Vec3`/`Vec2`, each enum as a `class(Enum)`, the constants, then per-class `__init__` signatures + method stubs (`generate_pyi_class`, `gen.py:2151`). Imported by IDEs / type-checkers for `import mcell`. |
| `documentation/generated/*.rst` + `api.rst` | `doc.py:generate_documentation` (`gen.py:2327` → `doc.py:135`) | One RST per category plus an `api.rst` toctree; attributes/methods/enums/constants rendered with their docs and example links. |

## 2. How `gen.py` runs & the regeneration command

`gen.py` is a **standalone script**, not wired into CMake or any `mcell_tools`
build script (confirmed: no `gen.py` reference anywhere in `mcell/` build files or
`mcell_tools/`). The generated files are checked into the source tree under
`generated/`; you re-run the generator by hand whenever you edit a YAML.

It uses **relative paths** (`constants.py:44-47`: `TARGET_DIRECTORY = '../generated'`,
`API_DIRECTORY = 'api'`, `WORK_DIRECTORY = '../work'`, `DOC_DIRECTORY =
'../documentation/generated'`), so it must be run **from the `definition/`
directory**:

```bash
cd "$ROOT/mcell/libmcell/definition"   # $ROOT = the build root (see CLAUDE.md)
python3 gen.py          # or:  python3 gen.py -v   (verbose)
```

Outputs land in `../generated/` (and `../work/` for the class templates,
`../documentation/generated/` for RST). `main` (`gen.py:2329`) just sets
`VERBOSE` from `-v` and calls `load_and_generate_data_classes`. Requires `pyyaml`
(provided by the `mcell_python` venv, §7). After regeneration, new generated
`.cpp` files must be added to `libmcell/CMakeLists.txt` `SOURCE_FILES`, the new
`define_pybinding_X` call added to `api/exports.cpp`, and any new class needs an
`api/x.{h,cpp}` (copied from the `work/x.h` template).

## 3. The hand-written ↔ generated split

The core pattern (see `api/species.h:27`):

```cpp
class Species: public GenSpecies {          // GenSpecies lives in generated/gen_species.h
public:
  SPECIES_CTOR()                            // ctor macro from gen_species.h
  // ... hand-written overrides ...
};
```

`GenX` (generated) holds the data members, getters/setters, equality, copy,
`to_str`, and `export_to_python`, and declares the API methods **pure virtual**
(`= 0`, `gen.py:587`). `X` (hand-written, in `api/`) supplies the implementations
and any custom validation. The generated `X_CTOR()` macro
(`gen_species.h:27-56`) is what wires it together — it sets `class_name`, assigns
each attribute, then calls `postprocess_in_ctor()` and `check_semantics()`:

```cpp
#define SPECIES_CTOR() \
    Species( const std::string& name_ = STR_UNSET, ... )
      : GenSpecies(name_,elementary_molecules_,orientation_,compartment_name_) {
        class_name = "Species"; name = name_; ...; postprocess_in_ctor(); check_semantics(); }
    Species(DefaultCtorArgType) : GenSpecies(DefaultCtorArgType()) { ... }
```

A developer **extends a generated class** by editing only `api/x.{h,cpp}`:
override the virtuals (`postprocess_in_ctor`, `check_semantics`, custom
`to_str`/`__eq__`, the declared methods), add helpers, etc. `api/species.h` is a
rich example — it overrides `set_name`, `postprocess_in_ctor`, `inst`,
`export_to_python`, and adds `check_no_extra_fields_are_set`. Regenerating never
touches `api/`.

Supporting hand-written infrastructure:

- **`api/api_common.h`** — included by every generated header (`INCLUDE_API_COMMON_H`).
  Pulls in nanobind (`<nanobind/nanobind.h>` + opt-in STL casters `stl/string.h`,
  `stl/shared_ptr.h`, `stl/vector.h`, `stl/map.h`, `stl/set.h`, `stl/function.h`,
  `stl/bind_vector.h`; `namespace py = nanobind;`) + `gen_vectors_make_opaque.h` first, defines the `is_set(...)`
  overloads (per-type "unset" checks), `ValueError`/`RuntimeError` typedefs, the
  `vec_*` string/equality helpers, the sentinels (`VEC3_UNSET`, `PTR_UNSET`), and
  the `DefaultCtorArgType` tag struct (`api_common.h:269`) that disambiguates the
  generated all-defaults ctor from the internal default ctor used for copies.
- **`api/globals.cpp`** — defines the global shared-pointer species superclasses
  `AllMolecules`/`AllVolumeMolecules`/`AllSurfaceMolecules` referenced as constants
  in `constants.yaml`.
- **`api/bindings.cpp`** — hand-written nanobind for the non-generated value types
  `Vec3`/`Vec2`/`IVec3` (`define_pybinding_Vec3` etc.), plus the `check_ctrl_c`
  signal helper. (Note: the `NB_MODULE` macro is *also* here at
  `bindings.cpp:176` — see §4.)
- **`api/exports.cpp`** — the module-entry shim compiled into the `.so` (§4).
- **`api/base_*_class.h`** — `BaseDataClass`, `BaseIntrospectionClass`,
  `BaseExportClass`: the roots that `GenX` derives from depending on `superclass`.
- **`api/python_export_*`, `api/python_exporter.*`, `api/mcell4_converter.*`,
  `api/bng_converter.*`** — hand-written converters that turn the API object tree
  into the internal simulation engine (`src4/`) and into exported Python scripts;
  they consume the `NAME_*` constants from `gen_names.h`.

## 4. nanobind module assembly (`mcell4_so`)

The module is assembled by **`nanobind_add_module(mcell4_so …)`**
(`mcell/CMakeLists.txt:512`; nanobind is found via `find_package(nanobind CONFIG)`
at `:374` against the vendored `libs/nanobind`). Crucially, the *only* libmcell-API
file listed in that target is `libmcell/api/exports.cpp` — all the actual bindings
come in by **linking the `libmcell` static library** (`target_link_libraries(mcell4_so
PUBLIC MCell::libmcell …)`, `CMakeLists.txt:578`), which compiles all the
`gen_*.cpp` binding functions and the hand-written `api/bindings.cpp`. Because
nanobind has a **static support library** (`nanobind-static`) where pybind11 was
header-only, `mcell4_so` also links `nanobind-static` *after* `libmcell`
(`CMakeLists.txt:584`) so the support symbols resolve under GNU ld's link order.

`api/exports.cpp` is a deliberate workaround (`exports.cpp:1`): it just declares
`extern "C" void PyInit_mcell();` and references it from a static initializer so
the linker keeps the symbol exported from the static lib into the shared module.
The real `NB_MODULE(mcell, m)` entry point lives in **`api/bindings.cpp:176`**,
which is where every `define_pybinding_*` is called in dependency order
(`bindings.cpp:176-242`):

```cpp
NB_MODULE(mcell, m) {
  m.attr("__version__") = MCELL_VERSION;   // nanobind casts const char* -> str
  gen_vectors_bind(m);            // opaque std::vector<…> types
  define_pybinding_enums(m);      // enums first (classes reference them)
  define_pybinding_Vec3(m); define_pybinding_Vec2(m); define_pybinding_IVec3(m);
  define_pybinding_ComponentType(m); ... define_pybinding_Model(m); ...   // every class
  define_pybinding_constants(m);  // constants last (may reference classes)
  define_pybinding_bngl_utils(m); ... define_pybinding_data_utils(m);     // submodules
}
```

`mcell4_so` properties (`CMakeLists.txt:588-594`): `PREFIX ""`,
`OUTPUT_NAME "mcell"`, `SUFFIX ".so"` (`.pyd` on Windows) — this explicit override
deliberately wins over the versioned suffix `nanobind_add_module` would otherwise
apply, so a plain `mcell.so`/`mcell.pyd` is produced — position-independent.
The result is `<build>/lib/mcell.so` (`CMAKE_LIBRARY_OUTPUT_DIRECTORY`,
`CMakeLists.txt:386`).

**Package layout for `import mcell`:** there is no Python *package* directory —
`mcell.so` is a single extension module placed directly on the path, accompanied
by the `mcell.pyi` stub. CMake `configure_file`s the checked-in stub
`libmcell/generated/mcell.pyi` → `<build>/utils/mcell.pyi` (`CMakeLists.txt:506`).
Users `import mcell` after adding the directory containing `mcell.so` to
`sys.path` (in the bundle, the mcell install dir; the `.pyi` rides alongside for
IDE/type-checker support).

---

# PART B — The build system

## 5. `mcell_tools/` — the build orchestrator

`mcell_tools/run.py` is the top-level driver; it puts `scripts/` on the path and
imports `repositories`, `build`, `bundle`, `cmake_builder`, `Options`,
`build_settings`. `main()` (`run.py:80`) phase sequence:

1. **Parse opts** — `Options().process_opts()` (`options.py`). Phase flags:
   `-1/--do-repos`, `-2/--do-build`, `-3/--do-bundle`, `-4/--do-test`
   (`options.py:128-159`). **Default when none given = repos + build only**
   (`options.py:226`). Other flags: `-d/--debug`, `-y/--mcell-python <ver>`,
   `-c/--clean`, `-b/--branch`, `-u/--update`, `-w/--only-pypi-wheel`,
   `-o/--only-cellblender-mcell`.
2. **Prerequisites** — `check_prerequisites` (`run.py:34`) hard-requires Python
   ≥ 3.13 and bootstraps a sufficiently-new cmake via
   `cmake_builder.build_cmake_if_version_is_insufficient` (downloads/builds
   cmake-3.15.5 if the system cmake < 3.25.1; constants at
   `build_settings.py:117-119`).
3. **Phase 1 — repos** — `repositories.get_or_update(opts)` (`run.py:110`).
4. **Phase 2 — build** — `repositories.create_version_file` then
   `build.build_all(opts)` (`run.py:115`).
5. **Phase 3 — bundle** — `bundle.create_bundle` + `extract_resulting_bundle`,
   or the cellblender-plugin / pypi-wheel variants (`run.py:127`).
6. **Phase 4 — test** — shells out to `mcell_tests/run_tests.py` (`run.py:144`).

`build.build_all` (`build.py:229`) order: **`build_vtk` first** (`build.py:232`),
then `build_mcell` (`build.py:234`), then cellblender, then mesh_tools.

## 5b. The CMake driver

There is no general cmake wrapper module; `scripts/cmake_builder.py` only
*bootstraps cmake itself*. The actual project cmake call is assembled inline in
`build.py:58` (`build_mcell`):

```python
cmd_cmake  = [opts.cmake_executable]
cmd_cmake += CMAKE_EXTRA_ARGS                                   # empty on all platforms
cmd_cmake.append(os.path.join(opts.top_dir, REPO_NAME_MCELL))  # source = ../mcell
cmd_cmake.append(get_cmake_build_type_arg(opts))               # -DCMAKE_BUILD_TYPE=Release|Debug
if BUILD_OPTS_USE_LTO: cmd_cmake.append('-DENABLE_LTO=ON')     # off by default
if opts.mcell_python:  cmd_cmake.append('-DPYTHON_VERSION=' + opts.mcell_python)
if os.name == 'nt':    cmd_cmake.append('-DSYSTEM_PYTHON_LIB_DIR=' + ...)  # Windows only
```

cmake runs *in* the build dir `work/build_mcell` (`run(cmd_cmake,
mcell_build_dir)`, `build.py:94`); the generator is cmake's default (no `-G`).
The build step is `make -j<N>` on non-Windows, `cmake --build . --target
ALL_BUILD --config Release -j N` on Windows (`build.py:17`). `-DPYTHON_VERSION` is
only passed when `--mcell-python` is given; otherwise the CMake default of `3.13`
applies (§7).

## 5c. Directory layout under `work/` & VTK handoff

`opts.work_dir = <root>/mcell_tools/work` (`options.py:61`). Path constants
(`build_settings.py:53-61`):

| Constant | Path | Purpose |
|---|---|---|
| `BUILD_DIR_VTK` | `work/build_vtk` | static VTK build |
| `BUILD_DIR_MCELL` | `work/build_mcell` | mcell/module build tree |
| `BUILD_DIR_MCELL_PYPI` | `work/build_mcell_pypi` | wheel-only build |
| `BUILD_DIR_CELLBLENDER` | `work/build_cellblender` | cellblender |
| `TEST_BUNDLE_DIR` | `work/bundle_install` | bundle re-extracted for tests |

`build_vtk` (`build.py:169`) builds VTK **static** (`-DBUILD_SHARED_LIBS=OFF`,
`-DVTK_BUILD_ALL_MODULES=OFF` + a curated module allowlist) into `work/build_vtk`.
The handoff is **not** via a cmake arg — the mcell `CMakeLists.txt:32` hardcodes
`set(VTK_DIR ${CMAKE_SOURCE_DIR}/../mcell_tools/work/build_vtk)` and then
`find_package(VTK REQUIRED COMPONENTS …)` consumes that prebuilt tree. This is why
VTK must be built before mcell.

## 5d. `repositories.py`

Base URL `https://github.com/mcellteam/` (or SSH for `*.salk.edu` hosts).
`BASE_REPOSITORIES` (`repositories.py:27`): `mcell`, `libbng`, `cellblender`,
`mcell_tests`, `mcell_tools`, `neuropil_tools`, `mesh_tools`, `VTK`.
`FORKED_REPOSITORIES`: `nfsim`, `nfsimCInterface`, `bionetgen`, `gamer`.
Default branch `master` (or `mcell4_dev` for `mcell4*` branches,
`build_settings.py:24`). **nanobind and jsoncpp are NOT separate repos** — they
are vendored inside `mcell/libs/` (§6). (pybind11 was likewise vendored before the
nanobind migration; `libs/pybind11` has since been removed entirely.)

## 5e. `scripts/bundle.py`

`create_bundle` (`bundle.py:189`) assembles a Blender + CellBlender tree under
`work/blender`: copies a prebuilt Blender-with-Python base, copies cellblender,
then copies the whole `work/build_mcell` tree into the mcell install subdir
(excluding `CMakeFiles`, `deps`, `*.a`, logs), installs neuropil/mesh tools,
optionally gamer, drops in the version file, and archives (`zip` on macOS/Windows,
`tar.gz` on Linux). `mcell.so` (from `build_mcell/lib/`) and `mcell.pyi` (from
`build_mcell/utils/`) ride along via that wholesale copy. The pypi-wheel path
(`pypi_wheel.py`) instead copies `build_mcell_pypi/lib/mcell.so` into
`pypi_wheel/` and runs `setup.py bdist_wheel`.

## 6. Top-level `mcell/CMakeLists.txt` build graph

`mcell/CMakeLists.txt` (project `mcell`) orchestrates the whole native build.
Build-orchestration view (dep internals are covered in the libbng/nfsim docs):

- **Python & VTK discovery** — `find_package(Python3 … EXACT)` (`:10`) and, for
  nanobind, a capital-P `find_package(Python … EXACT)` pinned to the same
  interpreter (`set(Python_EXECUTABLE ${Python3_EXECUTABLE})`, `:371-372`);
  `set(VTK_DIR …/work/build_vtk)` + `find_package(VTK …)` (`:32-52`);
  `find_package(nanobind CONFIG)` against the vendored `libs/nanobind` (`:374`).
- **Bundled subdirectories** via `add_subdirectory`:
  `libs/jsoncpp` (`:71`), sibling `../libbng` (`:376`), `../nfsim` (`:378`),
  `../nfsimCInterface` (`:379`), `libmcell` (`:381`), `src4` (`:382`). Plus
  `utils/data_model_to_pymcell` and `utils/` (`:574-575`). nanobind is **not** an
  `add_subdirectory` — it comes in via `find_package` above (`libs/pybind11` was
  removed in the migration). nanobind's include dirs are added to
  `mcell_project_options` (`:161`).
- **Shared flags** — a single `mcell_project_options` INTERFACE target
  (`:96-152`) carries the C++ standard (`MCELL_CXX_STANDARD`, default 17),
  warnings, `-fpermissive`, optimization, and `-DPYTHON_VERSION=…`; linked
  `PRIVATE` into every first-party + bundled target (`:574-583`).
- **Code-gen at configure time** — flex/bison for the MDL parser/scanner
  (`:312-331`), `version.h`/`version_flags.h` generation (`:288-349`).
- **Two binary targets:**
  - **`mcell`** executable (`add_executable`, `:533`) — the standalone MDL-based
    CLI, forced to the build root, links
    `MCell::mcell4 MCell::libmcell_dummy BNG::bng NFsimC NFsim JsonCpp Nauty
    ${VTK_LIBRARIES}` (`:556`). On **macOS** it does not link the static Python lib —
    Python C-API symbols are left undefined and resolved at runtime via
    `target_link_options(mcell PRIVATE …-undefined,dynamic_lookup)` (`:572`, the
    direct replacement for the old `pybind11::python_link_helper`). On **Windows**
    it (and `data_model_to_pymcell`) must additionally link `nanobind-static`
    (`:566`) — MSVC resolves every external when linking an executable.
  - **`mcell4_so`** module (`nanobind_add_module`, `:512`) — the Python `.so`,
    links `MCell::libmcell` (twice, for link-ordering, `:578-584`) + `nanobind-static`
    + `mcell4` + deps + VTK.
- **VTK autoinit** for all VTK-using targets (`vtk_module_autoinit`, `:602`).

## 7. Static Python 3.13 + `uv`

**CMake side** (`mcell/CMakeLists.txt:4-12`): if `PYTHON_VERSION` is empty it
defaults to **3.13**, derives `PYTHON_VERSION_NO_DOT` (`313`, used as the
`-DPYTHON_VERSION=313` compile define at `:117`), sets
`MCELL_PYTHON_VERSION` (renamed from `PYBIND11_PYTHON_VERSION` when pybind11 was
removed), and requires an **exact** interpreter+development match:

```cmake
find_package(Python3 ${MCELL_PYTHON_VERSION} EXACT REQUIRED COMPONENTS Interpreter Development)
```

(`libmcell/CMakeLists.txt:7` does the same.) `EXACT` means the build must find a
**3.13** interpreter *and* its development headers/libs — i.e. a fully-built
CPython 3.13, not just an interpreter. The orchestrator passes the version
through only when `--mcell-python` is given (`build.py:81`); otherwise the 3.13
default holds. Version constants live in `build_settings.py:90-109`
(`Python-3.13.9`, `python3.13`, `bin/python3.13`).

**The statically-built CPython 3.13** is a standalone interpreter that lives **outside `$ROOT`**,
wherever it was built. Unlike `$ROOT`, this path is **not a convention you set** (and nothing
asks you for it) — it is **recorded in the repo itself**. Read the real value on your machine:

```sh
cat mcell_python/.python-version
# → an absolute path like  <prefix>/Python-3.13.12/bin/python3.13
```

(`<prefix>` differs from machine to machine — below, it just stands in for whatever that file
actually contains. The pinned static binary is ~6.7 MB.) It supplies the `Python3::Python` / `Python3::Development` that
`find_package(Python3 3.13 EXACT)` resolves against and that the build links the
module to (via `nanobind_add_module`; on macOS the *executable* leaves these
undefined for runtime `dynamic_lookup`, §6).

**`mcell_python/` — the `uv` project/venv.** This is a small `uv` project whose
job is to provide a reproducible Python 3.13 environment for *running* MCell's
Python scripts (and for the codegen tooling, which needs `pyyaml`). Key files:

- `pyproject.toml` — `name = "mcell-python"`, `requires-python = ">=3.13"`, deps
  for the scientific/CellBlender stack (`numpy`, `scipy`, `pandas`, `matplotlib`,
  `lxml`, `pyparsing`, `pyyaml`, `python-libsbml`, `pyside6`, `distro`, `psutil`,
  `toml`, `setuptools`).
- `.python-version` — points uv at the static interpreter above.
- `uv.lock` — the locked dependency set.
- `.venv/pyvenv.cfg` — `home = <prefix>/Python-3.13.12/bin`,
  `version_info = 3.13.12`, `uv = 0.11.7` → confirms the venv was created by `uv`
  from the statically-built 3.13 (the `home` path is the same machine-specific one read
  from `.python-version` above — not something you configure).

So: the static CPython 3.13 is the one CMake links the `mcell.so` against; the
`mcell_python` uv venv (built from that same interpreter) is the runtime
environment that `import mcell` and the model scripts run inside, with all the
required third-party packages.

**Activating the venv.** Each build root has its own venv in `mcell_python/.venv`.
Activate it from `$ROOT` with

```sh
source mcell_python/.venv/bin/activate
```

— the same command on every machine, and always the venv of the root you are in, so
the wiki never hardcodes an interpreter path. After activating, `python3` is the venv
interpreter — e.g. pass it to the test runner as `run_tests.py -t "$(command -v python3)"`.
Being a plain `source`, it also works in non-interactive shells (scripts, agents):
`source "$ROOT/mcell_python/.venv/bin/activate" && …`. (A maintainer's personal shell
alias such as `mp` may point at another root's venv — check before relying on it.)

## 8. MDLr → MDL Python tooling & utils (pointers)

Two families of Python tooling are copied into the build tree (not generated):

- **`mcell/src/rules_py/*.py`** → `configure_file`'d into `<build>/`
  (`CMakeLists.txt:491-503`): `grammar_definition.py`, `mdlr2mdl.py`,
  `mcell3r.py`, `nfsim_python.py`, `read_mdl.py`, `write_mdl.py`,
  `read_bngxml.py`/`split_bngxml.py`/`write_bngxmle.py`, `small_structures.py`,
  `postprocess_mcell3r.py`. These convert **MDLr** (rule-based MDL) into plain MDL
  + NFsim XML for the rule-based simulation path.
- **`mcell/utils/`** — a C++ + Python toolbox; the build adds
  `utils/data_model_to_pymcell` (the `data_model_to_pymcell` target:
  `mcell4_generator.cpp`, `python_generator.cpp`, `bngl_generator.cpp` — converts a
  CellBlender **data model** JSON into a pymcell4 Python script) and `utils/`
  itself (`mcell4_runner`, `bng_analyzer`, plotting/visualization helpers). `bng2/`
  (BioNetGen) is copied to `<build>/bng2/` (`CMakeLists.txt:510`).
  - **The generated `model.py` guards its run block.** `MCell4Generator::generate_model`
    (`mcell/utils/data_model_to_pymcell/mcell4_generator.cpp:1245`) wraps the
    "initialization and execution" section in `if __name__ != '__mp_main__':` (`:1341`).
    That section is the `custom_init_and_run` hook, or `initialize()` → checkpoint
    scheduling → `run_iterations()` → `end_simulation()`. The guard exists so that
    `customization.py` can couple MCell to an external simulator (e.g. NEURON) through
    `multiprocessing`. A child started with the `spawn` method (the default on macOS and
    Windows) re-imports the main script as `__mp_main__`. Without the guard, the child
    re-executed that section: it re-ran the simulation, or failed with multiprocessing's
    "bootstrapping phase" `RuntimeError` when the section itself starts processes. Running
    the script directly (`__main__`) or importing it under any other name (e.g. `model`)
    still runs it. Landed as `98980098` on `mcell4_dev` and cherry-picked to
    `mcell4_nanobind` as `4587bf63`; the generator code is identical on both branches.
  - **Checkpoint resume works with the guard.** Tested on both branches (2026-10-15): a run
    saves a checkpoint and stops, and the next run resumes from it and finishes.
  - **One gap: a worker started *after* a checkpoint was written.** The code that checks for
    a checkpoint and resumes from it is emitted above the guard (`mcell4_generator.cpp:1295`,
    text from `get_resume_from_checkpoint_code()`,
    `mcell/libmcell/api/python_export_constants.cpp:53`), so a `spawn` worker runs it too. A
    worker started before any checkpoint exists, e.g. at the beginning of
    `custom_init_and_run`, finds none and does its task. A worker started once a checkpoint
    exists resumes the simulation from it instead: in the test it re-ran iterations 5 to 10
    alongside the parent and exited 0 without doing its task. Same on both branches.

These are referenced here at pointer level; they are downstream consumers of the
generated API, not part of the codegen/module-build path.

---

## Quick reference

**Regenerate the API from YAML:**
```bash
cd mcell/libmcell/definition && python3 gen.py        # writes ../generated/*, ../work/*, ../documentation/generated/*
```
**Build everything (repos + VTK + mcell + module):**
```bash
cd mcell_tools && python3 run.py                      # default = repos + build
python3 run.py -2 -d                                  # build only, Debug
```
**Direct cmake build of just the module:**
```bash
mkdir -p mcell_tools/work/build_mcell && cd mcell_tools/work/build_mcell
cmake ../../../mcell -DCMAKE_BUILD_TYPE=Release && make -j mcell4_so   # → lib/mcell.so
```
**Key artifacts:** `work/build_mcell/lib/mcell.so` · `work/build_mcell/utils/mcell.pyi`
· `work/build_mcell/mcell` (CLI).
