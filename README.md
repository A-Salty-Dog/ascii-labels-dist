# ascii-labels

**ASCII-driven label & document layout engine**

Language: Go | Output formats: SVG · PDF

---

## Overview

`ascii-labels` is a Go command-line tool that converts plain-text ASCII art layout templates into production-quality graphic output — individual SVG files and/or a multi-page PDF. The design intent is to enable non-programmers to define complex label, badge, or ticket layouts using nothing but a text editor, while keeping the rendering pipeline fully programmable.

The tool reads three inputs and produces one or more outputs:

- `template.txt` — the ASCII art canvas + INI configuration block
- `data.csv` — one data row per output label (filename configurable via `csv_file`)
- `fixed-text.txt` *(optional)* — a free-flow text document that fills in background frames (filename configurable via `text_file`)

From those inputs it generates:

- `layout_001.svg`, `layout_002.svg`, … — one SVG per CSV data row
- `labels.pdf` — a multi-page PDF when `pdf_output = 1` (filename configurable via `pdf_file`)

### Design Principles

- **Single source of truth:** every dimension is derived from the ASCII grid character metrics; no pixel values in the template.
- **Renderer-agnostic pipeline:** the same intermediate representation (`[]model.Block` + `GlobalConfig`) feeds both the SVG and PDF back-ends through a shared `Renderer` interface.
- **Zero dependencies on external layout engines:** rectangles, lines, points, and text slots are discovered by static analysis of the ASCII art.

---

## Project Structure

The repository is a single Go module (`ascii-labels`) divided into two packages:

| File / Package | Responsibility |
|---|---|
| `main.go` | Entry point; orchestrates parse → render pipeline |
| `parser/` | ASCII template analysis, CSV/text I/O, config parsing |
| `model/` | Shared data types: `Block`, `GlobalConfig` |
| `render/` | SVG and PDF rendering, text-flow engine |

**Parser package files:**

| File | Contents |
|---|---|
| `grid.go` | `Grid` type — bounding-box crop of the raw layout |
| `scanner.go` | `ScanSegments` — detects all `+…+` segments |
| `rectangles.go` | `findRectangles` — closed rectangles; IDs from inner digit |
| `lines.go` | `findLines` — independent lines; removes split frames |
| `points.go` | `findPoints` — isolated `+` markers |
| `parser.go` | `ParseTemplate`, `ReadCSV`, `ReadTextFile`, `ExtractLayout`, `ApplyConfig` |

**Render package files:**

| File | Contents |
|---|---|
| `render.go` | `Renderer` interface, `RenderRow`, `renderCodeBlock`, shared utilities |
| `render_svg.go` | `svgRenderer` — writes SVG markup to an `io.Writer` |
| `render_pdf.go` | `pdfRenderer` — wraps gofpdf, writes a multi-page PDF |
| `text_flow.go` | `FlowTextIntoFrames` — reflowable text layout engine |

---

## Template File Format (`template.txt`)

The template file has two sections separated by the first line that begins with `[` (an INI section header). Lines whose first character is `#` are treated as comments and discarded.

### ASCII Layout Section

The layout section is a plain-text canvas drawn with four characters:

| Character | Meaning |
|---|---|
| `+` | Corner / junction — the only structurally significant character |
| `-` | Horizontal segment between two `+` markers |
| `\|` | Vertical segment between two `+` markers |
| digit `1`–`9` | Written inside a rectangle to assign it a data-column ID |
| *(space)* | Open area — eligible for text-flow slots |

The bounding box of all `+` characters defines the canvas. Everything outside that bounding box is ignored. Coordinates throughout the system are expressed in character cells relative to the top-left `+` of the bounding box.

#### Rectangles

A rectangle is detected when four `+` markers form the four corners of a closed box whose sides are fully composed of `-` (top/bottom) and `|` (left/right). The inner digit on the second row, second column of the interior identifies the rectangle:

- Digit `1`–`9` → the rectangle is a data block; CSV column index = ID.
- No digit (or space) → the rectangle is an anonymous frame used for borders or text-flow.

