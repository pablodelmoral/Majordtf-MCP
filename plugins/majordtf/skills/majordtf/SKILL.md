---
name: majordtf
description: Prep, vectorize, recolor, or gang-sheet artwork for DTF/transfer printing via majordtf.com — the prep_artwork / vectorize_artwork / recolor_artwork / gangsheet_artwork MCP tools, or the POST /v1/prep, /v1/vectorize, /v1/recolor, /v1/gangsheet HTTP APIs. Use when someone wants a logo/design cleaned, background removed, upscaled, halftoned to a garment colour, traced to a vector SVG, recolored (e.g. white↔black), or several logos laid onto one 21" gang sheet.
---

# MajorDTF — artwork prep, vectorize, recolor & gang sheet

Four operations on an image URL, each returning a clean, signed file URL. Billed **on success
only**, **1 credit per call** (the gang sheet: 1 credit per logo, quantities free):

| Tool | Does | Returns |
|---|---|---|
| **prep_artwork** | Background removed, upscaled, edges hardened, sized to print | 300 DPI transparent **PNG** |
| **vectorize_artwork** | Deterministic Image-Trace of the raster — no AI redraw | clean **SVG** |
| **recolor_artwork** | Swap a colour (white→black) or tint the whole design | **PNG** or **SVG** |
| **gangsheet_artwork** | Prep several logos and lay them onto one 21" sheet | gang-sheet **PNG** |

Each is reachable two ways, same engine and params:

- **MCP tools** — for agents: `prep_artwork` / `vectorize_artwork` / `recolor_artwork` / `gangsheet_artwork`.
- **HTTP** — for scripts/curl: `POST https://worker.majordtf.com/v1/{prep,vectorize,recolor,gangsheet}`.
  Prep also has a CLI wrapper: `pip install majordtf`.

## Auth

Mint a key at <https://www.majordtf.com/account> → API keys (shown once). Format `mdtf_live_...`.
Send it as an `Authorization: Bearer mdtf_live_...` header from your MCP client / curl — keep it in
config, **never paste it into chat**. If a client can't set headers, pass it as the tool's
`api_key` argument instead.

## MCP — install

One line in Claude Code:

```bash
claude mcp add --transport http majordtf https://worker.majordtf.com/mcp/ --header "Authorization: Bearer mdtf_live_..."
```

Or in `.mcp.json` (remote, recommended):

```json
{ "mcpServers": { "majordtf": {
  "url": "https://worker.majordtf.com/mcp/",
  "headers": { "Authorization": "Bearer mdtf_live_..." }
} } }
```

Local stdio alternative: `services/mcp/server.py`, reads `MAJORDTF_API_KEY` from env.

## MCP — call

```python
prep_artwork(
  image_url="https://example.com/logo.png",
  width_in=11,                 # printed width in inches; height follows the art
  height_in=99,                # only honoured when fit="height" / "contain"
  fit="width",                 # width (default) | height | contain
  bg_method="auto",            # auto | key (flat-bg logo) | smart | ai (photo) | keep_alpha
  garment_color="#1a1a1a",     # shirt hex, keyed out for a clean cutout (omit to keep all colours)
  halftone=False,              # True → screen the design into dots that fade into garment_color
  thickness_protection=False,  # 0.5pt garment-colour stroke so thin lines keep their underbase
)
# auth is the Authorization header, NOT a tool arg
# -> { "file_url": "https://…/clean.png", "width": 3300, "height": 4436, "format": "png", ... }

vectorize_artwork(
  image_url="https://example.com/logo.png",
  n_colors=24,                 # >16 keeps ALL colours (default); <=16 reduces to N flat colours
  clean_first=True,            # remove the background before tracing
)
# -> { "file_url": "https://…/trace.svg", "format": "svg", ... }

recolor_artwork(
  image_url="https://example.com/logo.png",
  swap_from="#ffffff", swap_to="#000000",   # white -> black (only the matched colour changes)
  # ...or tint="#000000" to repaint every opaque pixel one colour
)
# -> { "file_url": "https://…/recolor.png", "format": "png", ... }

gangsheet_artwork(
  items=[
    { "image_url": "https://example.com/a.png", "width_in": 8, "qty": 3 },
    { "image_url": "https://example.com/b.png", "width_in": 4, "qty": 2, "bg_method": "key" },
  ],
  gap_in=0.25, sheet_width_in=21,
)
# each logo prepped, then placed at its width × qty on one 21" sheet (height grows to fit)
# -> { "file_url": "https://…/gangsheet.png", "logos": 2, "total_pieces": 5, "credits_charged": 2, ... }
```

