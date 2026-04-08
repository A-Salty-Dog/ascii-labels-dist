# ascii-labels

**Design labels, badges, and tickets with nothing but a text editor.**

ascii-labels is a Go command-line tool that turns plain-text ASCII art layouts into production-quality **SVG** and **PDF** output. You draw the layout with characters, point it at a CSV, and get one polished label per row — automatically.

No graphic design software. No pixel coordinates. No scripting.

---

## How it works

You define your label layout using ASCII art — boxes, lines, and a digit inside each area to tell the tool what goes there. Then you configure fonts, sizes, barcodes, and images in a simple INI block right below the drawing. Feed it a CSV and it renders one label per row.

```
+------------------------------------------+
| +----------------+  +----------------+   |
| |1               |  |2               |   |
| |                |  |                |   |
| +----------------+  +----------------+   |
|                  +                       |
+------------------------------------------+
|                                          |
| +------------------------------------+   |
| |3                                   |   |
| |                                    |   |
| +------------------------------------+   |
+------------------------------------------+

[0]
char_width  = 10
char_height = 20
pdf_output  = 1
csv_file    = data.csv

[1]
type  = text
font  = Arial
size  = 24
align = center

[2]
type   = image
stroke = 0

[3]
type    = code
format  = qr
caption = above
```

```csv
title,logo,qr
Alpha Corp,logo_alpha.png,https://example.com/alpha
Beta Ltd,logo_beta.png,https://example.com/beta
```

Running the tool produces `layout_001.svg`, `layout_002.svg`, and a multi-page `labels.pdf`. One row, one label.

---

## Features

- **ASCII-driven layout** — draw your template in any text editor, no coordinates required
- **SVG + PDF output** — pixel-perfect SVG per label, multi-page PDF with exact canvas dimensions
- **Text, images, barcodes** — each area can hold a text field, a local or remote image, or a code symbol
- **QR codes, EAN-13, Code 128** — just set `type = code` and `format = qr / ean13 / code128`
- **Reflowable background text** — point `text_file` at a local file or a URL and it flows into the background of every label automatically
- **Custom fonts** — drop a `.ttf` in the working directory and reference it by name; falls back gracefully to Helvetica
- **Rounded corners** — one `radius` key in the config, globally or per block
- **Zero external layout engine** — pure Go, static analysis of the ASCII grid, no JavaScript, no browser, no Electron

---

The tool reads `template.txt` (which contains both the ASCII layout and the INI config), then loads the CSV and optional text file referenced inside it. Outputs land in the current directory.

---

## Template quick reference

### Layout characters

| Character | Meaning |
|-----------|---------|
| `+` | Corner or junction |
| `-` | Horizontal segment |
| `\|` | Vertical segment |
| `1`–`9` | Data block ID (maps to CSV column) |
| ` ` | Open area for background text flow |

### Block types

| `type` | Effect |
|--------|--------|
| `text` | Renders the CSV column as a text label |
| `image` | Loads a file path or URL and fits the image proportionally |
| `code` | Encodes the value as a barcode or QR code |

### Barcode formats

| `format` | Effect |
|----------|--------|
| `qr` | QR code (error correction: `low` / `medium` / `high` / `highest`) |
| `ean13` | EAN-13 linear barcode |
| `code128` | Code 128 (full ASCII) |

### Key global settings (`[0]`)

| Key | Default | Description |
|-----|---------|-------------|
| `char_width` | `10` | Pixel width of one character cell |
| `char_height` | `20` | Pixel height of one character cell |
| `pdf_output` | `0` | Set to `1` to generate a PDF |
| `csv_file` | `data.csv` | Path to the data file |
| `pdf_file` | `labels.pdf` | Output PDF path |
| `text_file` | — | Path or URL of background text |
| `font` | Helvetica | Default font for all blocks |
| `size` | `12` | Default font size in points |
| `radius` | `0` | Corner radius in pixels (0 = sharp) |

---

## Installation

Download the Windows executable directly from the [ditribution project page](https://github.com/A-Salty-Dog/ascii-labels-dist).

No installation needed — just unzip and run.

---

## Dependencies included

| Module | Purpose |
|--------|---------|
| `github.com/jung-kurt/gofpdf` | PDF generation and font metrics |
| `github.com/boombuler/barcode` | EAN-13 and Code 128 encoding |
| `github.com/skip2/go-qrcode` | QR code matrix generation |

---

## License

Proprietary
