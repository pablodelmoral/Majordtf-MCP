# MajorDTF — Claude Code plugin

Prep, vectorize, recolor, and gang-sheet artwork for DTF/transfer printing, right inside Claude
Code. Bundles the MajorDTF skill **and** the hosted MCP tools (`prep_artwork`,
`vectorize_artwork`, `recolor_artwork`, `gangsheet_artwork`).

## Install

```
/plugin marketplace add pablodelmoral/Majordtf-MCP
/plugin install majordtf
```

## Set your API key

The MCP server authenticates with your MajorDTF key. Mint one at
<https://www.majordtf.com/account> → API keys, then expose it as an environment variable
before launching Claude Code:

```bash
export MAJORDTF_API_KEY="mdtf_live_..."   # macOS / Linux
setx MAJORDTF_API_KEY "mdtf_live_..."     # Windows (new terminals)
```

The key is read from the environment and sent as an `Authorization` header — it never enters
the chat. Each tool call costs 1 credit (billed on success only).

## What you get

- **prep_artwork** → print-ready 300 DPI transparent DTF PNG
- **vectorize_artwork** → clean vector SVG (deterministic Image-Trace)
- **recolor_artwork** → swap colours (e.g. white→black) or tint the whole design
- **gangsheet_artwork** → prep several logos and lay them onto one 21" sheet (1 credit per logo)
- the **majordtf** skill, so the agent knows when and how to use them

Docs: <https://www.majordtf.com/docs/mcp>
