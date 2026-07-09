# HelpBento — Claude Code plugin

Draft HelpBento content from your codebase, straight from Claude Code. Ask it to
document a feature / change / PR and Claude reads the relevant code or `git diff`,
writes Markdown grounded in the real code, and pushes it to your HelpBento account
as a **draft**. Nothing is ever auto-published — a human reviews and publishes
every draft in the HelpBento admin UI.

Three skills, picked automatically from what you ask for:

- **`draft-articles`** — help-center / knowledge-base articles, auto-categorised
  against your existing categories.
- **`draft-dev-docs`** — developer docs: pages, sections, API references (from an
  OpenAPI spec), doc versions, and a version's API release notes.
- **`draft-changelog`** — changelog / **product-update** entries (a "What's New"
  feed), grouped New / Improved / Fixed / Breaking from a PR, diff, or tag.

All three can also **create and update knowledge bases** (help-center, developer-
docs, or changelog). The plugin is **skills plus a remote MCP server**; you connect
once with a browser login (OAuth) — there's no API key to paste.

## What you need

- Claude Code (ships with Node 24).
- A HelpBento account that belongs to a company.

## Connect

1. Install the plugin.
2. Run `/mcp`, choose **helpbento-api**, and complete the **browser login** when
   Claude Code opens it. Sign in to HelpBento and click **Approve**.
3. That's it — Claude Code stores and refreshes the tokens. Drafts you create are
   attributed to your HelpBento user.

To disconnect or re-authenticate, use `/mcp` → **helpbento-api** → *Clear
authentication*.

## Usage

In any project, just ask — the matching skill triggers, fetches your knowledge
bases / categories, checks for duplicates, drafts Markdown grounded in your code,
creates the draft(s), and reports back a deep-link `adminUrl` for review:

```
> I just shipped passwordless login. Create help center articles for it.
> Refresh my API reference from openapi.json and add a "Webhooks" page.
> Draft a changelog entry for this release from the latest git tag.
```

## Notes

- **Never publishes or deletes.** Article / page / changelog-entry edits (new and
  updates) always land as **drafts** a human publishes. Categories/sections and
  knowledge bases are live structure with no draft state, so the plugin
  creates/updates those only when you ask or confirm (and KB management needs
  owner/admin rights).
- You author Markdown; the HelpBento server converts it to the editor's block
  format on ingestion. Don't repeat the title as a top `#` heading (the title is
  rendered separately).
