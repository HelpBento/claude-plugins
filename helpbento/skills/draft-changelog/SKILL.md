---
name: draft-changelog
description: >-
  Draft HelpBento CHANGELOG entries — company product updates / release notes /
  "What's New" — from the current codebase, and push them to HelpBento as DRAFTS.
  Use when the user asks to "draft a changelog entry", "write a product update",
  "add a What's New entry", "create a release note for this PR / diff / tag", or to
  revise an existing changelog entry. Reads the relevant code, diff, or PR, writes
  the entry in Markdown grouped by New / Improved / Fixed / Breaking, and creates a
  draft for human review. Targets a `changelog`-type knowledge base (NOT the
  developer-docs API release notes — that's `draft-dev-docs`). Never publishes.
allowed-tools:
  - mcp__plugin_helpbento_helpbento-api__list_knowledge_bases
  - mcp__plugin_helpbento_helpbento-api__create_knowledge_base
  - mcp__plugin_helpbento_helpbento-api__update_knowledge_base
  - mcp__plugin_helpbento_helpbento-api__list_changelog_entries
  - mcp__plugin_helpbento_helpbento-api__get_changelog_entry
  - mcp__plugin_helpbento_helpbento-api__create_changelog_entry
  - mcp__plugin_helpbento_helpbento-api__update_changelog_entry
  - mcp__plugin_helpbento_helpbento-api__upload_image
  - Bash(node:*)
  - Bash(git:*)
  - Read
  - Grep
  - Glob
---

# Draft HelpBento changelog entries

You turn work in this codebase into HelpBento **changelog entries** — the
company's public **product updates** feed (a "What's New" / release-notes stream).
Everything you write lands as a **DRAFT** for a human to review and publish in the
HelpBento admin UI. You can never **publish** or **delete**; updates only ever
touch an entry's draft version, never its live published content.

> [!IMPORTANT]
> A **changelog** here is a `changelog`-type **knowledge base** of company product
> updates. This is a DIFFERENT feature from a developer-docs version's **release
> notes** (API/SDK notes), which the `draft-dev-docs` skill handles via
> `update_dev_docs_release_notes`. If the user wants API/version release notes on a
> `developer_docs` KB, use `draft-dev-docs` instead.

You author entries in **Markdown**; the HelpBento server converts your Markdown to
the editor's format on ingestion. Reuse the exact Markdown blocks documented in the
`draft-articles` skill (headings `##`/`###`, lists, task lists, the five callout
types, code fences, tables, images, dividers). Do NOT emit raw editor JSON, and do
NOT start the body with a top-level `#` that repeats the title.

## How you talk to HelpBento

Use the plugin's **MCP tools** (authenticated by a one-time browser login; you
never handle credentials). If a tool returns an authentication error, tell the user
to run `/mcp`, choose **helpbento-api**, and complete the browser login, then retry
— don't retry blindly.

- `list_knowledge_bases` → `[{ id, name, slug, visibility, type }]`. Filter to
  `type === 'changelog'`.
- `create_knowledge_base { name, type: 'changelog', slug?, visibility?, description? }`
  — create a changelog KB when none exists (LIVE + owner/admin; needs the company's
  `changelog` plan entitlement). See **Step 1**.
- `update_knowledge_base { knowledgeBaseId, name?, slug?, visibility?, description? }`
  — edit a KB (a slug change moves the public URL — confirm first).
- `list_changelog_entries { knowledgeBaseId }` → `[{ id, title, versionNumber, releaseDate, status, order }]`. Use to check for duplicates.
- `get_changelog_entry { entryId }` → the entry body **as Markdown** plus `{ title, versionNumber, releaseDate, status, contentSource, hasUnpublishedDraft, knowledgeBaseId }`. Use before updating.
- `create_changelog_entry { knowledgeBaseId, title, markdown, versionNumber?, releaseDate? }` → `{ entryId, versionId, status, adminUrl }`.
- `update_changelog_entry { entryId, markdown, title?, versionNumber?, releaseDate? }` → `{ entryId, versionId, status, adminUrl }`.
- `upload_image { changelogEntryId, contentType, dataBase64 }` → `{ path, url }`. See **Images**.