Duplicate IDs are resolved by keeping the first detected rectangle; subsequent ones are demoted to ID = 0 (frame).

#### Lines

After all rectangles are identified, any remaining segment that is not a side of any rectangle becomes an independent line. A fused segment that completely crosses an existing rectangle from one side to the other causes that rectangle to be removed — the crossing line "splits" the frame.

#### Points

A `+` that is not the corner of any rectangle, not an endpoint of any line, and not a four-way intersection is an isolated point marker rendered as a small square or circle.

### INI Configuration Section

The configuration section starts immediately after the layout. Each block is introduced by `[N]` where `N` is `0` for global settings or `1`–`9` to target a specific rectangle block.

#### Global Section `[0]`

| Key | Description |
|---|---|
| `char_width` | Pixel width of one character cell. Default: `10` |
| `char_height` | Pixel height of one character cell. Default: `20` |
| `stroke_point` | Half-size (radius) in pixels of isolated point markers. Default: `3` |
| `stroke_line` | Stroke width in pixels of independent lines. Default: `2` |
| `stroke_frame` | Default stroke width in pixels for rectangle borders. Default: `1` |
| `radius` | Global corner radius in pixels (`0` = sharp corners). Default: `0` |
| `pdf_output` | `1` to generate the PDF; `0` to skip. Default: `0` |
| `csv_file` | Path to the CSV data file. If omitted or empty, the tool defaults to a single print. |
| `pdf_file` | Output path for the generated PDF. Default: `labels.pdf` |
| `text_file` | Path or HTTP(S) URL of the free-flow text document |
| `font` | Default font name for all blocks. Falls back to Helvetica in PDF |
| `size` | Default font size in points for all blocks. Default: `12` |
| `align` | Default text alignment: `left` \| `center` \| `right`. Default: `left` |
| `fg_color` | Default foreground color in hex format (e.g. `#000000`) and common color names format (e.g., `Black`). Default: `#000000` |
| `bg_color` | Default background color in hex format (e.g. `#FFFFFF`) and common color names format (e.g., `White`). |
| `qr_correction` | Default QR error-correction level: `low` \| `medium` \| `high` \| `highest`. Default: `medium` |

#### Block Sections `[1]`–`[9]`

| Key | Description |
|---|---|
| `type` | Content type: `text` (default) \| `image` \| `code` |
| `font` | Font name. Overrides global default. |
| `size` | Font size in points. Overrides global default. |
| `align` | Text alignment: `left` \| `center` \| `right` |
| `fg_color` | Foreground color. Overrides global default. |
| `bg_color` | Background color. Overrides global default. |
| `stroke` | Border stroke width in pixels (`0` = no border) |
| `radius` | Corner radius in pixels, overrides global radius |
| `format` | For `type=code`: `qr` \| `ean13` \| `code128` |
| `caption` | For `type=code`: `above` \| `below` — prints the raw data string as text |
| `qr_correction` | For `type=code, format=qr`: `low` \| `medium` \| `high` \| `highest`. Overrides global. |

---

## CSV Data File (`data.csv`)

The CSV file is comma-separated with no quoting support. The first row is treated as a header and skipped during rendering. Each subsequent row produces one output label.

If `csv_file` is omitted in the configuration or left empty, the tool defaults to generating exactly one label.

If the CSV file is explicitly specified but is missing, unreadable, or contains no data rows (only the header), the tool will issue a warning and default to generating exactly one label.

Columns map to block IDs: column index 0 → block ID 1, column index 1 → block ID 2, etc.

| Block type | Column value interpretation |
|---|---|
| `text` | Rendered as a text string in the block |
| `image` | Interpreted as a file path or URL; image loaded and fitted proportionally (contain mode, centered) |
| `code` | Encoded into a barcode or QR code; optionally printed as caption |

---

## Free-Flow Text (`text_file`)

When `text_file` is set in `[0]`, the tool loads that file and flows its content into all anonymous frames (ID = 0) found on the canvas. The same text appears on every generated label.

The value of `text_file` can be a relative or absolute file-system path, or an HTTP/HTTPS URL fetched at runtime.

---

