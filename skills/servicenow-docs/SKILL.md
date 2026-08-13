---
name: servicenow-docs
description: Search ServiceNow's official product documentation live, using the bundled Fetch connector. Use this skill whenever you need authoritative ServiceNow documentation - platform capabilities, APIs, application development, security/ACLs, ITSM/ITOM/HRSD, Flow Designer, integrations, Service Portal, scripting, and more - and always when the user explicitly asks to "search the ServiceNow docs", "check the docs", "look this up in the ServiceNow documentation", or cites a doc topic. It reads ServiceNow's official documentation (published as Markdown for LLMs) live from GitHub, so answers reflect the current published docs. Prefer this skill over training-data recall for anything ServiceNow-specific, version-sensitive, or where an authoritative citation is helpful.
---

# ServiceNow Docs (via the Fetch connector)

This skill answers ServiceNow questions from the official
`ServiceNow/ServiceNowDocs` repository - the docs ServiceNow publishes as one Markdown file
per topic, with a machine-readable `llms.txt` index designed for "read the index, then fetch
the file you need" retrieval.

## Use the Fetch connector for every retrieval

This plugin bundles the **Fetch** MCP server, exposed as the **`fetch` connector**. It
provides a `fetch` tool that takes a `url` and returns the page as Markdown.

**Every piece of content you cite in an answer must come from a `fetch` call made during this
conversation.** When the user invokes this skill, they are explicitly asking you to consult
the live documentation - so:

- **Always call `fetch` first.** Do not answer a ServiceNow question from memory and then
  offer to look it up. Look it up, then answer.
- **Never reconstruct doc content from recall** and present it as documentation, even when
  you are confident. Confidence is not a citation.
- **Never guess or fabricate a URL or file path.** Every path comes from the index (below).
- If the docs contradict your prior knowledge, **the fetched docs win** - say so plainly if
  the difference matters to the user.

## Resolve the branch first - always

**Documentation content does not live on `main`.** That branch carries only repo-level files
(`README.md`, `LICENSE`, `llms.txt`) and a `markdown/` directory holding nothing but a
placeholder. The actual docs live on release-family branches - `australia`, `zurich`,
`yokohama`, `xanadu` - and which one is current rotates as ServiceNow ships new families.

**Before the first content fetch in a conversation, resolve `{branch}`:**

1. `fetch` `https://api.github.com/repos/ServiceNow/ServiceNowDocs` and read the
   `default_branch` field. That value is `{branch}` for the rest of the conversation.
2. If the user names a release explicitly ("on Xanadu", "we're still on Yokohama"), use that
   family name as `{branch}` for that request instead.
3. **Never hardcode a family name**, and never fall back to `main` - it has no content.

Resolve once per conversation and reuse it; don't re-resolve before every fetch.

## Base locations

| Purpose | URL |
|---|---|
| **Branch index** - fetch right after resolving `{branch}` | `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/{branch}/llms.txt` |
| **Raw file base** - join with any path | `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/{branch}/` |
| **Directory listing** - files within one publication | `https://api.github.com/repos/ServiceNow/ServiceNowDocs/contents/markdown/{publication}?ref={branch}` |
| **Human-browsable repo** - for citing back to the user | `https://github.com/ServiceNow/ServiceNowDocs` |

Prefer `raw.githubusercontent.com` over `api.github.com` wherever both would work: the GitHub
API allows only **60 unauthenticated requests per hour per IP**, and spending that budget on
listings will stall retrieval mid-conversation.

## Retrieval workflow

1. **`fetch` the branch index** (`{branch}/llms.txt`, about 9 KB). It lists all ~56
   publications with human-readable names and a direct URL each - "Building applications" is
   `markdown/application-development/`, "App development and low-code" is
   `markdown/hyperautomation-low-code/`. Pick the 1-2 publications matching the question.
2. **Locate the topic file inside that publication - but do not fetch the publication's
   `index.md` to do it.** Those are complete tables of contents and some are enormous:
   `application-development/index.md` is roughly 624 KB, far more than the topic file you
   actually want. Fetch the directory listing instead, which returns descriptive file names
   (`ACL-access-checks.md`, `activate-RCA.md`, ...). Large publications run past 300 files, so
   the listing may truncate - raise `max_length` and page with `start_index`; entries are
   alphabetical, which makes paging predictable. If you have a web search tool, searching the
   topic plus "ServiceNowDocs github" is often the fastest way to land on the exact file.
3. **`fetch` the resolved file(s)** as `{raw file base}markdown/{publication}/{file}.md`.
4. **Answer from the fetched content**, and cite each topic with its GitHub URL so the user
   can open the source.

For multi-part questions, resolve each sub-topic separately, then fetch each relevant file.

## Handling long topic files

The `fetch` tool truncates by default (roughly 5,000 characters) and reports when content was
cut short. ServiceNow topic files are often longer.

- Raise `max_length` when you need more of a document in one call.
- Use `start_index` to continue from where a truncated response stopped, and keep paginating
  until you have the section you need.
- **Never answer from a half-read document as though you read all of it.** If you stopped
  early, either paginate or tell the user which part you read.

## Failure handling

- **A path 404s:** re-fetch the branch index (`{branch}/llms.txt`) - the topic likely moved -
  and retry with the updated path before giving up.
- **Everything 404s, or a listing returns only a `README.md`:** you are almost certainly still
  pointed at `main`. Re-resolve `{branch}` via `default_branch` and retry.
- **A GitHub API call returns 403 with a rate-limit message:** the 60/hour unauthenticated
  budget is spent. Switch to `raw.githubusercontent.com` - the branch index and the topic files
  themselves need no API call - and tell the user directory listings are unavailable until the
  hour rolls over.
- **No `fetch` tool available to you:** the connector is not running. Say so instead of
  answering from memory, and point the user at the fix for their client:
  - **Claude Code:** the server launches via `uvx`, so `uv` must be installed -
    `curl -LsSf https://astral.sh/uv/install.sh | sh` (macOS/Linux) or
    `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`
    (Windows). Reopen the terminal, restart Claude, then check `/mcp` for a connected `fetch`
    server.
  - **Claude desktop app:** the bundled connector cannot be activated from the plugin's
    Connectors panel, so the Fetch server has to be added to Claude Desktop's own config
    once. Tell the user to fully quit Claude Desktop, open **Settings → Developer → Edit
    Config**, merge this into the `mcpServers` object in `claude_desktop_config.json`
    (keeping any servers already there), then relaunch:

    ```json
    "fetch": { "command": "uvx", "args": ["--with", "mcp<2", "mcp-server-fetch"] }
    ```

    This also requires `uv` — same install commands as above.
- If you answer anyway despite having no `fetch` tool, **state up front** that the live
  documentation was unavailable and the answer comes from prior knowledge, which may be
  outdated or wrong on version-specific details.

## Notes

- Retrieval is online and file-at-a-time against the resolved release branch, so it reflects
  the currently published docs. There is no local copy to keep in sync.
- ServiceNow keeps only the three most recent families (four during an early-availability
  window) and deletes the oldest branch at GA, so a family the user names may no longer exist.
- Do not try to read `servicenow.com/docs` directly - it is a JavaScript app that returns no
  readable content. The repo is the only machine-readable source.
- Fetch narrowly - usually 1-3 files per question, not large swaths of the corpus - to keep
  context focused.
- Offline access or whole-corpus search would require a local clone with a filesystem/git MCP
  server instead; that is out of scope for this skill.