## Step 1 — Pick (or create) the changelog knowledge base

Call `list_knowledge_bases` and filter to `type === 'changelog'`.

- **One changelog KB** → use it.
- **Several** → recommend the best fit, then the alternatives, and let the user
  confirm.
- **None** → offer to **create one**: `create_knowledge_base { name, type: 'changelog' }`.
  Confirm the name first — it's a LIVE, **owner/admin** action, and it requires the
  company's **`changelog` plan entitlement** (if the tool returns an entitlement /
  permission error, tell the user their plan or role doesn't allow it, and stop —
  don't retry). Use the returned `knowledgeBaseId`.

## Step 2 — Understand what to announce

- Identify the release or change. If the user mentions a diff, branch, PR, or tag,
  read it (`git diff`, `git diff main...HEAD`, `git log`, the PR) and the source it
  touches, so the entry reflects real, shipped behaviour.
- Ground every claim in the actual code — real feature names, UI labels, endpoints,
  flags. Never invent. Flag anything you can't verify with `> [!NOTE] Verify: …`.
- A changelog entry is **user-facing** and marketing-adjacent: lead with what
  changed for the customer and why it matters, not internal refactors.

## Step 3 — Infer the version and date (confirm, don't guess)

- **`versionNumber`** is an optional tag (e.g. `v1.5.0`). Infer it from the latest
  git tag (`git describe --tags --abbrev=0`) or `package.json` `version`. If you
  can't find one, omit it or ask — don't fabricate.
- **`releaseDate`** defaults to today. Infer from the tag/commit date when the
  release already shipped; otherwise leave it to default.
- When you inferred either, **state your inference and let the user correct it**
  before creating.

## Step 4 — Check for duplicates

Call `list_changelog_entries { knowledgeBaseId }` and scan titles/versions. If a
close match exists (same version or same feature), surface it and ask whether to
update that entry (Step 6) rather than create a near-duplicate.

## Step 5 — Write and create the draft

- Write a concise, benefit-led **title** (e.g. "Smart Inbox: reach a human when AI
  can't help").
- Structure the body with `##` sections — typically **New**, **Improved**,
  **Fixed**, **Breaking** — using bullet lists; keep only the sections that apply.
- Call `create_changelog_entry { knowledgeBaseId, title, markdown, versionNumber?, releaseDate? }`.
  It returns `{ entryId, versionId, status: 'draft', adminUrl }`.

## Images

Product updates land better with a screenshot of the actual feature. Once the
entry exists (Step 5 created it, or you're revising one in Step 6), host a real
screenshot with `upload_image { changelogEntryId, contentType, dataBase64 }`:

1. **Get the image bytes as base64** — read a screenshot/mockup file on disk
   and base64-encode it, or encode one already in memory.
2. **Call `upload_image`** with the entry's `entryId` as `changelogEntryId`,
   `contentType` (`image/png` / `image/jpeg` / `image/gif` / `image/webp`; max
   10MB), and `dataBase64`. Returns `{ path, url }`.
3. **Embed the `url`** in the entry's Markdown as `![alt](url)`.

The image lives in that entry's own media folder and is deleted automatically
if the entry is deleted. It cannot be reused on a different entry.

## Step 6 — Updating an existing entry

To revise an entry (the release changed, a line is wrong, add a section) —
**edit it, don't recreate it**:

1. `list_changelog_entries` / confirm which entry, then `get_changelog_entry { entryId }`
   to read the current body **as Markdown** plus metadata.
2. If `hasUnpublishedDraft` is true, the entry already has unpublished draft edits
   (possibly a human's) — **warn and confirm before overwriting**.
3. Make the smallest change; keep the rest. Call
   `update_changelog_entry { entryId, markdown, title?, versionNumber?, releaseDate? }`.
   The edit lands on the entry's **draft** and never publishes. (Pass `versionNumber: ""`
   to clear the tag; omit a field to leave it unchanged.)

## Step 7 — Report back

For each entry, report the title, the version/date, and the returned `adminUrl`
deep-link. Remind the user these are **DRAFTS** — nothing goes live on the public
changelog until a human reviews and publishes each one in HelpBento.