## HTTP API — prep

```bash
curl -X POST https://worker.majordtf.com/v1/prep \
  -H "Authorization: Bearer mdtf_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "image_url": "https://example.com/logo.png",
    "print_width_in": 11,
    "garment_colors": ["#1a1a1a"],
    "thickness_protection": false
  }'
```

### Request fields

| Field | Type | Notes |
|---|---|---|
| `image_url` | string | Public source URL (PNG/JPG/WEBP). Either this or `image_data`. |
| `image_data` | string | Base64 data URL, instead of `image_url`. |
| `print_width_in` | number | **Required.** Printed width in inches. (MCP arg: `width_in`.) |
| `print_height_in` | number | Default 99. Use with `fit:"height"` to size by height. (MCP arg: `height_in`.) |
| `fit` | string | `width` (default) \| `height` \| `contain`. |
| `bg_method` | string | `auto` (default) \| `key` (flat-bg logo) \| `smart` \| `ai` (photo) \| `keep_alpha`. The web Auto/Logo/Smart/Photo art types. |
| `garment_colors` | string[] | One hex colour = the shirt this prints on, keyed out. Omit → auto bg-removal, keep every colour. (MCP arg: `garment_color`, single hex.) |
| `halftone_off` | boolean | Default **false** — bare `/v1/prep` with a `garment_colors` value auto-halftones (blends into the shirt). The web app, MCP `prep_artwork`, and CLI default to a clean **knockout** (`halftone_off:true`) and opt into the blend via `halftone` / `--halftone`. |
| `thickness_protection` | boolean | 0.5 pt garment-colour stroke so thin lines keep their white underbase. Needs exactly one garment colour. |
| `dpi` | number | Output DPI, default 300. |

### Response

```json
{
  "status": "ok",
  "file_url": "https://…supabase.co/…/clean.png?token=…",
  "width": 3300, "height": 4436,
  "print_in": { "w": 11.0, "h": 14.79 },
  "method": "ai", "quality": "clean",
  "halftoned": false, "thin_lines": false, "thin_pct": 0.0,
  "credits_charged": 1, "balance": 41
}
```

`file_url` is clean, un-watermarked, and signed for **24 h** — download/store it promptly.
`thin_lines: true` flags sub-1 pt strokes that adhere poorly on DTF — re-run with
`thickness_protection` (and a `garment_color`).

### Errors

| Code | Meaning |
|---|---|
| 401 | Missing/invalid API key |
| 402 | Out of credits — top up at /account |
| 400 | Missing image or print size |
| 500 | Prep failed (bad/unreadable image) |

Same error codes apply to `/v1/vectorize`, `/v1/recolor`, and `/v1/gangsheet`.

## HTTP API — vectorize (raster → SVG)

```bash
curl -X POST https://worker.majordtf.com/v1/vectorize \
  -H "Authorization: Bearer mdtf_live_..." \
  -H "Content-Type: application/json" \
  -d '{ "image_url": "https://example.com/logo.png", "clean_first": true, "n_colors": 24 }'
# -> { "status": "ok", "file_url": "https://…/trace.svg?token=…", "engine": "vtracer", "bytes": 48213, "credits_charged": 1, "balance": 40 }
```

