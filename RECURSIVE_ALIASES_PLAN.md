# Plan: Support recursive PEP 695 `type` aliases

Branch: `support-recursive-typealiases` (child of `fix-pep695-generic-typeddict-nameerror`).

## Goal

Make self-referential PEP 695 type aliases work end-to-end instead of raising
`RecursionError`:

```python
type JSON = int | str | bool | None | list[JSON] | dict[str, JSON]

msgspec.json.decode(b'{"a": [1, "x", null]}', type=JSON)   # currently RecursionError
msgspec.json.schema(JSON)                                   # currently RecursionError
msgspec.inspect.type_info(JSON)                             # currently RecursionError
```

Chosen design: **always-indirect `AliasInfo`** — every `type X = ...` alias is
represented by a refcounted indirection object cached on the alias *before* its
`__value__` is expanded, exactly mirroring how `StructInfo` breaks struct
recursion. Recursion "just works" because the back-edge lands on the cached
placeholder.

## Why the current code fails (verified)

- Aliases are expanded **inline** in `typenode_origin_args_metadata`
  (`_core.c:4966` bare, `:4922` parametrized): it reassigns `t = alias.__value__`
  and re-enters the unwrapping loop. No visited-set, no cache.
- Container element types are resolved by **fresh nested `TypeNode_Convert`
  calls** from `typenode_from_collect_state` (`:4053` dict, `:4065` fixtuple,
  `:4073` array). A self-referential alias re-expands on every hop until
  `Py_EnterRecursiveCall(" while analyzing a type")` (`:5313`) trips.
- **Hard constraint:** `TypeNode_Free` recursively `PyMem_Free`s child
  `TypeNode*` (owned, non-refcounted raw pointers). A cycle at the raw
  `TypeNode*` level would double-free / infinite-loop. **The back-edge must go
  through a refcounted PyObject.** This is precisely why structs indirect through
  `StructInfo` and why aliases must too.

## Prior art to mirror: `StructInfo`

`StructInfo_Convert_lock_held` (`_core.c:~7022`):
1. `get_msgspec_cache` early-return if already built (`:7037`).
2. Allocate the info object, zero its child slots (`:7077`).
3. **Insert into cache before descending** (`:7089` metaclass slot / `:7092`
   `__msgspec_cache__`), set `cache_set = true` (`:7094`).
4. Loop fields calling `TypeNode_Convert` (`:7101`); a recursive reference hits
   the cache from step 3 and stops.
5. On error after `cache_set`, delete the cache entry (`:7115`).

TypedDict / dataclass / NamedTuple use the same pattern via `__msgspec_cache__`.
Their Info objects are GC-tracked and traversed so the reference cycle is
collectable by Python's GC (not freed by raw recursion).

## ⚠️⚠️ Second finding: aliases must stay TRANSPARENT inside unions

Verified today (`/tmp/alias_union.py`, `/tmp/alias_union2.py`) that aliases are
currently expanded **before** union-invariant checks run, so an alias is fully
transparent to the surrounding union:

- `bytes | IntStr` (where `type IntStr = int | str`) → `TypeError: ... more than
  one str-like type`. The `str` *inside the alias* collides with... itself?
  No — the error shows the alias's members are flattened into the outer union and
  checked as peers.
- `set[str] | Lst` (where `type Lst = list[int]`) → `TypeError: ... more than one
  array-like`. The `list` inside `Lst` is seen by the "one array-like per union"
  invariant.
- `int | Lst` and `int | Flt` decode fine — members flatten and dispatch by
  distinct JSON tokens.

**Implication for the always-indirect design:** if an alias becomes an *opaque*
`MS_TYPE_ALIAS` node, the outer union can no longer see the types hidden inside
it. That would (a) silently bypass the union-invariant checks above (correctness
regression — two array-likes could slip through), and (b) break token-based
union dispatch (the decoder picks a union member by input token; an opaque alias
node has no token class the dispatcher understands).

So we **cannot** make every alias opaque. The alias indirection must only kick in
where it's actually needed to break a cycle. Refined model:

- **Non-recursive alias** (the overwhelmingly common case): keep TODAY's inline
  expansion. Zero behavior change, no opacity, unions keep working. This is also
  the zero-hot-path-cost outcome.
- **Recursive alias:** introduce the `AliasInfo` indirection ONLY on the
  back-edge that would otherwise recurse forever. i.e. the *first* expansion of
  `type JSON = ... list[JSON] ...` proceeds inline as a union; when `list[JSON]`
  re-references `JSON`, THAT inner reference resolves to the cached `AliasInfo`
  placeholder instead of re-expanding.

