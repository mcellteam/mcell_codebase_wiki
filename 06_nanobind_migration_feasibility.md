# 06 — Feasibility: migrating the Python bindings from pybind11 → nanobind

> **Status:** feasibility study → validation spike → **executed full port** on branch
> `mcell4_nanobind`, now **validated on all three platforms — Linux, macOS, and Windows/MSVC —
> with `libs/pybind11` removed entirely** (Linux/macOS at commit `06837640`, Windows at `c8d49386`).
> `gen.py` was rewritten to emit nanobind, nanobind vendored at
> `mcell/libs/nanobind`, CMake switched to `nanobind_add_module`, and **`mcell.so`/`mcell.pyd`
> compiles, links, imports, and passes the full `tests/pymcell4_positive` suite (82/82 on Linux;
> 82 pass + 2 pre-existing reference-data diffs on macOS and Windows)** — incl. reaction callbacks,
> checkpoint save/restore, and the risk-3 `enable_shared_from_this` paths at runtime. §8 risks were
> first verified with a standalone
> A/B harness (pybind11 2.9.2 vs nanobind 2.13.0) — **spike results in §11**, **build-time findings
> from the real port in §11.5**, **the executed macOS step (executable link rework + pybind11
> removal) in §11.7**, **the executed Windows step (`/bigobj` + nanobind-static on the exe/utils)
> in §11.8.**
> **Scope:** the `mcell.so` / `mcell4_so` Python module only. Does not touch the MCell3
> `src4/` engine. (The MCell *executable* is touched only for its link line — the Darwin
> `dynamic_lookup` rework, §11.7, and the Windows `nanobind-static` link, §11.8 — not its engine code.)
> **Remaining:** none — all three platforms build and validate with pybind11 fully removed.
> **Paths are relative to `$ROOT`** (the build root, the parent of this wiki folder; see `CLAUDE.md`).

---

## 1. Verdict (executive summary)

**Migration is feasible, and the multiple-inheritance objection does not actually apply to
MCell.** The one place MCell uses C++ multiple inheritance among Python-bound types (`Model`) is
already registered to Python as a *flat, base-less* class — exactly the shape nanobind needs.
Every other bound class uses **single inheritance**, which nanobind fully supports.

The realistic payoff is **developer ergonomics, not simulation speed**:

| Dimension | Expected outcome for MCell |
|---|---|
| **Multiple inheritance blocker** | ❌ Not a blocker — MCell exposes *zero* multi-base classes to Python |
| **Compile time / binary size** | ✅ Genuine win (nanobind ~4× compile, ~5× binary; no LTO needed) |
| **Per-instance memory** | ✅ Real win (24 B vs 56 B wrapper overhead) |
| **Simulation runtime** | ⚠️ Negligible — MCell's hot loop is pure C++ in `src4/`, it does not cross the binding |
| **Codegen simplicity** | ✅ Win — drop holder boilerplate, optionally delegate `.pyi` to nanobind stubgen |
| **Migration cost** | 🟡 Moderate, but *highly leveraged* — ~95% of the change is in one file (`gen.py`). Plan **three** mechanical sweeps (§11), not two |
| **Main real risks** | All verified in the spike (§11): `bind_vector` copy-on-`__getitem__` (narrow); `enum.export_values`; `enable_shared_from_this` (bites — has a fix); **plus** `.none()` on nullable args; `NB_MAKE_OPAQUE` for `bind_vector` |

Recommendation: a **1–2 day spike** (§8), not a blind commit. The leverage from code generation
makes this an unusually cheap migration to *attempt*, and the risks are concentrated in 3
verifiable behaviors.

---

## 2. The multiple-inheritance question — definitive answer

**nanobind's actual restriction** (porting guide): *"Multiple inheritance … is one of the main
casualties in creating nanobind."* Concretely, nanobind will not let you register a type with
**two or more Python-bound base classes** (`nb::class_<Derived, Base1, Base2>`), because it
co-locates the C++ instance with the Python object and assumes a single fixed pointer offset —
which breaks when a `Derived*` must be pointer-adjusted to a secondary base.

**What MCell actually does:** exactly **one** C++ class has multiple inheritance among bound
types — `GenModel`:

```cpp
// mcell/libmcell/generated/gen_model.h:55
class GenModel: public Subsystem, public Instantiation, public Observables, public Introspection {
```
driven by `definition/model.yaml:2`: `superclasses: [Subsystem, Instantiation, Observables, Introspection]`.

**But it is never exposed to Python as multiply-derived.** The generator emits it flat:

```cpp
// mcell/libmcell/generated/gen_model.cpp:149
return py::class_<Model, std::shared_ptr<Model>>(m, "Model", ...)   // ← NO bound bases
```

Because `Model` uses `superclasses` (plural) rather than `superclass`, `gen.py`'s
`has_single_superclass()` (`definition/gen.py:600`) returns false, which causes the generator to
(a) omit all bases from the `py::class_<>` declaration (`gen.py:1440–1442`, `1449–1450`) and
(b) **re-bind every inherited method and property directly onto `Model`** (`gen.py:1510`, `1519`).
So on the Python side `Model` is a single, self-contained class with ~70 of its own methods —
there is no Python-level MI to translate.

The decisive point: **nanobind's binding shape for `Model` is identical to pybind11's** —
`nb::class_<Model>` with all methods re-bound. nanobind never adjusts a `Model*` to a
`Subsystem*`/`Observables*`; the C++ compiler handles `this`-adjustment internally when an
inherited member `&Model::method` is called. That is plain C++, invisible to the binding layer.

**The other "multiple inheritance" cases are not bound MI either:**

- `ComponentType`, `Region`, `CountTerm` each inherit `public Gen…, public std::enable_shared_from_this<…>`
  (`api/component_type.h:22`, and the analogous `api/region.h`, `api/count_term.h`). The second
  base is a non-bound standard-library helper — also registered flat in Python
  (`py::class_<ComponentType, std::shared_ptr<ComponentType>>`, `generated/gen_component_type.cpp:77`).

**Every genuinely inherited Python class uses a single bound base** — natively supported by nanobind:

- `py::class_<Species, Complex, …>` (`generated/gen_species.cpp:127`)
- `py::class_<GeometryObject, Region, …>` (`generated/gen_geometry_object.cpp:245`)
- `py::class_<SurfaceRegion, Region, …>` (`generated/gen_surface_region.cpp:214`)
- `py::class_<SurfaceClass, SurfaceProperty, …>` (`generated/gen_surface_class.cpp:122`)
- `py::class_<ChkptSurfMol, BaseChkptMol, …>` (`generated/gen_chkpt_surf_mol.cpp:205`),
  `py::class_<ChkptVolMol, BaseChkptMol, …>` (`generated/gen_chkpt_vol_mol.cpp:145`)