## Coordinate System

All coordinates are expressed in character cells and converted to pixels at render time:

```
pixel_x = char_col  × char_width
pixel_y = char_row  × char_height
pixel_w = char_cols × char_width
pixel_h = char_rows × char_height
```

The canvas pixel dimensions are:

```
canvas_width  = (bounding_box_char_width  - 1) × char_width
canvas_height = (bounding_box_char_height - 1) × char_height
```

---

## Worked Example

### Template (`template.txt`)

```
+
 +--------------------------------------------+
 | +------------------+  +------------------+ |
 | |1                 |  |2                 | |
 | |                  |  |                  | |
 | |                  |  |                  | |
 | +------------------+  +------------------+ |
 |  +                                         |
 +--------------------------------------------+
 |                                            |
 | +----------------------------------------+ |
 | |3                                       | |
 | |                                        | |
 | |                                        | |
 | +----------------------------------------+ |
 +--------------------------------------------+
                                               +

[0]
char_width = 10
char_height = 20
stroke_point = 12
stroke_line = 3
stroke_frame = 1
pdf_output = 1
radius = 0
csv_file = data.csv
pdf_file = labels.pdf
text_file = fixed-text.txt
size = 26
align = left
fg_color = Blue
qr_correction = medium

[1]
type = text
font = Arial
size = 24
align = center
stroke = 2
radius = 8
bg_color = Yellow
fg_color = Red

[2]
type = image
stroke = 0

[3]
type = code
format = qr
caption = above
font = Arial
size = 16
stroke = 4
```

### Interpretation

- **Block 1** (top-left): renders the `title` CSV column as centered Arial 24pt text, stroke 2, rounded corners radius 8px.
- **Block 2** (top-right): renders the `logo` CSV column as a proportionally-fitted image, no border.
- **Standalone `+` point**: rendered as a 12px point marker.
- **Block 3** (lower): renders the `qr` CSV column as a QR code with medium error correction, raw string printed above in Arial 16pt.
- **Outer anonymous frame**: receives reflowed text from `fixed-text.txt` using the font configured in `[0]`, 26pt, left-aligned.

### Data File (`data.csv`)

```
title,logo,qr
Alpha Corp,logo_alpha.png,https://example.com/alpha
Beta Ltd,logo_beta.png,https://example.com/beta
Gamma Inc,logo_gamma.png,https://example.com/gamma
```

---

## Quick Reference

### Block `type` values

| Value | Effect |
|---|---|
| `text` *(default)* | CSV column rendered as a text label inside the block |
| `image` | CSV column is a file path or URL; image is contain-fitted and centred |
| `code` | CSV column is encoded as a barcode; `format` and `caption` sub-keys apply |

### `format` values (`type = code`)

| Value | Effect |
|---|---|
| `qr` | QR code; error-correction level from `qr_correction` |
| `ean13` | EAN-13 linear barcode |
| `code128` | Code 128 linear barcode (full ASCII) |

### `qr_correction` values

| Value | Effect |
|---|---|
| `low` | ≈7% codeword error recovery capacity |
| `medium` *(default)* | ≈15% codeword error recovery capacity |
| `high` | ≈25% codeword error recovery capacity |
| `highest` | ≈30% codeword error recovery capacity |

### `caption` values (`type = code`)

| Value | Effect |
|---|---|
| `above` | Prints the raw data string above the code symbol |
| `below` | Prints the raw data string below the code symbol |
| *(omitted)* | No text caption |

### `align` values

| Value | Effect |
|---|---|
| `left` | Text anchored to left edge |
| `center` | Text horizontally centred in the block |
| `right` | Text anchored to right edge |

---

## External Dependencies

| Module | Purpose |
|---|---|
| `github.com/jung-kurt/gofpdf` | PDF generation and font metrics |
| `github.com/boombuler/barcode` | EAN-13 and Code 128 barcode encoding |
| `github.com/skip2/go-qrcode` | QR code matrix generation |

---

## Installation

Download the executable directly from the [ditribution project page](https://github.com/A-Salty-Dog/ascii-labels-dist).

No installation needed.