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

**1. Install the skill:**

1. Go to Customize → Plugins → Add → Add marketplace
2. Paste `bikemeardsley/claude-servicenow-docs`
3. Set Sync automatically = true
4. Click Sync
5. Click the + to install the ServiceNow docs plugin

**2. Add the Fetch mcp server to Claude Desktop's config json.** Desktop lists the bundled server under the
plugin's **Connectors** panel but can't start it from there, so it needs adding once:

1. Open **Customize → Developer → Edit Config** to open `claude_desktop_config.json`.
2. Merge the `fetch` entry into `mcpServers`, keeping any servers already there:

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

3. Save the file.
4. **Fully quit Claude Desktop** — ⌘Q on macOS, or quit from the system tray on Windows. Relaunch Claude so it grabs the updated config file.

## Claude Code

```bash
/plugin marketplace add bikemeardsley/claude-servicenow-docs
/plugin install servicenow-docs@servicenow-docs
```

Restart Claude Code, then run `/mcp` — `fetch` should be listed and connected. No config
editing needed; the bundled server starts on its own. The first launch is slower while `uvx`
caches it.

## How it works

ServiceNow ships each release family as its own branch (`australia`, `zurich`, `yokohama`,
`xanadu`) — `main` holds no documentation at all. The skill uses `HEAD`, which always points
at the current family, so it keeps working across release cutovers with no update:

```
raw.githubusercontent.com/ServiceNow/ServiceNowDocs/HEAD/markdown/{publication}/{file}.md
```

Claude finds the topic file cheapest-first — deriving the path from a search result, probing
for a curated landing page, and only listing the publication directory as a last resort — then
fetches it and links the source. A derived path is only a candidate: if the fetch doesn't
return content, it isn't cited. Mention you're on an older release and it substitutes that
family for `HEAD`.

Retrieval is online and file-at-a-time, so it needs internet access.

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