> **Conclusion:** MCell exposes **no multiple inheritance to Python**. nanobind's MI limitation is
> irrelevant here. (Spike item: confirm nanobind accepts a *flat-registered* C++ type that
> internally uses MI — very likely yes, since no base-pointer casting occurs for `Model`.)

---

## 3. How MCell uses pybind11 today (the migration surface)

The bindings are **almost entirely code-generated**, the single biggest factor making this cheap.

- **Vendored pybind11 2.9.2** (`mcell/libs/pybind11/include/pybind11/detail/common.h:3-5`); wired
  via `add_subdirectory` (`mcell/CMakeLists.txt:361`) and `pybind11_add_module(mcell4_so …)`
  (`CMakeLists.txt:498-512`), with `pybind11::python_link_helper` on macOS (`:546`) and
  `PYBIND11_LTO_CXX_FLAGS` handling (`:353`).
- **~74 `py::class_`**, **~360 `.def`**, **~205 `.def_property`**, **~12 `py::enum_`**,
  **~23 `py::bind_vector`** — all from templates in `definition/gen.py` (class binding
  `gen.py:1449–1532`; enums `1763–1799`; vectors `1859–1954`).
- **Universal holder = `std::shared_ptr<T>`** everywhere (`gen.py:72-73`); no `smart_holder`,
  no custom holder macro.
- **STL/vector strategy:** `PYBIND11_MAKE_OPAQUE` per vector (`gen.py:1902`) + `py::bind_vector`
  (`:1941`) + `py::implicitly_convertible` from `list`/`tuple`/numpy array (`:1942-1949`), plus
  `py::return_value_policy::reference` on list-typed properties (`gen.py:1524`).
- **Hand-written binding code (the only non-generated parts):** `api/bindings.cpp`
  (`PYBIND11_MODULE(mcell,m)` at `:180`; hand-bound `Vec3`/`Vec2` with operator lambdas and
  `def_readwrite` at `:86-177`), `api/exports.cpp` (entry for `mcell4_so`), and `gen_vectors_bind`.
- **Callbacks are `std::function<… , py::object>`** (`api/callbacks.h:6-12`) invoked under
  `py::gil_scoped_acquire` (`api/callbacks.cpp:128`); **no trampolines / `PYBIND11_OVERRIDE`** —
  Python never subclasses C++ types.
- Minor: `py::overload_cast` (2 uses, `ComponentType`), `py::gil_scoped_release` around `Model.run`.

This is a **conservative, modern subset** of pybind11 (no custom casters, trampolines,
metaclasses, or module-local) — and that subset maps cleanly onto nanobind.

---

## 4. Feature-by-feature mapping (what `gen.py` would emit instead)

| pybind11 (today) | nanobind | Notes |
|---|---|---|
| `#include <pybind11/pybind11.h>`, `stl.h` | `<nanobind/nanobind.h>` + **opt-in** `stl/shared_ptr.h`, `stl/vector.h`, `stl/string.h`, `stl/function.h` | nanobind requires explicit STL caster includes |
| `PYBIND11_MODULE(mcell, m)` | `NB_MODULE(mcell, m)` | mechanical |
| `py::class_<X, Base, std::shared_ptr<X>>` | `nb::class_<X, Base>` | **drop the holder** — shared_ptr inferred from header |
| `py::init<…>()` / `py::arg("x")=d` | `nb::init<…>()` / `nb::arg("x")=d` | identical |
| `.def_property` / `.def_readwrite` | `.def_prop_rw` / `.def_rw` | rename |
| `py::return_value_policy::reference` | `nb::rv_policy::reference` | rename |
| `py::enum_<E>(m,"E", py::arithmetic())` … `.export_values()` | `nb::enum_<E>(m,"E", nb::is_arithmetic())` + manual `m.attr("NAME")=E::NAME` per value | ✅ **no `export_values()` in nanobind** — `gen.py` must emit the `m.attr` lines (§11) |
| `py::bind_vector<V>` + `PYBIND11_MAKE_OPAQUE` | `nb::bind_vector<V>` + **`NB_MAKE_OPAQUE(V)`** | ⚠️ **NOT "opaque automatically"** — conflicts with `stl/vector.h` caster without `NB_MAKE_OPAQUE`; `__getitem__` still copies (§11) |
| `py::implicitly_convertible<list,V>` | `bind_vector` accepts sequences; `nb::init_implicit<>` for class conversions | numpy paths move to `nb::ndarray` |
| arg may receive `None` (e.g. `context`, any `=None` default) — implicit in pybind11 | **`nb::arg("x").none()`** (and `.none() = nb::none()` for the default) | ⚠️ **new, pervasive** — nanobind rejects `None` for object/pointer args otherwise; MCell defaults *many* args to `None` (§11) |
| `py::array_t<double>` conversions | `nb::ndarray<>` | small rework for Vec3/numpy interop |
| `py::overload_cast<…>` | `nb::overload_cast<…>` | identical |
| `py::gil_scoped_acquire/release` | `nb::gil_scoped_acquire/release` | identical |
| `std::function<…, py::object>` callbacks | `std::function<…, nb::object>` + `stl/function.h` | mechanical |
| hand-written `mcell.pyi` generation in `gen.py` | optional: nanobind **built-in stubgen** | net simplification if adopted |

**The leverage:** all "mechanical/rename" rows live in `gen.py`'s template strings plus 3
hand-written files. Edit once, run `cd mcell/libmcell/definition && python3 gen.py`, and all 74
classes regenerate.

**Environment compatibility:** nanobind needs **C++17** (MCell already forces `MCELL_CXX_STANDARD=17`)
and **Python ≥ 3.9** (MCell pins 3.13) and **GCC 8+** (the existing gcc-8.3.0 hard-block is an
NFsim concern and unrelated to `mcell.so`, which does not link NFsim). All satisfied.

---

## 5. Performance analysis — realistic for MCell

nanobind's headline claims: **~4× faster compile, ~5× smaller binaries, ~10× lower runtime call
overhead**, and per-instance wrapper overhead **24 B vs 56 B**. How much lands for MCell:

- **Compile time & binary size — real win.** 74 generated `gen_*.cpp` TUs full of heavy pybind11
  templates are exactly nanobind's target; pybind11 also needs LTO to keep `mcell.so` small
  (`CMakeLists.txt:353`), which nanobind avoids via its precompiled support library. Directly
  improves the build loop.
- **Per-instance memory — modest win.** Helps models that build many API objects
  (`ReleaseSite`/`Species`/`Complex`/`Component`); setup-time, not per-timestep.
- **Simulation runtime — essentially no change (key honesty point).** MCell's compute loop is
  entirely in the C++ MCell4 engine (`src4/`); `Model.run()` is a *single* crossing that runs the
  whole simulation in C++. The binding is exercised at model build, introspection, and
  **callbacks**. Callbacks (`register_mol_wall_hit_callback`, `register_reaction_callback`) are the
  only per-event crossings, and are dominated by **GIL acquisition + the user's Python callback
  body**, which nanobind's faster dispatch does not remove. Do **not** expect faster simulations.

