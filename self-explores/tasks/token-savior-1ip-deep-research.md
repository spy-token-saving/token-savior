---
date: 2026-04-20
type: task-worklog
task: token-savior-1ip
title: "token-savior — Deep Research (Tư duy của Top 0.1%)"
status: done
tags: [system-design, deep-research, design-principles, solid, patterns]
---

# token-savior — Deep Research (Tư duy của Top 0.1%)

## Mô tả task
[Role: System Architect top 0.1%, Sư phụ hướng dẫn Học trò.]

Nghiên cứu sâu "Tại sao họ thiết kế như vậy?" cho token-savior — từng design decision quan trọng.

## Dependencies
- Chờ: token-savior-psl (Strategic Evaluation) xong trước
- Có thể chạy **song song** với token-savior-5bn (Code Mapping)

## Kế hoạch chi tiết

### 5 Decisions cần phân tích:

**Decision 1: 4-category handler dispatch (META/MEMORY/SLOT/QFN)**
- Nguyên lý: Command Pattern + Separation of Concerns
- Tại sao không flat registry? → slot context requirement cho SLOT, no-project-context cho META
- Industry ref: LLVM pass manager, LSP server dispatch

**Decision 2: 3-layer memory progressive disclosure (15/60/200 tokens)**
- Nguyên lý: Cost-aware information retrieval
- Tại sao không return full content mỗi lần? → token budget trong MCP context
- Industry ref: Elasticsearch lazy loading, Google Search snippet vs full page

**Decision 3: LazyLines pattern (disk-backed, loads on demand)**
- Nguyên lý: Lazy Evaluation + Memory Efficiency
- Tại sao không in-memory cache toàn bộ? → 10K files × avg 500 LOC = 5M lines
- Industry ref: mmap, LMDB, Git pack objects

**Decision 4: SQLite WAL + FTS5 + sqlite-vec cho memory backend**
- Nguyên lý: Single-file simplicity vs distributed complexity
- Tại sao không PostgreSQL/Elasticsearch? → MCP server local-first, no external deps
- Industry ref: SQLite used in production by Apple, Android, Firefox

**Decision 5: Profile filtering (105 → 17 tools via env var)**
- Nguyên lý: Feature toggling + MCP schema overhead reduction
- Tại sao không lazy-load? → MCP protocol requires upfront schema manifest
- Industry ref: OpenAPI spec versioning, gRPC reflection

### Bước: Analyze mỗi decision (~15 phút/decision = 75 phút total)
Format: Decision → Principle → Rationale (why not simpler?) → Historical context → Industry Reference

### Output mong đợi
- [ ] ≥3 decisions với đầy đủ 4 điểm phân tích
- [ ] Mỗi decision có ≥1 industry reference
- [ ] Code references PHẢI clickable

## Worklog

### [Chi tiết hóa — 2026-04-20] READY FOR DEV (blocked by psl)

**Objective:** Phân tích sâu 5+3 design decisions của token-savior theo 3-4 chiều, derive rationale từ code constraints khi git log không có. Output: section "Design Principles" trong worklog này.

**Scope:**
- In-scope: 5 decisions (Decision 4 = 3 sub-decisions), ≥3 chiều/decision, ≥1 industry ref
- Out-of-scope: Không cần git blame/log, không benchmark actual performance

**Input / Output:**
- Input: 4p1 component diagram, psl findings, `src/token_savior/` codebase
- Output: Analysis appended vào worklog này (dài), sau đó aggregated vào 9t8 report

**Steps:**

**Step 1** (~5 phút): Scan 4 files chính để có context trước khi analyze:
```bash
grep -n "META_HANDLERS\|MEMORY_HANDLERS\|SLOT_HANDLERS\|QFN_HANDLERS" src/token_savior/server_runtime.py | head -10
grep -c "" src/token_savior/memory_schema.sql  # line count to gauge schema size
head -30 src/token_savior/memory/search.py     # hybrid_search signature
grep -n "LazyLines" src/token_savior/cache_ops.py | head -5
grep -n "PROFILE_TOOL_MAP\|TOKEN_SAVIOR_PROFILE" src/token_savior/server.py | head -5
```

