---
date: 2026-04-20
type: task-worklog
task: token-savior-4p1
title: "token-savior — Contextual Awareness (SAD / Diagrams)"
status: in_progress
started_at: 2026-04-20 
tags: [system-design, architecture, diagrams, mermaid, sad]
---

# token-savior — Contextual Awareness (SAD / Diagrams)

## Mô tả task
[Role: System Architect top 0.1%, Sư phụ hướng dẫn Học trò.]

Bước 1 & 2: Tìm kiếm tài liệu kiến trúc đã có cho token-savior. Ưu tiên self-explores/ trước, rồi mới scan repo gốc. Nếu không tìm thấy → tự generate từ code.

## Kế hoạch chi tiết

### Bước 1: Scan self-explores/ (~5 phút)
```bash
find self-explores/ -name "*.md" | xargs grep -l "diagram\|SAD\|sequence\|architecture" 2>/dev/null
ls self-explores/context/ 2>/dev/null
```

### Bước 2: Scan repo gốc (~10 phút)
```bash
ls docs/ design/ architecture/ .github/ 2>/dev/null
grep -r "sequence\|diagram\|architecture" README.md docs/ 2>/dev/null | head -20
```

### Bước 3: Generate diagrams từ code (~15 phút)
Nếu không có tài liệu sẵn → tạo 3 diagrams:
- **Sequence diagram**: MCP request → `server.py` → `server_runtime.py` → handler → result
- **Component diagram**: 4 handler categories (META/MEMORY/SLOT/QFN) + ProjectSlot pipeline
- **Use-case diagram**: Claude Code client, MCP server, WORKSPACE_ROOTS projects

Format: Mermaid markdown.

### Bước 4: Bảng tóm tắt luồng (~5 phút)
| Flow name | Actors | Trigger | Output |
|-----------|--------|---------|--------|
| Tool call | Claude Code → MCP server | Tool invocation | JSON result |
| Index build | Indexer → AST → Cache | WORKSPACE_ROOTS startup | JSON index |
| Memory recall | Client → SQLite FTS5 | memory_search call | Ranked results |

### Output mong đợi
- [ ] 2+ diagrams (Mermaid) hoặc tóm tắt 3+ luồng có sẵn
- [ ] Bảng tóm tắt luồng (4 cột)
- [ ] File worklog đầy đủ

## Worklog

### [Bước 1+2] Scan tài liệu

**Kết quả scan:**
- `docs/progressive-disclosure.md` — tài liệu 3-layer memory search (Layer 1/2/3)
- `README.md` — architecture overview đầy đủ, benchmark table, memory engine table
- Không có file diagram/SAD sẵn có → tiến hành generate từ code

**Files đã đọc để hiểu architecture:**
- [`server.py`](../../src/token_savior/server.py) — entry point: `list_tools()`, `call_tool()`, profile filtering
- [`server_runtime.py`](../../src/token_savior/server_runtime.py) — `_dispatch_tool()` (dispatch 4 categories), `_register_roots()`
- [`slot_manager.py`](../../src/token_savior/slot_manager.py) — `SlotManager`, `_ProjectSlot`, `ensure()`, `build()`
- [`cache_ops.py`](../../src/token_savior/cache_ops.py) — `CacheManager`, `LazyLines`
- [`memory/search.py`](../../src/token_savior/memory/search.py) — `hybrid_search()` (FTS5 + vector → RRF)

---

### [Bước 3] Diagrams

#### Diagram 1 — Sequence: MCP Tool Call Flow

```mermaid
sequenceDiagram
    participant CC as Claude Code (MCP Client)
    participant SV as server.py
    participant RT as server_runtime.py
    participant SM as SlotManager
    participant MH as MEMORY_HANDLERS
    participant SH as SLOT_HANDLERS
    participant QH as QFN_HANDLERS
    participant MDB as SQLite (Memory DB)
    participant IDX as ProjectQueryEngine

    CC->>SV: tool_call(name, arguments)
    SV->>RT: _dispatch_tool(name, arguments)

    alt META_HANDLERS (stats, budget, health)
        RT->>RT: meta_handler(arguments)
        RT-->>SV: TextContent
    else MEMORY_HANDLERS (memory_save/search/get...)
        RT->>MH: mem_handler(arguments)
        MH->>MDB: FTS5 + vector query (hybrid_search)
        MDB-->>MH: ranked rows
        MH-->>RT: text result
    else SLOT_HANDLERS (edit, git, tests, checkpoints)
        RT->>SM: resolve(project_hint)
        SM-->>RT: ProjectSlot
        RT->>SH: handler(slot, arguments)
        SH-->>RT: result
        RT->>RT: _count_and_wrap_result()
    else QFN_HANDLERS (find_symbol, get_call_chain...)
        RT->>SM: resolve(project_hint)
        SM-->>RT: ProjectSlot
        RT->>SM: _prep(slot) — ensure index built
        SM->>IDX: ProjectIndexer.index() [lazy, cached]
        IDX-->>SM: ProjectQueryEngine
        RT->>QH: qfn_handler(slot.query_fns, arguments)
        QH->>IDX: query method (22 methods)
        IDX-->>QH: raw result
        RT->>RT: _maybe_compress() → @F/@S/@L tokens
        RT->>RT: _count_and_wrap_result()
    end

    RT-->>SV: list[TextContent]
    SV-->>CC: tool_result
```

#### Diagram 2 — Component: Kiến trúc hệ thống

