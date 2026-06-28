---
name: notion
description: >
  Use this skill for ANY task that uses the Notion MCP tools (names starting with
  notion-): reading or fetching pages and databases; creating or editing pages;
  writing and formatting content with bold/italic/underline/color; building
  tables — especially colored tables meant to match a screenshot or image the
  user sends; adding callouts, toggles, columns, code, equations, or media;
  creating and editing databases, schemas, and select/status option colors;
  querying with SQL or a saved view; creating or updating views; adding database
  rows; duplicating or moving pages; commenting; reading meeting notes; and
  replacing markers like //for claude. Also trigger on phrases in any language
  such as "vào Notion", "viết lên Notion", "tạo bảng Notion", "tô màu bảng".
  Always consult this skill before calling a Notion MCP tool: the
  enhanced-markdown syntax, SQL-DDL schema, and view DSL are non-obvious and easy
  to get wrong. Do not guess Notion syntax from memory — read here.
---

# Notion via MCP — read, write, format, and build

This skill is the source of truth for working with Notion through the `notion-*`
MCP tools. It already contains the full **enhanced-markdown** spec (in
`references/`), so you never need to fetch `notion://docs/enhanced-markdown-spec`
or any external doc — read the reference files in this skill instead.

## The golden workflow — do this every time

1. **Fetch first.** Call `notion-fetch` on the page/database before writing.
   This gives you (a) the exact current text so your `old_str` matches byte-for-
   byte, (b) the location of any marker like `//for claude`, (c) the database
   schema and `collection://` data-source IDs, and (d) the existing style to
   match. Skipping this is the #1 cause of `validation_error`.
2. **Plan the content in enhanced markdown.** Decide block types and colors
   before emitting. If anything about the syntax is uncertain, open the relevant
   file in `references/` — don't improvise.
3. **Make the smallest edit that does the job.** Prefer `update_content`
   (search-and-replace) or `insert_content` over `replace_content`. Resubmitting
   a whole page is slower and more likely to hit async limits, and risks
   deleting child pages.
4. **Verify by re-fetching.** Especially for tables and colors: fetch the page
   again and compare against what the user asked for (or the image they sent).
   Fix mismatches before declaring done.

```
notion-fetch(id=URL)                      # read current state
→ plan content (enhanced markdown)
→ notion-update-page(command="update_content",
      page_id=..., content_updates=[{old_str, new_str}])
→ notion-fetch(id=URL)                    # verify
```

## Tool catalog

All tools are `Notion:notion-*`. Fetch first whenever you need IDs or exact text.

| Tool | What it does / when to use |
|------|----------------------------|
| `notion-fetch` | Read a page, database, or data source by URL/ID. `id="self"` → workspace + your user identity. `include_discussions=true` shows comment markers; `include_transcript=true` includes meeting transcripts. **Always your first call.** |
| `notion-search` | Semantic search across the workspace (and Slack/Drive/GitHub/Jira if connected). `query_type:"user"` finds people by name/email. Use to locate a page/DB when you don't have the URL. |
| `notion-create-pages` | Create one or more pages. Parent = `page_id` (normal page) or `data_source_id` (database row). `content` is enhanced markdown; `properties` is a JSON map; `template_id` applies a template; `icon`/`cover` optional. |
| `notion-update-page` | Edit a page. `command` ∈ `update_content`, `insert_content`, `replace_content`, `update_properties`, `apply_template`, `update_verification`. See command table below. |
| `notion-duplicate-page` | Copy a page (async — the copy fills in shortly; tell the user to check back). |
| `notion-move-pages` | Move page(s) under a new parent page or database. |
| `notion-create-database` | Create a database from a **SQL DDL** `CREATE TABLE (...)` schema. Select/status/multi-select option colors are set here. |
| `notion-update-data-source` | Change a database's schema with DDL (`ADD/DROP/RENAME/ALTER COLUMN`), or its title/description/inline/trash. |
| `notion-query-data-sources` | Query rows with **SQLite** against `collection://` URLs, or run a saved view (`mode:"view"`). For filtering, aggregation, reporting. (Plan limits apply — see reference.) |
| `notion-query-database-view` | Run one saved view's filters/sorts by its `?v=` URL. |
| `notion-create-view` / `notion-update-view` | Create or edit a view (table/board/calendar/timeline/gallery/chart/form/etc.) using the **view DSL** (`FILTER`, `SORT BY`, `GROUP BY`, …). |
| `notion-create-comment` | Add a page comment, comment on a selection, or reply to a discussion thread. |
| `notion-get-comments` | List open comments/discussions on a page or block. |
| `notion-get-users` / `notion-get-teams` | List workspace members / teamspaces (for mentions, assignments, scoping). |
| `notion-query-meeting-notes` | Find the current user's AI meeting notes by date/attendee/title filters. |
| `notion-get-async-task` | Poll an async task (returned when a write sets `allow_async:true` or a big edit is queued). |

