---
date: 2026-04-20
source-tasks: token-savior-4p1, token-savior-psl, token-savior-5bn, token-savior-1ip, token-savior-yll
version: 1.0
---

# Design Principles — token-savior

Aggregated from 5 tasks (4p1 → psl → 5bn + 1ip parallel → yll → this report). Tổng hợp toàn bộ kiến trúc, leverage points, design decisions, và kỹ năng chuyển giao.

---

## Section 1 — Architecture Overview

**Xem diagrams đầy đủ tại:** [token-savior-4p1 worklog](../tasks/token-savior-4p1-contextual-awareness-sad-diagrams.md#diagram-1)

3 diagrams đã được tạo trong task 4p1:
- **Sequence Diagram:** MCP request → [`server.py:401`](../../src/token_savior/server.py#L401) (`call_tool`) → [`server.py:304`](../../src/token_savior/server.py#L304) (`_dispatch_tool`) → 4 handler categories → result
- **Component Diagram:** 7 nodes — server.py, server_handlers/__init__.py, SlotManager, ProjectIndexer, CacheManager, memory/search.py, SQLite+FTS5+sqlite-vec
- **Use-Case Diagram:** 8 use cases — code navigation, memory read/write, edit, git, tests, project stats, profile selection

**Bảng tóm tắt 8 luồng (từ 4p1):**

| Flow | Actors | Trigger | Entry Point | Output |
|------|--------|---------|-------------|--------|
| Tool call dispatch | CC → server → handler | MCP `call_tool` | [`server.py:401`](../../src/token_savior/server.py#L401) | TextContent JSON |
| Index build (lazy) | SlotManager → Indexer → Cache | First QFN call | [`slot_manager.py:104`](../../src/token_savior/slot_manager.py#L104) (`ensure()`) | JSON cache + QueryEngine |
| Incremental update | SlotManager → git diff | Cache stale ≤20 files | [`slot_manager.py:104`](../../src/token_savior/slot_manager.py#L104) | Partial index patch |
| Memory search | Client → hybrid_search | `memory_search` call | [`memory/search.py:107`](../../src/token_savior/memory/search.py#L107) | BM25+vector → RRF ranked |
| Memory progressive | Client (3 layers) | L1 → L2 → L3 | `memory_index` → `memory_search` → `memory_get` | 15 → 60 → 200 tokens |
| Profile filtering | env var at startup | `TOKEN_SAVIOR_PROFILE` | [`server.py:139`](../../src/token_savior/server.py#L139) | 105 → 17 tools (ultra) |
| Session rollup | Hooks lifecycle | SessionEnd | `hooks/memory-hooks-config.json` | Compact summary saved |
| Symbol staleness | CacheManager | Content hash change | [`memory/consistency.py`](../../src/token_savior/memory/consistency.py) | Linked obs invalidated |

---

## Section 2 — Core Components

6 components — focus: WHAT each component does (WHERE + leverage → Section 3).

**1. `_dispatch_tool()` — Central tool router**
[`server.py:304`](../../src/token_savior/server.py#L304) — 4 if/elif blocks routing all 105 MCP tools into categories (META → MEMORY → SLOT → QFN), priority-ordered. Single point of dispatch; không có alternative routing path. Remove this → all tools unreachable.

**2. Handler category dicts — Authoritative dispatch tables**
[`server_handlers/__init__.py:42`](../../src/token_savior/server_handlers/__init__.py#L42) — 4 dicts (`META_HANDLERS`, `MEMORY_HANDLERS`, `SLOT_HANDLERS`, `QFN_HANDLERS`) assembled via `_merge_disjoint()`. Import-time collision detection raises `RuntimeError` nếu tool name bị duplicate across categories.

**3. `SlotManager.ensure()` — Lazy index initialization gate**
[`slot_manager.py:104`](../../src/token_savior/slot_manager.py#L104) — 3-tier cache decision: (1) git-ref exact hit → restore cache, (2) ≤20 changed files → accept stale, (3) fallback → full rebuild. All SLOT + QFN tools call `ensure()` before any operation.

**4. `LazyLines` — Demand-driven file content loading**
[`models.py:92`](../../src/token_savior/models.py#L92) — Virtual Proxy pattern. Every `ProjectFileEntry.lines` field holds a `LazyLines` instance; disk I/O deferred until first `__getitem__` access. Eliminates constant ~400MB RAM for 10K-file project when tools don't need file content.

**5. `hybrid_search()` + `rrf_merge()` — Memory recall engine**
[`memory/search.py:107`](../../src/token_savior/memory/search.py#L107) — BM25 FTS5 keyword-rank + optional sqlite-vec k-NN → RRF fusion (`1/(k + rank)`, k=60). Degrades gracefully to FTS-only when sqlite-vec absent. All 21 memory tools route through here.

**6. `_PROFILE_EXCLUDES` — Tool manifest filter**
[`server.py:131`](../../src/token_savior/server.py#L131) — Dict mapping profile name → excluded tool set. Evaluated once at startup. Same handlers registered regardless of profile; only `list_tools()` manifest changes. Full: 105 tools. Ultra: 17 tools (~2,800 vs ~10,950 tokens injected into MCP client).

---

## Section 3 — Leverage Points

Định nghĩa: thay đổi ≤5 LOC ở đây → affect ≥10 tool behaviors. **Dedup với Section 2:** leverage points là SUBSET của core components — góc nhìn khác là WHERE + LEVERAGE METRIC thay vì WHAT.

| Component | File:line | Leverage test (≤5 LOC → N behaviors) | Metric |
|-----------|-----------|---------------------------------------|--------|
| Handler category dicts | [`server_handlers/__init__.py:42`](../../src/token_savior/server_handlers/__init__.py#L42) | Thêm 1 dict entry → route 1 tool vào đúng tier; thay category → toàn bộ context requirement thay đổi cho tool đó | 1 line → 1 tool behavior; restructure dict → all 105 affected |
| `_PROFILE_EXCLUDES` | [`server.py:131`](../../src/token_savior/server.py#L131) | 1 line thay đổi trong profile list → toàn bộ profile manifest khác (e.g., ultra: 17→18 tools, MCP client phải reconnect) | 1 line → N tools in profile |
| `rrf_merge()` RRF constant | [`memory/search.py:27`](../../src/token_savior/memory/search.py#L27) | Thay `RRF_K = 60` → 1 line → ranking của tất cả memory search results thay đổi toàn bộ | 1 constant → all memory recall results |
| `_EXTENSION_MAP` + `_ANNOTATOR_MAP` | [`annotator.py:25`](../../src/token_savior/annotator.py#L25), [`annotator.py:65`](../../src/token_savior/annotator.py#L65) | Thêm 2 dict entries + 1 import → 1 ngôn ngữ mới được index toàn bộ project; 0 core file changes | 3 lines → full language support |
| `LazyLines.__slots__` + `_loaded` | [`models.py:92`](../../src/token_savior/models.py#L92) | Thay lazy loading thành eager → 400MB RAM constant overhead; cache deserialization behavior thay đổi cho tất cả files | Strategy switch → all indexed files |

**Key insight từ psl analysis:** Handler category dicts và `_PROFILE_EXCLUDES` là leverage points mạnh nhất vì chúng quyết định toàn bộ ROUTING và VISIBILITY của 105 tools — thay đổi ở đây không cần sửa bất kỳ business logic nào.

---

## Section 4 — Design Principles & Rationale

5+3 decisions đầy đủ tại: [token-savior-1ip worklog](../tasks/token-savior-1ip-deep-research.md#decision-1-4-category-handler-dispatch-metamemoryslotqfn). Summary dưới đây.

### Decision 1: 4-Category Handler Dispatch
**Nguyên lý:** Command Pattern + SRP — 4 call signatures khác nhau (no context / string / slot / query_fns) không thể flatten vào 1 registry mà không lặp null-check ở 105 handlers. Import-time guard `_merge_disjoint()` tại [`server_handlers/__init__.py:56`](../../src/token_savior/server_handlers/__init__.py#L56) phát hiện duplicate tool name trước runtime.
**Industry ref:** LSP capabilities dict — `textDocument/hover` vs `workspace/symbol` vs `initialize` có context requirement khác nhau, cùng dispatch-tier pattern.

### Decision 2: 3-Layer Memory Progressive Disclosure
**Nguyên lý:** Cost-aware lazy loading — 20 results × 200 tokens = 4,000 tokens eager vs 20 × 15 = 300 tokens shortlist → 93% token saving. Full content only at Layer 3 `memory_get`. Code: [`memory/search.py:107`](../../src/token_savior/memory/search.py#L107).
**Industry ref:** Elasticsearch highlight→source, Google SERP snippet→full page — same 3-tier cost structure.

### Decision 3: LazyLines Disk-Backed
**Nguyên lý:** Virtual Proxy pattern — constraint math: 10K files × 500 LOC × 80B = 400MB RAM constant overhead; 95% of tool calls never need file content. Code: [`models.py:92`](../../src/token_savior/models.py#L92).
**Industry ref:** Git pack objects (inflate on access), SQLite B-tree pages (load on traverse), LMDB copy-on-write pages.

### Decision 4a: SQLite WAL Mode
**Nguyên lý:** MVCC-lite concurrent reads/writes without external daemon. WAL: concurrent readers don't block writer; writer appends to WAL, periodic checkpoint. Alternative (Redis) = external daemon + network round-trip, violates local-first principle. Code: [`db_core.py:94`](../../src/token_savior/db_core.py#L94).
**Industry ref:** iOS Core Data, Android SQLiteDatabase, Firefox IndexedDB, Obsidian — all WAL for same local-embedded-concurrent constraint.

### Decision 4b: FTS5 Full-Text Search
**Nguyên lý:** Zero-dependency collocated search — index trong cùng `.db` file, trigger-synced, BM25 built-in. Tantivy = Rust binary dep. Elasticsearch = JVM + network. Code: [`memory_schema.sql:63`](../../src/token_savior/memory_schema.sql#L63).
**Industry ref:** Signal messenger, Obsidian — SQLite FTS5 vì cùng local-first, no external deps constraint.

### Decision 4c: sqlite-vec for Vector Similarity
**Nguyên lý:** Vector k-NN as SQLite extension — `vec0` virtual table trong cùng `.db`, graceful degrade khi extension absent (`VECTOR_SEARCH_AVAILABLE` flag). pgvector = Postgres daemon. Qdrant/Chroma = separate process. Code: [`db_core.py:206`](../../src/token_savior/db_core.py#L206), [`memory/search.py:72`](../../src/token_savior/memory/search.py#L72).
**Industry ref:** DuckDB-VSS — embed vector math in query engine, no external vector DB.

### Decision 5: Profile Filtering via Env Var
**Nguyên lý:** Feature Toggling tại startup — MCP `list_tools()` phải static (client caches at connect). Lazy loading vi phạm MCP spec. Full profile = ~10,950 tokens injected; ultra = ~2,800 tokens. Env var available trước khi server khởi động. Code: [`server.py:131`](../../src/token_savior/server.py#L131).
**Industry ref:** OpenAPI spec versioning, gRPC service reflection — "static manifest declared at startup" model.

---

## Section 5 — Mental Shortcuts & Exercises

Từ [token-savior-yll worklog](../tasks/token-savior-yll-skill-transfer.md) với **corrected file locations** (khác với giả định ban đầu trong task plan).

### 3 Mental Shortcuts

**Shortcut 1 — Routing** *(Làm thế nào tool X được xử lý?)*
```
Đọc _dispatch_tool() tại server.py:304
→ ~80 LOC, 4 if/elif blocks → biết routing cho tất cả 105 tools
→ Không cần đọc 105 tool schemas hay handler files
```
**Clickable:** [`server.py:304`](../../src/token_savior/server.py#L304) *(correction: NOT server_runtime.py)*

**Shortcut 2 — Memory Recall** *(Kết quả memory_search từ đâu ra?)*
```
Đọc hybrid_search() + rrf_merge() tại memory/search.py:107
→ BM25 FTS5 + optional vector → RRF(k=60) → decay score → result
→ Debug recall: check RRF_K constant trước, rồi check FTS5 query
→ Không cần trace toàn bộ memory subsystem (7+ files)
```
**Clickable:** [`memory/search.py:107`](../../src/token_savior/memory/search.py#L107)

**Shortcut 3 — Extension** *(Thêm ngôn ngữ mới tốn bao nhiêu files?)*
```
Đọc _EXTENSION_MAP tại annotator.py:25 + _ANNOTATOR_MAP tại annotator.py:65
→ Thêm ngôn ngữ = 1 file mới + 2 dict entries (extension → lang, lang → fn)
→ AnnotatorProtocol: models.py:266 — interface: (source_text, source_name) → StructuralMetadata
→ 0 core file changes
```
**Clickable:** [`annotator.py:25`](../../src/token_savior/annotator.py#L25) → [`annotator.py:65`](../../src/token_savior/annotator.py#L65) → [`models.py:266`](../../src/token_savior/models.py#L266)

### 2 Bài Tập Thực Hành

**Exercise 1 — Add Kotlin Annotator (~1.5 giờ)**
Goal: Thực hành Open/Closed Principle.
```bash
git checkout -b practice/add-kotlin-annotator
# 1. src/token_savior/kotlin_annotator.py
#    annotate_kotlin(source_text: str, source_name: str) -> StructuralMetadata
# 2. annotator.py: _EXTENSION_MAP[".kt"] = "kotlin"
#                 _ANNOTATOR_MAP["kotlin"] = annotate_kotlin
# 3. cp tests/test_prisma_annotator.py tests/test_kotlin_annotator.py
#    Adapt: extension, fixture source, class names
```
Verify: `pytest tests/test_kotlin_annotator.py -v` → all PASSED. `git diff --stat | grep -v kotlin` → empty.

**Exercise 2 — Add Observation Type (~45 phút)**
Goal: Thực hành SRP — type validation tách khỏi storage.
Key insight: `observations.type` là `TEXT NOT NULL` (NO CHECK constraint) → schema không cần migration!
```bash
# memory/auto_extract.py:42 — thêm "hypothesis" vào VALID_TYPES set
# tests/: viết test dùng sqlite3.connect(":memory:") — no migration
```
Verify: `pytest tests/test_hypothesis_obs.py -v`

**Thought Experiment — Redis Backend**
*"Nếu thêm Redis backend thay SQLite — sửa files nào, theo thứ tự nào?"*
1. [`memory/search.py:107`](../../src/token_savior/memory/search.py#L107) → abstract `hybrid_search()` behind interface
2. `memory/redis_backend.py` → implement interface (RediSearch + RedisVL)
3. `memory/observations.py` → inject backend (DI)
4. `server.py` → config-based backend selection
5. `memory_schema.sql` → giữ cho SQLite; tạo redis equivalent init script

Nguyên lý: **Dependency Inversion** — high-level (`hybrid_search`) không depend trực tiếp vào SQLite.

---

## Verify Pass

```bash
grep -n "\.py:" self-explores/context/token-savior-design-principles.md | grep -v "\[.*\](" | grep -v '```'
```
→ Phải return empty (tất cả refs clickable).
