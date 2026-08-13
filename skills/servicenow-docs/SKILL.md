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

## Base locations

| Purpose | URL |
|---|---|
| **Index** - always fetch first | `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/main/llms.txt` |
| **Raw file base** - join with any path from the index | `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/main/` |
| **Human-browsable repo** - for citing back to the user | `https://github.com/ServiceNow/ServiceNowDocs` |

## Retrieval workflow

1. **`fetch` the index.** Retrieve `llms.txt` and scan it for topics matching the question.
   The index lists topics with short descriptions and their file paths.
2. **Resolve to specific file(s).** Pick the 1-3 most relevant entries and build each raw URL
   by joining the raw file base with the path from the index.
3. **`fetch` the file(s).** Retrieve the actual Markdown topic file(s).
4. **Answer from the fetched content**, and cite each topic with its GitHub URL so the user
   can open the source.

For multi-part questions, resolve each sub-topic against the index separately, then fetch each
relevant file.

## Handling long topic files

The `fetch` tool truncates by default (roughly 5,000 characters) and reports when content was
cut short. ServiceNow topic files are often longer.

- Raise `max_length` when you need more of a document in one call.
- Use `start_index` to continue from where a truncated response stopped, and keep paginating
  until you have the section you need.
- **Never answer from a half-read document as though you read all of it.** If you stopped
  early, either paginate or tell the user which part you read.

## Failure handling

- **A path 404s:** re-fetch `llms.txt` - the topic likely moved - and retry with the updated
  path before giving up.
- **No `fetch` tool available to you:** the connector is not running. Say so instead of
  answering from memory, and point the user at the fix for their client:
  - **Claude Code:** the server launches via `uvx`, so `uv` must be installed -
    `curl -LsSf https://astral.sh/uv/install.sh | sh` (macOS/Linux) or
    `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`
    (Windows). Reopen the terminal, restart Claude, then check `/mcp` for a connected `fetch`
    server.
  - **Claude desktop app:** open the plugin's **Connectors** tab and connect `fetch`, then
    restart the app.
- If you answer anyway despite having no `fetch` tool, **state up front** that the live
  documentation was unavailable and the answer comes from prior knowledge, which may be
  outdated or wrong on version-specific details.

## Notes

- Retrieval is online and file-at-a-time against the `main` branch, so it reflects the latest
  published docs. There is no local copy to keep in sync.
- Fetch narrowly - usually 1-3 files per question, not large swaths of the corpus - to keep
  context focused.
- Offline access or whole-corpus search would require a local clone with a filesystem/git MCP
  server instead; that is out of scope for this skill.
