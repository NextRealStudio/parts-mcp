# Next Everything: electronic parts MCP server

Your AI assistant checks your BOM and orders the parts.

Next Everything is a hosted MCP server for people who design electronics with an AI assistant. Connect it to Claude Code, Claude Desktop, Cursor, Codex, Windsurf or VS Code with one snippet, and your assistant can source the parts for your board.

Give it a BOM exported from KiCad, Altium, EasyEDA or a spreadsheet and the number of boards. It matches every line to an orderable part, prices the build with price breaks and minimum order quantities applied, and flags what would stop you from ordering: out of stock, not enough stock, parts being discontinued, uncertain matches and JLCPCB extended-part fees. Problem lines come with replacements checked spec by spec, in the same package so the PCB does not change.

Part data comes from the JLCPCB/LCSC catalog (630,000+ in-stock parts, refreshed daily) and live DigiKey and Mouser lookups. Every build gets a project page with the current BOM, prices and each step the assistant took.

When the BOM is right, the assistant prepares an order. Nothing is charged by that step: the team re-checks stock and price and emails you one payment link for the whole build, which you approve yourself. Delivery to the US, Europe and the UK.

- Website: https://nexteverything.io/
- Setup docs and tool reference: https://nexteverything.io/docs/
- Guides: https://nexteverything.io/guides/
- MCP endpoint: `https://mcp.nexteverything.io/mcp` (Streamable HTTP)
- Official MCP Registry: `io.nexteverything/parts`

This repository holds documentation, client configs and examples. The server itself is hosted; there is nothing to install or run locally.

## Features

- Check a whole BOM in one call: matches, prices for your board count, problems flagged
- 630,000+ in-stock JLCPCB/LCSC parts, refreshed daily, plus live DigiKey and Mouser prices
- Search by part number or by specs (e.g. `rds_on <= 10mΩ`, package, manufacturer)
- Replacements checked spec by spec, same package by default, with a confidence level
- JLCPCB basic vs extended parts and the $3 extended-part fee counted in the price
- Price breaks and MOQ handled; buying up to the next break when that is cheaper overall
- Live project page per build: BOM, prices, stock, every assistant step
- Orders need your approval: one emailed payment link per build

## Tools

| Tool | What it does |
|---|---|
| `create_project` | Starts a project for one board. Returns a `project_id` and a `dashboard_url` with a live view of the BOM, the assistant's steps and the orders. |
| `check_bom` | Takes a BOM (CSV from KiCad, Altium, EasyEDA or a spreadsheet) and a board count. Returns the match for every line, prices with price breaks, MOQ and pack sizes, problems (out of stock, not enough stock, uncertain match, MPN and DigiKey/Mouser number pointing at different parts, missing footprint, JLCPCB extended-part fees), alternatives, similar parts for unmatched lines and shipments per distributor. Returns a `quote_id`. |
| `update_bom_line` | Changes one line (part, quantity, pack, remove, add) or the board count. Returns the totals, the problems and only the changed line. |
| `find_parts` | Searches by part number (typos too; a number missing from the catalog is looked up live at Mouser and DigiKey), words with values (`resistor 33 ohm 1/4W axial`), category, package, manufacturer and specs (`vds_max >= 30V`, `rds_on <= 10mΩ`). |
| `get_part` | Everything about one part: specs, offers with their price ladder, datasheet, pinout, footprint and stock trend. |
| `find_alternatives` | Replacements checked by rules (polarity, voltage, current, tolerance, dielectric and more), with a spec comparison and a confidence level. Works for parts only Mouser or DigiKey sell, and shows the replacement the distributor suggests. |
| `place_order` | Turns a quote into an order. Nothing is charged by this call: the team re-checks live stock and price and emails you one payment link. |
| `order_status` | Status, history, payment link, shipment tracking and `waiting_on` (who the order is waiting for). |

