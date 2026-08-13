# ServiceNow Docs in Claude as a skill

[![License: CC0-1.0](https://img.shields.io/badge/License-CC0_1.0-lightgrey.svg)](LICENSE)
[![ServiceNow docs.md](https://img.shields.io/badge/ServiceNow-docs.md-62D84E.svg?logo=servicenow&logoColor=white)](https://github.com/ServiceNow/ServiceNowDocs)

Ask Claude a ServiceNow question, get an answer from ServiceNow's official documentation —
fetched live, cited, and never from training-data recall.

The plugin bundles the **Fetch** MCP server plus a skill that reads the `llms.txt` index in
[`ServiceNow/ServiceNowDocs`](https://github.com/ServiceNow/ServiceNowDocs), pulls the one or
two topic files your question needs, and links them. Nothing to clone or keep in sync.

## Prerequisite: `uv`

Both clients need [`uv`](https://docs.astral.sh/uv/) — the Fetch server launches via `uvx`. It
brings its own Python.

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
```
```powershell
# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Reopen your terminal so `uvx` lands on `PATH`, then check `uvx --version`.

## Claude Desktop

**1. Install the skill.** Customize → **Skills** → **+** next to *Personal plugins* → **+
Create plugin** → **Add marketplace** → paste `bikemeardsley/claude-servicenow-docs` → **Sync**
→ **Install**.

**2. Add the Fetch server to Desktop's config.** Desktop lists the bundled server under the
plugin's **Connectors** panel but can't start it from there, so it needs adding once:

1. **Fully quit Claude Desktop** — ⌘Q on macOS, or quit from the system tray on Windows. It
   rewrites its config on exit.
2. Open **Settings → Developer → Edit Config** to open `claude_desktop_config.json`.
3. Merge the `fetch` entry into `mcpServers`, keeping any servers already there:

   ```json
   {
     "mcpServers": {
       "fetch": {
         "command": "uvx",
         "args": ["--with", "mcp<2", "mcp-server-fetch"]
       }
     }
   }
   ```

4. Save and relaunch.

Keep the `mcp<2` pin exactly as written — `mcp-server-fetch` crashes on startup under MCP
Python SDK 2.0.

## Claude Code

```bash
/plugin marketplace add bikemeardsley/claude-servicenow-docs
/plugin install servicenow-docs@servicenow-docs
```

Restart Claude Code, then run `/mcp` — `fetch` should be listed and connected. No config
editing needed; the bundled server starts on its own. The first launch is slower while `uvx`
caches it.

## How it works

| | |
|---|---|
| Index | `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/main/llms.txt` |
| Raw file base | `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/main/` |

Claude reads the index, picks the 1–3 entries matching your question, fetches those topic
files, answers from them, and links the source. Paths always come from the index — the docs
get reorganized, so guessing from memory doesn't work.

Retrieval is online and file-at-a-time, so it needs internet access and reads the `main`
branch live.

## Pairs with GlideGrail.md

[**GlideGrail.md**](https://github.com/bikemeardsley/GlideGrail.md) is the judgment layer — how
you want ServiceNow code written. This is the reference layer — what the platform actually
does. Separate installs, good together:

```bash
/plugin marketplace add bikemeardsley/GlideGrail.md
/plugin install glidegrail@glidegrail
```

## License

[CC0 1.0 Universal](LICENSE) — public domain. Use it however you like, no attribution needed.

The documentation this plugin retrieves is published by ServiceNow under its own terms; see
[`ServiceNow/ServiceNowDocs`](https://github.com/ServiceNow/ServiceNowDocs). Not affiliated
with or endorsed by ServiceNow.
