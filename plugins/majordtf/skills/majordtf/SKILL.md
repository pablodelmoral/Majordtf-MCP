---
name: majordtf
description: Prep, vectorize, or recolor artwork for DTF/transfer printing via majordtf.com — the prep_artwork / vectorize_artwork / recolor_artwork MCP tools or the POST /v1/prep, /v1/vectorize, /v1/recolor HTTP APIs. Use when someone wants a logo/design cleaned, background removed, upscaled, halftoned to a garment colour, traced to a vector SVG, or recolored (e.g. white↔black).
---

# MajorDTF — artwork prep, vectorize & recolor

Three operations on a raw image URL, each returning a clean, signed file URL. Each call costs
**1 credit** (billed on success only):

- **Prep** → print-ready 300 DPI transparent DTF **PNG** (bg removed, upscaled, edges hardened, sized).
- **Vectorize** → clean vector **SVG** (deterministic Image-Trace replica, no AI redraw).
- **Recolor** → swap colours (e.g. white→black) or tint the whole design. Works on a PNG or SVG.

Two ways to call each — same engine, same params:

- **MCP tools** `prep_artwork` / `vectorize_artwork` / `recolor_artwork` — for agents.
- **HTTP** `POST https://worker.majordtf.com/v1/{prep,vectorize,recolor}` — for scripts/curl.
  Prep also has a CLI wrapper: `pip install majordtf`.

## Auth

Mint a key at <https://www.majordtf.com/account> → API keys (shown once). Format `mdtf_live_...`.
Keep it in the client/MCP config as a header — **never paste it into chat**.

## MCP

Install in Claude Code (one line):

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

Call it:

```
prep_artwork(
  image_url="https://example.com/logo.png",
  width_in=11,                 # printed width in inches; height follows the artwork
  garment_color="#1a1a1a",     # optional — see below
  thickness_protection=false,  # optional — see below
)
# auth is the Authorization header, NOT a tool arg
# -> { "file_url": "https://…/clean.png", "width": 3300, "height": 4436, ... }
```

If a client can't set headers, pass the key as the `api_key` argument instead.

Vectorize and recolor are sibling tools, same auth:

```
vectorize_artwork(
  image_url="https://example.com/logo.png",
  n_colors=24,         # >16 keeps ALL colours (default); <=16 reduces to N flat colours
  clean_first=true,    # remove the background before tracing
)
# -> { "file_url": "https://…/trace.svg", "format": "svg", ... }

recolor_artwork(
  image_url="https://example.com/logo.png",
  swap_from="#ffffff", swap_to="#000000",   # white -> black (only matched colour changes)
  # ...or tint="#000000" to repaint every opaque pixel one colour
)
# -> { "file_url": "https://…/recolor.png", "format": "png", ... }
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
| `print_height_in` | number | Default 99. Use with `fit:"height"` to size by height instead. |
| `fit` | string | `width` (default) \| `height` \| `contain`. |
| `garment_colors` | string[] | One hex colour → keyed out + design halftoned to blend into that shirt colour. Omit/none → auto bg-removal, keep every colour. (MCP arg: `garment_color`, single hex.) |
| `thickness_protection` | boolean | Adds a 0.5 pt garment-colour stroke so thin lines keep their white underbase. Needs exactly one garment colour. |
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

Same error codes apply to `/v1/vectorize` and `/v1/recolor`.

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

## Notes

- Billed **on success only**, 1 credit per call (prep, vectorize, or recolor).
- No `garment_color` = transparent cutout keeping all colours (the common case). Pass one only
  when the design will press onto that single shirt colour and should fade into it.
- Practical crisp-print ceiling ≈ 14" @ 300 DPI (upscale input ≤ 4 MP / 4096 px); larger prints
  at ~200 DPI. Source upload ≤ 50 MB.
