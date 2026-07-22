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

## Layers affected

| Layer | File | Recursive today? | Work |
|---|---|---|---|
| decode | `_core.c` | ❌ RecursionError | **new AliasInfo + flag + dispatch** |
| convert | `_core.c` (shared TypeNode) | ❌ | covered by the same node |
| encode | `_core.c` | ✅ already (object-driven, not type-driven) | none |
| `to_builtins` | `_core.c` | ✅ already (object-driven) | none |
| `inspect.type_info` | `inspect.py` | ❌ | ref/cache for aliases |
| `json.schema` | `_json_schema.py` | ❌ | **transitive** — built on `inspect.multi_type_info`; fixing inspect fixes schema (it already emits `$ref` for cycles) |

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
4. **`AliasInfo_Convert`:** mirror `StructInfo_Convert`:
   `get_msgspec_cache` → alloc → cache on alias via `__msgspec_cache__` →
   `TypeNode_Convert(alias.__value__ [substituted])` → fill `info->type`.
   Back-references resolve to the cached placeholder. Handle error-path cache
   deletion.
5. **`TypeNode_get_alias_info`** accessor (pop-count slot indexing like
   `TypeNode_get_struct_info` `:3140`).
6. **Decode dispatch:** at each decode entry (json + msgpack), when
   `MS_TYPE_ALIAS` is set, fetch `AliasInfo`, and recurse the decoder on
   `info->type`. Same for the **convert** dispatcher.
7. **`TypeNode_traverse` / `TypeNode_Free`:** treat the `AliasInfo*` slot as a
   PyObject (Py_DECREF), NOT as a raw `TypeNode*` (so no raw-recursion cycle).
8. **msgspec_cache invalidation / GC:** ensure alias `__msgspec_cache__` entries
   participate in the same overwrite-detection (`get_msgspec_cache` `:4131`) and
   are cleared on module teardown paths already handled for other Info types.

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
- **`__msgspec_cache__` on `TypeAliasType`:** confirm the object permits
  arbitrary attribute assignment (structs use a metaclass slot; TypedDict etc.
  use `__msgspec_cache__` on the type — a `TypeAliasType` is a plain object, so
  this should work, but verify on 3.12/3.13/3.14).
- **Generic aliases:** the placeholder must be keyed such that `Alias` and
  `Alias[int]` don't collide incorrectly. Structs solve this by caching
  `StructInfo` on the *generic alias* via `__msgspec_cache__` while plain
  structs use the metaclass slot — follow the same split.
- **This is a C-extension change** → requires `just rebuild=1 test`. Every test
  run in this branch must rebuild.

## Build/verify workflow

- `just rebuild=1 test` after each C change.
- `just test-all`, `just test-typing`, `just check`, `just doc-build` before PR
  (same gate as the parent branch).
- Verify on Python 3.12, 3.13, 3.14.