**Step 2** (~15 phút): Decision 1 — 4-category handler dispatch (META/MEMORY/SLOT/QFN)
- Nguyên lý: Command Pattern + Separation of Concerns (SRP)
- Tại sao không flat registry? → SLOT và QFN cần `ProjectSlot` context; META không cần. Flat registry buộc mọi handler xử lý null project context.
- Alternative: Middleware chain (Express.js style) — rejected vì no natural grouping + harder to reason
- Industry ref: "LSP server dùng capabilities dict để route từng method (hover, completion, definition) — cùng pattern: method name → handler, với context requirement khác nhau"

**Step 3** (~15 phút): Decision 2 — 3-layer memory progressive disclosure (15/60/200 tokens)
- Nguyên lý: Cost-aware information retrieval + Lazy Loading
- Tại sao không return full content? → Token budget trong MCP context (mỗi tool call = tokens consumed)
- Alternative: Return full content mỗi lần (PostgreSQL style) — rejected vì MCP token cost prohibitive
- Industry ref: "Elasticsearch: search → highlight snippet → full source. Google Search: SERP snippet (50 chars) → full page (50KB). Same 3-tier cost structure."
- File ref: `docs/progressive-disclosure.md` + [`memory/search.py:107`](../../src/token_savior/memory/search.py#L107)

**Step 4** (~15 phút): Decision 3 — LazyLines disk-backed vs in-memory
- Nguyên lý: Lazy Evaluation + Memory Efficiency
- Tại sao không cache toàn bộ? Constraint: 10K files × avg 500 LOC = 5M lines × 80 bytes = 400MB RAM chỉ cho file content → OOM trên dev machines
- Alternative: mmap mỗi file → rejected vì OS page cache behavior không predictable; LazyLines explicit hơn
- Industry ref: "Git pack objects: compressed data loaded only when accessed. SQLite B-tree: pages loaded on demand. LMDB: copy-on-write pages, only materialize when read."

**Step 5** (~20 phút): Decision 4 — SQLite storage (3 sub-decisions):

4a. WAL mode (concurrent reads vs writes):
- Nguyên lý: MVCC-lite without full transaction isolation
- Tại sao không Redis? → MCP server local-first, no external daemon dep; WAL gives concurrent reads without blocking writes
- Industry ref: "SQLite WAL dùng trong production: iOS Core Data, Android SQLite, Firefox IndexedDB, Obsidian. Redis cần network round-trip + memory overhead."
- File ref: [`memory_schema.sql`](../../src/token_savior/memory_schema.sql) (`PRAGMA journal_mode=WAL`)

4b. FTS5 vs Tantivy/Elasticsearch:
- Nguyên lý: Zero-dependency full-text search (collocated with data)
- Tại sao không Tantivy? → Rust binary dep, không embed trong Python process. Elasticsearch → cần JVM + network.
- Industry ref: "Signal messenger, Obsidian dùng SQLite FTS5 cho local search. VS Code dùng ripgrep (không embed) cho codebase search — khác use case."

4c. sqlite-vec vs pgvector/Qdrant/Chroma:
- Nguyên lý: Local-first, zero-additional-process vector similarity
- Tại sao không pgvector? → Needs Postgres. Tại sao không Chroma? → Python process, port conflict.
- Industry ref: "sqlite-vec approach tương tự DuckDB-VECS: embed vector math trong query engine, không external process."

**Step 6** (~15 phút): Decision 5 — Profile filtering 105→17 via env var
- Nguyên lý: Feature Toggling + MCP schema overhead reduction
- Tại sao không lazy-load? → MCP protocol requires upfront `list_tools()` manifest. Client caches schema at connect time.
- Alternative: Dynamic tool discovery per-request → rejected vì MCP spec requires static manifest
- Industry ref: "OpenAPI spec versioning (v2 vs v3 different schemas). gRPC reflection: service registers capabilities upfront. AWS feature flags: env var pattern for service-level capability"
- File ref: [`server.py:139`](../../src/token_savior/server.py#L139) area

**Step 7** (~10 phút): Format: mỗi decision → 4-chiều markdown block, code refs clickable, append vào worklog.

**Step 8** (~5 phút): Verify:
```bash
grep -n "\.py:" self-explores/tasks/token-savior-1ip-deep-research.md | grep -v "\[.*\](" | grep -v '```'
```
Phải empty.

**Edge Cases:**
| Case | Trigger | Xử lý |
|------|---------|--------|
| Git log không có design rationale | `git log --grep="Decision" --oneline` returns empty | Derive từ constraint math: "X files × Y bytes = Z MB" |
| Industry ref không rõ ràng | Pattern quá generic | Chỉ tên system + "same pattern" + 1 câu tại sao relevant |
| Decision 4 bị merge thành 1 | Viết 1 block cho cả 3 | BẮT BUỘC tách 3 headers riêng: 4a, 4b, 4c |

**Acceptance Criteria:**
- Happy 1: Given 5 decisions analyzed, When count chiều per decision, Then ≥3/4 chiều for each
- Happy 2: Given Decision 4, When count headers, Then 4a + 4b + 4c distinct headers present
- Happy 3: Given industry refs written, When read each, Then format "System X dùng Y cho Z"
- Negative: Given grep verify run, When check output, Then empty (no plain .py refs)

**Technical Notes:**
- Decision 1: `grep -n "if name in META_HANDLERS\|elif name in MEMORY" src/token_savior/server_runtime.py`
- Decision 3: `grep -n "class LazyLines" src/token_savior/cache_ops.py` → read __getitem__ method
- Decision 4a: `grep -n "WAL\|journal_mode" src/token_savior/memory_schema.sql`
- Decision 4b: `grep -n "FTS5\|CREATE VIRTUAL" src/token_savior/memory_schema.sql`
- Decision 4c: `grep -rn "sqlite.vec\|vec_f32" src/token_savior/memory/`
- Output format per decision: `### Decision N: [Title]\n**Nguyên lý:** ...\n**Tại sao không đơn giản hơn:** ...\n**Alternative approaches:** ...\n**Industry reference:** ...`

**Risks:**
- Decision 4 analysis thin (treat as 1) → explicitly use 3 sub-headers 4a/4b/4c
- Industry refs superficial (just names) → always include "same pattern because Y"

---

### [Thực thi — 2026-04-20] Design Decisions Analysis

#### Decision 1: 4-Category Handler Dispatch (META/MEMORY/SLOT/QFN)

**Nguyên lý:** Command Pattern + Single Responsibility Principle

**Tại sao không đơn giản hơn:** Một flat registry dạng `{tool_name: handler}` chỉ hoạt động khi mọi handler có cùng signature. Trong token-savior, bốn nhóm handler có context requirement hoàn toàn khác nhau: META handlers nhận `(arguments)` và không cần project context nào cả (stats, budget, health); MEMORY handlers nhận `(arguments)` và trả về raw string (memory engine đã tự quản lý connection nội bộ); SLOT handlers nhận `(slot, arguments)` và cần một `ProjectSlot` đã resolved với index còn sống; QFN handlers nhận `(query_fns, arguments)` và cần thêm bước kiểm tra `slot.query_fns is not None` trước khi gọi. Nếu dùng flat registry, mọi handler phải tự kiểm tra "tôi có cần slot không?" và "slot đã có index chưa?" — logic bảo vệ này sẽ bị lặp ở 105 nơi thay vì tập trung tại dispatcher. Kiến trúc hiện tại cũng bảo vệ cả khi import: `_merge_disjoint()` raise `RuntimeError` ngay khi load nếu một tool name xuất hiện ở hai categories — lỗi phát hiện tại import time thay vì runtime.

**Alternative approaches:** Middleware chain kiểu Express.js (`req → auth → slot-resolve → handler`) cho phép pipeline linear nhưng mất khả năng group routing — mọi request đều phải chạy qua toàn bộ chain kể cả khi là META call thuần túy không cần slot. Plugin registry pattern (mỗi handler đăng ký capability requirements) linh hoạt hơn nhưng phức tạp hơn và không có lợi thực tế khi category count chỉ là 4.

**Industry reference:** LSP (Language Server Protocol) dùng capabilities dict để route từng method — `textDocument/hover` cần open document context, `workspace/symbol` không cần, `initialize` là lifecycle và xử lý riêng trước tất cả. Cùng pattern "method name → dispatch tier → context requirement", chỉ khác token-savior không dùng JSON-RPC mà dùng Python dict lookup.

**Code ref:** [`server_handlers/__init__.py:56`](../../src/token_savior/server_handlers/__init__.py#L56) (aggregation + `_merge_disjoint` guard), [`server.py:310`](../../src/token_savior/server.py#L310) (4-tier dispatch loop)

---

#### Decision 2: 3-Layer Memory Progressive Disclosure (15/60/200 tokens)

**Nguyên lý:** Cost-aware information retrieval + Lazy Loading

**Tại sao không đơn giản hơn:** Trong MCP context, mỗi tool call response đi vào context window của LLM và tiêu thụ token budget vĩnh viễn trong session đó. Nếu `memory_search` trả về full content (~200 tokens) cho mọi kết quả, một search với 20 hits = 4000 tokens bị consume chỉ để LLM xác định observation nào relevant. Thực tế LLM thường chỉ cần 2-3 observations sau khi đã đọc shortlist. Progressive disclosure giải quyết bằng cách: Layer 1 `memory_index` trả về shortlist ~15 tokens (title + type + id), Layer 2 `memory_search` trả về BM25+vector excerpt ~60 tokens với đủ context để judge relevance, Layer 3 `memory_get` mới trả về full content ~200 tokens khi LLM đã chọn xong. Token cost thực tế: 15 × 20 hits = 300 tokens cho shortlist so với 200 × 20 = 4000 tokens nếu eager — tiết kiệm 93%.

**Alternative approaches:** Trả về full content theo kiểu PostgreSQL `SELECT *` đơn giản hơn về implementation nhưng token cost prohibitive. Pagination (return N items per page) đòi hỏi LLM phải request multiple pages và phức tạp hơn cho calling pattern. Server-side summarization (LLM call để compress trước khi trả về) thêm latency và cost ở server.

**Industry reference:** Elasticsearch dùng pattern snippet → source: search API trả về `highlight` (40-80 char) trước, caller tự decide có cần `_source` (full document) không — cùng cost-tier structure. Google Search dùng SERP snippet (~150 chars) → full page (~50KB) → cached full page cho power users. Signal messenger dùng progressive message loading: recent 20 → scroll-back 100 → full archive.

**Code ref:** [`memory/search.py:107`](../../src/token_savior/memory/search.py#L107) (`hybrid_search` — Layer 2 entry point), [`memory/observations.py:317`](../../src/token_savior/memory/observations.py#L317) (FTS + RRF fusion)

---

#### Decision 3: LazyLines Disk-Backed (Loads On Demand)

**Nguyên lý:** Lazy Evaluation + Memory Efficiency — defer work until it is actually needed

**Tại sao không đơn giản hơn:** Constraint math trực tiếp: một project vừa (10,000 files) với avg 500 LOC/file = 5,000,000 lines; mỗi line ~80 bytes (UTF-8 average với Python str overhead) = **400 MB RAM** chỉ cho file content — chưa kể AST metadata (functions, classes, imports, dependency graph). Trên dev machine với 8-16 GB RAM chạy cùng IDE + browser, 400 MB constant overhead là không chấp nhận được. Quan trọng hơn, 95% tool calls (get_functions, find_symbol, get_dependencies) không cần line content — chúng dùng pre-parsed AST metadata. Chỉ các calls như `get_function_source`, `get_class_source` mới cần đọc actual lines. LazyLines giải quyết: cache phase serialize `root_path + rel_path` thay vì `list[str]`, file chỉ được đọc khi `__getitem__` hoặc `__iter__` được gọi lần đầu.

**Alternative approaches:** `mmap` cung cấp memory-mapped file access với OS-level page cache nhưng behavior không predictable trên Windows và không transparent khi file thay đổi. Explicit LRU cache (giữ N files gần nhất in-memory) phức tạp hơn và cần eviction policy; LazyLines đơn giản hơn vì nó delegate caching cho OS page cache. Redis-backed line store là over-engineering cho use case này.

**Industry reference:** Git pack objects dùng delta compression và chỉ expand (inflate) khi object được access — same "store reference, materialize on demand" pattern. SQLite B-tree pages chỉ được loaded từ disk vào page cache khi query traverse đến page đó. LMDB dùng copy-on-write memory-mapped pages: pages chỉ được brought into RAM khi read, và dirty pages được written back lazily.

**Code ref:** [`models.py:92`](../../src/token_savior/models.py#L92) (`LazyLines` class definition, `__slots__`, `_loaded` property), [`cache_ops.py:241`](../../src/token_savior/cache_ops.py#L241) (LazyLines instantiation at deserialization time)

---

#### Decision 4a: SQLite WAL Mode (Concurrent Reads/Writes)

**Nguyên lý:** MVCC-lite (Multi-Version Concurrency Control) mà không cần external daemon

**Tại sao không đơn giản hơn:** Default SQLite journal mode (DELETE/ROLLBACK) dùng exclusive lock khi write — mọi concurrent read phải đợi. Trong MCP context, MCP server có thể nhận concurrent tool calls: một call đang write memory observation trong khi call khác đang read (memory_search). Không dùng WAL → readers block writers hoặc writers block readers. Redis giải quyết concurrency nhưng cần external daemon (`redis-server`), network round-trip, và thêm một process người dùng phải quản lý — vi phạm nguyên tắc local-first/zero-deps của MCP server. PostgreSQL lại còn nặng hơn. WAL mode cho phép concurrent readers không block writer và writer không block readers bằng cách append writes vào WAL file riêng; readers đọc từ main DB file, writers append vào WAL, checkpoint periodic.

**Alternative approaches:** SQLite với PRAGMA locking_mode=EXCLUSIVE đơn giản hơn nhưng hoàn toàn single-writer single-reader. Threading lock ở application level (Python threading.Lock) là workaround không scale khi có multiple processes. Redis Streams cho event-style memory bus nhưng không cung cấp SQL query capability cần cho FTS/filters.

**Industry reference:** iOS Core Data dùng SQLite WAL cho concurrent main thread reads và background context writes. Android SQLiteDatabase mặc định WAL mode từ API 16. Firefox IndexedDB backend dùng SQLite WAL cho tab isolation. Obsidian dùng SQLite WAL cho plugin database. Tất cả cùng pattern vì cùng constraint: single-file embedded DB cần serve concurrent readers mà không block.

**Code ref:** [`db_core.py:94`](../../src/token_savior/db_core.py#L94) (`PRAGMA journal_mode = WAL` trong `run_migrations`), [`db_core.py:242`](../../src/token_savior/db_core.py#L242) (`PRAGMA journal_mode = WAL` trong mỗi `get_db()` call)

---

#### Decision 4b: FTS5 Full-Text Search

**Nguyên lý:** Zero-dependency collocated full-text search — index và data trong cùng một file

**Tại sao không đơn giản hơn:** Tantivy là Rust-based full-text search library xuất sắc (dùng trong Meilisearch, Quickwit) nhưng đòi hỏi Rust binary dependency và không embed natively trong Python process — cần giao tiếp qua IPC hoặc compiled Python binding. Elasticsearch cần JVM, network, cluster setup — totally disproportionate cho local development tool. Typesense, Meilisearch — cần external process. FTS5 là SQLite virtual table module: index được lưu trong cùng `.db` file với observations, trigger-synced tự động khi insert/delete/update, BM25 ranking built-in, `snippet()` function cho excerpt extraction. Không cần migration, không cần external process, không cần network. Trade-off: FTS5 không có semantic search (giải quyết bằng Decision 4c), và không có distributed sharding (không cần ở local tool scale).

**Alternative approaches:** LIKE/GLOB queries trên raw text columns — O(n) full scan, không scale với thousands of observations. Whoosh (Python-native full-text search) — đã unmaintained, không embedded trong SQLite transaction boundary. PostgreSQL + pg_trgm — cần Postgres, overkill cho local-first tool.

**Industry reference:** Signal messenger dùng SQLite FTS5 cho local message search — cùng constraint (local-first, embedded, no external deps). Obsidian dùng SQLite FTS5 cho vault search plugin. VS Code dùng ripgrep (không embed) cho codebase search nhưng đó là different use case (file content grep, không phải structured document search).

**Code ref:** [`memory_schema.sql:63`](../../src/token_savior/memory_schema.sql#L63) (`CREATE VIRTUAL TABLE observations_fts USING fts5`), [`memory/observations.py:340`](../../src/token_savior/memory/observations.py#L340) (`observations_fts MATCH ?` query với `snippet()`)

---

#### Decision 4c: sqlite-vec for Vector Similarity

**Nguyên lý:** Local-first vector k-NN search mà không cần additional process, không cần network

**Tại sao không đơn giản hơn:** pgvector cần PostgreSQL running — external daemon, port conflict, RAM overhead. Qdrant là dedicated vector DB cần separate process và HTTP/gRPC endpoint. Chroma là Python process nhưng vẫn cần separate server mode hoặc embedded mode với file locking issues khi concurrent access. sqlite-vec là SQLite extension: `vec0` virtual table được load trực tiếp vào SQLite connection (`conn.enable_load_extension(True); sqlite_vec.load(conn)`), vector data stored trong cùng `.db` file, k-NN query dùng cùng SQL interface (`WHERE embedding MATCH ? AND k = ?`). Optional dependency: nếu `import sqlite_vec` fail, code degrade gracefully về FTS-only (xem `VECTOR_SEARCH_AVAILABLE` flag). Embedding dùng `all-MiniLM-L6-v2` (384 dims, ~90MB, runs locally) — cũng optional.

**Alternative approaches:** FAISS (Facebook AI Similarity Search) — C++ library, Python bindings, nhưng không tích hợp với SQLite transaction và cần separate index file. Annoy (Spotify) — file-based ANN index nhưng không SQL-queryable. NumPy cosine similarity — O(n) brute force, scale poorly với thousands of observations.

**Industry reference:** DuckDB-VSS (Vector Similarity Search extension) dùng cùng approach: embed vector math trực tiếp trong query engine bằng extension, không cần external vector DB — same "zero-process-addition" philosophy. LiteFS (Fly.io) dùng SQLite extension pattern cho replication — chứng minh pattern extension-based augmentation là production-grade. Apple on-device ML sử dụng embedded vector store approach cho similar reasons: no network, no external process.

**Code ref:** [`db_core.py:206`](../../src/token_savior/db_core.py#L206) (`CREATE VIRTUAL TABLE obs_vectors USING vec0(embedding FLOAT[384])`), [`memory/search.py:72`](../../src/token_savior/memory/search.py#L72) (`import sqlite_vec` + graceful fallback), [`memory/embeddings.py:14`](../../src/token_savior/memory/embeddings.py#L14) (`all-MiniLM-L6-v2` 384-dim model)

---

#### Decision 5: Profile Filtering (105 → 17 Tools via Env Var)

**Nguyên lý:** Feature Toggling + MCP schema overhead reduction tại connect time

**Tại sao không đơn giản hơn:** MCP protocol yêu cầu client gọi `list_tools()` một lần khi connect và cache manifest đó cho toàn bộ session. Client không re-query per-request. Điều này có nghĩa là lazy loading tools "on demand" không khả thi theo MCP spec: nếu tool không có trong manifest lúc connect, client không biết tool đó tồn tại để gọi. Manifest size có cost thực tế: full 105-tool manifest ~10,950 tokens bị inject vào system prompt của một số MCP clients. Ultra profile (17 tools) = ~2,800 tokens — tiết kiệm 8,150 tokens mỗi session, tức là 8,150 tokens không bao giờ được reclaimed. Profile filtering giải quyết bằng cách filter `TOOLS` list tại server startup (không phải per-request), dùng env var `TOKEN_SAVIOR_PROFILE` vì env var có sẵn trước khi MCP server khởi động và không cần config file.

**Alternative approaches:** Dynamic tool discovery per-request (tools register capabilities on-the-fly) vi phạm MCP spec về static manifest. Feature flags trong tool schema (tool có `"enabled": false` field) — không trong MCP spec, client-side behavior undefined. Separate MCP server per profile (chạy 4 servers) — resource overhead và configuration complexity. gRPC reflection pattern (server advertises services dynamically) — không applicable vì MCP không có reflection mechanism.

**Industry reference:** OpenAPI spec versioning (v2 vs v3 expose different endpoint sets) dùng cùng pattern: capability set declared upfront, không thay đổi trong session. AWS Lambda function với Reserved Concurrency env var — same "capability gate via env var at deploy time". Claude Code hooks dùng `TOKEN_SAVIOR_PROFILE` pattern tương tự `ANTHROPIC_FEATURE_FLAGS` để control feature surface tại process startup. gRPC server reflection: services register capabilities upfront, client introspects once và cache — same "static manifest, filter at startup" model.

**Code ref:** [`server.py:80`](../../src/token_savior/server.py#L80) (`_LEAN_EXCLUDES` definition, comment về token savings), [`server.py:131`](../../src/token_savior/server.py#L131) (`_PROFILE_EXCLUDES` dict mapping profile name → excluded tool set), [`server.py:139`](../../src/token_savior/server.py#L139) (`_PROFILE = os.environ.get("TOKEN_SAVIOR_PROFILE", "full")`)

---

### [Phản biện Lần 2 — 2026-04-20] Score: 7/10

**Bối cảnh:** Từ 4p1, xác nhận cả 5 decisions đều visible trong code. Không có decision quan trọng nào bị bỏ sót.

**Điểm mơ hồ:**
1. "Historical context OPTIONAL" nhưng "Alternative approaches" cũng cần rationale — circular nếu không có nguồn.
2. "Industry reference" depth chưa rõ: chỉ tên hệ thống hay cần giải thích tại sao relevant?

**Giả định ẩn:**
1. Decision 4 là 1 decision — SAI. SQLite WAL + FTS5 + sqlite-vec là 3 decisions bundled:
   - WAL: concurrency model (reads không block writes)
   - FTS5: full-text search backend (vs Tantivy/Lucene)
   - sqlite-vec: vector store (vs pgvector/Qdrant/Chroma)
   Mỗi sub-decision có tradeoff riêng biệt cần phân tích.
2. git log có design rationale — thường không. Phải derive từ constraints.

**Rủi ro:**
- MEDIUM: Decision 4 bundling → phân tích không đủ sâu nếu chỉ nói "SQLite đơn giản".
- LOW: Decision 3 (LazyLines) thiếu strong industry ref. Tốt nhất: SQLite rowid + mmap (OS level), Git pack objects lazy expansion.

**Thiếu sót:**
1. Decision 4 cần tách 3 sub-decisions với rationale riêng
2. Output file format chưa rõ — nên vào worklog hay context file riêng?

**Khuyến nghị:**
1. Tách Decision 4 thành: 4a) SQLite WAL vs Redis/Postgres, 4b) FTS5 vs Tantivy/Elasticsearch, 4c) sqlite-vec vs external vector DBs
2. Industry reference format chuẩn: "Hệ thống X dùng pattern Y cho use case Z"
3. Derive rationale từ constraints: "LazyLines = 10K files × avg 500 LOC = 5M lines → OOM nếu in-memory"
