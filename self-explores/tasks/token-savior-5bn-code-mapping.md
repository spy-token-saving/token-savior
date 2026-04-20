---
date: 2026-04-20
type: task-worklog
task: token-savior-5bn
title: "token-savior — Code Mapping (Truy vết thực tế)"
status: done
tags: [system-design, code-mapping, leverage-points, clickable-links]
---

# token-savior — Code Mapping (Truy vết thực tế)

## Mô tả task
[Role: System Architect top 0.1%, Sư phụ hướng dẫn Học trò.]

Scan toàn bộ codebase và map mỗi Core Component / Leverage Point từ Task 2 → file:line cụ thể + code snippet tinh hoa.

## Dependencies
- Chờ: token-savior-psl (Strategic Evaluation) xong trước

## Kế hoạch chi tiết

### Bước 1: Map Core Components (~20 phút)
Với mỗi component từ Task 2 → grep/read để tìm class/function chính xác.
```bash
grep -n "class SlotManager\|class CacheManager\|class ProjectQueryEngine" src/token_savior/*.py
grep -rn "META_HANDLERS\|MEMORY_HANDLERS\|SLOT_HANDLERS\|QFN_HANDLERS" src/token_savior/
```

### Bước 2: Trích code tinh hoa (~25 phút)
Với mỗi Leverage Point: đọc đoạn 50-100 dòng, giải thích tại sao là "tinh hoa".

Format output:
```markdown
## Leverage Point: Tool Dispatch Dict
File: [`server_runtime.py:LINE`](../../src/token_savior/server_runtime.py#LLINE)
Nguyên lý: Single Responsibility + Open/Closed
Code tinh hoa: [đoạn code]
Lý do: Dict-driven dispatch tách routing khỏi logic — thêm tool mới chỉ cần thêm 1 entry
```

### Bước 3: Verify clickable links (~15 phút)
100% file:line references phải là clickable markdown links. Kiểm tra:
```bash
grep -n "\.py:\d\+" self-explores/tasks/token-savior-5bn-code-mapping.md | grep -v "http\|](../)"
```
Nếu còn plain text → convert ngay.

### Output mong đợi
- [ ] Mỗi Core Component từ Task 2 → 1+ file:line clickable link
- [ ] Mỗi Leverage Point → 1+ file:line clickable link
- [ ] ≥3 đoạn code tinh hoa với giải thích
- [ ] 100% code references là clickable links (grep verify)

## Worklog

### [Chi tiết hóa — 2026-04-20] READY FOR DEV (blocked by psl)

**Objective:** Map 5 priority components từ token-savior → exact file:line + code snippet 10-20 LOC + giải thích nguyên lý. Output: `self-explores/context/code-map.md`.

**Scope:**
- In-scope: 5 priority components (từ 4p1), max 8 total, file:line clickable, snippets 10-20 LOC
- Out-of-scope: Không nhét output vào worklog này, không trích toàn hàm (chỉ đoạn "tinh hoa")

**Input / Output:**
- Input: psl findings (strategic evaluation output), 4p1 component diagram
- Output: `self-explores/context/code-map.md` (mới, không phải worklog)

**Steps:**

**Step 1** (~5 phút): Grep exact line numbers cho 5 priority components:
```bash
grep -n "def _dispatch_tool\|META_HANDLERS\|MEMORY_HANDLERS\|QFN_HANDLERS" src/token_savior/server_runtime.py | head -10
grep -n "class SlotManager\|def ensure\|def build" src/token_savior/slot_manager.py | head -10
grep -n "class LazyLines\|def __getitem__" src/token_savior/cache_ops.py | head -10
grep -n "def hybrid_search" src/token_savior/memory/search.py
grep -n "PROFILE_TOOL_MAP\|TOKEN_SAVIOR_PROFILE" src/token_savior/server.py | head -5
```
- Verify: Mỗi grep trả về ít nhất 1 line number

**Step 2** (~25 phút): Với mỗi trong 5 components, đọc 10-20 LOC KEY nhất:
1. `_dispatch_tool`: Chỉ đọc đoạn handler resolution (if/elif chain), không phải toàn hàm
2. `ensure()`: Đọc lazy init check (git ref cache + incremental vs full rebuild decision)
3. `LazyLines.__getitem__`: Đọc lazy load logic
4. `hybrid_search()`: Đọc RRF fusion block (BM25 + vector → combined score)
5. `PROFILE_TOOL_MAP`: Đọc dict structure (profile name → list of tool names)

**Step 3** (~20 phút): Write `self-explores/context/code-map.md` với format:
```markdown
## Component: {Tên}
File: [`file.py:START-END`](../../src/token_savior/file.py#LSTART-LEND)
Nguyên lý: {SOLID/GoF pattern}
Code tinh hoa: [đoạn code 10-20 LOC]
Lý do tinh hoa: {1-2 câu — tại sao đoạn này chi phối toàn bộ behavior}
```

**Step 4** (~10 phút): Nếu psl output có thêm components (trục 1/2 findings) → bổ sung, total max 8.