## Picking the right `notion-update-page` command

| Goal | command + key args |
|------|--------------------|
| Replace a `//for claude` marker or edit an exact existing snippet | `update_content`, `content_updates=[{old_str, new_str}]` |
| Append to the end of a page | `insert_content`, `content=...` (omit `position`, or `position={"type":"end"}`) |
| Prepend to the top | `insert_content`, `content=...`, `position={"type":"start"}` |
| Rewrite the whole page | `replace_content`, `new_str=...` (last resort; may need `allow_deleting_content`) |
| Update database row fields | `update_properties`, `properties={...}` |
| Apply a template to an existing page | `apply_template`, `template_id=...` |
| Set/replace icon or cover | any command + `icon=...` and/or `cover=...` |

Notes that prevent errors:
- `update_content` takes a **`content_updates` array**, not loose `old_str`/`new_str`. Each item's `old_str` must match exactly **once**; if it appears multiple times, add `"replace_all_matches": true` or extend the snippet.
- `replace_content` refuses to delete child pages/databases unless you set `allow_deleting_content: true`. **Never set this silently** — list the affected pages and confirm with the user first.

## Enhanced markdown — inline formatting

```
**bold**      *italic*      ~~strikethrough~~      `inline code`
<span underline="true">underline</span>
[link text](https://url.com)
<br>                                   ← line break inside the same block
$E = mc^2$                             ← inline math
<span color="red">colored text</span>
<span color="red">**bold + colored**</span>
```

### The color palette — these 9 hues are the ONLY colors that exist

| Text color | Background color |
|------------|------------------|
| `gray` `brown` `orange` `yellow` `green` `blue` `purple` `pink` `red` | `gray_bg` `brown_bg` `orange_bg` `yellow_bg` `green_bg` `blue_bg` `purple_bg` `pink_bg` `red_bg` |

There is **no** `teal`, `cyan`, `navy`, `sky`, `lime`, `magenta`, `gold`,
`light_*`, or `*_light`. Map any such color to the nearest hue above (e.g. teal →
`blue` or `green`, navy/sky → `blue`, magenta/rose → `pink`, gold/cream →
`yellow`). Inventing a token makes the color silently fail.

**Block-level color** — add `{color="..."}` to the first line of any block:
```
## Heading {color="blue"}
- List item {color="green_bg"}
> Quote {color="gray_bg"}
Paragraph text {color="yellow_bg"}
```

## Block types (quick reference)

```
# H1   ## H2   ### H3   #### H4          (headings take no children; H5/H6 → H4)
- bullet            (TAB to nest children)
1. numbered
- [ ] todo          - [x] done
> quote             (multi-line: > Line 1<br>Line 2)
---                 (divider)
<empty-block/>      (preserve a blank line; bare blank lines are stripped)
```

````
```python
code goes here          # never escape inside code fences
```
````

```html
<callout icon="💡" color="blue_bg">
	Callout text. Can contain multiple indented child blocks.
</callout>

<details color="gray_bg">
<summary>Toggle title</summary>
	Hidden content (indent children with TAB)
</details>

<columns>
	<column>Left</column>
	<column>Right</column>
</columns>
```

Toggle heading: `## Section {toggle="true" color="blue_bg"}` with TAB-indented
children. For callouts, toggles, columns, mentions, equations, media, synced
blocks, page/database links, and TOC, see `references/blocks-and-extras.md`.

