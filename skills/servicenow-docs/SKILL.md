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
- **Never present content as read without a successful `fetch` confirming it.** Deriving a
  candidate path - from a search result, a landing-page link, or a directory listing - is fine
  and expected. Treating an unconfirmed derived path as documentation is not: if the fetch
  404s, that path told you nothing.
- If the docs contradict your prior knowledge, **the fetched docs win** - say so plainly if
  the difference matters to the user.

## Base locations

`HEAD` works as a branch alias everywhere on this repo and always resolves to the current
release family (`australia` today; ServiceNow rotates it as new families ship). **Use the
literal string `HEAD` in every URL below.** There is no branch to resolve, and no reason to
call `api.github.com/repos/ServiceNow/ServiceNowDocs` for `default_branch`.

| Purpose | URL |
|---|---|
| **Topic file** - the thing you actually want | `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/HEAD/markdown/{publication}/{file}.md` |
| **Landing page** - curated hub for a subtopic | `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/HEAD/markdown/{publication}/{topic}-landing.md` |
| **Directory listing** - last resort, see workflow | `https://api.github.com/repos/ServiceNow/ServiceNowDocs/git/trees/HEAD:markdown/{publication}` |
| **Publication list** - orientation only | `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/HEAD/llms.txt` |
| **Human-browsable repo** - for citing back to the user | `https://github.com/ServiceNow/ServiceNowDocs` |

Prefer `raw.githubusercontent.com` over `api.github.com` wherever both would work: the GitHub
API allows only **60 unauthenticated requests per hour per IP**, and spending that budget on
listings will stall retrieval mid-conversation.

## Retrieval workflow

Ordered cheapest-first. **Stop as soon as a `fetch` returns content.**

1. **Search, then derive the path.** If you have a web search tool, search the topic plus
   "ServiceNow" or "servicenow.com/docs". Results are dominated by
   `servicenow.com/docs/r/{publication}/{slug}.html` URLs. **Do not fetch that page** - it is a
   JavaScript app that returns nothing readable. Instead derive
   `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/HEAD/markdown/{publication}/{slug}.md`
   and fetch that. The slug usually maps exactly to the filename, but not always: it is a
   candidate, not a citation, until the fetch succeeds. A 404 means go to step 2 - **not** a
   second guess at the URL.
2. **Probe for a landing page.** Try
   `https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/HEAD/markdown/{publication}/{topic}-landing.md`
   with your best guess at the publication and topic slug. Many publications carry these
   curated hubs - short pages of links pointing at the right sub-pages
   (`order-management/configure-price-quote-landing.md` is a real one). One cheap fetch, and a
   hit usually names the exact file you need. Same rule: a 404 means move on, not guess again.
3. **List the publication directory - last resort.** `fetch`
   `https://api.github.com/repos/ServiceNow/ServiceNowDocs/git/trees/HEAD:markdown/{publication}`,
   pick the file from the returned paths, then fetch it via the topic-file URL pattern.
   **Omit the `recursive` parameter entirely.** GitHub checks only whether it is *present*, not
   what it is set to - `?recursive=0` and `?recursive=false` both return the full recursive
   tree, 49,498 entries instead of 56 on `markdown/`.
   Even done correctly this is expensive: large publications run past 1,000 files
   (`order-management` is 1,041, about 274,000 characters of JSON), so raise `max_length` and
   page with `start_index`, or narrow down the publication first.
4. **`llms.txt` for orientation only.** It lists all ~56 publications with human-readable
   names - "App development and low-code" is `markdown/hyperautomation-low-code/`. Use it for
   "what does the documentation cover" questions, or when you can't guess which publication
   owns a topic. It is **not** the first step for a known topic. It runs about 9,200
   characters, so pass `max_length` of at least 12,000 or the first call truncates and costs
   you a second round-trip.

Then **answer from the fetched content**, and cite each topic with its GitHub URL so the user
can open the source. For multi-part questions, resolve each sub-topic separately.

## Handling long topic files

The `fetch` tool truncates by default (roughly 5,000 characters) and reports when content was
cut short. ServiceNow topic files are often longer.

- Raise `max_length` when you need more of a document in one call.
- Use `start_index` to continue from where a truncated response stopped, and keep paginating
  until you have the section you need.
- **Never answer from a half-read document as though you read all of it.** If you stopped
  early, either paginate or tell the user which part you read.

## Failure handling

Walk the workflow in order; each failure has exactly one next move.

- **A slug-derived path 404s (step 1):** fall through to the landing-page probe (step 2). Do
  not guess a second URL from the same search result, and do not re-run the search with
  different terms.
- **The landing-page probe 404s (step 2):** escalate to the directory listing (step 3). That
  listing is authoritative for what files exist in a publication - once you have it, you are
  no longer guessing.
- **The listing has nothing relevant:** you probably picked the wrong publication. Fetch
  `llms.txt` (step 4) for the full publication list, then retry step 3 against the right one.
- **A GitHub API call returns 403 with a rate-limit message:** the 60/hour unauthenticated
  budget is spent, so step 3 is unavailable until the hour rolls over. Steps 1, 2 and 4 need no
  API call - they are all `raw.githubusercontent.com` - so keep working from those and tell the
  user directory listings are temporarily out.
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

- Retrieval is online and file-at-a-time against `HEAD`, so it reflects the currently
  published docs. There is no local copy to keep in sync.
- **If the user explicitly names an older release** ("we're still on Xanadu"), substitute that
  family name for `HEAD` in the URL patterns. ServiceNow keeps only the three most recent
  families (four during an early-availability window) and deletes the oldest branch at GA, so
  a family they name may no longer exist.
- Do not try to read `servicenow.com/docs` directly - it is a JavaScript app that returns no
  readable content. The repo is the only machine-readable source.
- Fetch narrowly - usually 1-3 files per question, not large swaths of the corpus - to keep
  context focused.
- **If a synced local clone is available** through a filesystem or git MCP server, prefer it
  for open-ended discovery: real glob and grep across the whole corpus, with no network
  round-trips, is structurally cheaper than any sequence of fetches. This skill remains the
  right choice when the content must be guaranteed live, or when no local clone exists.
