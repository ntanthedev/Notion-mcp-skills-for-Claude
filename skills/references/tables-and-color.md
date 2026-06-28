# Tables & color — deep reference

Read this when building any colored table, and especially when reproducing a
table from an image so the colors match. The core rules live in `SKILL.md`; this
file adds the full color map, header behavior, and worked patterns.

## Contents
- [Valid colors and image→Notion mapping](#valid-colors-and-imagenotion-mapping)
- [Fill vs text, with examples](#fill-vs-text-with-examples)
- [Precedence and the colgroup mechanism](#precedence-and-the-colgroup-mechanism)
- [Header row / header column behavior](#header-row--header-column-behavior)
- [Reproducing a table from an image](#reproducing-a-table-from-an-image)
- [Worked patterns](#worked-patterns)
- [Troubleshooting](#troubleshooting)

---

## Valid colors and image→Notion mapping

Notion has exactly **9 hues**, each available as text color (`red`) or background
(`red_bg`), plus `default`. Nothing else exists. When a source image shows a
color outside this set, snap it to the nearest hue — never invent a token.

| If the image looks… | Use hue | Fill token | Text token |
|---------------------|---------|-----------|-----------|
| red, crimson, coral, salmon, tomato | red | `red_bg` | `red` |
| orange, amber, peach, apricot | orange | `orange_bg` | `orange` |
| yellow, gold, mustard, cream, light tan-yellow | yellow | `yellow_bg` | `yellow` |
| green, lime, mint, olive, emerald | green | `green_bg` | `green` |
| blue, cyan, sky, navy, **teal** (cool), azure, indigo-blue | blue | `blue_bg` | `blue` |
| purple, violet, lavender, indigo-purple, plum | purple | `purple_bg` | `purple` |
| pink, magenta, rose, fuchsia, hot pink | pink | `pink_bg` | `pink` |
| brown, tan, beige, khaki, taupe, chocolate | brown | `brown_bg` | `brown` |
| gray, grey, silver, slate, light-black | gray | `gray_bg` | `gray` |
| white / no fill / transparent | default | *(omit color)* | `default` |

Ambiguous cases:
- **Teal/aqua** sits between blue and green. Pick whichever it's visually closer
  to; if unsure, `blue`/`blue_bg`. Mention the choice if precision matters.
- **Very light pastel fills** still map to the same hue's `_bg` — there is only
  one shade per hue, so a pale yellow and a strong yellow both become
  `yellow_bg`. You cannot reproduce two different intensities of the same hue.
- **Light gray vs white:** if the cell is clearly shaded (even faintly), use
  `gray_bg`; if it's the page background, omit color.

## Fill vs text, with examples

Read the cell as two independent channels — background fill and text color —
and reproduce each.

| What you see in the image | Markup |
|---------------------------|--------|
| Cell filled yellow, black text | `<td color="yellow_bg">text</td>` |
| White cell, red text | `<td><span color="red">text</span></td>` |
| Yellow cell, red text | `<td color="yellow_bg"><span color="red">text</span></td>` |
| Yellow cell, bold red text | `<td color="yellow_bg"><span color="red">**text**</span></td>` |
| White cell, bold black text (header look) | `<td>**text**</td>` |
| Green cell, two lines | `<td color="green_bg">line 1<br>line 2</td>` |

A cell's `color=` attribute always sets its **background**. Text color always
comes from an inline `<span color="...">`. Bold/italic are `**`/`*` as usual and
combine freely with color.

## Precedence and the colgroup mechanism

When a cell, its row, and its column all specify a color, **cell wins, then row,
then column**. Use this to set the common case broadly and override exceptions.

`<colgroup>` defines per-column defaults. It must contain exactly one `<col>`
per column, in order — including empty `<col>` entries for columns with no fill,
or every subsequent color lands one column to the left.

```html
<table fit-page-width="true" header-row="false" header-column="false">
	<colgroup>
		<col color="gray_bg">   <!-- col 1 all gray -->
		<col>                   <!-- col 2 no default -->
		<col color="blue_bg">   <!-- col 3 all blue -->
	</colgroup>
	<tr>
		<td>Label</td>
		<td>Value</td>
		<td color="red_bg">Exception: this one cell is red, overriding blue</td>
	</tr>
	<tr color="yellow_bg">
		<td>This whole row is yellow…</td>
		<td>…including this cell…</td>
		<td>…and this one (row beats the column's blue)</td>
	</tr>
</table>
```

## Header row / header column behavior

`header-row="true"` makes Notion render the first row with its built-in header
styling (bold text, subtle shading). `header-column="true"` does the same for the
first column. This is convenient but the shading is Notion's default — you don't
control its exact color.

Therefore:
- If the target's header is just **bold with light/no shading**, `header-row="true"`
  is the simplest faithful choice.
- If the target's header is a **specific color** (e.g. a dark blue header band
  with white text), set `header-row="false"` and style the header cells yourself:
  ```html
  <tr>
  	<td color="blue_bg"><span color="default">**Name**</span></td>
  	<td color="blue_bg"><span color="default">**Score**</span></td>
  </tr>
  ```
  (Notion has no white text token; on a dark fill, leaving text `default` renders
  as the theme's contrast color. If the design truly needs white-on-color, note
  that exact white text isn't an API-controllable option.)

## Reproducing a table from an image

A reliable, repeatable procedure:

1. **Transcribe structure first.** Write out the grid as rows × columns in plain
   text, filling each cell's text content. Confirm the column count is identical
   across rows. Flag any visually merged cells (Notion can't merge — repeat the
   value, or leave a blank cell, and tell the user).
2. **Annotate colors per cell.** For every cell, record two things: fill
   (background) and text color. A quick notation helps, e.g. `R1C2: fill=yellow,
   text=red,bold`.
3. **Snap to the 9 hues** using the mapping table above. Decide `_bg` for fills
   and plain hue for text.
4. **Factor the fills by level.** If an entire column shares a fill, move it to
   `<colgroup>`; an entire row → `<tr color>`; leftover unique cells → `<td color>`.
   This keeps the markup small and the precedence correct.
5. **Decide headers** per the section above.
6. **Choose width:** `fit-page-width="true"` for wide/reference tables;
   `"false"` for compact ones the user wants narrow.
7. **Emit the `<table>`**, then **re-fetch and compare to the image** cell by
   cell. Most defects are: pipe table used by mistake, a fill rendered as text
   color (or vice-versa), an invented color token, or a `<colgroup>` count
   mismatch shifting colors. Fix and re-verify.

## Worked patterns

### Pattern A — vocabulary / two-section reference table
Label columns gray, term columns bold+red, example columns plain. Two logical
sections side by side (common for IELTS/language notes).

```html
<table fit-page-width="true" header-row="false" header-column="false">
	<colgroup>
		<col color="gray_bg">
		<col>
		<col>
		<col color="gray_bg">
		<col>
		<col>
	</colgroup>
	<tr>
		<td>Đổi mục đích sử dụng</td>
		<td><span color="red">**Converted into**<br>**Repurposed**<br>**Transformed into**</span></td>
		<td>The studio was <span color="red">**converted into**</span> apartments.</td>
		<td>Xây mới</td>
		<td><span color="red">**Constructed**<br>**Established**<br>**Erected**</span></td>
		<td>A new center <span color="red">**was constructed**</span>.</td>
	</tr>
</table>
```

### Pattern B — colored direction / category grid
Each column a different fill, no header.

```html
<table fit-page-width="false" header-row="false" header-column="false">
	<colgroup>
		<col color="yellow_bg">
		<col color="green_bg">
		<col color="pink_bg">
	</colgroup>
	<tr>
		<td>In the northwest,<br>In the northwestern part,</td>
		<td>In the north,<br>In the northern part,</td>
		<td>In the northeast,<br>In the northeastern corner,</td>
	</tr>
</table>
```

### Pattern C — comparison table with a colored header band
Header row a single color; body cells signal status with text color.

```html
<table fit-page-width="true" header-row="false" header-column="true">
	<colgroup>
		<col color="gray_bg">
		<col>
		<col>
	</colgroup>
	<tr>
		<td color="blue_bg">**Feature**</td>
		<td color="blue_bg">**Plan A**</td>
		<td color="blue_bg">**Plan B**</td>
	</tr>
	<tr>
		<td>**Price**</td>
		<td><span color="green">$10</span></td>
		<td><span color="red">$25</span></td>
	</tr>
	<tr>
		<td>**Seats**</td>
		<td>5</td>
		<td><span color="green">Unlimited</span></td>
	</tr>
</table>
```

### Pattern D — status matrix where individual cells differ
Most cells inherit a column fill; a few cells override.

```html
<table fit-page-width="true" header-row="true" header-column="false">
	<colgroup>
		<col>
		<col color="gray_bg">
		<col color="gray_bg">
	</colgroup>
	<tr>
		<td>**Task**</td>
		<td>**Q1**</td>
		<td>**Q2**</td>
	</tr>
	<tr>
		<td>Design</td>
		<td color="green_bg">Done</td>
		<td color="yellow_bg">In progress</td>
	</tr>
	<tr>
		<td>Build</td>
		<td color="red_bg">Blocked</td>
		<td>Planned</td>
	</tr>
</table>
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| No colors at all | Used `|`-pipe table | Rewrite as `<table>` |
| Color appears on wrong columns | `<colgroup>` has fewer `<col>` than columns | Add an empty `<col>` for every uncolored column |
| Cell shaded but text was meant to be colored | Put the hue on `<td color>` instead of an inline span | Move text color to `<span color="...">` inside the cell |
| Requested teal/navy/etc. missing | Invented a nonexistent token | Snap to nearest of the 9 hues |
| Two shades of the same color look identical | Notion has one shade per hue | Use a different hue, or accept a single shade |
| Header color won't match the image | Relied on `header-row="true"` default shading | Set `header-row="false"` and color header cells explicitly |
| Cells dropped a list/heading I put inside | Cells accept rich text only | Put block content outside the table; keep cells to inline text + `<br>` |