| Field | Type | Notes |
|---|---|---|
| `image_url` / `image_data` | string | Source raster (URL or base64). |
| `clean_first` | boolean | Default false (MCP/UI send true). Remove the background first so the trace is on transparent art. |
| `bg_method` | string | Used with `clean_first`: `auto` (default) \| `key` \| `smart` \| `ai` \| `keep_alpha`. |
| `n_colors` | number | Default 24. **>16 keeps ALL colours faithfully** (correct for multi-colour logos). ≤16 reduces to exactly N flat colours — simple/poster art only; a low value mutes a colourful logo. |
| `filter_speckle` | number | Default 8. Drop traced blobs < N px (Image-Trace Noise). |
| `corner_threshold` | number | Default 60. Higher = smoother corners. |
| `mode` | string | `spline` (curves, default) \| `polygon` \| `pixel`. |
| `path_precision` | number | Default 8. Path coordinate decimals. |
| `print_width_in` | number | Default 11. Only used with `clean_first` (scale before tracing). |

## HTTP API — recolor (swap / tint)

```bash
curl -X POST https://worker.majordtf.com/v1/recolor \
  -H "Authorization: Bearer mdtf_live_..." \
  -H "Content-Type: application/json" \
  -d '{ "image_url": "https://example.com/logo.png",
        "swaps": [{ "from": "#ffffff", "to": "#000000", "tolerance": 0.16 }] }'
# -> { "status": "ok", "file_url": "https://…/recolor.png?token=…", "ext": "png", "bytes": 603627, "credits_charged": 1, "balance": 39 }
```

| Field | Type | Notes |
|---|---|---|
| `image_url` / `image_data` | string | Source PNG **or** SVG (URL or base64). SVG just rewrites fills — lossless. |
| `swaps` | array | `[{ from, to, tolerance? }]` hex swaps. `tolerance` 0–1 colour distance (default 0.14). Matched against the original first, so a simultaneous white↔black flip is safe. **Provide `swaps` or `tint`.** |
| `tint` | string | Repaint every opaque pixel to this hex (keeps the alpha shape) — for single-colour designs. |

`swap` is targeted (only the matched colour changes — keeps the rest of a multi-colour logo);
`tint` repaints everything. The MCP tool takes one `swap_from`/`swap_to` or a `tint`; use the
HTTP `swaps` array for multiple swaps at once.

## HTTP API — gang sheet (many logos → one 21" sheet)

```bash
curl -X POST https://worker.majordtf.com/v1/gangsheet \
  -H "Authorization: Bearer mdtf_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      { "image_url": "https://example.com/a.png", "width_in": 8, "qty": 3 },
      { "image_url": "https://example.com/b.png", "width_in": 4, "qty": 2, "bg_method": "key" }
    ],
    "gap_in": 0.25, "sheet_width_in": 21
  }'
# -> { "status": "ok", "file_url": "https://…/gangsheet.png?token=…",
#      "logos": 2, "total_pieces": 5, "credits_charged": 2, "balance": 38, ... }
```

| Field | Type | Notes |
|---|---|---|
| `items` | array | **Required.** One entry per logo (see item fields below). |
| `gap_in` | number | Spacing between logos. Default 0.25". |
| `sheet_width_in` | number | Sheet width. Default 21". |
| `dpi` | number | Default 300. |

**Item fields:** `image_url` (required), `width_in` (print width, default 11), `qty` (copies,
default 1), `bg_method` (default `auto`), `garment_color` (hex to knock out, optional),
`halftone` (bool, default false). Each logo is prepped, then placed at its width × its qty,
row-packed left→right, sheet height grows to fit. **Charged 1 credit per distinct logo**
(quantities are free); nothing is charged if the sheet can't be built.

## Notes

- Billed **on success only**, 1 credit per call (the gang sheet: 1 per logo, quantities free).
- No `garment_color` = transparent cutout keeping all colours (the common case). Pass one only
  when the design presses onto that single shirt colour and should fade into it.
- Practical crisp-print ceiling ≈ 14" @ 300 DPI (upscale input ≤ 4 MP / 4096 px); larger prints
  at ~200 DPI. Source upload ≤ 50 MB.
