---
date: 2026-04-20
type: task-worklog
task: token-savior-yll
title: "token-savior — Skill Transfer (Lối tắt & Thực hành)"
status: done
tags: [system-design, skill-transfer, mental-models, exercises]
---

# token-savior — Skill Transfer (Lối tắt & Thực hành)

## Mô tả task
[Role: System Architect top 0.1%, Sư phụ hướng dẫn Học trò.]

Chuyển giao kỹ năng: 3-5 mental shortcuts + 2-3 bài tập thực hành để "cảm" nguyên lý thiết kế của token-savior.

## Dependencies
- Chờ: token-savior-5bn (Code Mapping) xong
- Chờ: token-savior-1ip (Deep Research) xong

## Kế hoạch chi tiết

### Bước 1: Mental Shortcuts (~20 phút)
Dựa trên output Task 3+4, chắt lọc 3-5 shortcuts:
- Shortcut 1: Muốn hiểu tool routing nhanh → đọc handler dict trong server_runtime.py, không cần đọc 105 schemas
- Shortcut 2: Muốn debug memory recall → xem decay_score formula trước, rồi mới trace FTS5 query
- Shortcut 3: Muốn extend annotator → chỉ cần implement 2 methods (tên class + extract symbols), không cần đọc indexer
- (Thêm từ Task 3+4 findings)

### Bước 2: Bài tập thực hành (~25 phút để design)

**Bài 1: Thêm language annotator mới (~1.5 giờ để làm)**
```bash
git checkout -b practice/add-zig-annotator
# Chỉ tạo src/token_savior/zig_annotator.py
# Register trong project_indexer.py tại 1 dòng
# Không sửa bất kỳ file core nào khác
```
Verify: `pytest tests/test_zig_annotator.py` — pattern giống `test_rust_annotator.py`

**Bài 2: Thêm observation type thứ 13 (~1 giờ để làm)**
```bash
git checkout -b practice/add-obs-type
# Sửa memory_schema.sql: thêm CHECK constraint
# Sửa 1 constant trong memory/ nơi list 12 types
# Viết test verify type được lưu/query
```
Verify: new type xuất hiện trong memory_index khi search

**Bài 3: Thought experiment (không cần code)**
"Nếu muốn thêm remote backend (Redis thay SQLite) cho memory engine — bạn sẽ modify files nào? Tại sao?"

### Output mong đợi
- [ ] ≥3 mental shortcuts (khác nhau về góc nhìn)
- [ ] ≥2 bài tập có: mô tả, commands, verify criteria, estimated time
- [ ] Bài tập áp dụng nguyên lý từ Task 4
- [ ] File references trong shortcuts PHẢI clickable

## Worklog

### [Chi tiết hóa — 2026-04-20] READY FOR DEV (blocked by 5bn + 1ip)

**Objective:** Tạo 3 mental shortcuts (với exact file:line) + 2 practice exercises (với verify commands). Mỗi shortcut = "đọc FILE:LINE → biết X mà không cần đọc Y".

**Scope:**
- In-scope: 3 shortcuts (routing/storage/extension), 2 exercises hoặc 1 exercise + 1 thought experiment
- Out-of-scope: Không implement exercises (chỉ design), không chạy actual tests

**Input / Output:**
- Input: 4p1 component diagram, 5bn code-map.md, 1ip design decisions analysis
- Output: Shortcuts + exercises appended vào worklog này

**Steps:**

**Step 0a** (~5 phút): Verify feasibility — Zig grammar:
```bash
grep -n "LANGUAGE_MAP\|zig\|lua\|LANGUAGES" src/token_savior/project_indexer.py | head -20
```
- Nếu "zig" có trong LANGUAGE_MAP → Exercise 1 = Zig annotator (đã có grammar support)
- Nếu không → Exercise 1 = Lua annotator (hoặc ngôn ngữ đơn giản nhất chưa có)
- Verify: Biết chính xác ngôn ngữ target trước khi viết exercise

