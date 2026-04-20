---
date: 2026-04-20
source-tasks: token-savior-5bn, token-savior-psl, token-savior-4p1
version: 1.0
---
# Code Map — token-savior Leverage Points

Five components that control the system's core behavior. Each entry shows exact file location, the design principle at work, the 10-20 LOC "essence" of that component, and why that specific block governs overall behavior.

---

## Component: `_dispatch_tool` — Four-category handler dispatch

File: [`server.py:304-352`](../../src/token_savior/server.py#L304)

Nguyên lý: Chain of Responsibility (GoF) + Open/Closed Principle (SOLID)

Code tinh hoa:
```python
def _dispatch_tool(name: str, arguments: dict[str, Any], record_symbol: str) -> list[types.TextContent]:
    meta_handler = _META_HANDLERS.get(name)
    if meta_handler is not None:
        return meta_handler(arguments)

    mem_handler = _MEMORY_HANDLERS.get(name)
    if mem_handler is not None:
        return [TextContent(type="text", text=mem_handler(arguments))]

    slot, err = s._slot_mgr.resolve(arguments.get("project"))
    if err:
        return [TextContent(type="text", text=f"Error: {err}")]

    handler = _SLOT_HANDLERS.get(name)
    if handler is not None:
        return _count_and_wrap_result(slot, name, arguments, handler(slot, arguments))

    qfn_handler = _QFN_HANDLERS.get(name)
    if qfn_handler is not None:
        ...
        result = qfn_handler(slot.query_fns, arguments)
        return _count_and_wrap_result(slot, name, arguments, result)

    return [TextContent(type="text", text=f"Error: unknown tool '{name}'")]
```

Lý do tinh hoa: This is the single routing decision point for all 105 MCP tools — every tool call passes through this exact if/elif chain in priority order (META → MEMORY → SLOT → QFN). Adding a new tool category requires only adding a new dict and one `if` branch here; no other code changes.

---

## Component: Handler dict aggregation — `server_handlers/__init__.py`

File: [`server_handlers/__init__.py:42-87`](../../src/token_savior/server_handlers/__init__.py#L42)

Nguyên lý: Fail-Fast (import-time collision detection) + Single Source of Truth

Code tinh hoa:
```python
def _merge_disjoint(label: str, *parts: dict[str, Any]) -> dict[str, Any]:
    """Merge handler dicts; raise on duplicate tool names within a category."""
    merged: dict[str, Any] = {}
    for part in parts:
        overlap = merged.keys() & part.keys()
        if overlap:
            raise RuntimeError(
                f"{label}: duplicate tool name(s) across handler modules: "
                f"{sorted(overlap)}"
            )
        merged.update(part)
    return merged

META_HANDLERS: dict[str, Any] = _merge_disjoint(
    "META_HANDLERS", _STATS_HANDLERS, _MEMORY_ADMIN_HANDLERS, _PROJECT_HANDLERS,
)
MEMORY_HANDLERS: dict[str, Any] = dict(_MEMORY_HANDLERS_RAW)
SLOT_HANDLERS: dict[str, Any] = _merge_disjoint(
    "SLOT_HANDLERS", _GIT_HANDLERS, _CHECKPOINT_HANDLERS, _EDIT_HANDLERS,
    _TESTS_HANDLERS, _PROJECT_ACTION_HANDLERS, _ANALYSIS_HANDLERS,
)
QFN_HANDLERS: dict[str, Any] = dict(_QFN_HANDLERS)
```

Lý do tinh hoa: `_merge_disjoint` enforces at import time that no tool name appears in two handler categories — a silent shadowing bug would be invisible at runtime. The four exported dicts are the authoritative dispatch tables consumed by `_dispatch_tool`.

---

## Component: `SlotManager.ensure()` + `build()` — Lazy index initialization

File: [`slot_manager.py:104-182`](../../src/token_savior/slot_manager.py#L104)

Nguyên lý: Lazy Evaluation + Strategy (incremental vs. full rebuild decision)

Code tinh hoa:
```python
def ensure(self, slot: _ProjectSlot) -> None:
    if slot.indexer is not None:
        return                          # already initialized — fast path

    cached_index = slot.cache.load()
    if cached_index is not None and slot.is_git and cached_index.last_indexed_git_ref:
        current_head = get_head_commit(root)
        if current_head == cached_index.last_indexed_git_ref:
            # exact git-ref hit — restore from cache, skip full re-index
            slot.indexer = ProjectIndexer(root)
            slot.indexer._project_index = cached_index
            slot.query_fns = create_project_query_functions(cached_index)
            slot.cache_gen += 1
            return

        changeset = get_changed_files(root, cached_index.last_indexed_git_ref)
        total_changes = len(changeset.modified) + len(changeset.added) + len(changeset.deleted)
        if not changeset.is_empty and total_changes <= 20:
            # small delta — accept stale cache, mark for incremental patch
            slot.query_fns = create_project_query_functions(cached_index)
            slot.cache_gen += 1
            return

    self.build(slot)                    # fallback: full AST re-index
```

Lý do tinh hoa: Three-tier cache decision (fresh git-ref hit → incremental ≤20 files → full rebuild) means cold-start cost is paid only once per project per git ref; subsequent tool calls on the same slot return instantly via the `slot.indexer is not None` guard on line 106.

---

## Component: `LazyLines.__getitem__` — Demand-driven file content loading

File: [`models.py:92-147`](../../src/token_savior/models.py#L92)

Nguyên lý: Virtual Proxy pattern (GoF) — defers expensive I/O until first access

Code tinh hoa:
```python
class LazyLines:
    """Defers reading file lines from disk until first access.
    Saves ~80 % idle RAM for cached indexes."""

    __slots__ = ("_data", "_root_path", "_rel_path")

    @property
    def _loaded(self) -> list[str]:
        if self._data is None:
            full_path = os.path.join(self._root_path, self._rel_path)
            try:
                with open(full_path, "r", encoding="utf-8", errors="replace") as f:
                    self._data = f.read().splitlines()
            except OSError:
                self._data = []
        return self._data

    def __getitem__(self, key):
        return self._loaded[key]        # triggers load on first [] access

    def __iter__(self):
        return iter(self._loaded)

    def __len__(self):
        return len(self._loaded)
```

Lý do tinh hoa: Every `ProjectFileEntry.lines` field in the cached index is a `LazyLines` instance; file content is never read from disk unless a tool actually requests it. When a project has thousands of files loaded from cache, `_data is None` holds for nearly all of them, keeping memory near zero for idle files.

---

## Component: `hybrid_search()` + `rrf_merge()` — BM25 + vector RRF fusion

File: [`memory/search.py:27-142`](../../src/token_savior/memory/search.py#L27)

Nguyên lý: Strategy (pluggable retrieval back-ends) + Graceful Degradation

Code tinh hoa:
```python
RRF_K = 60  # Reciprocal Rank Fusion constant (Cormack et al. 2009)

def rrf_merge(*ranked_lists, limit=20, k=RRF_K):
    scores: dict[int, float] = {}
    metadata: dict[int, dict] = {}
    for rows in ranked_lists:
        for rank, row in enumerate(rows, start=1):
            oid = row.get("id")
            scores[oid] = scores.get(oid, 0.0) + 1.0 / (k + rank)
            if oid not in metadata:
                metadata[oid] = row
    ranked = sorted(scores.items(), key=lambda kv: kv[1], reverse=True)
    return [dict(metadata[oid]) | {"_rrf_score": round(score, 6)}
            for oid, score in ranked[:limit]]

def hybrid_search(conn, fts_rows, query, project_root, *, limit=20, ...):
    if not VECTOR_SEARCH_AVAILABLE:
        return fts_rows[:limit]         # degrade gracefully: FTS-only
    vec = embed(query)
    if vec is None:
        return fts_rows[:limit]
    vec_rows = vec_search_rows(conn, vec, project_root, limit=limit * 2, ...)
    if not vec_rows:
        return fts_rows[:limit]
    return rrf_merge(fts_rows, vec_rows, limit=limit)
```

Lý do tinh hoa: `rrf_merge` combines the BM25 keyword rank-list and the k-NN vector rank-list using `1/(k + rank)` scoring — a rank-position formula that is scale-invariant across dissimilar scoring systems. `hybrid_search` degrades silently to pure FTS when `sqlite-vec` is absent, so the memory engine ships with zero hard dependencies on the vector stack.

---

## Component: `_PROFILE_EXCLUDES` — Profile-based tool manifest filtering

File: [`server.py:131-151`](../../src/token_savior/server.py#L131)

Nguyên lý: Open/Closed Principle — TOOLS list filtered by exclusion sets, not branching logic per tool

Code tinh hoa:
```python
_PROFILE_EXCLUDES: dict[str, set[str]] = {
    "full": set(),
    "core": set(_MEMORY_HANDLERS) | set(_META_HANDLERS),
    "nav":  set(_MEMORY_HANDLERS) | set(_META_HANDLERS) | set(_SLOT_HANDLERS),
    "lean": _LEAN_EXCLUDES,
    "ultra": set(TOOL_SCHEMAS) - _ULTRA_INCLUDES,
}

_PROFILE = os.environ.get("TOKEN_SAVIOR_PROFILE", "full").lower()
if _PROFILE not in _PROFILE_EXCLUDES:
    _PROFILE = "full"

if _PROFILE != "full":
    _excluded = _PROFILE_EXCLUDES[_PROFILE]
    TOOLS = [t for t in TOOLS if t.name not in _excluded]
```

Lý do tinh hoa: A single dict maps each profile name to the set of tool names hidden from `list_tools` — the handlers remain registered and callable by name regardless of profile. This means the same server binary advertises anywhere from 17 (`ultra`) to 106 (`full`) tools to the MCP client without any code-path changes; manifest size is the only variable controlled here.