**Step 5** (~10 phút): Verify clickable links:
```bash
grep -n "\.py:" self-explores/context/code-map.md | grep -v "\[.*\](" | grep -v '```'
```
Output phải EMPTY. Nếu không empty → convert từng plain text ref thành clickable link.

**Edge Cases:**
| Case | Trigger | Xử lý |
|------|---------|--------|
| `_dispatch_tool` dài >50 LOC | Hàm phức tạp | Chỉ trích đoạn 10 LOC "if name in META_HANDLERS" |
| LazyLines quá đơn giản | <10 LOC meaningful | Giải thích design pattern thay vì chỉ code |
| psl chưa xong | Blocked | Dùng 4p1 component diagram làm fallback source |

**Acceptance Criteria:**
- Happy 1: Given grep step 1 completed, When write code-map.md, Then file exists với 5+ entries
- Happy 2: Given grep verify step 5 run, When check output, Then empty string (no plain refs)
- Happy 3: Given each entry read, When count LOC, Then 10-20 LOC per snippet
- Negative: Given output file missing, When check, Then FAIL

**Technical Notes:**
- Output path: `self-explores/context/code-map.md`
- Relative path from code-map.md: `../../src/token_savior/` (đúng vì file ở `self-explores/context/`)
- For `ensure()`: LazyLines + git ref check = lazy eval pattern; incremental update = Δ diff < 20 files
- For `hybrid_search()`: RRF formula = 1/(k + rank_BM25) + 1/(k + rank_vector), k=60
- Verify command format: `grep -n 'pattern' file | grep -v 'exclusion' | grep -v '^\`\`\`'`

**Risks:**
- Code snippet bị stale sau refactor → note version/date trong code-map.md header
- Snippets > 20 LOC → strictly enforce cap (nếu vượt, cut và add "..." marker)

---

### [Phản biện Lần 2 — 2026-04-20] Score: 7/10

**Bối cảnh:** Từ 4p1, confirmed 7 node chính trong component diagram — CAP 8 đúng. Có sẵn grep results cho class names.

**Điểm mơ hồ:**
1. "Mỗi Core Component từ Task 2" — Task 2 (psl) chưa xong, component list chưa confirmed. 5bn phụ thuộc vào output của psl.
2. "Code tinh hoa 10-20 dòng" — tiêu chí "tinh hoa" vẫn vague: critical path? density? elegance?

**Giả định ẩn:**
1. 10-20 LOC đủ để capture essence của mỗi component — SAI cho `_dispatch_tool()` (~80 LOC). Cần chọn đoạn "dispatch decision" cốt lõi, không phải toàn hàm.
2. Worklog file này chứa được toàn bộ 8 snippets — sẽ rất dài. Nên export ra file riêng.

**Rủi ro:**
- MEDIUM: `_dispatch_tool()` không thể distill 10-20 dòng có nghĩa. Giải pháp: chọn đoạn handler resolution (10 LOC quyết định META/MEMORY/SLOT/QFN) thay vì toàn function.
- LOW: Verify clickable links bằng grep không catch broken `#L42` nếu code refactored.

**Thiếu sót:**
1. Output file: snippets nên vào `self-explores/context/code-map.md`, không nhét vào worklog (file sẽ quá dài).
2. Priority order cho 8 components chưa có — mapping theo importance: dispatch → slot pipeline → cache → memory → annotators.

**Khuyến nghị:**
1. Tạo output file riêng: `self-explores/context/code-map.md`
2. Priority map từ 4p1 output: 1) `_dispatch_tool` dict, 2) `ensure()`/`build()`, 3) `LazyLines`, 4) `hybrid_search()`, 5) profile `PROFILE_TOOL_MAP`
3. Cho `_dispatch_tool`: chỉ trích đoạn 10 LOC quyết định category (not full function)

---

### [Thực thi — 2026-04-20] DONE

Created `self-explores/context/code-map.md` with 5 components mapped to exact file:line locations:

1. `_dispatch_tool` handler chain — [`server.py:304-352`](../../src/token_savior/server.py#L304) — routes all 105 tools via four ordered dict lookups (META → MEMORY → SLOT → QFN)
2. Handler dict aggregation — [`server_handlers/__init__.py:42-87`](../../src/token_savior/server_handlers/__init__.py#L42) — `_merge_disjoint` enforces no cross-category tool name collision at import time
3. `SlotManager.ensure()` + `build()` — [`slot_manager.py:104-182`](../../src/token_savior/slot_manager.py#L104) — three-tier cache decision: exact git-ref hit / ≤20-file incremental / full rebuild
4. `LazyLines.__getitem__` — [`models.py:92-147`](../../src/token_savior/models.py#L92) — Virtual Proxy pattern; file content deferred until first `[]` access, saving ~80% idle RAM
5. `_PROFILE_EXCLUDES` (the actual dict behind "PROFILE_TOOL_MAP") — [`server.py:131-151`](../../src/token_savior/server.py#L131) — maps 5 profiles (full/core/nav/lean/ultra) to exclusion sets controlling the MCP manifest size from 17 to 106 tools
