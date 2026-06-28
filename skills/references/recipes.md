# Recipes — end-to-end playbooks

Copy-and-adapt workflows for the most common Notion jobs. Each recipe lists the
goal, the exact sequence of tool calls, and the mistakes to avoid. The guiding
case is an AI writing study/homework content into Notion accurately, but the
patterns generalize.

## Contents
1. [Write homework/lesson content into an existing page](#1-write-homeworklesson-content-into-an-existing-page)
2. [Recreate a colored table from a photo](#2-recreate-a-colored-table-from-a-photo)
3. [Create a study / assignment tracker database](#3-create-a-study--assignment-tracker-database)
4. [Build a structured study or vocabulary page](#4-build-a-structured-study-or-vocabulary-page)
5. [Safe re-runs (idempotency) for agents](#5-safe-re-runs-idempotency-for-agents)
6. [Bulk operations efficiently](#6-bulk-operations-efficiently)

---

## 1. Write homework/lesson content into an existing page

**Goal:** the user has a page (often with a marker like `//for claude`) and wants
the answer/lesson written in, well-formatted, without disturbing the rest.

1. `notion-fetch(id=PAGE_URL)` — read the current content; locate the marker or
   the exact heading/sentence you'll write under. Copy that text verbatim.
2. Compose the content in enhanced markdown: a clear heading hierarchy
   (`#`/`##`/`###`), short paragraphs, callouts for key takeaways, code fences
   for code/formulas, and tables only when tabular. Keep Vietnamese (or any)
   text as-is — don't escape normal punctuation.
3. Replace the marker precisely:
   ```
   notion-update-page(command="update_content", page_id=PAGE_ID,
     content_updates=[{ "old_str": "//for claude",
                        "new_str": "## Bài giải\n...formatted answer..." }])
   ```
   If there's no marker, append with `insert_content` (`position:{type:"end"}`)
   or insert under a section by matching its heading text as `old_str` and
   re-emitting heading + new body.
4. `notion-fetch(id=PAGE_URL)` — verify the content rendered and the rest of the
   page is intact.

**Gotchas:** `old_str` must match exactly once (whitespace/case included). Never
use `replace_content` here — it can wipe sibling content and child pages.

## 2. Recreate a colored table from a photo

**Goal:** the user sends a screenshot/photo of a colored table and wants it
rebuilt faithfully in Notion. This is the highest-risk task — follow it exactly.
Full rules: `references/tables-and-color.md`.

1. **Transcribe structure.** Count rows × columns; write out each cell's text.
   Note any merged cells (Notion can't merge — repeat the value or leave blank,
   and say so).
2. **Build a color map.** For every distinctly colored cell, record fill and
   text color separately. Example notation:
   ```
   header row:  fill=blue,  text=bold dark
   col 1:       fill=gray,  text=normal
   "Done" cells: fill=green, text=normal
   "Blocked":    fill=red,   text=normal
   warnings:     fill=none,  text=red bold
   ```
3. **Snap to the 9 hues** (`*_bg` for fills, plain hue for text). Teal→blue/green,
   navy/sky→blue, magenta/rose→pink, gold/cream→yellow, etc. Never invent a token.
4. **Emit an HTML `<table>`** (never pipes when color is involved). Put
   whole-column fills in `<colgroup>` (one `<col>` per column, empty `<col>` for
   uncolored ones), whole-row fills on `<tr color>`, and exceptions on
   `<td color>`. For a specifically-colored header, use `header-row="false"` and
   color the header cells yourself.
5. **Write it**, then `notion-fetch` and compare to the photo cell by cell. Fix
   any fill/text/precedence mismatch and re-verify.

**Gotchas:** the classic failures are pipe tables (color lost), fill-vs-text
swapped, invented colors, and `<colgroup>` count mismatches shifting colors.

## 3. Create a study / assignment tracker database

**Goal:** a database to track assignments with colored statuses.

1. Decide the schema. Create it with SQL DDL — option colors are set here:
   ```
   notion-create-database(title="Assignments", parent={"page_id": PARENT_ID},
     schema='CREATE TABLE ("Assignment" TITLE, "Subject" SELECT('Math':blue, 'IELTS':green, 'Science':orange), "Status" SELECT('To Do':red, 'Doing':yellow, 'Done':green), "Due" DATE, "Score" NUMBER, "Done?" CHECKBOX)')
   ```
2. From the response (or a follow-up `notion-fetch`), grab the
   `collection://<id>` data-source id.
3. Add rows with `notion-create-pages` (up to 100 per call):
   ```
   notion-create-pages(parent={"data_source_id": "<collection-uuid>"}, pages=[
     {"properties": {"Assignment": "Essay 1", "Subject": "IELTS", "Status": "Doing",
        "date:Due:start": "2026-07-05", "date:Due:is_datetime": 0, "Done?": "__NO__"}},
     {"properties": {"Assignment": "Algebra set", "Subject": "Math", "Status": "To Do",
        "date:Due:start": "2026-07-08", "date:Due:is_datetime": 0, "Done?": "__NO__"}}
   ])
   ```
4. Optional: add a board view grouped by status, or a calendar by due date:
   ```
   notion-create-view(database_id=DB_ID, data_source_id="<collection-uuid>",
     name="Board", type="board", configure='GROUP BY "Status"')
   ```

**Gotchas:** dates use the expanded `date:Prop:start` / `:is_datetime` form;
checkboxes are `"__YES__"`/`"__NO__"`; numbers are real numbers; always `fetch`
the schema first to use exact property names.

## 4. Build a structured study or vocabulary page

**Goal:** a clean, scannable page mixing callouts, a colored reference table, and
toggles.

1. `notion-fetch` the parent (or create a new page with `notion-create-pages`,
   `parent={"page_id": ...}`, a title, and `content`).
2. Compose content: a top callout for the objective, a colored `<table>` for the
   vocabulary/reference grid (see recipe 2 / tables reference), and toggles for
   "examples" or "answers" so the page stays compact:
   ```
   <callout icon="🎯" color="blue_bg">Mục tiêu: ...</callout>

   <table fit-page-width="true" header-row="false">
     <colgroup><col color="gray_bg"><col><col></colgroup>
     <tr><td>Từ</td><td><span color="red">**Synonyms**</span></td><td>Ví dụ</td></tr>
   </table>

   <details color="gray_bg"><summary>Đáp án</summary>
     - point 1
   </details>
   ```
3. Verify by re-fetching.

**Gotchas:** table cells hold inline text only — put lists/headings outside the
table. Use `<empty-block/>` if you need a deliberate blank line.

## 5. Safe re-runs (idempotency) for agents

**Goal:** an agent may run the same task twice; avoid duplicating content.

- Before inserting, `notion-fetch` and check whether the target already contains
  what you'd add (e.g. the marker `//for claude` is gone → it was already
  replaced; don't append again).
- Prefer `update_content` keyed on a stable marker over blind `insert_content`.
- When updating database rows, use `update_properties` on the known `page_id`
  rather than creating a new row.
- If a write returns an async task, poll `notion-get-async-task` before deciding
  whether to retry, so you don't double-write.

## 6. Bulk operations efficiently

- **Many rows:** one `notion-create-pages` call accepts up to 100 pages under the
  same parent — batch them instead of looping one-by-one.
- **Several edits to one page:** one `update_content` call accepts a
  `content_updates` array — make all the search-replaces in a single call.
- **Large writes:** set `allow_async:true` and poll `notion-get-async-task`.
- **Large reads/exports:** window a big database by `created_time` (see
  `references/databases-and-queries.md`) so you don't silently stop at 10k rows.