Every parameter is in the [tool reference](https://nexteverything.io/docs/#tools).

## Setup

1. Create an account at https://nexteverything.io/signup/ and copy your API key from the dashboard.
2. Add the server to your client with one of the snippets below. Replace `nx_live_your_key` with your key.

Treat the key like a password. Do not commit a config file that contains your real key.

### Claude Code

Run in your terminal:

```bash
claude mcp add --transport http nexteverything https://mcp.nexteverything.io/mcp --header "Authorization: Bearer nx_live_your_key"
```

Keep the command on one line: a trailing `\` line break in PowerShell or cmd silently drops the `--header` part and every call fails with 401. Add `--scope user` to use it in every project.

Or put this into `.mcp.json` in your project folder (or `"mcpServers"` in `~/.claude.json`):

```json
{
  "mcpServers": {
    "nexteverything": {
      "type": "http",
      "url": "https://mcp.nexteverything.io/mcp",
      "headers": {
        "Authorization": "Bearer nx_live_your_key"
      }
    }
  }
}
```

Check with `claude mcp list` or `/mcp` inside Claude Code. [Full guide](https://nexteverything.io/docs/claude-code/)

### Claude Desktop

Settings → Connectors → Add custom connector → name it Next Everything and paste this URL:

```
https://mcp.nexteverything.io/mcp?key=nx_live_your_key
```

Or edit `claude_desktop_config.json` (Settings → Developer → Edit Config), then restart Claude. This route runs `mcp-remote` through `npx`, so it needs Node.js:

```json
{
  "mcpServers": {
    "nexteverything": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://mcp.nexteverything.io/mcp",
        "--header",
        "Authorization:${NX_AUTH}"
      ],
      "env": {
        "NX_AUTH": "Bearer nx_live_your_key"
      }
    }
  }
}
```

[Full guide](https://nexteverything.io/docs/claude-desktop/)

### Cursor

`~/.cursor/mcp.json` (every project) or `.cursor/mcp.json` (one project):

```json
{
  "mcpServers": {
    "nexteverything": {
      "url": "https://mcp.nexteverything.io/mcp",
      "headers": {
        "Authorization": "Bearer nx_live_your_key"
      }
    }
  }
}
```

Use the agent in Agent mode: MCP tools are not called from plain chat completions. [Full guide](https://nexteverything.io/docs/cursor/)

### Codex CLI

`~/.codex/config.toml`:

```toml
[mcp_servers.nexteverything]
url = "https://mcp.nexteverything.io/mcp"
http_headers = { "Authorization" = "Bearer nx_live_your_key" }
```

Restart running Codex sessions after editing the file. [Full guide](https://nexteverything.io/docs/codex/)

### Windsurf

`~/.codeium/windsurf/mcp_config.json` (Windsurf uses `serverUrl` instead of `url`):

```json
{
  "mcpServers": {
    "nexteverything": {
      "serverUrl": "https://mcp.nexteverything.io/mcp",
      "headers": {
        "Authorization": "Bearer nx_live_your_key"
      }
    }
  }
}
```

[Full guide](https://nexteverything.io/docs/windsurf/)

### VS Code (GitHub Copilot agent mode)

`.vscode/mcp.json` (VS Code uses `servers` instead of `mcpServers`):

```json
{
  "servers": {
    "nexteverything": {
      "type": "http",
      "url": "https://mcp.nexteverything.io/mcp",
      "headers": {
        "Authorization": "Bearer nx_live_your_key"
      }
    }
  }
}
```

Click Start above the server, then open Copilot Chat in Agent mode. Add `.vscode/mcp.json` to `.gitignore` if it holds your real key. [Full guide](https://nexteverything.io/docs/vscode/)

### Other clients

Any MCP client that supports Streamable HTTP works: point it at `https://mcp.nexteverything.io/mcp` and send the header `Authorization: Bearer nx_live_your_key`.

## Example prompts

Talk to it the way you would to a colleague who does your purchasing:

- "Use Next Everything to check my BOM for 50 boards and fix anything that is out of stock."
- "Find an in-stock 3.3 V LDO in SOT-23 that can supply 500 mA, cheapest first."
- "This board uses an SPH0645 microphone. Is it still in production, and what can replace it?"
- "Compare JLCPCB, DigiKey and Mouser prices for every part in bom.csv at 100 boards."
- "Check this KiCad BOM for JLCPCB assembly of 20 boards and replace extended parts with basic ones where possible."
- "Which parts in this BOM are end of life or not recommended for new designs?"
- "Order the parts for this project and send me the payment link."

Try them on [examples/sample-bom.csv](examples/sample-bom.csv).

## BOM files

CSV, TSV or semicolon-separated exports from KiCad, Altium, EasyEDA/JLCPCB and hand-made spreadsheets. Columns such as Designator, Quantity, MPN, Manufacturer, Value, Footprint, LCSC Part, DigiKey and Mouser part numbers and DNP are recognised. A line needs at least a part number or a value. Excel files and schematics are not read yet: export the BOM as CSV first.

## Guides

- [Mouser vs DigiKey: price comparison on a real 48-line BOM](https://nexteverything.io/guides/mouser-vs-digikey-bom-price-comparison/)
- [How much JLCPCB extended part fees really cost](https://nexteverything.io/guides/jlcpcb-extended-part-fees/)
- [AI BOM checker](https://nexteverything.io/tools/bom-checker/)
- [Find alternative parts with AI](https://nexteverything.io/tools/find-alternatives/)
- [JLCPCB / LCSC](https://nexteverything.io/mcp/jlcpcb/), [DigiKey](https://nexteverything.io/mcp/digikey/) and [Mouser](https://nexteverything.io/mcp/mouser/) in your AI assistant

## Pricing

No subscription and no per-call fee. You pay only for the parts in an order you approve through the emailed payment link.

## Support

hello@nexteverything.io

Next Everything is an independent service and is not affiliated with or endorsed by JLCPCB, LCSC, DigiKey or Mouser Electronics. Prices and stock come from their public data at the time of the check and are re-checked before you pay.

## License

The documentation and examples in this repository are under the [MIT License](LICENSE). The hosted service is operated by NextEverything Inc.; its source code is not part of this repository.
