# Notion Skill for Claude

> A Claude **Agent Skill** that lets AI assistants write to Notion *accurately* —
> correct formatting, faithful colored tables, and reliable database/page
> operations — through the official Notion **MCP** tools.

This skill teaches the model Notion's non-obvious syntax (enhanced
"Notion-flavored Markdown", the SQL-DDL database schema, and the view DSL) and
the exact argument shapes of the `notion-*` MCP tools, so it stops guessing and
starts producing pages that match what you asked for.

## Why this exists

When you ask an AI to put content into Notion, three things commonly go wrong:

1. **Tables lose their colors.** The model emits a plain Markdown pipe table
   (`| a | b |`), which cannot carry color, so a colored table you copied from a
   screenshot comes back gray.
2. **Formatting is approximate.** Background fills and text colors get confused,
   invented color names silently fail, and column colors land on the wrong
   columns.
3. **Tool calls are malformed.** The model uses the wrong command shape (e.g.
   for search-and-replace edits) or the wrong way to define a database, and the
   call errors out or does the wrong thing.

This skill fixes all three. Its headline feature is a step-by-step method for
**recreating a colored table from an image** so the result matches the original.

## What it can do

- **Write & edit page content** in enhanced markdown: headings, lists, to-dos,
  quotes, callouts, toggles, columns, dividers, code blocks, equations, and
  inline formatting (bold/italic/underline/strikethrough/links/color).
- **Build colored tables that match a screenshot** — the core use case — using
  HTML `<table>` with correct fill-vs-text color, the 9-color palette, column/
  row/cell precedence, and a verification step.
- **Targeted edits** via search-and-replace (including replacing markers like
  `//for claude`) without disturbing the rest of the page.
- **Create & manage databases**: define schemas with SQL DDL, set select/status
  option colors, add and update rows, and change schemas safely.
- **Query databases** with SQL or a saved view, including windowing very large
  databases past the 10,000-row query cap.
- **Create & update views** (table, board, calendar, timeline, gallery, chart,
  form, …) with the view DSL.
- **More**: comments & discussions, @mentions, media/files, synced blocks,
  meeting notes, workspace identity, duplicating/moving pages, and async handling.

## Who it's for

- People who use **Claude (or any MCP-capable AI)** to put structured content
  into Notion and want it to come out right the first time.
- The original motivation: having an AI **write lesson and homework content into
  Notion accurately** — for example recreating a colored vocabulary table from a
  photo, or filling an answer under a marker — but the skill is general-purpose
  for any Notion authoring and database work.

## How it works

The skill uses **progressive disclosure** so it stays light until needed:

```
notion/
├── SKILL.md                         # always-on core: workflow, tools, formatting,
│                                    #   the tables-&-color rules, quick references
└── references/                      # loaded only when the task needs them
    ├── tables-and-color.md          # full color map, header behavior, worked tables
    ├── databases-and-queries.md     # DDL schema, querying, views DSL, templates
    ├── blocks-and-extras.md         # callouts/toggles/columns, mentions, comments,
    │                                #   meeting notes, files, identity, async
    └── recipes.md                   # end-to-end playbooks for common jobs
```

Only the short `name` + `description` is always in context; the body loads when
the skill triggers; the reference files load only when a task actually needs
them.

## Requirements

- A Claude client (Claude apps, or Claude Code) with the **Notion MCP connector
  enabled**, so the `notion-*` tools are available.
- The Notion connection must have access (and the right capabilities) to the
  pages/databases you want it to read or edit.

## Installation

**Claude apps (claude.ai / desktop / mobile):** add the packaged `notion.skill`
file in your client's Skills / Capabilities settings.

**Claude Code:** place the `notion/` folder in your skills directory (e.g.
`~/.claude/skills/notion/`).

See Anthropic's current Skills documentation for the exact location in your
client, as the UI evolves.

## Usage examples

Once installed, just ask naturally — the skill triggers on Notion tasks:

- "Here's a screenshot of an IELTS vocabulary table — recreate it in this Notion
  page with the same colors: `<page url>`."
- "Write the solution to this problem under the `//for claude` marker on `<page>`."
- "Create an Assignments database with Status (To Do red, Doing yellow, Done
  green), Subject, Due date, and Score, then add these three rows…"
- "Add a board view grouped by Status to my Tasks database."

## Limitations

Some things are not possible through the API and the skill won't promise them:
merging/spanning table cells, custom table column widths, more than one shade per
color, per-row background color *inside a database*, font size/family changes,
and page background color.

## Credits

Built on Notion's official developer documentation
(<https://developers.notion.com>). Not affiliated with or endorsed by Notion.

## License

No license is set yet. If you intend this to be freely used and modified, add an
open-source license such as **MIT** (recommended for broad reuse).