---

## Tables & colors — the part people get wrong

This is the most common failure, so follow it closely. A Notion table is a
`<table>` of `<tr>` rows and `<td>` cells. **Cells hold rich text only** — no
headings, lists, callouts, or nested tables inside a cell.

### Rule 0 — pipes vs HTML
If the table needs **any** color or per-cell formatting, you **must** use the
HTML `<table>` form. Markdown pipe tables (`| a | b |` / `|---|`) cannot carry
color — every color you write there is silently dropped. Use pipes only for
plain, uncolored data where simplicity helps.

### The four pieces of table syntax
```html
<table fit-page-width="true" header-row="true" header-column="false">
	<colgroup>
		<col color="gray_bg">    <!-- column 1 background -->
		<col>                    <!-- column 2: no background -->
		<col color="yellow_bg">  <!-- column 3 background -->
	</colgroup>
	<tr>
		<td>**Header A**</td>
		<td>**Header B**</td>
		<td color="green_bg">**Header C**</td>   <!-- cell overrides its column -->
	</tr>
	<tr>
		<td>plain</td>
		<td><span color="red">**red text**</span></td>
		<td>line one<br>line two</td>
	</tr>
</table>
```
- `fit-page-width` `"true"`=full width, `"false"`=natural width.
- `header-row` / `header-column` style the first row / first column as a header.
- The number of `<col>` in `<colgroup>` must equal the number of `<td>` per row
  — include an empty `<col>` for uncolored columns so colors don't shift.

### Color precedence: cell > row > column
Set the broadest level that's true, then override exceptions at a finer level:
- whole **column** one fill → set it once in `<colgroup>` via `<col color="…">`
- whole **row** one fill → `<tr color="…">`
- a single **cell** differs → `<td color="…">` (this wins over row and column)

### Fill vs text — the distinction that fixes most "wrong color" bugs
Look at each colored thing in the target and ask *what* is colored:
- **The whole cell is shaded** → use a **background** token (`*_bg`) at the
  cell/row/column level. e.g. `<td color="yellow_bg">`.
- **Only the letters are colored, cell stays white** → use a **plain** hue on an
  inline span: `<td><span color="red">text</span></td>`.
- **Both** (e.g. red text on a yellow cell) → combine:
  `<td color="yellow_bg"><span color="red">**text**</span></td>`.

