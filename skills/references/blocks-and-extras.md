# Blocks & extras — reference

Everything beyond the core: full block syntax, mentions, comments, meeting
notes, files/media, workspace identity, and async handling.

## Contents
- [Callouts, toggles, columns](#callouts-toggles-columns)
- [Media blocks](#media-blocks)
- [Page / database references & TOC](#page--database-references--toc)
- [Synced blocks](#synced-blocks)
- [Equations](#equations)
- [Mentions & custom emoji](#mentions--custom-emoji)
- [Escaping](#escaping)
- [Comments & discussions](#comments--discussions)
- [Meeting notes](#meeting-notes)
- [Workspace identity, users, teams](#workspace-identity-users-teams)
- [Files & media (upload/retrieve)](#files--media-uploadretrieve)
- [Duplicate & move pages](#duplicate--move-pages)
- [Async writes](#async-writes)
- [Reading fetch output](#reading-fetch-output)

---

## Callouts, toggles, columns

```html
<callout icon="💡" color="blue_bg">
	Main text. A callout can hold multiple **indented** child blocks:
	- a list item
	- another line
</callout>

<details color="gray_bg">
<summary>Toggle title</summary>
	Hidden content. Indent children with TAB.
	Can contain lists, paragraphs, even tables.
</details>

<!-- Toggle heading variant -->
## Section title {toggle="true" color="blue_bg"}
	Indented children appear when expanded.

<columns>
	<column>
		Left column blocks
	</column>
	<column>
		Right column blocks
	</column>
</columns>
```
`icon` accepts an emoji, a `:custom_emoji:` name, or an external image URL.

## Media blocks

```
![Caption](https://host/image.png) {color="gray_bg"}
```
```html
<video src="https://host/clip.mp4">Caption</video>
<audio src="https://host/track.mp3">Caption</audio>
<file src="https://host/doc.pdf">Display name</file>
<pdf src="https://host/report.pdf">Report</pdf>
```
URLs may be public links or Notion file-upload references (see Files below). In
`fetch` output, media URLs are pre-signed and expire in ~1 hour — re-fetch to
refresh.

## Page / database references & TOC

```html
<page url="https://notion.so/...">Linked page title</page>
<database url="https://notion.so/..." inline="false" icon="📊">DB title</database>
<table_of_contents color="gray"/>
```
A `<page url=...>` block embeds a link to a child/peer page. When using
`replace_content`, re-include existing child `<page>`/`<database>` tags you want
to keep, or the operation refuses to delete them (and you'd need
`allow_deleting_content`).

## Synced blocks

```html
<!-- Source of truth -->
<synced_block url="https://notion.so/...">
	Content edited here syncs everywhere it's referenced.
</synced_block>

<!-- A reference to an existing synced block -->
<synced_block_reference url="https://notion.so/...">
	Mirrors the source content.
</synced_block_reference>
```

## Equations

Inline: `$E = mc^2$`. Block:
```
$$
\int_0^\infty e^{-x^2}\,dx = \frac{\sqrt{\pi}}{2}
$$
```
KaTeX syntax. Inline math also works inside comments.

## Mentions & custom emoji

```html
<mention-user url="USER_ID">Name</mention-user>
<mention-page url="https://notion.so/...">Page title</mention-page>
<mention-database url="https://notion.so/...">DB name</mention-database>
<mention-data-source url="collection://...">Source name</mention-data-source>
<mention-date start="2026-07-01"/>
<mention-date start="2026-07-01" end="2026-07-05"/>
<mention-date start="2026-07-01" startTime="09:00" timeZone="Asia/Ho_Chi_Minh"/>
```
Self-closing forms like `<mention-user url="..."/>` are valid. Get user ids from
`notion-get-users` or `notion-search(query_type="user")`. Custom emoji inline:
`:emoji_name:`. Citations: `[^https://source-url]`.

Do **not** use editor shortcuts (`@today`, `@name`, `[[page]]`) — those are UI
affordances, not markdown; they won't render through the API.

## Escaping

Outside code blocks, these characters are special and may need a leading
backslash to render literally: `\ * ~ ` $ [ ] < > { } | ^`. Example:
`\*not italic\*` → literal `*not italic*`. **Never** escape inside code fences —
code content is literal.

## Comments & discussions

```
# Page-level comment
notion-create-comment(page_id="PAGE", markdown="Looks good — **ship it**.")

# Comment anchored to a selection (give ~10 chars from start and end)
notion-create-comment(page_id="PAGE",
  selection_with_ellipsis="# Section ti...last words", markdown="Note here.")

# Reply to an existing discussion thread
notion-create-comment(page_id="PAGE",
  discussion_id="discussion://pageId/blockId/discussionId", markdown="Reply.")

# Read open comments
notion-get-comments(...)   # list open discussions on a page/block
```
Comments support inline formatting, inline math (`$x^2$`), and user/page/
database/date mention tags — but block-level markdown (headings, lists, tables,
fenced code) is stored as plain text, not rendered. To find a `discussion_id`,
fetch the page with `include_discussions=true` (the `<page-discussions>` summary
lists `discussion://` ids), or read it from the Notion UI's "Copy link to
discussion". The API can reply to and read open discussions but cannot start a
brand-new inline discussion or read resolved ones.

## Meeting notes

`notion-query-meeting-notes` searches the current user's AI meeting notes by
filters on `title`, attendees (`notion://meeting_notes/attendees`),
`created_time`, `created_by`, `last_edited_time`, `last_edited_by`. By default it
already scopes to meetings where the user is attendee/creator.

Guidance:
- Treat "summaries/notes/todos/action items" as a request for meeting *outcomes*,
  not a title filter — don't filter the title on those words.
- Generic time phrases ("recent", "this week", "yesterday") → date filters, using
  `date_is_within` with relative values like `the_past_week`, or a custom
  relative `{direction, unit, count}`, or an exact range.
- Filter by attendee by resolving the person id first (`notion-search`
  `query_type:"user"`), then `person_contains`.
- Combinator filters use `{ "operator": "and"|"or", "filters": [...] }`.

To read a meeting's generated content, `notion-fetch` the meeting page; add
`include_transcript=true` for the full transcript (otherwise a placeholder shows).

## Workspace identity, users, teams

- `notion-fetch(id="self")` → connected workspace (id, name) and the
  authenticated user (id, name, email). Use to label the connection or get a
  stable user id.
- `notion-get-users` → workspace members (for PEOPLE properties, assignments,
  @mentions).
- `notion-get-teams(query?)` → teamspaces (id, name, your membership/role); useful
  to scope `notion-search` with `teamspace_id`.

## Files & media (upload/retrieve)

Whether file *upload* is available depends on the connected MCP surface; the core
content tools attach files by URL or by a file-upload id. Reference for the
underlying REST behavior:

- **Supported attach targets:** `image`, `file`, `video`, `audio`, `pdf` blocks;
  database `files` properties; page `icon`/`cover`.
- **Direct upload (≤ 20 MB):** create a file-upload object → POST the bytes as
  `multipart/form-data` under the `file` key → attach by its `id` (type
  `file_upload`). Must attach within 1 hour or it's archived. Reusable after the
  first attach.
- **Multi-part (> 20 MB, paid workspaces, up to 5 GB):** create with
  `mode:"multi_part"` + `number_of_parts` → send 10 MB parts (last may be
  smaller) → complete → attach.
- **External URL import:** create with `mode:"external_url"` + `external_url` +
  `filename`; the URL must be public, SSL, expose `Content-Type`, and be within
  the size limit; import is async (poll or use webhooks).
- **Size limits:** free workspaces 5 MiB/file, paid 5 GiB/file. A bot's limit is
  in its user object (`workspace_limits.max_file_upload_size_in_bytes`).
- **Retrieve:** file objects in `fetch` output carry a temporary signed `url`
  that expires in ~1 hour — re-fetch the page/property to refresh. Types:
  `external` (public URL), `file` (uploaded via UI), `file_upload` (uploaded via
  API, becomes `file` after attach).

## Duplicate & move pages

- `notion-duplicate-page(page_id=...)` — copies a page **asynchronously**; the
  returned id/URL isn't populated immediately. Tell the user it's processing and
  to check back via `notion-fetch` or by opening the URL.
- `notion-move-pages(...)` — re-parent page(s) under a new page or database.

## Async writes

Many writes can run in the background. Set `allow_async:true` on
`notion-create-pages`/`notion-update-page` (or a large edit may be queued
automatically). You'll get an `async_task` with a `task_id`; poll
`notion-get-async-task(task_id=...)` until status is `succeeded` (or `failed`),
respecting the suggested backoff. Template application and page duplication are
inherently async.

## Reading fetch output

`notion-fetch` returns Notion-flavored markdown wrapped in tags:
```
<page url="...">
	<ancestor-path> ... </ancestor-path>
	<properties> {...} </properties>
	<content> ...enhanced markdown... </content>
</page>
```
For databases:
```
<database url="...">
	<data-source url="collection://<id>" name="...">
		...schema...
		<templates> ...template ids... </templates>
		<rows> <row ...> ... </row> </rows>
	</data-source>
</database>
```
The `collection://<id>` is the value to pass as `data_source_id` for creating
rows, updating schema, and querying. Copy exact text from `<content>` when
building `old_str` for `update_content`.