**Net:** better build & lighter objects, not faster science.

---

## 6. Simplicity / maintainability analysis

Wins:
1. **Less per-class boilerplate** — dropping `, std::shared_ptr<X>` and `PYBIND11_MAKE_OPAQUE`
   shrinks the generator templates.
2. **Stub generation could be delegated.** `gen.py` hand-emits `mcell.pyi` today; nanobind ships a
   stub generator. Adopting it removes generator code and reduces binding↔stub drift (optional).
3. **Potential build-coupling relief (conditional).** A standing wiki gotcha is
   `find_package(Python3 3.13 EXACT)` — `mcell.so` is ABI-bound to the exact static 3.13. nanobind
   supports the **CPython stable ABI (abi3)** and **free-threaded Python**, which *could* decouple
   the module from one interpreter and future-proof for no-GIL. **Caveat:** MCell ships its own
   statically-built Python + link helper, so the practical benefit is smaller than for a typical
   wheel — treat as "investigate," not a headline.

Cost: nanobind is a **smaller, stricter** library; you permanently give up pybind11-only escape
hatches. Given MCell already uses only a conservative subset, this cost is low.

---

## 7. Migration plan & effort

Concentrated and mechanical, thanks to code generation:

1. **Add nanobind** (vendored sibling/submodule); swap `pybind11_add_module` →
   `nanobind_add_module` in `mcell/CMakeLists.txt` (and adjust the macOS link helper).
2. **Rewrite `gen.py` template strings** per §4 (bulk of the work — but one file). The spike (§11)
   shows this is **three** mechanical sweeps: (a) the §4 renames; (b) `.none()` on every nullable /
   `=None`-defaulted arg; (c) `nb::new_(make_shared)` factory ctors for the `enable_shared_from_this`
   classes. Plus `gen_vectors_make_opaque.h` swaps `PYBIND11_MAKE_OPAQUE`→`NB_MAKE_OPAQUE`, and the
   enum generator emits `m.attr("NAME")=E::NAME` lines.
3. **Port the 3 hand-written files:** `api/bindings.cpp` (module macro, `Vec3`/`Vec2`, numpy
   interop), `api/exports.cpp`, `gen_vectors_bind`.
4. **Port callbacks:** `py::object` → `nb::object`, include `stl/function.h`.
5. **Regenerate** (`python3 gen.py`) and build.
6. **Run the pymcell4 test suite** (models under `tests/pymcell4/…` referenced in YAML `examples:`)
   and diff outputs.

Rough effort: **a few days** for someone fluent in the codegen, dominated by the 3 risk areas in
§8, not by the mechanical rewrite.

---

## 8. Risks & validation spike checklist (ordered by risk)

*(All items 1–4 were exercised by the spike — see §11 for the harness and verdicts.)*

