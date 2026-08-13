# claude-servicenow-docs - live ServiceNow documentation in Claude as a skill

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-blue.svg)](LICENSE)
[![ServiceNow docs.md](https://img.shields.io/badge/ServiceNow-docs.md-62D84E.svg?logo=servicenow&logoColor=white)](https://github.com/ServiceNow/ServiceNowDocs)

A Claude plugin that gives your assistant authoritative ServiceNow product documentation —
with nothing to clone, download, or keep in sync.

ServiceNow publishes its AI Platform docs as [`ServiceNow/ServiceNowDocs`](https://github.com/ServiceNow/ServiceNowDocs):
one Markdown file per topic, plus a machine-readable `llms.txt` index. This plugin bundles the
**Fetch** MCP server and a skill that reads that index first, pulls only the one or two topics
it actually needs, and cites them back to you. Answers reflect what ServiceNow publishes
today rather than training-data recall.

## Install

**1. Install [`uv`](https://docs.astral.sh/uv/)** — required, since the Fetch server launches
via `uvx`. It provisions its own Python, so Python is not a separate prerequisite.

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
```
```powershell
# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Reopen your terminal so `uvx` lands on `PATH`, then confirm with `uvx --version`.

**2. Add the marketplace and install the plugin** (Claude Code):

```bash
/plugin marketplace add bikemeardsley/claude-servicenow-docs
/plugin install servicenow-docs@servicenow-docs
```

**3. Restart Claude**, then run `/mcp` — a `fetch` server should be listed and connected. The
first launch is slower while `uvx` downloads and caches the server.

If `fetch` isn't there, `uv` is almost certainly missing or not yet on `PATH`. On Claude
Desktop the bundled connector needs one extra manual step — see
[Claude Desktop setup](#claude-desktop-setup) below.

## Claude Desktop setup

**Claude Code users:** nothing to do here — the bundled Fetch server auto-starts with the
plugin. Skip to [How it works](#how-it-works).

**Known Claude Desktop limitation:** a plugin's bundled local (stdio) MCP server appears in
the plugin's **Connectors** panel, but the Install button there doesn't activate it. This
affects every plugin that bundles a local server — including Anthropic's own — and isn't
specific to this one. The skill itself installs fine; the Fetch server just never starts, so
lookups fail with the server "not connected."

The one-time fix is to add the Fetch server to Claude Desktop's own config:

1. **Fully quit Claude Desktop** — ⌘Q on macOS; on Windows quit from the system tray rather
   than just closing the window. It rewrites its config on exit, so edits made while it's
   running can be overwritten.
2. Open **Settings → Developer → Edit Config**, which opens `claude_desktop_config.json`
   (`~/Library/Application Support/Claude/` on macOS, `%APPDATA%\Claude\` on Windows).
3. Add the `fetch` entry inside the existing `mcpServers` object — **merge it in, don't
   overwrite** any servers already there:

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

4. Save and relaunch Claude Desktop. The `servicenow-docs` skill now has a live Fetch server
   to call.

This is the same server definition the plugin ships in `.mcp.json`, [`mcp<2` pin](#why-mcp2-is-pinned)
included — MCP Python SDK 2.0 removed the `@server.list_tools()` API that `mcp-server-fetch`
relies on, so an unpinned launch crashes on startup.

**Verify (optional):** run `uvx --with "mcp<2" mcp-server-fetch` in a terminal. Silence is
healthy — it's an stdio server waiting for a client, so press Ctrl+C to exit. An error means
`uv` isn't installed or isn't on `PATH`.

## How it works

The plugin bundles the Fetch MCP server as the **`fetch` connector**, and the skill routes
every retrieval through it: anything Claude cites must come from a `fetch` call made during
that conversation, never from recall. Ask it to "search the ServiceNow docs for X" and it
looks X up rather than answering from memory and offering to check.

The skill points Claude at two locations and one rule: **read the index before fetching.**

| | |
|---|---|
| Index | `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/main/llms.txt` |
| Raw file base | `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/main/` |

Claude reads `llms.txt`, picks the 1–3 entries matching your question, fetches those topic
files, answers from them, and links the source. Paths are never guessed from memory — the
docs get reorganized, and the index is the only reliable map.

## Why `mcp<2` is pinned

`.mcp.json` launches the server as `uvx --with "mcp<2" mcp-server-fetch`. Under MCP Python
SDK 2.0 the Fetch server crashes on launch with:

```
AttributeError: 'Server' object has no attribute 'list_tools'
```

The pin holds the SDK on the 1.x line so the server starts cleanly. Leave it in place until
`mcp-server-fetch` ships a 2.0-compatible release.

## Tradeoffs

Retrieval is **online and file-at-a-time**:

- ✅ Always current — reflects the docs as ServiceNow publishes them, nothing to re-sync.
- ✅ Light on context — 1–3 topic files per question, not the whole corpus.
- ❌ Requires internet access.
- ❌ No whole-corpus search and no offline use. If you need either, clone the docs repo
  locally and point a filesystem or git MCP server at it instead.

## Pairs with GlideGrail.md

[**GlideGrail.md**](https://github.com/bikemeardsley/GlideGrail.md) is the judgment layer —
how you want ServiceNow code written (naming, server/client patterns, Flow Designer, ACLs,
integrations, Service Portal, ATF). This plugin is the reference layer — what the platform
actually does. They're separate installs and work well together:

```bash
/plugin marketplace add bikemeardsley/GlideGrail.md
/plugin install glidegrail@glidegrail
```

## License

Licensed under a [Creative Commons Attribution 4.0 International License (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

The ServiceNow documentation this plugin retrieves is published by ServiceNow under its own
terms — see [`ServiceNow/ServiceNowDocs`](https://github.com/ServiceNow/ServiceNowDocs). This
project is not affiliated with or endorsed by ServiceNow.