This is effectively the "detect-and-indirect only recursive aliases" option
that was listed (and not chosen) in the original design question — the union
semantics force it. The module-level cache dict (option 1) is still the storage
mechanism; what changes is that the top-level/non-recursive expansion is NOT
wrapped in an alias node, only the recursive back-reference is.

### DECISION (locked): container-guarded recursion only

Support recursion that passes **through a container** (list/dict/set/tuple, or a
struct field — anything that gives the recursive `TypeNode` its own nested slot
that is not a union peer). Leave *unguarded* self-references erroring cleanly.

Rationale — the hard cases are exactly the degenerate ones:

| Alias | Recurses through | Inhabitable? | Decision |
|---|---|---|---|
| `type JSON = ... \| list[JSON] \| dict[str,JSON]` | list/dict (empty = base case) | yes | **support** |
| `type Tree[T] = tuple[T, list[Tree[T]]]` | list | yes | **support** |
| `type A = list[B]; type B = list[A]` | list | yes | **support** |
| `type Ex = Ex \| None` | nothing (direct union member) | only `None` | **error cleanly** |
| `type Ex = tuple[Ex, int]` | fixtuple, no base case | no (uninhabitable) | **error cleanly** |

`type Ex = Ex | None` collapses to just `None` (the recursive branch adds no
inhabitants); `type Ex = tuple[Ex, int]` has no base case so no finite value
satisfies it. Neither is a real-world type — they exist only as
"error-cleanly" torture inputs. This mirrors recursive **Structs**, which only
ever recurse behind a named field (structurally = "behind a container") and thus
never hit union opacity.

Practically: the `AliasInfo` back-edge is only ever installed when the recursive
reference is reached inside a container element `TypeNode_Convert` (dict/array/
fixtuple/struct-field), never as a direct union member. A back-reference that
surfaces as a bare union member remains a `RecursionError` (kept by
`test_recursive_typealias_errors`).

**Nuance — fixtuple counts as a container.** `tuple[Ex, int]` recurses through a
fixtuple element, which IS a nested `TypeNode` slot. So `type Ex = tuple[Ex, int]`
(and the generic `tuple`-based cases 3–5 in `test_recursive_typealias_errors`)
will now **build successfully** — the cycle breaks via `AliasInfo`. They're
uninhabitable, so decode fails at runtime (`ValidationError` once finite input
runs out of nesting), not at build. Net effect when the feature lands:
`test_recursive_typealias_errors` narrows to only the genuinely-unguarded
`type Ex = Ex | None` (still build-time `RecursionError`); the tuple cases move to
"builds, but no finite value decodes." Finalize these expectations empirically
once the C lands rather than guessing now.

## ★ Linchpin architecture: intercept at the `TypeNode_Convert` boundary

The single key decision that makes everything else fall out correctly:

> **Aliases are indirected (→ `AliasInfo`/`MS_TYPE_ALIAS`) ONLY at the
> `TypeNode_Convert` boundary. Inline collection
> (`typenode_collect_type` / `typenode_origin_args_metadata`) keeps expanding
> aliases inline, unchanged.**

Why this works — the two paths a type reference can travel:

1. **Inline path** — union members. The union branch (`:5223`) collects each
   member via `typenode_collect_type(state, arg)` on the *shared* state;
   `typenode_origin_args_metadata` expands an alias member's `__value__` inline.
   → **Union transparency preserved for free.** `bytes | IntStr` still flattens
   and still fires the "one str-like type" invariant; `set[str] | Lst` still
   fires "one array-like". No change to these.
2. **Fresh-conversion path** — the root type, and every container element
   (dict key/val `:4053`/`:4056`, fixtuple `:4065`, array `:4073`). These call a
   *fresh* `TypeNode_Convert(element)`. → This is where we intercept: if the
   object is a `TypeAliasType` (or a parametrized alias whose origin is one),
   build a standalone `{types: MS_TYPE_ALIAS, details[0] = AliasInfo}` node
   instead of expanding.

Union members never cross the fresh-conversion boundary, so intercepting there
NEVER touches a union member. An `MS_TYPE_ALIAS` node is therefore always
standalone (never OR-ed with other type bits) → detail-slot logic is trivial
(exclusive, like `MS_TYPE_CUSTOM`).

### Why the degenerate cases still behave exactly as decided

