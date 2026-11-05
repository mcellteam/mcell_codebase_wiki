# Design note — replace nested `vector<vector<…>>` geometry with a contiguous representation

> **Status: PROPOSAL (not implemented).** This is "option B" referenced in
> `06_nanobind_migration_feasibility.md` §11.5. It is an **API/engine design improvement**,
> **independent of the pybind11→nanobind migration** (pybind11 had the identical inefficiency).
> Paths are relative to `$ROOT` (the build root; see `CLAUDE.md`).
> **Decision required before implementing** — see §7.

---

## 1. Summary

MCell's Python API stores large geometry as **nested** `std::vector<std::vector<double>>`
(`vertex_list`) and `std::vector<std::vector<int>>` (`wall_list`). This is the wrong data
structure: it fragments memory, bloats it ~2–3×, is cache-hostile, forces a copy at every
boundary (numpy, glm, the engine, mesh files), and makes fast numpy ingestion impossible without
a binding-layer hack. **MCell already works in contiguous `Vec3` internally** and converts *back*
to the nested form purely for the API surface.

**Proposal:** change the geometry-bearing API attributes to a **contiguous** representation —
`std::vector<Vec3>` for vertices and `std::vector<glm::ivec3>` (or a flat `std::vector<int>`) for
walls — and expose them to Python as zero-copy `nb::ndarray` (with backward-compatible
list/tuple/numpy *input* and Vec3 made list-like). This unlocks ~zero-copy numpy interop (the real
performance ceiling, ~100× over the current path), cuts memory, likely *simplifies* the converter,
and dissolves "risk-1" (the bind_vector copy-on-`__getitem__` nested-mutation issue) for free.

## 2. Why — the evidence

### 2.1 The nested representation is redundant: MCell already uses `Vec3`
- `Vec3` is `glm::dvec3` (`mcell/src4/defines.h:358`) — a standard-layout struct of 3 contiguous
  scalars. A `std::vector<Vec3>` **is** a contiguous `(N,3)` buffer.
- The engine is `Vec3`-based throughout; `geometry_utils.cpp` builds `std::vector<Vec3>& vertex_list`
  (the `create_box`/`create_icosphere` helpers, `mcell/libmcell/api/geometry_utils.cpp:124-141`),
  then the result is presented to Python as `vector<vector<double>>`. So the nested form is **extra
  conversion work** at the boundary, and it is the odd one out everywhere else (numpy, glm, VTK,
  OBJ/PLY mesh files).

### 2.2 Cost of nested `vector<vector<double>>`
- **N separate heap allocations** (one tiny 3-element vector per vertex), each ~24 B of vector
  bookkeeping + heap-header overhead for 24 B of payload → **~2–3× memory bloat** and fragmentation.
- **Cache-hostile**: pointer-chasing N inner vectors instead of striding one contiguous buffer —
  costs the engine on every geometry traversal, not just at ingestion.

### 2.3 Measured ingestion cost (nanobind build, current path)
From the benchmark (`scratchpad/bench_numpy.py`):

| N vertices (2N walls) | `VectorVectorFloat(Nx3 np)` | `VectorVectorInt(2Nx3 np)` | full `GeometryObject` |
|---|---|---|---|
| 1M | 1.26 s | 2.74 s | **4.47 s** |
| 4M | 5.10 s | 10.98 s | **18.3 s** |

~90 % of `GeometryObject` construction time is the numpy→nested-vector conversion. For realistic
neuropil reconstructions (millions of vertices, often many objects), this is seconds-to-tens-of-
seconds of avoidable setup cost. A contiguous `vector<Vec3>` viewed as an `(N,3)` ndarray is a
`memcpy` (~0.001–0.005 s/1M floats — the "bulk floor" in the same benchmark), i.e. **~100× faster**,
versus only ~12× for the binding-only hack (option A).

### 2.4 Independent of the binding library
This inefficiency exists identically under pybind11. Fixing it benefits both `mcell4_dev` (pybind11)
and `mcell4_nanobind`. nanobind merely makes the *good* representation easier to expose (first-class
`nb::ndarray`, zero-copy).

## 3. The proposed change

### 3.1 Affected API attributes (all `List[List[float/int]]`)
| YAML | attribute | current C++ | proposed C++ | notes |
|---|---|---|---|---|
| `geometry.yaml:140` | `GeometryObject.vertex_list` | `vector<vector<double>>` | **`vector<Vec3>`** | the primary win |
| `geometry.yaml:145` | `GeometryObject.wall_list` | `vector<vector<int>>` | **`vector<glm::ivec3>`** | triangle vertex indices |
| `introspection.yaml:93` | `Wall.vertices` | `vector<vector<double>>` | `vector<Vec3>` | introspection (read-only) |
| `subsystem.yaml:540` | `ReactionRule.variable_rate` | `vector<vector<double>>` | *leave as-is* | small, `[time,rate]` pairs — not geometry; no benefit |
| `submodules.yaml:125` | `data_utils.load_dat_file` return | `vector<vector<double>>` | *leave as-is* | small two-column data |

