---
date: 2026-04-20
type: task-worklog
task: token-savior-9t8
title: "token-savior — Design Principles Report & Notion Sync"
status: done
tags: [system-design, report, notion-sync, design-principles]
---

# token-savior — Design Principles Report & Notion Sync

## Mô tả task
[Role: System Architect top 0.1%, Sư phụ hướng dẫn Học trò.]

Tổng hợp toàn bộ output Task 1-5 thành báo cáo Design Principles hoàn chỉnh, lưu local + sync Notion.

## Dependencies
- Chờ: token-savior-yll (Skill Transfer) xong

## Kế hoạch chi tiết

### Bước 1: Aggregate output (~15 phút)
Đọc tất cả worklogs:
```bash
cat self-explores/tasks/token-savior-4p1-*.md
cat self-explores/tasks/token-savior-psl-*.md
cat self-explores/tasks/token-savior-5bn-*.md
cat self-explores/tasks/token-savior-1ip-*.md
cat self-explores/tasks/token-savior-yll-*.md
```

### Bước 2: Tạo report file (~10 phút)
Tạo `self-explores/context/token-savior-design-principles.md` với 5 sections:
1. Architecture Overview (diagrams từ Task 1)
2. Core Components (từ Task 2)
3. Leverage Points (từ Task 2+3 — file paths clickable)
4. Design Principles & Rationale (từ Task 4)
5. Mental Shortcuts & Exercises (từ Task 5)

### Bước 3: Notion sync (~5 phút)
Dùng `/viec review` để push:
- Page: Experiments > token-savior > Design Principles
- Title: "Design Principles: token-savior"
- Tags: architecture, design-principles, token-savior, mcp-server

### Output mong đợi
- [ ] `self-explores/context/token-savior-design-principles.md` tồn tại với đủ 5 sections
- [ ] Notion page accessible (URL trả về)
- [ ] 100% code references clickable trong report

## Worklog

### [Chi tiết hóa — 2026-04-20] READY FOR DEV (blocked by yll)

**Objective:** Aggregate 5 task outputs thành report 5 sections + sync Notion (hoặc fallback note). Output: `self-explores/context/token-savior-design-principles.md`.

**Scope:**
- In-scope: 5 sections (200+ words each, ≥2 code refs each), Notion sync attempt
- Out-of-scope: Không tạo new content (chỉ aggregate), không benchmark

**Input / Output:**
- Input: 5 task worklogs + `self-explores/context/code-map.md` (từ 5bn)
- Output: `self-explores/context/token-savior-design-principles.md`

**Steps:**

**Step 0** (~10 phút): Đọc 5 worklogs, map overlap, tạo outline:
```bash
# Sections từ mỗi task:
# 4p1 → Section 1 (diagrams + flow table)
# psl → Section 2 (core components) + partial Section 3 (leverage points)
# 5bn (code-map.md) → Section 3 main (leverage with file:line)
# 1ip → Section 4 (design decisions analysis)
# yll → Section 5 (shortcuts + exercises)
```
- Dedup plan: Section 2 = "WHAT each component does", Section 3 = "WHERE leverage is (file:line) + WHY"
- Verify: outline thể hiện rõ đâu content từ đâu, không duplicate

**Step 1** (~5 phút): Tạo file output:
```bash
mkdir -p self-explores/context
touch self-explores/context/token-savior-design-principles.md
```

**Step 2** (~30 phút): Write 5 sections theo thứ tự:

**Section 1 — Architecture Overview** (~250 words):
- REFERENCE (embed link, không copy-paste) diagrams từ 4p1: Sequence diagram, Component diagram, Use-case diagram
- Summary 2-3 câu per diagram (không embed full Mermaid)
- Include 8-row flow table (copy từ 4p1, format compact)
- ≥2 code refs: [`server.py:401`](../../src/token_savior/server.py#L401), [`server_runtime.py:304`](../../src/token_savior/server_runtime.py#L304)

**Section 2 — Core Components** (~250 words):
- List 5-7 core components với 1-2 câu mô tả mỗi component
- Focus: "cái này làm gì" (WHAT)
- ≥2 code refs từ psl findings

**Section 3 — Leverage Points** (~300 words):
- Import từ `self-explores/context/code-map.md` (5bn output)
- Focus: "thay đổi ≤5 LOC ở đây → affect ≥10 behaviors" (WHERE + WHY)
- Explicit note: dedup với Section 2 — leverage points là SUBSET của core components + file:line
- ≥2 code refs với file:line

**Section 4 — Design Principles & Rationale** (~400 words):
- Import từ 1ip analysis: 5+3 decisions (Decision 4 = 3 sub-decisions)
- Format: Decision header + 2-3 dòng rationale + industry reference
- ≥2 code refs (1ip đã có clickable refs)

**Section 5 — Mental Shortcuts & Exercises** (~250 words):
- Import từ yll: 3 shortcuts (exact file:line) + 2 exercises/thought experiments
- ≥2 code refs (shortcuts đã có file:line)

**Step 3** (~5 phút): Attempt Notion sync:
```
→ Dùng mcp__notion MCP để tạo page
→ Parent: Experiments > token-savior > Design Principles
→ Nếu MCP không available: ghi note 'Notion sync pending — /viec review để sync'
```

**Step 4** (~5 phút): Verify pass:
```bash
grep -n "\.py:" self-explores/context/token-savior-design-principles.md | grep -v "\[.*\](" | grep -v '^\`\`\`'
```
Output PHẢI empty. Nếu không → fix từng plain ref.

**Step 5** (~3 phút): Word count check:
```bash
wc -w self-explores/context/token-savior-design-principles.md
```
Target: 1400-2000 words total.

**Edge Cases:**
| Case | Trigger | Xử lý |
|------|---------|--------|
| code-map.md chưa có (5bn chưa xong) | File not found | Dùng psl findings làm Section 3 placeholder, note "TODO: update when 5bn done" |
| Sections 2+3 vẫn overlap sau dedup | Content trùng | Section 2 = component list, Section 3 = pure table: Component | File:line | Leverage metric |
| Report > 2000 words | Word count exceeds | Trim Section 1 (reference diagrams thay vì describe), trim Section 4 (1 decision/paragraph) |
| Notion parent page missing | MCP call fails | Create hierarchy: Experiments → token-savior → Design Principles (3 create calls) |

**Acceptance Criteria:**
- Happy 1: Given aggregate complete, When read file, Then 5 sections present, each ≥200 words
- Happy 2: Given verify pass run, When check output, Then empty (no plain .py refs)
- Happy 3: Given Notion sync attempted, When result checked, Then URL returned OR fallback note recorded
- Negative: Given sections < 200 words, When check, Then expand with rationale (no padding)

**Technical Notes:**
- Verify command: `grep -n '\.py:' self-explores/context/token-savior-design-principles.md | grep -v '\[.*\](' | grep -v '^\`\`\`'`
- Word count per section: S1~250, S2~250, S3~300, S4~400, S5~250 = ~1450 total
- Section 1 reference format: `[Xem Sequence Diagram](../tasks/token-savior-4p1-contextual-awareness-sad-diagrams.md#diagram-1)`
- Notion page content: plaintext version of markdown (Notion doesn't render Mermaid natively)
- File header: include date, source tasks list, version

**Risks:**
- Section 1 thành mini-copy của 4p1 → strictly: link-only, no Mermaid embed
- Report redundant between S2 và S3 → enforce different angle: S2=WHAT, S3=WHERE+LEVERAGE

---

### [Thực thi — 2026-04-20] DONE

**Output:** [`self-explores/context/token-savior-design-principles.md`](../context/token-savior-design-principles.md) — 1668 words, 5 sections.

**Section word counts:** S1~220, S2~290, S3~280, S4~450, S5~380 (all ≥200 ✓)

**Clickable refs:** 30+ code refs, verify pass clean (all plain refs inside ``` code blocks only).

**Key corrections discovered during execution:**
- `_dispatch_tool()` is in [`server.py:304`](../../src/token_savior/server.py#L304) — NOT `server_runtime.py` (4p1 worklog had this wrong)
- `LazyLines` is in [`models.py:92`](../../src/token_savior/models.py#L92) — NOT `cache_ops.py`
- Annotator dispatch = `_EXTENSION_MAP` + `_ANNOTATOR_MAP` in [`annotator.py:25`](../../src/token_savior/annotator.py#L25) + [`annotator.py:65`](../../src/token_savior/annotator.py#L65) — NOT `LANGUAGE_MAP` in `project_indexer.py`
- `AnnotatorProtocol` at [`models.py:266`](../../src/token_savior/models.py#L266)
- `observations.type` has NO CHECK constraint → adding obs type requires only updating `VALID_TYPES` in [`memory/auto_extract.py:42`](../../src/token_savior/memory/auto_extract.py#L42)

**Notion sync:** Fallback — MCP requires OAuth flow. Notion sync pending — dùng `/viec review` để sync khi token available.
- Target page: Experiments > token-savior > Design Principles
- Title: "Design Principles: token-savior"
- Tags: architecture, design-principles, token-savior, mcp-server

---

### [Phản biện Lần 2 — 2026-04-20] Score: 7/10 (từ 6/10)

**Bối cảnh:** Fallback Notion đã được thêm từ lần 1 → risk HIGH giải quyết. Còn gaps về overlap + dedup.

**Điểm mơ hồ:**
1. "200 words minimum" là length, không phải quality requirement.
2. "Notion sync" — cần Notion API token và parent page ID; chưa document.

**Giả định ẩn:**
1. Sections 2 (Core Components) và 3 (Leverage Points) không overlap — SAI. Chúng rất liên quan: một leverage point thường LÀ một core component. Cần dedup layer.
2. Aggregating 5 worklogs là đủ — một số worklogs chứa phanbien analysis, không phải findings. Cần filter đúng sections.

**Rủi ro:**
- MEDIUM: Sections 2+3 overlap → report redundant. Cần dedup step.
- LOW: Paste code refs từ worklogs → có thể miss #L numbers khi verify. Add grep check cuối cùng.
- LOW (fallback added): Notion MCP không available → OK.

**Thiếu sót:**
1. Không có dedup/editing step trước khi finalize
2. Sections 1 (Architecture Overview) và task 4p1 overlap — cần reference, không copy-paste
3. Không rõ "≥2 clickable code refs per section" target — section 5 (Mental Shortcuts) có natural refs không?

**Khuyến nghị:**
1. Thêm Bước 0 (trước aggregate): đọc 5 worklogs → map overlap → tạo outline không duplicate
2. Section 1: REFERENCE 4p1 diagrams (đừng copy toàn bộ Mermaid code), chỉ embed link
3. Verify pass cuối: `grep -n "\.py:" report.md | grep -v "\[.*\]("`  → phải return empty
4. Score: 7/10 — đáng làm