- `type Ex = Ex | None`: `TypeNode_Convert(Ex)` → `AliasInfo_Convert(Ex)` →
  `info->type = TypeNode_Convert(Ex|None)` → union branch collects `Ex` *inline*
  via `typenode_collect_type` → re-expands `Ex.__value__ = Ex|None` inline →
  inline union recursion → `Py_EnterRecursiveCall` (`:5219`) → **RecursionError**.
  The back-edge is a direct union member, never crosses the fresh-conversion
  boundary, so it is never indirected. ✅ still errors cleanly.
- `type Ex = tuple[Ex, int]`: the `Ex` back-edge is a *fixtuple element* → fresh
  `TypeNode_Convert(Ex)` → hits the cached placeholder → **builds**; uninhabitable
  so decode fails at runtime. ✅ matches the nuance above.
- `type JSON = ... | list[JSON] | ...`: `list[JSON]` element → fresh
  `TypeNode_Convert(JSON)` → cached placeholder → terminates, builds. ✅

### `AliasInfo_Convert(alias)` sequence (module-dict cache)

1. `dict[alias]` lookup → hit returns cached `AliasInfo` (handles both the
   recursive back-edge and cross-references between distinct types).
2. miss: alloc `AliasInfo` with `type = NULL`; `dict[alias] = info` (placeholder
   inserted *before* expansion).
3. `info->type = TypeNode_Convert(alias.__value__)` — for a parametrized alias,
   substitute args first (`alias.__value__[args]`, cf. `:4922`). Recursive
   container references to `alias` resolve to the placeholder from step 1.
4. return `info`. `->type` is non-NULL before any decode runs.
5. error after insert → remove `dict[alias]` (mirror `StructInfo`'s cache
   rollback at `:7115`).

Note: this indirects *non-recursive* aliases that appear as container elements /
root too (e.g. `list[Pair]` for a non-recursive `Pair`). That's harmless — one
extra pointer hop, no opacity issue since it's not a union member — and keeps the
logic uniform. Union-member aliases remain fully inlined.

## ⚠️ Revision: `__msgspec_cache__`-on-the-alias does NOT work

Verified on 3.12 and 3.13 (`/tmp/alias_attr.py`, `/tmp/cache_strategy.py`,
`/tmp/key_props.py`):

- **`TypeAliasType`** (bare `type JSON = ...`) has **no `__dict__`** and rejects
  attribute assignment: `AttributeError: ... no __dict__ for setting new
  attributes`. So we cannot stash a placeholder on it the way structs do.
- **Parametrized `Pair[int]`** is a `types.GenericAlias` (NOT the interned
  `typing._GenericAlias` that generic *structs* produce). It is likewise
  attribute-less, AND it is **transient**: `Pair[int] is Pair[int]` is `False`.
  So there's no stable object to hang an attribute on either.
  - (This is exactly why generic-alias *structs* can use `__msgspec_cache__`:
    `S[int]` is a `typing._GenericAlias`, which is interned and writable. Aliases
    are a different, unwritable type — the struct precedent does not transfer.)

What IS true of aliases:
- Bare `JSON` has **stable identity** (`JSON is ns['JSON']`) and is **hashable**.
- Parametrized `Pair[int]` is **hashable** and **value-equal**
  (`Pair[int] == Pair[int]` is `True`), so it works as a plain-dict key even
  though each instance is a fresh object.
- Neither is **weakref-able** (`TypeError: cannot create weak reference`), so a
  `WeakKeyDictionary` / `WeakValueDictionary` is out.

### Revised caching strategy

Use a **module-level cache dict keyed by the alias object**, not an attribute:

- Store `alias -> AliasInfo` (and `parametrized_alias -> AliasInfo`) in a dict
  held in module state (like the existing `struct_lookup_cache` at `:485`, which
  is already a module-level cache rather than a per-type attribute).
- Insert the placeholder `AliasInfo` into this dict **before** expanding
  `__value__`; the recursive `TypeNode_Convert` re-entry looks up the dict, finds
  the in-flight `AliasInfo`, and stops. Same cycle-break, different storage.
- Because aliases aren't weakref-able, entries are **strong refs** and live for
  the process (or until an explicit cap/eviction, cf. the 64-entry LRU on
  `struct_lookup_cache` at `:4772`). Acceptable — a program has a bounded set of
  alias types — but note it in the risks.
- Keying: bare alias by identity works; parametrized by value-equality works.
  Confirm `hash`/`==` are consistent enough that `Pair[int]` and `Pair[str]`
  don't collide (they hash differently — verified) and that
  `Pair[int] == Pair[int]` across instances is stable (verified True).