Only the **geometry** attributes (vertex/wall) need to change; `variable_rate` and `load_dat_file`
are small and semantically not Nx3 arrays.

### 3.2 Make `Vec3`/`IVec3` list-like (the compatibility keystone)
The break "feels" large only because `vertex_list[i][j]` and list-of-lists construction are common.
Both are recoverable cheaply because **glm vectors already support `operator[]`**:
- Add `__getitem__`/`__len__` to the `Vec3`/`IVec3` bindings (trivial — `glm::dvec3::operator[]`).
  → `obj.vertex_list[i][0]` keeps working when `vertex_list` is `List[Vec3]`.
- Register implicit construction `Vec3` from a 3-`tuple`/`list`/`ndarray`-row.
  → `GeometryObject(vertex_list=[[x,y,z], …])` keeps working (each row → `Vec3`).
With these two additions, `List[Vec3]` is **near-drop-in** for both access and construction.

### 3.3 Python-facing shape (the API-compat decision — see §7)
Three coherent options, in increasing ambition:
- **(C1) `List[Vec3]` + list-like Vec3.** Smallest conceptual change; `bind_vector<vector<Vec3>>`.
  numpy *input* via an `nb::ndarray` ctor on the vector (still needs the option-A ordering care, but
  Vec3 elements make even the slow path cheaper). Reads return the bound vector.
- **(C2) numpy-in / numpy-out.** `vertex_list` setter accepts `nb::ndarray<double, ndim<2>>`
  (zero-copy view or one bulk copy into `vector<Vec3>`); getter returns a **zero-copy** `nb::ndarray`
  view over the contiguous `vector<Vec3>`. Best performance; most "numpy-native".
- **(C3) hybrid / back-compat shim (recommended).** Accept **any** of list-of-lists, numpy `(N,3)`,
  or `List[Vec3]` on input; store `vector<Vec3>`; return a zero-copy numpy view (with `.tolist()`
  available). Maximizes compatibility for existing models/cellblender while giving the fast path.

## 4. Benefits
- **~Zero-copy numpy** both directions (the ~100× ceiling, not the ~12× of option A).
- **~2–3× less memory** + contiguous → better cache behavior for the **engine**, not just ingestion.
- **Likely simplifies the converter** — deletes the existing `Vec3`↔nested round-trips
  (`geometry_utils.cpp`, and the per-element reads in `mcell4_converter.cpp`).
- **Dissolves risk-1** (`06_*.md` §8/§11.5): the bind_vector copy-on-`__getitem__` nested-mutation
  hazard only exists because elements are *inner vectors*; with `Vec3` elements it disappears.
- Makes the **option-A binding hack unnecessary** — no forked `bind_vector` to maintain.

## 5. Consequences & ecosystem impact

### 5.1 In-tree (mcell)
- **`mcell4_converter.cpp`** — the main engine consumer (`:185`, `:749`, `:760`, `:921`, `:931`,
  `:1049`). Iterates `for (auto& vert : o->vertex_list)` and `o->wall_list[side][i]`. With
  `vector<Vec3>`/`vector<ivec3>` the loops are *simpler* (glm `operator[]` still serves `wall_list[side][i]`).
  **Net: likely a simplification, fully internal.**
- **`api/geometry_object.cpp`, `api/compartment_utils.cpp`** — read `vertex_list` element-wise;
  small mechanical updates (`v[0]` → `v.x` or keep `v[0]` via Vec3 `operator[]`).
- **`gen.py` codegen** — the type map (`yaml_type_to_cpp_type`), the getter/setter signatures, the
  `make_opaque`/`bind_vector` emission for these types, the `.pyi` type, and `export_to_python`
  (must emit the chosen Python form — list-of-lists *or* `m.Vec3(...)` rows). New YAML type spellings
  (e.g. `List[Vec3]`) or a dedicated `VertexList`/`WallList` type.
- **`export_to_python` / checkpoints** (`gen_geometry_object.cpp`) — checkpoint/script export emits
  `vertex_list = [...]`; must emit a form the new API accepts (trivially compatible under C3).
- **`.pyi` stub + RST docs** — regenerated; type annotations change (`List[Vec3]` / `numpy.ndarray`).

### 5.2 The CellBlender data-model → pymcell4 path (mcell utility)
- **`utils/data_model_to_pymcell/`** (`python_generator.cpp:744`, `data_model_geometry.cpp:31-47`):
  reads `KEY_VERTEX_LIST` / `KEY_ELEMENT_CONNECTIONS` from the **JSON data model** and *generates
  pymcell4 Python source*. It emits the literal Python for `vertex_list`. **It must be updated to
  emit the new API form** (under C3, the existing `[[x,y,z],…]` literal still works → minimal/no change).