1. **`bind_vector` copy-on-`__getitem__` (highest risk).** pybind11 here uses `MAKE_OPAQUE` +
   `return_value_policy::reference` so Python can mutate list elements **in place** with write-back.
   nanobind's `bind_vector` **copies** on `__getitem__` by default; in-place element mutation would
   silently not persist. Audit whether any workflow mutates an element fetched from a bound list
   property (vs reassigning the whole list); if so, use `rv_policy::reference_internal` (which
   nanobind warns is footgun-prone). **Most likely to bite.**
   → **Spike result: real but NARROW.** Of MCell's three list-element kinds, only nested
   `List[List[float/int]]` breaks: `m[i][j]=v` (in-place inner mutation) is lost in nanobind, kept
   in pybind11. `List[X*]` (`vector<shared_ptr>`) field mutation and scalar-list `__setitem__`
   **both persist in both** (pointer element / write-through `__setitem__`).
   → **Usage audit (done): MOOT for MCell.** The only nested-list properties are
   `GeometryObject.vertex_list` / `.wall_list`, `Wall.vertices`, `ReactionRule.variable_rate`
   (`load_dat_file`'s return is a copy). A full sweep of the in-tree Python corpus
   (`mcell_tests`, `mcell`, `mcell_tools`, `cellblender`; ~2100 files) found **zero** in-place
   nested mutations — every access is whole-list assignment at construction, `len(prop)`, or a
   nested-index **read** (which nanobind serves correctly from the copy). Geometry is mutated via
   API methods (`add_vertex_move`/`apply_vertex_moves`, `set_wall_color`), never by writing into the
   bound list. **No `rv_policy::reference_internal` or special handling needed.**
   → **Follow-up (2026-10-28): lifetime was missed.** The audit above covered mutation only.
   `reference` hands Python a view of the list with no tie to the object that owns it, so a view
   taken from a temporary owner outlives it: `model.get_molecule(i).pos3d` read freed memory
   (garbage, or another molecule's position). Fixed in `e1e5ded2` by emitting
   `rv_policy::reference_internal`, which is also nanobind's own default for `def_prop_rw` getters
   (`mcell/libs/nanobind/include/nanobind/nb_class.h:701`): in-place mutation still works and the
   view keeps its owner alive. See doc 05 §1.7. The pybind11 build on `mcell4_dev` had the same
   bug and got the same fix as `1d47c148` (see §12).
2. **`enum … export_values()`.** MCell relies on exported enum values and module-level constants
   (`STATE_UNSET`, `BOND_UNBOUND`, orientation enums, …). Confirm the nanobind enum equivalent
   reproduces the exact Python-visible names the API and `.pyi` promise.
   → **Spike result: real, trivial.** nanobind has **no `.export_values()`**. Emitting
   `m.attr("NAME") = E::NAME` per enumerator reproduces the bare module-level names; `int(E::UP)`,
   arithmetic, and module constants then match pybind11 exactly.
3. **`std::enable_shared_from_this` classes** (`ComponentType`, `Region`, `CountTerm`). nanobind's
   support is "limited" — wants these constructed via `make_shared` and passed as `shared_ptr`.
   Verify their construction/return paths across the boundary.
   → **Spike result: real, BITES.** A naive `nb::init` port throws `bad_weak_ptr` from
   `shared_from_this()` on a Python-constructed instance (nanobind owns it directly, not via a
   `shared_ptr`). **Fix (verified):** bind the constructor as a factory —
   `.def(nb::new_([](…){ return std::make_shared<T>(…); }))` — so nanobind adopts a real
   `shared_ptr` and primes the weak_ptr.
4. **Flat-registered MI `Model`.** Confirm nanobind accepts `nb::class_<Model>` where C++ `Model`
   internally inherits 4 bases (expected yes — no base casting occurs).
   → **Spike result: confirmed non-issue.** `nb::class_<Model>` with inherited methods re-bound
   flat behaves identically to pybind11; the C++ bases stay invisible to Python.
5. **numpy ↔ `Vec3`/vector conversions** via `nb::ndarray` replacing `py::array_t`. *(Deferred in
   the spike — Vec3 value class, operators, and `def_rw` all verified; the `py::array_t`→`nb::ndarray`
   rework is still TODO in the real port.)*
6. **shared_ptr crossing overhead** — nanobind allocates a control block per crossing; confirm this
   only happens at setup (it should), not in a callback hot path. *(Not measured; setup-only by
   design — callbacks were verified functionally, not benchmarked.)*

---

## 9. Recommendation

Run a **time-boxed spike** rather than committing. Port a *vertical slice* — `Vec3`, one value
class (`Color`), one single-inheritance chain (`Complex`→`Species`), the flat-MI `Model`, one
enum, one `bind_vector` property, and one callback — and run the relevant `tests/pymcell4` cases.
That slice exercises every §8 risk in ~1–2 days. If items 1–3 behave, the rest is mechanical
template editing in `gen.py`.

The multiple-inheritance fear that motivated the study is a non-issue for MCell. The honest trade:
**meaningfully better build/dev ergonomics and a simpler generator, with no expected simulation
speedup** — gated on the three concrete behavioral checks above.

---

## 10. Cross-platform considerations (Linux / macOS Intel + Apple Silicon / Windows)

> **Executed status:** Linux ✅, macOS (Apple Silicon) ✅, **and Windows/MSVC ✅ are all done.**
> See **§11.7** for the macOS step (confirms §10.1's `python_link_helper` replacement, keeps the
> plain `.so` suffix per §10.3 option (a), exercises §10.2's arm64 path) and **§11.8** for the
> Windows step (confirms the §10.3 `.pyd` suffix override wins over `nanobind_add_module`'s
> versioned suffix, and resolves §10.4 — `/bigobj` added as a safeguard, plus a Windows-only
> `nanobind-static` link on the executable and `data_model_to_pymcell`). The subsections below are
> the original analysis; read them with §11.7–§11.8 for what actually landed.

The spike can be developed on **Debian Linux**, but the CMake and codegen changes must stay
correct for **Linux, macOS (Intel x86_64 and Apple Silicon arm64), and Windows/MSVC**. Good news:
nanobind *centralizes* most of the per-OS binding plumbing that MCell currently spells out by hand,
so this is mostly a simplification — provided we let `nanobind_add_module` own the binding-specific
flags and keep MCell's legacy-code flags (`-fpermissive` / `/Zc:strictStrings-`) untouched.

### 10.1 What `nanobind_add_module` handles automatically (per platform)
- **macOS undefined-symbol linking** — nanobind's link options set the Apple "undefined →
  dynamic_lookup" behavior for Python C-API symbols automatically. This **replaces** MCell's
  current `pybind11::python_link_helper` and the Darwin/non-Darwin Python-link split
  (`mcell/CMakeLists.txt:535-546`). Net removal of fragile platform code.
- **Extension suffix** — assigned from CMake `FindPython` (`.cpython-3xx-darwin.so`,
  `…-x86_64-linux-gnu.so`, `….pyd` on Windows). ⚠️ See §10.3 — MCell currently *forces* a plain
  `mcell.so` / `mcell.pyd`.
- **Size optimization + symbol stripping** in release builds (`-Os`, strip on Linux/macOS),
  replacing MCell's manual `-flto` plumbing (`mcell/CMakeLists.txt:188-195`) and the
  `PYBIND11_LTO_CXX_FLAGS=""` centos7 workaround (`:353`). LTO becomes an opt-in `LTO` flag.
- **Compiler floor:** C++17, GCC 8+/Clang 8+/MSVC 2019+ on all three OSes — already satisfied.
  MCell's `cmake_minimum_required(VERSION 3.25…)` (`CMakeLists.txt:1`) comfortably exceeds
  nanobind's CMake ≥ 3.15 (FindPython) requirement.

### 10.2 Apple Silicon vs Intel macOS
- The vendored **pybind11 2.9.2 dates from 2022** and predates mature arm64 support; **nanobind is
  current** and handles arm64, x86_64, and `universal2` cleanly — a net win for Apple Silicon.
- For a fat binary, set `CMAKE_OSX_ARCHITECTURES="x86_64;arm64"` (MCell sets no arch today).
- **The real macOS constraint is unchanged by this migration:** MCell ships its *own statically
  built Python*, and `find_package(Python3 3.13 EXACT)` couples the module to that interpreter's
  arch/ABI. The static Python must match the target arch (or be universal). nanobind doesn't change
  that coupling — but **STABLE_ABI** (§10.5) can loosen the *version* half of it.

### 10.3 Extension filename — a concrete tooling touch-point
MCell deliberately forces a plain module name today:
```cmake
# mcell/CMakeLists.txt:567-570
if (WIN32) set_target_properties(mcell4_so PROPERTIES SUFFIX ".pyd")
else()     set_target_properties(mcell4_so PROPERTIES SUFFIX ".so")  # plain .so even on macOS
```
`nanobind_add_module` instead emits the versioned suffix (e.g. `mcell.cpython-313-darwin.so`),
which Python still imports as `mcell`. **Decision for the spike:** either (a) keep the explicit
`set_target_properties(... SUFFIX ...)` override to preserve the plain name, or (b) accept the
versioned suffix and update the bundler. Note the bundling tooling already encodes filenames and is
**stale** — `mcell_tools/scripts/pypi_wheel.py:36-43` still references `cpython-35m-…` (Python 3.5)
names. Whichever path is chosen, reconcile it with `pypi_wheel.py` and `build.py` (target
`mcell4_so`, `build.py:22,107`).

### 10.4 Windows / MSVC specifics
- nanobind needs **MSVC 2019+**; MCell already targets modern MSVC with `/EHsc` and
  `/Zc:strictStrings-` (the legacy-code counterpart of `-fpermissive`, `mcell/CMakeLists.txt:116-123`).
  Those stay as-is — they're for `src/`, not the bindings.
- ⚠️ **`/bigobj` is not currently set** (`grep` confirms none). The ~74 large generated
  `gen_*.cpp` translation units full of heavy template instantiations are exactly the case that
  trips MSVC's object-section limit. Expect to add `/bigobj` for the `mcell4_so` target on MSVC
  (nanobind's own examples commonly do). Add it as a Windows-guarded option.
- Keep the existing MSVC "searches for python library when linking utils" workaround
  (`mcell/CMakeLists.txt:504-506`) in mind when reworking the link section.

### 10.5 Stable ABI & free-threading (cross-platform, optional)
- **`STABLE_ABI`** (nanobind option) requires **Python 3.12+** — MCell's 3.13 qualifies. A
  stable-ABI build is reusable across Python *minor* versions instead of only patch versions, at a
  small perf cost. Because MCell ships its own Python, the benefit is modest, but it would
  uniformly relax the "exact 3.13" coupling across all three OSes (esp. helpful for Windows
  packaging). Treat as optional, evaluate after the core port works.
- **`FREE_THREADED`** (3.13+) is available uniformly across platforms — future-proofing for no-GIL,
  relevant since MCell's engine is already C++-threaded. Not part of the initial spike.

### 10.6 Cross-platform guardrails for the spike
- Develop on Debian, but **gate every CMake change with the same idioms already in the file**
  (`if(APPLE)`, `if(WIN32)`, `$<PLATFORM_ID:…>` generator expressions) — do not hard-code
  Linux-only assumptions.
- Prefer **deleting** MCell's hand-rolled binding-link/LTO/suffix plumbing in favor of
  `nanobind_add_module` options, rather than layering nanobind on top of it.
- Add to the §8 spike exit criteria: a **clean configure+build on at least Linux and one macOS
  arch**, and a **CMake-level (configure) check on Windows/MSVC** even if a full Windows build is
  deferred — enough to confirm `/bigobj`, the suffix decision, and the link rework are sound on all
  three before committing to the full template rewrite.

---

## 11. Spike results (executed validation)

A standalone **A/B harness** was built and run: the same C++ domain — mirroring MCell's actual
binding shapes audited from `libmcell/` — bound under **both pybind11 2.9.2** (vendored, baseline)
and **nanobind 2.13.0**, with a pytest asserting Python-visible behavior matches. Built with
`nanobind_add_module` against the MCell venv (Python 3.13.12, g++ 12.2.0, CMake 4.3). **19/19 A/B
tests pass.** The harness lives outside the tree (a throwaway spike, per the no-commit-into-`mcell/`
convention); it does **not** touch `gen.py` or the build.

### 11.1 Confirmed verdicts (§8 risks)
- **MI is a non-issue** — flat `nb::class_<Model>` works identically (risk 4).
- **Risk 1 is narrow, and the usage audit closes it.** Only nested `List[List[…]]` in-place inner
  mutation (`m[i][j]=v`) is lost in nanobind; `List[X*]` element-field mutation and scalar
  `__setitem__` are safe. A full in-tree Python sweep (`mcell_tests`/`mcell`/`mcell_tools`/
  `cellblender`, ~2100 files) found **zero** in-place nested mutations — all access is
  construction-time whole assignment, `len(prop)`, or nested-index **reads** (served correctly from
  the copy). Geometry is mutated via API methods, not list writes. **Risk 1 is moot for MCell; no
  special binding handling needed.**
- **Risk 2 trivial** — no `export_values()`; emit `m.attr("NAME")=E::NAME` per enumerator.
- **Risk 3 bites, has a fix** — naive `nb::init` → `bad_weak_ptr`; bind ctor via
  `nb::new_(make_shared)`.
- Single bound-base inheritance + virtual override dispatch, `def_prop_rw`/`def_rw`, arithmetic
  enums, and `std::function` callbacks (incl. bool-returning cancel semantics + `gil_scoped_acquire`)
  all behave identically.

### 11.2 New findings beyond the original §8 (widen the `gen.py` change)
1. **`.none()` on nullable args (pervasive).** nanobind rejects `None` for an `nb::object`/pointer
   arg unless the arg is `nb::arg("x").none()`; pybind11's `py::object` accepts `None` implicitly.
   MCell defaults *many* optional params (and callback `context`) to `None`, so `gen.py` must emit
   `.none()` (and `.none() = nb::none()` for the default) on each nullable arg. **Largest mechanical
   surface-area item — bigger than the original draft anticipated.**
2. **`bind_vector` needs `NB_MAKE_OPAQUE`.** Contrary to "opaque automatically", nanobind's
   `bind_vector` conflicts with the `stl/vector.h` caster and requires an explicit
   `NB_MAKE_OPAQUE(V)` — the direct analogue of `PYBIND11_MAKE_OPAQUE`. Structure of
   `gen_vectors_make_opaque.h` is unchanged; only the macro name differs.

### 11.3 Toolchain note
**pybind11 2.9.2 compiles and runs against Python 3.13** for this binding subset (g++ 12.2) — so
MCell's pinned combo is viable, tempering the §10.2 "2.9.2 is old" caveat (it is old, but it builds).

### 11.4 Net
Migration remains feasible and unusually cheap (leveraged through `gen.py`). Plan **three**
mechanical sweeps, not two (§7 step 2). **The risk-1 usage audit is now done and closes the last
behavioral unknown** — no in-tree workflow does in-place nested-list mutation, so no special
binding handling is required. With all §8 risks verified and the two new mechanical items scoped,
recommend proceeding to the real `gen.py` rewrite on the `mcell4_nanobind` branch.

## 11.5 Build-time findings (executed full port)

The full port was carried out on `mcell4_nanobind`: `gen.py`/`constants.py` rewritten, hand-written
files ported (`bindings.cpp`, `callbacks.h`, `globals.h`, `api_common.h`, `pybind11_stl_include.h`),
nanobind vendored at `mcell/libs/nanobind`, CMake switched to `nanobind_add_module`. `mcell.so`
**compiles, links, imports, and passes smoke checks** — incl. `ComponentType.inst()` /
`ElementaryMoleculeType.inst()` / `Region` boolean ops, which exercise the risk-3 factory fix at
runtime. Beyond the spike's §11.2 items, the real build surfaced these (all fixed):

1. **`nb::class_<X, Base>` does NOT slice to `nb::class_<X>`.** pybind11 allowed returning the
   derived-template `class_` as the base template via the shared `object` base; nanobind rejects it.
   Fix: `define_pybinding_*` now return **`void`** (the return value was never used anyway).
2. **`nb::str` has no `std::string` constructor** (pybind11's `py::str(std::string)` worked). Fix:
   assign the raw C++ value to the module attr and let the string caster convert
   (`m.attr("X") = X;`), in both the constants generator and `bindings.cpp`'s `__version__`.
3. **nanobind has a support library (`libnanobind-static`) — pybind11 was header-only.** This makes
   static-link **order** matter: it must appear *after* `libmcell`/`mcell4` (which contain the
   compiled binding code) on the GNU `ld` line, or their nanobind symbols are undefined. Fix:
   re-list `nanobind-static` after `libmcell` in `target_link_libraries(mcell4_so …)` (mirrors the
   pre-existing double-listing of `libmcell`).
4. **Opt-in STL casters are per-container.** pybind11's `stl.h` bundled them all; nanobind needs an
   explicit include per type. A `std::map<std::string,double>` default argument threw
   **`std::bad_cast` at module init** (nanobind's `cast_error` *is* `std::bad_cast`,
   `nb_error.h:90`) because `<nanobind/stl/map.h>` wasn't included. Fix: add `stl/map.h` +
   `stl/set.h` (the only STL containers in the API beyond string/vector) to `api_common.h`.
   → Generalize for §4: the "opt-in STL caster includes" row must enumerate **every** STL container
   the API uses as a value/argument type, not just vector/string/function.
5. **`std::function<bool(...)>` callback + `None` return throws.** nanobind's `std::function`
   caster strictly casts the return value, and a Python callback with no `return` yields `None`,
   which `cast<bool>(None)` rejects (→ `std::bad_cast`). pybind11 coerced `None`→`False`. Fix:
   declare the reaction-callback type as `std::function<py::object(...)>` (in `model.yaml`,
   `gen_model.h`, `api/model.{h,cpp}`, `callbacks.h`) and interpret truthiness in
   `do_rxn_callback` via `PyObject_IsTrue` (`None`/`False`/no-return ⇒ don't cancel). Confirmed:
   `cast<py::object>(None)` is fine; only `cast<bool>(None)` failed. (9 rxn-callback tests.)
6. **nanobind rejects `int` for a `bool` argument** (pybind11 accepted `0`/`1`). MCell's
   checkpoint *export* (`export_to_python` codegen) streamed bools via `<< value` → `1`/`0`,
   producing checkpoint Python that nanobind then refused to load. Fix: emit Python `True`/`False`
   for `YAML_TYPE_BOOL` attributes in `gen.py`'s `write_export_to_python_implementation`.
   (Test `2700_save_checkpoint_rxn_in_box`.)
7. **The `enable_shared_from_this` factory must cover the whole subclass closure, not just the
   declaring classes.** `GeometryObject`/`SurfaceRegion` inherit `Region` (which has
   `enable_shared_from_this<Region>`), so `box1 - box2` (`Region::__sub__` → `shared_from_this()`
   on a *Python-constructed* `GeometryObject`) threw `bad_weak_ptr` — even though `Region` itself
   was factory-bound. Fix: `gen.py` now computes the **transitive closure** of
   `ENABLE_SHARED_FROM_THIS_CLASSES` over the YAML `superclass` chain (`g_esft_classes`) and binds
   every such class's ctor via `py::new_`. (Heavier `tests/pymcell4` region-expression tests
   `1700`/`1720`/`1250`.)

Findings 5–6 came from `tests/pymcell4_positive` (82/82), finding 7 from the heavier
`tests/pymcell4` sim suite (**49/50**; the one failure, `2840_overlapping_icospheres_transparent`,
"Processing of counted volumes failed", **fails identically on the original pybind11 build** — a
pre-existing `src4` engine issue, not a migration regression). Common theme: **nanobind is stricter
at the Python↔C++ boundary** (no `None`→`bool`, no `int`→`bool`, weak_ptr must be primed via
make_shared) where pybind11 silently coerced / its always-`shared_ptr` holder hid the issue.

8. **Argument-mismatch error messages differ — nanobind reports argument *types*, pybind11
   (as patched by MCell) reported truncated argument *values*.** One nutmeg test
   (`nutmeg4_pymcell4/2000_shorter_error_message`) asserts the truncated-value format
   (`element_connections=[[0,1,2],[1,3,4],…,…`), a behavior MCell obtained by **patching vendored
   `pybind11.h`'s dispatcher**. nanobind emits `Invoked with types: … element_connections: list`
   instead — the error is still raised with exit code 1, only the message format differs. This is
   a **test-expectation update**, not a code regression (no pybind dispatcher patch carries to
   nanobind). `nutmeg4_pymcell4` is otherwise **50/51**. **Expectation updated (2026-10-14,
   §12):** `matchPattern = "element_connections: list"` on `mcell_tests` `mcell4_nanobind` → PASS.

Test suites so far (nanobind build): `pymcell4_positive` **82/82**, `pymcell4` **49/50** (the 1 is
the pre-existing `src4` `2840` engine failure), `nutmeg4_pymcell4` **50/51** (the 1 is the
error-message-format difference above). No functional regressions attributable to the migration.

### 11.6 Full-suite pybind11-vs-nanobind equivalence (the `run.py test_all` run)

Ran the **entire** `run_tests.py` suite (what `mcell_tools/run.py`'s test step runs — all categories:
mdl, mcell3, nutmeg, every data_model / bngl / pymcell4 conversion path; ~2600 result entries) on
**both** builds and diffed the per-test PASS/FAIL sets. Baseline is a **same-commit (`e5c71769`),
Release** pybind11 build (the `mcell4_dev` checkout) — so the *only* variable is pybind11 vs nanobind:

| build | passed | failed | skipped | TODO | ignored |
|---|---|---|---|---|---|
| pybind11 (Release, same commit) | 2262 | 383 | 143 | 46 | 332 |
| nanobind  (Release)             | 2262 | 383 | 143 | 46 | 332 |

**Identical counts.** Diffing the failed sets (381 entries each):
- **Exactly one** test fails on nanobind but passes on pybind11:
  `nutmeg4_pymcell4/2000_shorter_error_message` — the cosmetic error-message-format difference
  (finding 8). Needs an expected-pattern update, not a code fix (**done — §12**).
- **One** test fails on pybind11 but passes on nanobind:
  `pymcell4/3100_dyn_vert_blocking_and_intramembrane_rxns` — nanobind is strictly no worse. The
  cause is not reference data: pybind11 ≥ 2.6 sets `__hash__ = None` on any class that defines
  `__eq__`, so the test's model, which hashes API objects, fails with
  `TypeError: unhashable type: 'mcell.GeometryObject'`. nanobind keeps the default identity
  hash. (`mcell4_dev` fixed the pybind11 side in `7e5be891`; see §12.)
- **~380 failures are common to both** — pre-existing/environmental in this checkout (reference-data
  mismatches; the set includes pure-MDL tests that never load `mcell.so`), so unrelated to the
  binding library.

Conclusion: **the pybind11→nanobind migration changes no test outcomes** beyond the single expected
cosmetic error-message format. (An earlier run against an older *Debug* pybind build agreed, modulo
`0000_out_of_partition_error` which failed only in Debug — a build-type artifact, not a binding one.
The ~380 shared failures predate and are independent of this work and should be triaged separately.)

Remaining follow-ups: the data-model / MDL conversion suites; numpy `py::array_t`→`nb::ndarray`
for `VectorFloat`/`VectorInt` (currently relies on `bind_vector`'s sequence ctor — note
`0090_numpy_array_as_release_location` already passes); ~~the Darwin link-helper on the `mcell`
*executable*~~ (**done — §11.7**); ~~update `2000_shorter_error_message`'s expected pattern to
nanobind's format~~ (**done — §12**); Windows/MSVC build (§10.4).

> **numpy follow-up — investigated (June 2026).** Benchmarking showed the *real* issue is the
> nested `vector<vector<double>>` geometry representation, not the binding. A binding-only
> `nb::ndarray` fast path ("option A") is ~12× but is shadowed by `bind_vector`'s `iterable`
> converter (requires hand-rolling, prototype-confirmed). The proper fix ("option B") is to make
> the geometry attributes contiguous (`vector<Vec3>`), which gives ~zero-copy numpy (~100×), cuts
> memory, simplifies the converter, and dissolves risk-1 — but it is an API/engine refactor,
> independent of this migration. **Full design + ecosystem impact (cellblender, data-model
> converters, etc.): see `DESIGN_geometry_vec3_representation.md`.** Recommendation: do option B
> deliberately on its own branch (not the binding hack) once test baselines are settled.

## 11.7 Executed macOS step (executable link rework + `libs/pybind11` removed)

After the Linux validation above, the migration was completed on **macOS (Apple Silicon, Apple
Clang `/usr/bin/c++`, MCell's static Python 3.13.12)** and `libs/pybind11` was deleted from the
tree (commit `06837640` on `mcell4_nanobind`). What was done:

1. **The `mcell` executable was the last consumer of pybind11.** It linked
   `pybind11::python_link_helper` on Darwin — which (per its definition in pybind11's now-removed
   `tools/pybind11Common.cmake`) is *only* the link option `-undefined dynamic_lookup` on Darwin
   (and a no-op elsewhere; despite its "Adds link to Python libraries" comment it adds none). The
   executable deliberately does **not** link the static
   Python lib on macOS — Python C-API symbols are left undefined and resolved at runtime. Replaced
   with the equivalent spelled out directly, dropping the pybind11 target:
   ```cmake
   # mcell/CMakeLists.txt:548
   target_link_options(mcell PRIVATE $<$<PLATFORM_ID:Darwin>:LINKER:-undefined,dynamic_lookup>)
   ```
   The **module** (`mcell4_so`) needed nothing — `nanobind_add_module` already applies this on
   macOS (§10.1). Chose this over "link `Python3::Python` on Darwin too" because that would have
   *changed* behavior (linking the static lib in); this preserves the exact prior behavior.
2. **`libs/pybind11` (vendored 2.9.2, 236 files) removed**, with its now-dead CMake feeders:
   `add_subdirectory(libs/pybind11)` deleted; `PYBIND11_LTO_CXX_FLAGS` (set, never read in-tree)
   deleted; `PYBIND11_PYTHON_VERSION` → **`MCELL_PYTHON_VERSION`** (`mcell/CMakeLists.txt:8,10` +
   `libmcell/CMakeLists.txt:7`); the `api_config.h` MSVC-include-collision comment generalized so it
   no longer implies a live pybind11 dependency. The `.so` suffix override
   (`mcell/CMakeLists.txt:570`) is **kept** — i.e. §10.3 option (a), plain `mcell.so` on macOS.
3. **Python resolution on macOS is correct.** A clean configure resolves both
   `find_package(Python3 … EXACT)` and nanobind's `find_package(Python … EXACT)` to the *same*
   venv interpreter (`mcell_python/.venv/bin/python3.13`, 3.13.12) — the latter pinned via
   `set(Python_EXECUTABLE ${Python3_EXECUTABLE})` just before it (`mcell/CMakeLists.txt:347`).

**Verification (macOS):**
- Clean configure+build **from scratch with `libs/pybind11` deleted** — zero `pybind11` mentions in
  CMake output (previously printed `pybind11 v2.9.2`); both `lib/mcell.so` and the `mcell`
  executable build with no link warnings.
- `import mcell` works; `m.ComponentType('c', states=['0','1']).inst('0')` exercises the risk-3
  `enable_shared_from_this` factory path at runtime; the `mcell` executable runs (`--version`).
- **`pymcell4_positive`: 82 pass, 2 `FAILED_DIFF`** (`1802_vol_rxn_callback_w_cancellation`,
  `4210_organelle_bng_units`). Those **two fail *identically* on the same-commit reference
  pybind11 macOS build** (`mcell_tools/work/build_mcell_pybind11`) — i.e. pre-existing
  macOS-vs-Linux reference-data diffs, **not** nanobind regressions. This is the §11.6 equivalence
  methodology applied per-platform: on macOS, nanobind and pybind11 produce the **identical**
  pass/fail set.

**Net:** pybind11 is gone from the build entirely; nanobind is the sole binding library on both
validated platforms at this point. Windows/MSVC (§10.4) was the only unbuilt target — now done
in §11.8.

## 11.8 Executed Windows step (`/bigobj` + nanobind-static on exe/utils — final platform)

Windows/MSVC, the last platform, was built and validated on **Windows 11 (Visual Studio 17 2022,
MSVC 14.43; system-wide Python 3.13.12 at `C:\mcell\tools\Python313`, NOT a uv venv — Windows uses
the system interpreter)** (commit `c8d49386` on `mcell4_nanobind`). The binding migration itself was
already complete; this step only made both artifacts build and pass on Windows.

**What was already correct (verified, not assumed):**
- **`.pyd` suffix (§10.3 option a).** The explicit `set_target_properties(mcell4_so PROPERTIES
  SUFFIX ".pyd")` override (`mcell/CMakeLists.txt:568`) wins over `nanobind_add_module`'s versioned
  suffix — a plain `mcell.pyd` is produced and `import mcell` loads it.
- **Python resolution (§10).** A clean configure resolves **both** `find_package(Python3 … EXACT)`
  and nanobind's `find_package(Python … EXACT)` (pinned via `set(Python_EXECUTABLE
  ${Python3_EXECUTABLE})`, `mcell/CMakeLists.txt:347`) to the **same** system interpreter
  (`C:/mcell/tools/Python313/python3.exe`, 3.13.12). Zero `pybind11` mentions in CMake output.
- **`mcell.pyd` itself links cleanly** — `mcell4_so` already lists `nanobind-static` (§11.5 #3),
  which is all the *module* needs.

**What had to change (two Windows-only CMake fixes, both committed):**
1. **`nanobind-static` on the `mcell` executable and `data_model_to_pymcell` (the real finding).**
   nanobind has a static support library where pybind11 was header-only (§11.5 #3). The `mcell`
   executable and the `data_model_to_pymcell` utility compile/link `mcell4`/`libmcell` objects that
   reference nanobind support symbols — `nanobind::python_error::restore` and
   `builtin_exception`'s destructor (pulled in via exception-handling paths) — but neither target
   linked `nanobind-static`. MSVC strictly resolves **every** external when linking an executable,
   so both failed with `LNK2001`/`LNK1120: 2 unresolved externals`. Fix: link `nanobind-static`,
   **Windows-guarded** (`$<$<PLATFORM_ID:Windows>:nanobind-static>`, listed last for link-order) on
   `target_link_libraries(mcell …)` (`mcell/CMakeLists.txt:532`, the guarded entry at `:542`) and on
   `data_model_to_pymcell` (`utils/data_model_to_pymcell/CMakeLists.txt:32`). GNU ld and macOS `dynamic_lookup` did not
   require this, so the Linux/macOS link lines are untouched. **This is the Windows generalization
   of §11.5 #3 from the module to the executable/utilities.**
2. **`/bigobj` (MSVC), as a safeguard.** Added build-wide via `mcell_project_options`
   (`mcell/CMakeLists.txt:130`, `$<$<CXX_COMPILER_ID:MSVC>:/bigobj>`). **Empirical correction to the
   §10.4 prediction:** the **Release** build of the ~74 dense generated `gen_*.cpp` TUs **stays under
   MSVC's section limit without `/bigobj`** — nanobind instantiates far fewer templates than
   pybind11, so the predicted `C1128` did not occur. `/bigobj` is kept as defense for Debug builds
   (far more sections) and future TU growth; it is harmless on the other TUs.

**Verification (Windows):**
- Clean configure + full Release rebuild: `mcell.pyd`, `mcell.exe`, and `data_model_to_pymcell.exe`
  all build with zero errors.
- `import mcell` → `4.2.0`; `m.ComponentType('c', states=['0','1']).inst('0')` and
  `m.geometry_utils.create_box(...)` exercise the risk-3 / §11.5 #7 `enable_shared_from_this`
  factory + transitive-closure paths at runtime; `mcell.exe --version` runs (commit `06837640`).
- **`pymcell4_positive`: 82 pass, 2 `FAILED_DIFF`** (`1802_vol_rxn_callback_w_cancellation`,
  `4210_organelle_bng_units`) — the **same two** as macOS (§11.7). Both fail **identically** on the
  same-tree reference pybind11 build (`mcell_tools/work/build_mcell_pybind11`, 0 pass / 2
  `FAILED_DIFF`) — i.e. pre-existing Windows reference-data diffs, **not** nanobind regressions. The
  §11.6 equivalence methodology applied per-platform: on Windows, nanobind and pybind11 produce the
  **identical** pass/fail set.
- Binary-size win is visible: nanobind `mcell.pyd` ≈ 10.79 MB vs the pybind11 reference's ≈ 11.97 MB.

**Net:** all three platforms (Linux, macOS, Windows) now build and validate with pybind11 removed
entirely; nanobind is the sole binding library everywhere. The migration is complete.

## 12. Merge-time TODO (resolved)

- ☑ **nutmeg `2000_shorter_error_message`** — its `matchPattern` was written for MCell's patched
  pybind11 dispatcher (truncated arg *values*); nanobind prints arg *types*, so the pattern matched
  0 lines and the test reported `FAILED_NUTMEG_SPEC`. Error is still raised + exit 1 (format parity
  only, not a regression). It had been deferred because `mcell_tests` only had the shared
  `mcell4_dev` branch, where the pybind11 build is still tested. **Fixed (2026-10-14)** on a
  dedicated `mcell_tests` branch **`mcell4_nanobind`** (commit `3b0094ba`):
  `matchPattern = "element_connections: list"` → test **PASSED** on the nanobind build. Remaining:
  merge `mcell_tests` `mcell4_nanobind` → `mcell4_dev` together with `mcell`'s. Full write-up:
  [`TODO_track_b_nutmeg_error_message.md`](TODO_track_b_nutmeg_error_message.md).
- ☑ **`mcell` fixes from `mcell4_dev` (checked 2026-10-15).** Two `mcell4_dev` commits made after
  this branch's base `e5c71769` were reviewed:
  - **`98980098` — `__mp_main__` guard in the generated `model.py`:** cherry-picked as
    **`4587bf63`** and pushed. `mcell4_generator.cpp` is now byte-identical on both branches, so
    the merge sees the same change on both sides. Verified on Linux: regenerated output for four
    test models differs only by the guard in `model.py`; direct runs, checkpoint save + resume,
    and a `spawn` child behave as intended; the data-model and nutmeg conversion modes of
    `mcell_tests` (1,385 results) show no new failures against the Jul 7 full-suite run. Details
    in doc 05 §8.
  - **Follow-up fix (2026-10-16), both branches:** the guard left the checkpoint-resume code
    unguarded, so a `spawn` worker started after a checkpoint was written resumed the
    simulation instead of doing its task. Fixed on `mcell4_dev` as `a972bce4` and
    cherry-picked here as `a62d7233`, with the regression test
    `0830_customization_spawn_after_checkpoint` in `mcell_tests` (`0ac2e40c` → `36d1772a`).
  - **Second follow-up (2026-10-20), made on this branch first:** the resume code set
    `sys.path` to the `mcell.VectorStr` returned by `m.run_utils.remove_cwd()`, so a `spawn`
    start after a resume failed. Fixed here as `e78a630d` and cherry-picked to `mcell4_dev` as
    `575328ae` (`python_export_constants.cpp` stays byte-identical on both branches), with the
    regression test `0840_customization_spawn_after_resume` in `mcell_tests`
    (`3b0159db` → `ecbd4f69`). Details: doc 05 §8.
    Details: doc 05 §8.
  - **`7e5be891` — identity `__hash__` emitted by `gen.py`: not ported, not needed.** It fixes the
    pybind11 regression behind §11.6's `3100` failure. nanobind doesn't set `__hash__ = None`
    when `__eq__` is defined: on the nanobind `mcell.so`, `Species`, `Complex`, `Color`,
    `GeometryObject`, and `ReleaseSite` hash and work as dict keys and set members. The commit's
    other change, the `register_reaction_callback` docstring, is already in this branch's
    generated code. At merge time, don't carry its pybind11 `__hash__` lines into the nanobind
    `gen.py` or `generated/`.
- ☑ **List-property lifetime (2026-10-28), both branches.** `gen.py` now emits
  `rv_policy::reference_internal` for list attributes (`e1e5ded2`, also on
  `mcell4_nanobind_novtk`); see §8 item 1 and doc 05 §1.7. `mcell4_dev`'s pybind11 `gen.py` used
  the same pattern (`PYBIND11_MAKE_OPAQUE(std::vector<double>)` plus
  `return_value_policy::reference` on list properties) and was affected too: reads through a
  temporary owner returned garbage, and a nested read could raise `MemoryError`. Fixed there as
  `1d47c148` with `return_value_policy::reference_internal`; the same 49 properties change, the
  `pymcell4_api` results are unchanged and the test outputs are byte-identical. At merge time keep
  the nanobind line.

---

*Authored from a source audit (June 2026), then extended with an executed A/B spike (pybind11 2.9.2
vs nanobind 2.13.0; 19/19 A/B tests green) and the executed Linux + macOS + Windows port
(§11.5–§11.8). Citations are `file_path:line` against the tree at the
time of writing — re-verify with `wiki_check.py` if `gen.py` or `libmcell/` has since changed.*