**Step 0b** (~5 phút): Verify Exercise 2 feasibility — obs type 13:
```bash
grep -n "CHECK\|observation_type\|obs_type" src/token_savior/memory_schema.sql | head -10
grep -rn "observation_type\|OBS_TYPES\|obs_types" src/token_savior/memory/ | head -10
```
- Nếu schema có CHECK constraint → note migration pattern (DROP+recreate in test, not ALTER)
- Nếu enum/list trong code → note list location (update 1 constant)
- Decision: nếu migration risk cao → chuyển Exercise 2 thành Thought Experiment

**Step 1** (~20 phút): Viết 3 mental shortcuts (sau khi đã verify line numbers từ 5bn/4p1):

**Shortcut 1 — Routing (góc nhìn: làm thế nào tool X được xử lý?):**
```
Đọc `_dispatch_tool()` tại [`server_runtime.py:304`](../../src/token_savior/server_runtime.py#L304)
→ 80 LOC, 4 if/elif blocks → biết routing cho tất cả 105 tools
→ Không cần đọc 105 tool schemas trong tool_schemas.py
```

**Shortcut 2 — Storage/Memory (góc nhìn: kết quả memory_search từ đâu ra?):**
```
Đọc `hybrid_search()` tại [`memory/search.py:107`](../../src/token_savior/memory/search.py#L107)
→ BM25 FTS5 + optional vector → RRF fusion → decay score
→ Không cần trace toàn bộ memory subsystem (7 files)
→ Debug recall issue: check decay formula trước, rồi mới check FTS5 query
```

**Shortcut 3 — Extension points (góc nhìn: thêm support ngôn ngữ mới tốn bao nhiêu files?):**
```
Đọc LANGUAGE_MAP tại project_indexer.py (line từ Step 0a)
→ 1 dict entry: {extension: annotator_class}
→ Implement 1 file mới: class XAnnotator với method _extract_symbols()
→ Không cần sửa bất kỳ file core nào khác
→ Reference: test_rust_annotator.py là template test
```

**Step 2** (~30 phút): Design Exercise 1 — Add Language Annotator:
```bash
# Setup
git checkout -b practice/add-{lang}-annotator

# Implement (1 file mới)
# src/token_savior/{lang}_annotator.py
# class {Lang}Annotator:
#     def _extract_symbols(self, source: str, path: str) -> list[Symbol]: ...

# Register (1 dict entry)
# project_indexer.py: LANGUAGE_MAP[".{ext}"] = {Lang}Annotator

# Test (reference pattern)
# cp tests/test_rust_annotator.py tests/test_{lang}_annotator.py
# Sửa: extension, class name, test fixture (source code mẫu)
```
- Verify: `pytest tests/test_{lang}_annotator.py -v` → all PASSED
- Estimate: ~1.5 giờ (30 min annotator + 30 min test + 30 min debug)

**Step 3** (~20 phút): Design Exercise 2 hoặc Thought Experiment:

**Nếu obs type 13 feasible (từ Step 0b):**
```bash
git checkout -b practice/add-obs-type

# Bước 1: Update constant (1 file)
# Tìm OBS_TYPES list/enum trong memory/ → thêm type mới

# Bước 2: Update schema (in test mode: recreate in-memory)
# KHÔNG ALTER TABLE trong SQLite — phải DROP + CREATE
# Test mode: dùng in-memory SQLite (`:memory:`) → không cần migration

# Bước 3: Viết test
# Verify new type có thể save + query
```
- Verify: `pytest tests/ -k "obs_type" -v` → new type appears in results

**Nếu migration risk cao → Thought Experiment:**
```
Câu hỏi: "Nếu muốn thêm Redis backend thay SQLite cho memory engine — sửa files nào? Theo thứ tự nào?"

Trả lời mong đợi:
1. `memory_schema.sql` → không cần (Redis không dùng SQL)
2. `memory/search.py` → abstract hybrid_search() behind interface
3. `memory/` → tạo `redis_backend.py` implement interface
4. `server_runtime.py` → inject backend based on config
5. `memory_schema.sql` → giữ cho SQLite backend, tạo redis equivalent init script

Nguyên lý áp dụng: Dependency Inversion Principle — high-level module (search) không depend trực tiếp vào SQLite
```

**Step 4** (~5 phút): Verify shortcuts có exact file:line:
```bash
grep -o "server_runtime\.py:[0-9]*\|search\.py:[0-9]*\|project_indexer\.py" \
  self-explores/tasks/token-savior-yll-skill-transfer.md
```
Output phải có đúng line numbers (verify với grep trực tiếp vào source).