### DECISION (locked): module-level dict in `MsgspecState`

Chosen: **option 1 — module-level cache dict in `MsgspecState`**, keyed by the
alias object, mirroring `struct_lookup_cache`. Needs GC traverse/clear of that
dict and thread-safety via the existing critical-section pattern. (Option 2,
Python-side pre-resolution, was rejected to keep alias resolution next to the
type-collection machinery.)

Everything downstream of "how the placeholder is stored" (AliasInfo object,
MS_TYPE_ALIAS flag, dispatch, traverse) is unchanged from steps A2/A5/A6/A7.

## Layers affected

| Layer | File | Recursive today? | Work |
|---|---|---|---|
| decode | `_core.c` | ❌ RecursionError | **new AliasInfo + flag + dispatch** |
| convert | `_core.c` (shared TypeNode) | ❌ | covered by the same node |
| encode | `_core.c` | ✅ already (object-driven, not type-driven) | none |
| `to_builtins` | `_core.c` | ✅ already (object-driven) | none |
| `inspect.type_info` | `inspect.py` | ❌ | ref/cache for aliases |
| `json.schema` | `_json_schema.py` | ❌ | **NOT transitive after all** — see finding below |

### ⚠️⚠️⚠️ Third finding: schema is NOT "transitive/free" for union aliases

Original assumption (❌ wrong): "fixing inspect fixes schema, it already emits
`$ref` for cycles." Verified false today:

- `_collect_component_types` / `to_schema` only emit a `$ref` for types with a
  `.cls` attribute — i.e. **nominal** types: Struct, TypedDict, Dataclass,
  NamedTuple, Enum (`_json_schema.py:129-137`, `:235-238`). Recursive *structs*
  work in schema precisely because they're nominal.
- A recursive alias to a **union/collection** (`type JSON = int | str |
  list[JSON]`) has **no `.cls`**. `to_schema` walks it structurally
  (`UnionType`→`CollectionType`→…) with **no cycle guard**, so even a correctly
  cyclic `inspect` graph → infinite recursion in `to_schema`.

Implication: real `json.schema` support for union/collection aliases needs the
schema layer to treat a recursive alias as a **nameable `$ref` component**. That
means either (a) a dedicated `mi.AliasType` node carrying the alias's name/cls so
`_collect_component_types` + `name_map` + `to_schema`'s `$ref` short-circuit pick
it up, or (b) a generic cycle-guard in `to_schema`/`collect` for non-nominal
nodes. Both are meaningfully larger than "free."

**DECISION (locked): full parity.** Implement (a) — a dedicated `mi.AliasType`
node carrying the alias object as `.cls`, so `_collect_component_types`,
`_build_name_map`, and `to_schema`'s `$ref` short-circuit treat a recursive
alias as a nameable `$defs` component exactly like a recursive Struct. This also
solves the `inspect` cycle: `AliasType` is a mutable placeholder cached in
`_Translator.cache` (keyed by the alias) *before* its `type` is filled, breaking
the cycle the same way `StructType` does.

## Implementation steps

### A. `_core.c` — new `AliasInfo` indirection

1. **Flag:** add `MS_TYPE_ALIAS` to the `MS_TYPE_*` bitset (next free bit;
   audit the 64-bit space near `:2815-2854`).