- **The CellBlender JSON *data model* format itself is unaffected** — `KEY_VERTEX_LIST` /
  `KEY_ELEMENT_CONNECTIONS` are an external on-disk schema (list-of-lists JSON). This change is about
  the in-memory **C++/Python API**, not the data-model file format. The converters bridge the two.

### 5.3 CellBlender (external repo)
- **`cellblender/cellblender_objects.py`** constructs `mcell.GeometryObject(...)` with `vertex_list`.
  It is a **direct API consumer in a separate repo.** Impact depends on the chosen option:
  - **C3 (recommended):** cellblender keeps passing list-of-lists (or could switch to numpy for
    speed) → **little or no change required**, but it *gains* the fast path if it passes numpy.
  - **C1/C2 strict:** cellblender must adopt `List[Vec3]` or numpy → coordinated cross-repo change.
- Anything reading `obj.vertex_list` back (introspection, viz) sees `List[Vec3]`/ndarray — covered by
  Vec3 `__getitem__` (C1/C3) so `[i][j]` access survives.

### 5.4 User models / tests
- Models that build geometry via list-of-lists (most `tests/pymcell4*` geometry cases, neuropil
  pipelines) — **unaffected under C3**; under C1/C2 they need updating (or rely on Vec3 list-likeness).
- The risk-1 audit (`06_*.md` §8) already found **no** in-tree code does in-place nested mutation, so
  no workflow depends on the `vector<vector<…>>` mutability that this change removes.

### 5.5 Precision footgun (must decide)
`Vec3` is `glm::dvec3` (double) when MCell is built in double precision, `glm::fvec3` (float) in
single precision (`defines.h:358-364`, the `pos_t` switch). For **true zero-copy** numpy the buffer
dtype must match (`float64` vs `float32`). The API should pin a dtype (recommend always exposing
`float64` and copy-converting if the engine is single-precision), or expose the engine dtype and
document it. A bulk copy is still ~100× better than the nested path.

## 6. Touch-point checklist
- `mcell/libmcell/definition/geometry.yaml` (+ `introspection.yaml`) — types of vertex/wall props.
- `mcell/libmcell/definition/gen.py`, `constants.py` — type map, codegen, `.pyi`, `export_to_python`.
- `mcell/libmcell/api/geometry_object.cpp`, `compartment_utils.cpp`, `geometry_utils.cpp` — consumers.
- `mcell/libmcell/api/mcell4_converter.cpp` — engine builder (likely simplified).
- `mcell/utils/data_model_to_pymcell/{python_generator,data_model_geometry}.cpp` — script emitter.
- `mcell/libmcell/api/bindings.cpp` — add `Vec3`/`IVec3` `__getitem__`/`__len__` + sequence/ndarray ctor.
- **cellblender** (separate repo) — `cellblender_objects.py` and any `vertex_list` consumers.
- Regenerate `generated/*`, `mcell.pyi`, RST docs; update `tests/` baselines as needed.

## 7. Decision required & recommended sequencing
**Open decisions:**
1. **API shape** — C1 / C2 / **C3 (recommended)**. C3 maximizes back-compat (existing models &
   cellblender keep working) while delivering the fast path; it is the lowest-risk way to land the
   right representation.
2. **Precision/dtype policy** (§5.5).
3. **Walls** — `vector<glm::ivec3>` vs flat `vector<int>` (ivec3 keeps `[i][j]` access; recommended).

**Recommended sequencing:**
- Treat this as its **own focused branch/project, separate from the nanobind migration**, and start it
  **after** the `mcell4_dev`/`mcell4_nanobind` test-baseline questions are resolved — because it
  shifts test expectations, which is exactly the churn currently being contained.
- Phase 1: add `Vec3`/`IVec3` list-likeness (`__getitem__`, sequence/ndarray construction) — pure
  addition, no break, independently useful.
- Phase 2: switch `vertex_list`/`wall_list` to `vector<Vec3>`/`vector<ivec3>` with C3 input/output;
  update converter + `data_model_to_pymcell`; regenerate; fix test baselines.
- Phase 3: zero-copy numpy getter/returns; coordinate the optional cellblender numpy fast-path.
- **Do not** ship the option-A binding hack on the nanobind branch in the meantime — leave the current
  correct-but-slower behavior and let this design supersede it.

---

*Authored as a design proposal alongside the nanobind migration work (June 2026). Numbers from
`scratchpad/bench_numpy.py`; citations are `file:line` against the `mcell4_nanobind` tree at writing.*