### Recreating a colored table from an image — systematic method
When the user sends a screenshot/photo of a table to reproduce:
1. **Map the grid.** Count rows and columns exactly. Note merged cells (Notion
   can't merge — approximate by repeating or leaving blank, and tell the user).
2. **Read each cell's two color channels separately:** its fill (background) and
   its text color. Write them down per cell before emitting anything.
3. **Snap every color to one of the 9 hues** (use the mapping in Rule "palette"
   above; full table in `references/tables-and-color.md`). Pick `*_bg` for fills,
   plain hue for text.
4. **Choose the level** for each fill (column via `<colgroup>`, row via `<tr>`,
   exceptions via `<td>`) to keep the markup clean and correct.
5. **Headers:** if the image's header row is a *specific* color, prefer
   `header-row="false"` and color the header cells yourself so the color matches
   exactly. Use `header-row="true"` only when Notion's default header shading is
   acceptable.
6. **Emit, then verify.** Re-fetch the page and compare the rendered table to the
   image cell by cell; correct any fill/text/precedence mismatch.

For a full color-mapping table, header-behavior details, and several worked
patterns (vocabulary tables, comparison grids, direction tables), read
`references/tables-and-color.md`.

---

## Databases — quick start (full details in references/databases-and-queries.md)

- **Create** with SQL DDL. Option colors live in the schema — column names in
  double quotes, option names in single quotes:
  ```
  notion-create-database(title="Tasks",
    schema='CREATE TABLE ("Task" TITLE, "Status" SELECT('To Do':red, 'Doing':yellow, 'Done':green), "Due" DATE, "Done?" CHECKBOX)')
  ```
- **Change schema** with `notion-update-data-source` DDL (`ADD COLUMN "P" SELECT(...)`, `RENAME`, `DROP`, `ALTER COLUMN ... SET`).
- **Add a row** with `notion-create-pages`, `parent={data_source_id: "<collection-uuid>"}`. Expanded property formats: dates split into `date:Prop:start` / `date:Prop:end` / `date:Prop:is_datetime`; checkboxes use `"__YES__"`/`"__NO__"`; numbers are real numbers; a property literally named `id`/`url` must be prefixed `userDefined:`.
- **Query** with `notion-query-data-sources` (SQLite over `collection://…`) or run a saved view.

Read `references/databases-and-queries.md` for the DDL type catalog (relations,
rollups, formulas, unique IDs), property formats, SQL querying, large-database
windowing, and templates.

## Views — quick start (full details in references/databases-and-queries.md)

Create/update views with a DSL string in `configure`:
```
notion-create-view(database_id=..., data_source_id=..., name="Active",
    type="table", configure='FILTER "Status" = "In Progress"; SORT BY "Due" ASC')
notion-update-view(view_id=..., configure='CLEAR FILTER; GROUP BY "Priority"')
```
Board needs `GROUP BY`, calendar needs `CALENDAR BY`, timeline needs
`TIMELINE BY "Start" TO "End"`, map needs `MAP BY`, chart needs `CHART …`. The
MCP view DSL is **not** the REST API's JSON filter object — use the DSL.

## Other capabilities

Comments & discussions, @mentions, equations, media/files, synced blocks,
meeting notes, workspace identity (`self`), duplicate/move, and async handling
are covered in `references/blocks-and-extras.md`.

**Common end-to-end workflows** — writing homework/answers into a page,
recreating a colored table from a photo, building a tracker database, making a
study page, idempotent re-runs, and bulk operations — have ready-to-adapt
playbooks in `references/recipes.md`. Reach for it when starting a multi-step
Notion job.

---

## Hard limits (impossible via the API — don't promise these)
- Merging/spanning table cells; custom table column widths.
- More than one shade per hue (only one `gray_bg`, etc.).
- Per-row background color *inside a database* (table-block rows can be colored;
  database rows cannot).
- Font size/family changes; page background color.

Possible: column- and cell-level table backgrounds, inline colored/bold/italic
text anywhere, callouts with emoji+background, block-level color on
headings/lists/quotes, toggles, columns, dividers, full-width tables, and
select/status option colors in databases.

## Error quick-reference

| Error | Cause | Fix |
|-------|-------|-----|
| `validation_error` — `old_str` not found | Snippet doesn't match the page | Re-fetch; copy exact text incl. whitespace/case |
| `validation_error` — multiple matches | `old_str` occurs >1 time | Add `"replace_all_matches": true` or lengthen the snippet |
| `validation_error` — would delete children | `replace_content` would drop a child page/db | Confirm with user, then set `allow_deleting_content: true`, or re-include the child via `<page url=...>` |
| `object_not_found` | Wrong ID or no access | Check ID; ensure the Notion connection has access to the page |
| Color/format ignored | Pipe table used, or invented color token, or fill/text confused | Switch to `<table>`; snap to the 9 hues; pick `_bg` vs plain hue correctly |
| Stuck "in progress" | Large/async write was queued | Poll `notion-get-async-task` with the returned `task_id` |

When any syntax is uncertain, the answer is in this skill's `references/` files —
read them rather than guessing.

## Before you finish — checklist

Run this quick check before telling the user a Notion task is done:
- **Verified by re-fetch?** You fetched the page again and the change is actually
  there (not just that the tool returned 200).
- **Colors right?** If a table or content was meant to match an image/spec: fills
  use `*_bg`, text uses plain hues, every color is one of the 9, and a `<table>`
  (not pipes) was used wherever color appears.
- **Nothing clobbered?** No sibling content or child pages were deleted; you used
  the smallest edit (`update_content`/`insert_content`) rather than a full
  `replace_content`.
- **No duplication?** On a possible re-run, you checked the marker/row state first
  so content wasn't added twice.
- **Async settled?** If a write or duplicate/template was queued, you confirmed it
  finished (or told the user it's still processing and how to check).