**Edge Cases:**
| Case | Trigger | Xử lý |
|------|---------|--------|
| Zig grammar không tồn tại | `grep -n "zig" project_indexer.py` returns empty | Switch sang Lua hoặc bất kỳ ngôn ngữ đơn giản chưa có |
| Exercise 2 migration quá phức tạp | Step 0b: CHECK constraint syntax phức tạp | Replace Exercise 2 với Thought Experiment |
| Shortcuts line numbers bị lệch | Code đã refactored | Re-grep và update line numbers trước khi submit |

**Acceptance Criteria:**
- Happy 1: Given shortcuts written, When check each, Then format "đọc FILE:LINE → biết X" present
- Happy 2: Given Exercise 1 designed, When read verify section, Then `pytest` command runnable
- Happy 3: Given thought experiment (if used), When read, Then clear answer "sửa files: A → B → C → D + lý do"
- Negative: Given shortcuts missing file:line, When check, Then FAIL — phải có exact refs

**Technical Notes:**
- Shortcut line numbers (verified): [`server.py:304`](../../src/token_savior/server.py#L304) (_dispatch_tool), [`memory/search.py:107`](../../src/token_savior/memory/search.py#L107) (hybrid_search)
- LANGUAGE_MAP grep: `grep -n "LANGUAGE_MAP\|AnnotatorClass\|annotator" src/token_savior/project_indexer.py | head -10`
- Exercise 1 test template: `tests/test_rust_annotator.py` → copy + adapt fixtures
- Exercise 2 in-memory SQLite: `conn = sqlite3.connect(":memory:")` — no file, no migration
- Thought experiment format: list files in order + DIP/SRP principle applied

**Risks:**
- Shortcuts too abstract (no file:line) → each shortcut MUST start with exact ref
- Exercise 1 setup non-obvious → add "Prerequisites: virtual env activated, tree-sitter-X installed (or skip annotator if grammar missing)"

---

### [Phản biện Lần 2 — 2026-04-20] Score: 7/10

**Bối cảnh:** Bước 0 (feasibility verify) đã được thêm từ lần 1. Còn 2 risks mới phát hiện từ 4p1 architecture read.

**Điểm mơ hồ:**
1. "Shortcut" vs "best practice" — shortcut PHẢI có dạng: "đọc X:LINE → hiểu Y mà không cần đọc Z". Cần exact file:line.
2. Tiêu chí nào để decide thought experiment thay code exercise? Bước 0 chưa có decision criteria rõ.

**Giả định ẩn:**
1. Zig annotator exercise: giả định tree-sitter-zig grammar tồn tại và có thể install. Cần verify trong `pyproject.toml` hoặc `project_indexer.py` LANGUAGE_MAP.
2. Exercise 2 (add obs type 13): giả định ALTER TABLE SQLite schema. SAI — SQLite không support `ALTER TABLE ADD CONSTRAINT`. Cần DROP + recreate schema hoặc dùng migration script.

**Rủi ro:**
- MEDIUM: Exercise 2 SQLite migration risk — CHECK constraint trong `memory_schema.sql` là DDL, không thể ADD sau khi create. Phải recreate.
- LOW: "Nhánh git riêng" — repo này có remote không? Kiểm tra `git remote` trước khi hướng dẫn git workflow.

**Thiếu sót:**
1. Mental shortcuts thiếu EXACT file:line — "đọc server_runtime.py" quá vague. Phải là: "đọc `_dispatch_tool()` tại [`server_runtime.py:304`](../../src/token_savior/server_runtime.py#L304)"
2. Exercise 1 reference pattern: `test_rust_annotator.py` chưa được mention — đây là template test pattern

**Khuyến nghị:**
1. Shortcut 1 (routing): "`_dispatch_tool()` tại [`server.py:304`](../../src/token_savior/server.py#L304) — 80 LOC quyết định toàn bộ 105 tool routing" (correction: server.py, không phải server_runtime.py)
2. Shortcut 3 (extension): "Annotator chỉ cần 2 methods; xem interface tại `project_indexer.py` LANGUAGE_MAP"
3. Exercise 1: add verify note "reference pattern: `test_rust_annotator.py`"
4. Exercise 2: note rõ SQLite migration pattern (test mode: drop+recreate in-memory)

---

### [Thực thi — 2026-04-20] Skill Transfer Content

**Step 0 findings (feasibility verify):**
- `_EXTENSION_MAP` tại [`annotator.py:25`](../../src/token_savior/annotator.py#L25) — dict: `.ext → language_name`
- `_ANNOTATOR_MAP` tại [`annotator.py:65`](../../src/token_savior/annotator.py#L65) — dict: `language_name → annotator_fn`
- `AnnotatorProtocol` tại [`models.py:266`](../../src/token_savior/models.py#L266) — `__call__(source_text, source_name) -> StructuralMetadata`
- Không có Zig annotator. Language không có: Kotlin, Swift, Lua, Zig, Ruby, PHP, Scala...
- **Exercise 1 target:** bất kỳ ngôn ngữ trên — recommend Kotlin (`.kt`) vì syntax đơn giản hơn Zig
- Schema: `type TEXT NOT NULL` — **KHÔNG có CHECK constraint** → không cần migration để thêm obs type
- `VALID_TYPES` tại [`memory/auto_extract.py:42`](../../src/token_savior/memory/auto_extract.py#L42) = 6 types (chỉ cho LLM auto-extraction)
- **Exercise 2:** thêm type = thêm vào `VALID_TYPES` + viết test → không cần DROP+recreate schema
- Test template: `tests/test_prisma_annotator.py` (không phải test_rust_annotator.py — file đó không tồn tại)
- `_dispatch_tool()` tại [`server.py:304`](../../src/token_savior/server.py#L304) — **không phải server_runtime.py** (correction từ 4p1 worklog)

---

#### Mental Shortcuts

**Shortcut 1 — Routing (Làm thế nào tool X được xử lý?)**
```
Đọc _dispatch_tool() tại server.py:304
→ ~80 LOC, 4 if/elif blocks → biết routing cho tất cả 105 tools
  - META_HANDLERS: stats, budget, health (no project context)
  - MEMORY_HANDLERS: 21 memory tools
  - SLOT_HANDLERS: code_nav, edit, git, tests (need ProjectSlot)
  - QFN_HANDLERS: 50+ structural query tools
→ Không cần đọc 105 tool schemas hay bất kỳ handler file nào
→ 1 function hiểu toàn bộ routing logic
```
**Clickable:** [`server.py:304`](../../src/token_savior/server.py#L304)

**Shortcut 2 — Memory Recall (Kết quả memory_search từ đâu ra?)**
```
Đọc hybrid_search() tại memory/search.py:107
→ BM25 FTS5 + optional vector k-NN → RRF fusion (k=60)
→ Formula: 1/(k + rank_BM25) + 1/(k + rank_vector) → combined score
→ Decay score apply sau → final ranked list
→ Không cần trace toàn bộ memory subsystem (7+ files)
→ Debug recall issue: check decay formula trước, rồi mới check FTS5 query
```
**Clickable:** [`memory/search.py:107`](../../src/token_savior/memory/search.py#L107)

**Shortcut 3 — Extension (Thêm support ngôn ngữ mới tốn bao nhiêu files?)**
```
Đọc _EXTENSION_MAP tại annotator.py:25 + _ANNOTATOR_MAP tại annotator.py:65
→ Extension pattern:
  Step 1: 1 file mới: {lang}_annotator.py
    - 1 function: annotate_{lang}(source_text: str, source_name: str) -> StructuralMetadata
    - AnnotatorProtocol: models.py:266
  Step 2: 2 dict entries (annotator.py):
    - _EXTENSION_MAP[".kt"] = "kotlin"
    - _ANNOTATOR_MAP["kotlin"] = annotate_kotlin
→ Không cần sửa bất kỳ file core nào khác (project_indexer, server, handlers)
→ Test template: copy tests/test_prisma_annotator.py + adapt
```
**Clickable:** [`annotator.py:25`](../../src/token_savior/annotator.py#L25) → [`annotator.py:65`](../../src/token_savior/annotator.py#L65) → [`models.py:266`](../../src/token_savior/models.py#L266)

---

#### Exercise 1 — Add Language Annotator (~1.5 giờ)

**Goal:** Thực hành Open/Closed Principle — thêm Kotlin support mà không sửa core files.

```bash
# Setup
cd /path/to/token-savior
git checkout -b practice/add-kotlin-annotator

# Step 1: Create annotator (1 file mới)
# src/token_savior/kotlin_annotator.py
# - 1 function: annotate_kotlin(source_text: str, source_name: str) -> StructuralMetadata
# - Parse: class definitions, fun declarations, val/var at top-level
# - Reference: prisma_annotator.py for StructuralMetadata construction pattern

# Step 2: Register (2 lines in annotator.py)
# Line ~25: _EXTENSION_MAP[".kt"] = "kotlin"
# Line ~65: _ANNOTATOR_MAP["kotlin"] = annotate_kotlin
# Add import: from token_savior.kotlin_annotator import annotate_kotlin

# Step 3: Write test
# cp tests/test_prisma_annotator.py tests/test_kotlin_annotator.py
# Adapt: extension=".kt", fixture source code = simple Kotlin class
```

**Verify:**
```bash
pytest tests/test_kotlin_annotator.py -v  # all PASSED
# Kiểm tra không sửa file nào ngoài: kotlin_annotator.py + annotator.py + test file
git diff --stat | grep -v "kotlin\|annotator\.py\|test_kotlin"  # phải empty
```

**Estimate:** ~1.5 giờ (30 min annotator + 30 min test + 30 min debug tree-sitter patterns)

**Nguyên lý học được:** Open/Closed — extend without modifying `project_indexer.py`, `server.py`, hay bất kỳ handler nào.

---

#### Exercise 2 — Add Observation Type (~45 phút)

**Goal:** Thực hành hiểu schema separation — type validation tách khỏi storage.

**Key insight:** `observations.type` là TEXT NOT NULL (không có CHECK constraint) → schema không cần migration! Type validation chỉ ở application layer.

```bash
# Setup
git checkout -b practice/add-obs-type

# Target: thêm type "hypothesis" (chưa có trong 12 types hiện tại)

# Step 1: Thêm vào auto_extract VALID_TYPES (auto-extraction support)
# memory/auto_extract.py:42 — thêm "hypothesis" vào set

# Step 2: Viết test — dùng in-memory SQLite (không cần migration)
# tests/test_hypothesis_obs.py:
# conn = sqlite3.connect(":memory:")
# Khởi tạo schema từ memory_schema.sql
# Lưu observation với type="hypothesis"
# Query lại và verify type được preserve
```

**Verify:**
```bash
pytest tests/test_hypothesis_obs.py -v
# Verify "hypothesis" type xuất hiện khi query
```

**Estimate:** ~45 phút (15 min schema study + 20 min test + 10 min debug)

**Nguyên lý học được:** SRP — type validation (application layer) tách biệt khỏi storage (SQLite TEXT). Không phải mọi constraint phải là database constraint.

---

#### Thought Experiment — Remote Memory Backend

**Câu hỏi:** "Nếu muốn thêm Redis backend thay SQLite cho memory engine — sửa files nào? Theo thứ tự nào?"

**Trả lời mong đợi (bottom-up, dependency order):**
1. [`memory/search.py:107`](../../src/token_savior/memory/search.py#L107) → abstract `hybrid_search()` behind `MemoryBackend` interface
2. `memory/` → tạo `redis_backend.py` implementing interface (FTS5 → RediSearch, sqlite-vec → RedisVL)
3. `memory/observations.py` → inject backend (Dependency Injection)
4. `server.py` → inject backend based on config/env var
5. `memory_schema.sql` → giữ cho SQLite; tạo `redis_schema.py` (HSET init commands)

**Nguyên lý áp dụng:** Dependency Inversion Principle — high-level module (`hybrid_search`) không được depend trực tiếp vào SQLite connection. "Depend on abstractions, not concretions."

**Câu hỏi mở rộng:** "Tại sao không dùng Redis cho production hiện tại?" → Xem Decision 4a analysis trong 1ip worklog: local-first + no external daemon dep.

---

#### Verify pass
```bash
grep -n "\.py:" self-explores/tasks/token-savior-yll-skill-transfer.md | grep -v "\[.*\](" | grep -v '```'
```
→ Phải return empty.