```mermaid
graph TB
    subgraph Client["Claude Code (MCP Client)"]
        CC[Tool calls]
    end

    subgraph Server["token-savior MCP Server"]
        SV["server.py<br/>list_tools() / call_tool()<br/>Profile filtering (env var)"]
        RT["server_runtime.py<br/>_dispatch_tool()<br/>_track_call() / _maybe_compress()"]

        subgraph Handlers["4 Handler Categories"]
            MET["META_HANDLERS<br/>stats, budget, health, project<br/>(no project context)"]
            MEM["MEMORY_HANDLERS<br/>21 tools: save/search/get<br/>index/timeline/distill..."]
            SLT["SLOT_HANDLERS<br/>code_nav, edit, git<br/>tests, analysis, checkpoints"]
            QFN["QFN_HANDLERS<br/>~50 tools → 22 query methods<br/>find_symbol, get_call_chain..."]
        end

        SV --> RT
        RT --> MET
        RT --> MEM
        RT --> SLT
        RT --> QFN
    end

    subgraph Indexing["Project Indexing Pipeline"]
        SM["SlotManager<br/>1 ProjectSlot per WORKSPACE_ROOT"]
        PI["ProjectIndexer<br/>tree-sitter AST parse<br/>language annotators"]
        CM["CacheManager<br/>JSON index + LazyLines<br/>(disk-backed, loads on demand)"]
        QE["ProjectQueryEngine<br/>22 query methods"]

        SM --> PI
        PI --> CM
        CM --> QE
    end

    subgraph MemEngine["Memory Engine (memory/)"]
        DB["SQLite WAL + FTS5 + sqlite-vec<br/>12 obs types · decay · dedup"]
        HS["hybrid_search()<br/>BM25 + vector → RRF fusion"]
        PD["Progressive Disclosure<br/>Layer1: ~15tk · L2: ~60tk · L3: ~200tk"]

        DB --> HS
        HS --> PD
    end

    CC --> Server
    QFN --> SM
    SLT --> SM
    MEM --> DB
    WORKSPACE["WORKSPACE_ROOTS (env var)"] --> SM
```

#### Diagram 3 — Use-Case: Actors và Interactions

```mermaid
graph LR
    subgraph Actors
        Dev["Developer<br/>(indirectly via CC)"]
        CC["Claude Code<br/>(MCP Client)"]
        ENV["Environment<br/>(WORKSPACE_ROOTS,<br/>TOKEN_SAVIOR_PROFILE)"]
    end

    subgraph UseCases["token-savior Use Cases"]
        UC1["Navigate code<br/>find_symbol, get_call_chain<br/>get_backward_slice, get_imports"]
        UC2["Read memory<br/>memory_index → memory_search<br/>→ memory_get (3-layer)"]
        UC3["Save memory<br/>memory_save (12 obs types)"]
        UC4["Edit code<br/>replace_symbol_source<br/>insert_near_symbol"]
        UC5["Git operations<br/>create_checkpoint<br/>build_commit_summary"]
        UC6["Run tests<br/>run_impacted_tests<br/>find_impacted_test_files"]
        UC7["Project stats<br/>get_project_summary<br/>get_budget_status"]
        UC8["Profile selection<br/>full/core/nav/lean/ultra<br/>105→17 tools"]
    end

    CC --> UC1
    CC --> UC2
    CC --> UC3
    CC --> UC4
    CC --> UC5
    CC --> UC6
    CC --> UC7
    ENV --> UC8
    UC8 --> CC
```

---

### [Bước 4] Bảng tóm tắt luồng

| Flow name | Actors | Trigger | Entry point | Output |
|-----------|--------|---------|-------------|--------|
| **Tool call dispatch** | CC → server → handler | MCP `call_tool` | [`server.py:401`](../../src/token_savior/server.py#L401) → [`server_runtime.py:304`](../../src/token_savior/server_runtime.py#L304) | TextContent JSON |
| **Index build (lazy)** | SlotManager → ProjectIndexer → CacheManager | First QFN call | [`slot_manager.py:104`](../../src/token_savior/slot_manager.py#L104) (`ensure()`) | JSON cache + ProjectQueryEngine |
| **Incremental update** | SlotManager → git diff | Cache stale ≤20 files changed | [`slot_manager.py:104`](../../src/token_savior/slot_manager.py#L104) | Partial index update |
| **Memory search** | Client → hybrid_search | `memory_search` / `memory_index` | [`memory/search.py:107`](../../src/token_savior/memory/search.py#L107) (`hybrid_search`) | BM25 + vector → RRF ranked list |
| **Memory progressive** | Client (3 layers) | Layer 1 → 2 → 3 contract | `memory_index` → `memory_search` → `memory_get` | 15 → 60 → 200 tokens/result |
| **Profile filtering** | env var at startup | `TOKEN_SAVIOR_PROFILE` set | [`server.py:139`](../../src/token_savior/server.py#L139) | 105 tools → 17 (ultra) |
| **Session rollup** | Hooks | SessionEnd lifecycle | `hooks/memory-hooks-config.json` | Compact summary saved to memory |
| **Symbol staleness** | CacheManager | Content hash change detected | [`memory/consistency.py`](../../src/token_savior/memory/consistency.py) | Linked obs invalidated |

---

### Output mong đợi — trạng thái

- [x] 3 diagrams Mermaid (sequence + component + use-case)
- [x] Bảng tóm tắt 8 luồng (4 cột: actors, trigger, entry point, output)
- [x] 100% code references clickable
- [x] File worklog đầy đủ