2. **Info object:** define `AliasInfo` (PyObject) holding a single
   `TypeNode *type` (the expanded value's node) + GC support
   (`tp_traverse`/`tp_clear`/`dealloc`), analogous to the lighter Info types.
   Note it must be `Py_TPFLAGS_HAVE_GC` and traverse into its `TypeNode` child's
   PyObjects so the alias↔value cycle is collectable.
3. **Detect aliases during collection:** in `typenode_origin_args_metadata`,
   instead of inline-expanding a `TypeAliasType`, record it (like
   `typenode_collect_struct` records the class) OR route through a new
   `typenode_collect_alias`. Decision to finalize during impl: cleanest is a
   dedicated collector that sets `MS_TYPE_ALIAS` + stashes the alias object.
   ⚠️ Must still support the existing non-recursive expansion semantics and the
   parametrized-generic-alias substitution (`alias.__value__[args]`, `:4922`).
4. **`AliasInfo_Convert`:** mirror `StructInfo_Convert`'s *shape*, but store the
   placeholder in a **module-level dict keyed by the alias object** (see
   "Revised caching strategy" above), NOT `__msgspec_cache__` (aliases have no
   `__dict__`). Sequence: dict lookup → alloc → insert placeholder into dict →
   `TypeNode_Convert(alias.__value__ [substituted])` → fill `info->type`.
   Back-references resolve to the in-flight `AliasInfo`. Handle error-path
   dict-entry removal. ⚠️ storage form pending the OPEN DECISION above.
5. **`TypeNode_get_alias_info`** accessor (pop-count slot indexing like
   `TypeNode_get_struct_info` `:3140`).
6. **Decode dispatch:** at each decode entry (json + msgpack), when
   `MS_TYPE_ALIAS` is set, fetch `AliasInfo`, and recurse the decoder on
   `info->type`. Same for the **convert** dispatcher.
7. **`TypeNode_traverse` / `TypeNode_Free`:** treat the `AliasInfo*` slot as a
   PyObject (Py_DECREF), NOT as a raw `TypeNode*` (so no raw-recursion cycle).
8. **Cache lifecycle / GC:** the module-level alias cache dict must be added to
   `MsgspecState`, GC-traversed and cleared alongside the other module-state
   members (cf. the module-state clearing already done for `struct_lookup_cache`
   and friends), and guarded by the existing critical-section pattern for
   free-threaded builds.

### B. `inspect.py`

- In `_origin_args_metadata` (`:675`, `:685`) aliases are expanded inline.
  Introduce alias handling in `_Translator` that mirrors the struct `self.cache`
  approach (`:930`): insert a placeholder (a `Metadata`/ref node or a lazily
  filled type) keyed by the alias object *before* translating `__value__`, so a
  recursive alias resolves to the cached node.
- Confirm `type_info` returns a structure that can express the cycle (the
  existing `type_info` types already form Python object cycles for recursive
  structs — reuse that shape).

### C. `json.schema`

- No direct change expected: `schema` → `mi.multi_type_info` →
  `_collect_component_types` already emits `$ref`/`$defs` for shared/recursive
  component types (`_json_schema.py:103-119`). Once `inspect` yields a cyclic
  type graph for aliases, schema should follow. **Verify**, and add a
  component-naming case if aliases need a stable `$ref` name.

### D. Tests

- **Rewrite** `test_recursive_typealias_errors`
  (`test_common.py:4768-4785`) — it currently *asserts* `RecursionError`. New
  behavior: those same 5 sources should decode/roundtrip correctly. Keep a
  genuinely-unrepresentable case (if any) erroring cleanly.
- Add decode + encode roundtrip tests for a canonical `JSON` alias across both
  protocols (`proto` fixture).
- Add `inspect.type_info` and `json.schema` tests for a recursive alias
  (schema should contain a `$ref` back to the alias's `$def`).
- Mutual recursion: `type A = list[B]; type B = list[A]`.
- Generic recursive alias: `type Tree[T] = tuple[T, list[Tree[T]]]` +
  `Tree[int]`.
- Guard: parametrized-arity error (`Pair[int, int]`) still raises `TypeError`
  (`test_typealias_parametrized_generic_too_many_parameters` must still pass).

## Risks / open questions

- **Hot-path cost:** one extra pointer-hop + branch per alias-typed value at
  decode time. Paid only by alias-typed fields; acceptable per design decision.
- **RESOLVED — cache storage:** `__msgspec_cache__` on the alias is impossible
  (`TypeAliasType`/`types.GenericAlias` have no `__dict__` and aren't
  weakref-able). Superseded by the module-level dict keyed by alias object (see
  the ⚠️ Revision section). Storage form is the OPEN DECISION above.
- **Cache is a strong ref → aliases are pinned for process lifetime** (not
  weakref-able). Bounded in practice; consider the same LRU cap as
  `struct_lookup_cache` if unbounded growth is a concern.
- **Generic aliases keying:** bare alias keys by identity; `Alias[int]` keys by
  value-equality (`==`/`hash`, verified stable and non-colliding across
  `Alias[int]`/`Alias[str]`). Each `Alias[int]` is a *fresh transient* object,
  so identity-keying would miss — must use value-equality.
- **This is a C-extension change** → requires `just rebuild=1 test`. Every test
  run in this branch must rebuild.

## Build/verify workflow

- `just rebuild=1 test` after each C change.
- `just test-all`, `just test-typing`, `just check`, `just doc-build` before PR
  (same gate as the parent branch).
- Verify on Python 3.12, 3.13, 3.14.
