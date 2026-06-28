# Databases, queries, views & templates — reference

How databases work through the MCP tools. Notion's model: a **database** is a
container; under it sit one or more **data sources** (collections); each data
source holds **pages** (rows) and defines the **schema** (columns/properties).
`notion-fetch` on a database shows its data sources as
`<data-source url="collection://<id>">` — that `collection://` id is what you
pass to row creation, schema updates, and queries.

## Contents
- [Create a database (SQL DDL)](#create-a-database-sql-ddl)
- [Property type catalog](#property-type-catalog)
- [Select / status / multi-select option colors](#select--status--multi-select-option-colors)
- [Update a schema](#update-a-schema)
- [Add rows (pages) and property formats](#add-rows-pages-and-property-formats)
- [Update row properties](#update-row-properties)
- [Query rows](#query-rows)
- [Very large databases](#very-large-databases)
- [Views (DSL)](#views-dsl)
- [Templates](#templates)

---

## Create a database (SQL DDL)

`notion-create-database` takes a **`schema`** that is a SQL `CREATE TABLE`
statement. Column names are double-quoted; option values use single quotes.
`parent` (a `page_id`) and `title`/`description` are optional. If no `TITLE`
column is given, a `Name` title column is added automatically.

```
notion-create-database(
  title="Tasks",
  parent={"page_id": "PARENT_PAGE_ID"},
  schema='CREATE TABLE (
    "Task" TITLE,
    "Status" SELECT('To Do':red, 'Doing':yellow, 'Done':green),
    "Priority" SELECT('High':red, 'Med':yellow, 'Low':blue),
    "Tags" MULTI_SELECT('eng':blue, 'design':pink),
    "Due" DATE,
    "Budget" NUMBER FORMAT 'dollar',
    "Done?" CHECKBOX,
    "Ticket" UNIQUE_ID PREFIX 'TSK'
  )'
)
```

The response includes the new data source id in a `<data-source>` tag — save it
for adding rows and querying.

> Quoting: column **names** go in double quotes (`"Task"`), option **names** and
> string options go in single quotes (`'Done':green`, `FORMAT 'dollar'`). Don't
> double the single quotes — write each option once as `'name':color`. Colors are
> bare words (`red`, not `'red'`).

## Property type catalog

Simple types: `TITLE` (exactly one per data source), `RICH_TEXT`, `DATE`,
`PEOPLE`, `CHECKBOX`, `URL`, `EMAIL`, `PHONE_NUMBER`, `STATUS`, `FILES`,
`CREATED_TIME`, `LAST_EDITED_TIME`.

Configured types:
- `SELECT('opt':color, ...)` and `MULTI_SELECT('opt':color, ...)`
- `NUMBER` or `NUMBER FORMAT 'dollar'` (other formats: `number`, `percent`,
  `euro`, etc.)
- `FORMULA('expression')`
- `RELATION('data_source_id')` — one-way link to another data source
- `RELATION('data_source_id', DUAL)` — two-way link
- `RELATION('data_source_id', DUAL 'synced_name')` — two-way, names the back-link
- `RELATION('data_source_id', DUAL 'synced_name' 'synced_id')` — needed for a
  **self-relation** (a data source related to itself, e.g. parent/sub-items)
- `ROLLUP('relation_prop', 'target_prop', 'function')` — e.g. `'sum'`, `'count'`
- `UNIQUE_ID` or `UNIQUE_ID PREFIX 'X'`
- Append `COMMENT 'description'` to any column to document it.

Option/relation colors: `default, gray, brown, orange, yellow, green, blue,
purple, pink, red`.

Self-relations are a two-step move: create the database first, then use
`notion-update-data-source` with the data source's own id to add the paired
relation columns (so the id is known).

## Select / status / multi-select option colors

This is a distinct kind of color from table-cell backgrounds: here you're
coloring the **pills/tags** shown in a database. They are set in the schema, not
in markdown. To match a screenshot of a database, read each option's pill color
and map it to one of the 10 option colors above, then encode it in the
`SELECT(...)`/`MULTI_SELECT(...)`/status definition. `STATUS` options and their
groups (To-do / In progress / Complete) are managed similarly via DDL.

## Update a schema

`notion-update-data-source` with `statements` (semicolon-separated DDL):
```
notion-update-data-source(data_source_id="collection://<id>", statements='
  ADD COLUMN "Owner" PEOPLE;
  ADD COLUMN "Risk" SELECT('High':red, 'Low':green);
  RENAME COLUMN "Status" TO "State";
  ALTER COLUMN "Priority" SET SELECT('P0':red, 'P1':orange, 'P2':yellow);
  DROP COLUMN "Obsolete"
')
```
Also supports `title`, `description`, `is_inline` (single-source DBs), and
`in_trash:true` (trash the data source). You can't drop/create the title
property, can have at most one `UNIQUE_ID`, and can't update synced databases.

## Add rows (pages) and property formats

Rows are pages whose parent is the data source. Always `notion-fetch` the
database first to get exact property names and the `collection://` id.

```
notion-create-pages(
  parent={"data_source_id": "<collection-uuid>"},
  pages=[{
    "properties": {
      "Task": "Write spec",              // title (name may differ — use schema)
      "Status": "Doing",                 // select/status: option name
      "Tags": "eng,design",              // multi-select: comma-separated
      "Priority": 1,                     // number: real number, not a string
      "Done?": "__YES__",                // checkbox: __YES__ / __NO__
      "date:Due:start": "2026-07-01",    // date start
      "date:Due:end": "2026-07-05",      // date end (optional)
      "date:Due:is_datetime": 0          // 0 = date only, 1 = has time
    },
    "content": "## Notes\nOptional page body in enhanced markdown."
  }]
)
```

Expanded property formats (the parts that trip people up):
- **Date:** split into `date:{Prop}:start`, `date:{Prop}:end` (optional),
  `date:{Prop}:is_datetime` (0/1). For a time, also include a time on the start.
- **Place:** `place:{Prop}:name`, `:address`, `:latitude`, `:longitude`,
  `:google_place_id` (optional).
- **Number:** JavaScript numbers, not strings.
- **Checkbox:** `"__YES__"` / `"__NO__"`.
- **Property literally named `id` or `url`** (case-insensitive): prefix with
  `userDefined:` → `"userDefined:URL"`, `"userDefined:id"`.
- **Files** properties: attach via file-upload ids / external URLs (see
  `blocks-and-extras.md`).

You can create up to 100 pages in one call (same parent). Set `allow_async:true`
for large batches and poll `notion-get-async-task`.

## Update row properties

```
notion-update-page(
  command="update_properties",
  page_id="ROW_PAGE_ID",
  properties={ "Status": "Done", "Priority": 0, "date:Due:start": "2026-07-10" }
)
```
Omitted properties are unchanged; pass `null` to clear a value.

## Query rows

Two MCP paths:

**SQL (SQLite) over data sources** — `notion-query-data-sources`. Use the
`collection://` URL as the table name. Supports parameters; checkboxes use
`__YES__`/`__NO__`.
```
notion-query-data-sources(data={
  "data_source_urls": ["collection://<id>"],
  "query": 'SELECT * FROM "collection://<id>" WHERE "Status" = ? AND "Done?" = ?',
  "params": ["Doing", "__NO__"]
})
```
- Single-source SQL and view mode require a Business plan with Notion AI.
  Cross-source joins in one query require Enterprise + Notion AI. On other plans,
  query one source at a time (or use a saved view via `notion-query-database-view`).
- Great for filtering, aggregation (`COUNT`, `SUM`, `GROUP BY`), and reports.

**Saved view** — run a view's existing filters/sorts:
```
notion-query-database-view(view_url="https://www.notion.so/ws/DB-abc?v=def")
# or:
notion-query-data-sources(data={"mode":"view","view_url":"...?v=def"})
```
When a response has `"has_more": true`, pass its `next_cursor` as `start_cursor`.

## Very large databases

A single query returns at most ~10,000 rows; a naive `next_cursor` loop on a
bigger table silently stops at 10k. To read everything, **window by
`created_time`**: sort ascending, page through, and when a window is capped,
start a new query filtered to `created_time >= last_seen`, de-duplicating by row
id. Use `created_time` (immutable), not `last_edited_time` (shifts rows between
windows). If one minute holds >10k rows (bulk import), add a second narrowing
filter (e.g. a status/category) per window. For ongoing sync, prefer webhooks
over repeated full scans.

## Views (DSL)

`notion-create-view` and `notion-update-view` configure a view with a **DSL
string** in `configure` — not the REST JSON filter object. Provide
`data_source_id` plus exactly one of `database_id` (add a view tab) or
`parent_page_id` (inline linked view on a page).

View types: `table, board, list, calendar, timeline, gallery, form, chart, map,
dashboard`.

Key DSL directives:
- `FILTER "Prop" = "value"` — filter rows (combine with `AND`/`OR`)
- `SORT BY "Prop" ASC|DESC`
- `GROUP BY "Prop"` — **required for board**
- `CALENDAR BY "DateProp"` — **required for calendar**
- `TIMELINE BY "Start" TO "End"` — **required for timeline**
- `MAP BY "LocationProp"` — **required for map**
- `CHART column|bar|line|donut|number` — with optional `AGGREGATE`, `COLOR`,
  `HEIGHT`, `SORT`, `STACK BY`, `CAPTION`
- `FORM CLOSE|OPEN`, `FORM ANONYMOUS true|false`, `FORM PERMISSIONS none|reader|editor`
- `SHOW "P1", "P2"` / `HIDE "P3"` — visible properties
- `COVER "Prop"`, `WRAP CELLS`, `FREEZE COLUMNS n`
- `CLEAR FILTER` / `CLEAR SORT` / `CLEAR GROUP BY` — remove settings (update only)

```
notion-create-view(database_id="DB", data_source_id="collection://<id>",
  name="Sprint board", type="board",
  configure='GROUP BY "Status"; FILTER "Sprint" = "Current"; SORT BY "Priority" DESC')

notion-update-view(view_id="VIEW", configure='CLEAR FILTER; SORT BY "Created" DESC')
```
Selects/status/multi-select filters accept multiple values; people filters accept
`"me"` for the current user.

## Templates

Database templates pre-fill a new row's properties and content.
- Discover template ids: `notion-fetch` the database → `<templates>` section per
  data source.
- Apply on create: pass `template_id` in the page object (and **omit `content`**;
  the template provides it). You may still set `properties` to override defaults.
- Apply to an existing page: `notion-update-page(command="apply_template",
  template_id=...)` (appends template content).
- Template application is **async** — the page appears blank first, then fills in.
  Tell the user, and re-fetch (or poll) to confirm before further edits.
