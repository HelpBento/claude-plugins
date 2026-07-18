---
name: draft-dev-docs
description: >-
  Draft and refresh HelpBento DEVELOPER DOCS from the current codebase — doc
  pages, sections, API references (from an OpenAPI spec), doc versions, and a
  version's release notes (which form this dev-docs KB's own built-in changelog).
  Use when the user asks to "document this API in HelpBento", "refresh my API
  reference from openapi.json/openapi.yaml", "draft dev docs / developer docs / doc
  pages", "create a new docs version", "write release notes / update the changelog
  for this API/docs version", or to revise an existing dev-docs page. NOT for a
  company-wide product-update / "What's New" changelog — that's the standalone
  changelog KB (`draft-changelog` skill). Pages and new versions land as DRAFTS /
  hidden for human review; sections, published-API-reference overwrites, and
  public-version release-notes edits are LIVE and require explicit confirmation.
  Never publishes a page, never makes a version public or default, never deletes.
allowed-tools:
  - mcp__plugin_helpbento_helpbento-api__list_knowledge_bases
  - mcp__plugin_helpbento_helpbento-api__get_writing_settings
  - mcp__plugin_helpbento_helpbento-api__create_knowledge_base
  - mcp__plugin_helpbento_helpbento-api__update_knowledge_base
  - mcp__plugin_helpbento_helpbento-api__list_categories
  - mcp__plugin_helpbento_helpbento-api__list_doc_versions
  - mcp__plugin_helpbento_helpbento-api__list_api_specs
  - mcp__plugin_helpbento_helpbento-api__search_articles
  - mcp__plugin_helpbento_helpbento-api__find_articles_for_symbols
  - mcp__plugin_helpbento_helpbento-api__suggest_linked_articles
  - mcp__plugin_helpbento_helpbento-api__get_article
  - mcp__plugin_helpbento_helpbento-api__create_draft_article
  - mcp__plugin_helpbento_helpbento-api__update_article
  - mcp__plugin_helpbento_helpbento-api__create_category
  - mcp__plugin_helpbento_helpbento-api__update_category
  - mcp__plugin_helpbento_helpbento-api__create_doc_version
  - mcp__plugin_helpbento_helpbento-api__refresh_api_spec
  - mcp__plugin_helpbento_helpbento-api__update_dev_docs_release_notes
  - mcp__plugin_helpbento_helpbento-api__upload_image
  - Bash(node:*)
  - Bash(git:*)
  - AskUserQuestion
  - Read
  - Grep
  - Glob
---

# Draft HelpBento developer docs

You turn work in this codebase into HelpBento **developer documentation** —
doc pages, sections, API references, versions, and release notes. A developer-docs
knowledge base reuses the same infrastructure as the help center, relabeled:
**Pages are articles, Sections are categories**, plus **API references** (OpenAPI)
and **versions** (v1 / v2 / next). Each version has **release notes**, and the KB's
`/changelog` page aggregates them into the KB's own built-in **changelog**.

For drafting plain help-center **articles** (not dev docs), use the
`draft-articles` skill instead. This skill is for `developer_docs` knowledge bases.

> [!IMPORTANT]
> **Two different "changelogs" exist — tell them apart by scope:**
> - A `developer_docs` KB has its **own built-in changelog**: its `/changelog` page
>   (titled "<KB> changelog") aggregates each version's **release notes**. Writing
>   those notes is THIS skill's Step 7 — so "update the changelog / release notes
>   for my API v2" belongs here.
> - A standalone `changelog`-type KB is the **company product-updates / "What's
>   New"** feed — a different KB with its own entries. That's the `draft-changelog`
>   skill. Route "write a product update / What's New entry" there, not here.

## What you can and cannot do

Mirror the article model's safety, but two things in dev docs have **no draft
layer**, so be deliberate:

- **Pages** (articles) — draft-safe. A NEW page, and EVERY edit (body AND
  **title** / **category** / excerpt / tags), ALWAYS land as a DRAFT. On an
  already-PUBLISHED page those metadata edits are staged on the draft too and only
  go live when a human clicks Publish — the public title, URL, and placement stay
  unchanged until then. So renaming or re-homing a page is draft-safe (no live
  confirmation needed); just make sure the user actually wants it, since it takes
  effect on their next publish. ✅
- **New versions** — created HIDDEN (status `draft`). You never make a version
  public or the default; a human does that in HelpBento. ✅
- **Sections** (categories) — **LIVE** structure (no draft state). Creating or
  renaming one takes effect immediately (a rename changes the public URL). Only
  do it when the user asked or confirmed. ⚠️
- **API references** — **no draft copy.** A NEW reference lands as a draft (safe),
  but refreshing a **published** reference overwrites the LIVE public reference
  immediately. That requires explicit user confirmation (`confirmLiveOverwrite`). ⚠️
- **Release notes** — a version-level field (the dev-docs version's `releaseNotes`,
  NOT the changelog KB). On a HIDDEN draft version it's safe; on a **public**
  version it updates that dev-docs KB's live release-notes page immediately. That
  requires explicit user confirmation (`confirmLiveUpdate`). ⚠️

You can **never** publish a page, make a version public/default, delete anything,
or reorder. Those stay in the HelpBento admin UI.

## How you talk to HelpBento

Use the plugin's **MCP tools** (authenticated by a one-time browser login; you
never handle credentials). If a tool returns an authentication error, tell the
user to run `/mcp`, choose **helpbento-api**, and complete the browser login, then
retry — don't retry blindly.

- `list_knowledge_bases` → `[{ id, name, slug, visibility, type }]`. **`type`
  tells you which KBs are `developer_docs`** — dev-docs tools only work on those.
- `create_knowledge_base { name, type, slug?, visibility?, description? }` /
  `update_knowledge_base { knowledgeBaseId, name?, slug?, visibility?, description? }`
  — add or edit a KB. To start docs for a new product, create one with
  `type: 'developer_docs'`. LIVE + owner/admin only; a KB's type can't change. See
  **Managing the knowledge base**.
- `list_doc_versions { knowledgeBaseId }` → `[{ id, label, slug, status, isDefault, order, basedOnVersionId? }]`. `status: 'draft'` is HIDDEN (the next-release working set); `beta`/`stable`/`deprecated` are public.
- `list_api_specs { knowledgeBaseId, docVersionId? }` → `[{ id, title, slug, status, operationCount, … }]`. `status` tells you whether a refresh would hit a live published reference.
- `search_articles { q, mode?, knowledgeBaseId? }`, `get_article { articleId }`, `create_draft_article`, `update_article` — pages are articles; see Step 3. `search_articles` `mode` is `full-text` (default) / `title` / `semantic`.
- `find_articles_for_symbols { symbols, knowledgeBaseId? }` — after an API changes (renamed endpoint, param, or flag), find the doc pages that still mention the OLD names, then offer to update them (Step 3 revise flow). Discovery only.
- `create_category`, `update_category` — sections are categories; see Step 4.
- `create_doc_version { knowledgeBaseId, label, slug?, fromVersionId? }` — see Step 6.
- `refresh_api_spec { spec, knowledgeBaseId?, apiSpecId?, …, confirmLiveOverwrite? }` — see Step 5.
- `update_dev_docs_release_notes { knowledgeBaseId, docVersionId, markdown, confirmLiveUpdate? }` — see Step 7.

## Managing the knowledge base

You can **create** or **update** a developer-docs KB — but a KB is LIVE structure
(no draft state) and needs **owner/admin** rights, so only do it when the user
asked or confirmed the name.

- **Create:** `create_knowledge_base { name, type: 'developer_docs', slug?, visibility?, description? }`.
  Use when the user is starting docs for a new product/API and no dev-docs KB
  exists. The returned `knowledgeBaseId` is where you author. `type` is fixed at
  creation.
- **Update:** `update_knowledge_base { knowledgeBaseId, name?, slug?, visibility?, description? }`.
  A slug change moves the public URL — confirm first. You **cannot** change a KB's
  type (create a new KB instead), nor delete a KB (that's the admin UI).

## Step 0 — Kickoff: settings + run preferences

When the run will write PROSE — doc pages or release notes — settle preferences
once, up front (a pure `refresh_api_spec` run needs no kickoff):

1. **Fetch writing settings** — call `get_writing_settings`
   (`{ aiAssistantEnabled, defaultTone, companyContext }`). A set
   `companyContext` is always part of your brief (product names, terminology,
   audience, style), even when `aiAssistantEnabled` is false.
2. **Check mockup viability** — exactly as `draft-articles` Step 0: mockups
   are only reliable on **Opus 4.8-or-stronger models** (smaller models skip
   them for the run and say so), and the theme question applies only when the
   repo's design tokens define both a light and a dark mode.
3. **Ask ONE `AskUserQuestion` dialog** with whichever of these the user's
   request hasn't already answered (skip the dialog if none are open):
   - **Visuals** — "Create UI mockup images for the doc pages?"
     `Yes, where they help` (recommended; each mockup still subject to
     `draft-articles`'s faithfulness gate) / `You decide per page` /
     `Text only`.
   - **Mockup theme** (both modes exist) — `App default` / `Light` / `Dark`;
     draw every mockup in the chosen mode.
   - **Voice** — if `defaultTone` or `companyContext` is set:
     `Use my Article Assistant settings` (recommended; name the tone) /
     `Different tone for this run` / `Neutral technical`. If neither is set,
     skip this question — dev docs default to a neutral technical voice.
4. Apply the answers for the whole run.

## Step 1 — Pick the developer-docs knowledge base

Call `list_knowledge_bases`. Filter to `type === 'developer_docs'`. If there's
one, use it. If there are several, work out the best fit and **present your
recommendation, then the alternatives**, and let the user confirm. If none are
`developer_docs`, offer to **create one** with
`create_knowledge_base { name, type: 'developer_docs' }` (confirm the name first —
it's a live, owner/admin action; see **Managing the knowledge base**), then author
into it.

## Step 2 — Pick the version FIRST (ask if unsure)

> [!IMPORTANT]
> Before you create or edit ANY page, section, API reference, or changelog in a
> developer-docs KB, you must know **which version** it targets. If the user
> hasn't told you, **ask — don't guess and don't silently default.**

Call `list_doc_versions`. Then:

- **One version, or the user named a version** → use it (resolve a named version
  against the list).
- **The user didn't specify** → ASK. Present the versions with a recommendation,
  e.g.:

  > Which version should this go in?
  > - **Recommended: vNext (draft, hidden)** — the working set for the next release
  > - v2 (stable, default) — the live current docs
  > - v1 (deprecated) — older, still public
  > - …or I can create a **new version** for this.

  Prefer the hidden **`draft`** (next-release) version when one exists — that's
  the normal authoring target and keeps your work hidden until publish. If the
  user wants to document a brand-new release, offer to create a new version
  (Step 6) before authoring into it.
- **No versions yet** → the KB is unversioned; pages/sections you create are the
  implicit single version. You may omit `docVersionId`. (Do NOT call
  `create_doc_version` on a KB that already has content — see Step 6.)

Pass the chosen `docVersionId` to `create_draft_article` / `create_category` /
`refresh_api_spec`. (A page/section under a section inherits that section's
version automatically.) Remember: even when you author into the **default**
(public) version, a new page still lands as a hidden **draft** until published.

## Step 3 — Pages (= articles)

Pages use the exact same Markdown authoring rules and supported blocks as the
`draft-articles` skill (headings ##/###, lists, task lists, the five callout
types, code fences, tables, images, dividers — author Markdown, never raw editor
JSON). Reuse that skill's block reference.

- **Create:** `create_draft_article { title, markdown, knowledgeBaseId, docVersionId?, categoryId?, excerpt?, tags? }`. Lands as a DRAFT page. Don't start the body with an `#` that repeats the title.
- **Revise an existing page:** read it first with `get_article { articleId }` (it
  returns the body as Markdown plus `docVersionId` and `hasUnpublishedDraft`),
  make the smallest change, then `update_article { articleId, markdown, … }`. If
  `hasUnpublishedDraft` is true, warn the user before overwriting. Every edit —
  body AND `title` / `excerpt` / `tags` / `categoryId` — lands on the page's draft
  and never publishes.
  > [!NOTE]
  > On an already-PUBLISHED page, `title` (→ public title + URL slug) and
  > `categoryId` (→ public nav placement) are staged on the draft alongside the
  > body and only go live when a human publishes — the public title, URL, and
  > placement stay unchanged until then. So renaming or re-homing a published page
  > is draft-safe; there is no live-metadata confirmation. Still, only rename or
  > re-home when the user asked, since it takes effect on their next publish.
- Ground every claim in the real code (endpoint paths, flags, labels). Flag
  anything you can't verify with `> [!NOTE] Verify: …`.

### Images

Pages are articles, so the same options apply: a generated
`data:image/svg+xml;base64,…` mockup (see `draft-articles`'s **Generating UI
mockups**), or a real screenshot hosted via
`upload_image { articleId, contentType, dataBase64 }` (the page's own
`articleId` — see `draft-articles`'s **Uploading real images**), returning
`{ path, url }` to embed as `![alt](url)`.

### Cross-linking

Before finishing a page (new or revised), call
`suggest_linked_articles { articleId, topic?, knowledgeBaseId? }` — same tool
and behaviour as `draft-articles`'s **Cross-linking related articles**: it
matches by semantic similarity among PUBLISHED articles/pages (never drafts or
archived), excluding the page itself. Review the suggestions yourself, then add
your own Markdown cross-references using the returned `slug` — never
auto-insert unreviewed links.

## Step 4 — Sections (= categories)

Dev-docs sections are categories with **no draft state** — creating or renaming
one is LIVE. Only do it when the user asked or confirmed the name + KB + version.

- **Create:** `create_category { name, knowledgeBaseId, docVersionId?, description?, parentId? }` (created unpublished; a human publishes). Use the returned `categoryId` as a page's/spec's `categoryId`.
- **Rename / re-nest:** `update_category { categoryId, … }` — confirm first (a rename changes the public URL).

## Step 5 — API references (from an OpenAPI spec)

- **Find the spec** in the repo: an `openapi.json` / `openapi.yaml` / `swagger.json`
  (parse YAML to a JSON object first), or generate one from route/controller
  definitions. The tool needs the **full OpenAPI 3.0/3.1 document as a JSON
  object** — NOT Markdown, NOT a YAML string. (Swagger 2.0 is rejected — convert it.)
- **Create a new reference:** `refresh_api_spec { spec, knowledgeBaseId, docVersionId?, title?, description?, slug?, categoryId? }`. Lands as a DRAFT. Title defaults to the spec's `info.title`.
- **Refresh an existing reference:** first `list_api_specs` to find its id **and
  status**.
  - If the reference is a **draft** → `refresh_api_spec { apiSpecId, spec }` is safe.
  - If the reference is **published** → refreshing overwrites the LIVE public
    reference immediately (references have no draft copy). **STOP and tell the
    user**, e.g. *"`Payments API` is published — refreshing replaces the live
    public reference now. Proceed?"* Only on an explicit yes, retry with
    `confirmLiveOverwrite: true`. A refresh never changes published/draft status.
- Report `operationCount` / `tagSummaries` back so the user sees what landed.

## Step 6 — New version

`create_doc_version { knowledgeBaseId, label, slug?, fromVersionId? }`. The new
version is ALWAYS created **hidden (`draft`)** and is never public or default —
tell the user they make it public/default in the admin UI when ready.

- **Empty** (omit `fromVersionId`) — start fresh.
- **Clone** (`fromVersionId` from `list_doc_versions`) — copies ALL pages,
  sections, and API references of that version (can be large). Say so, and report
  the `copied` counts.
- **First-version edge case:** if the KB has **no versions yet but already has
  content**, the tool will refuse (creating the first version would publish that
  content live as the default). Direct the user to do it once in the admin UI.

## Step 7 — Release notes (this dev-docs KB's changelog)

`update_dev_docs_release_notes { knowledgeBaseId, docVersionId, markdown, confirmLiveUpdate? }`
writes the chosen version's release notes (the per-version `releaseNotes`). The
KB's `/changelog` page aggregates every version's notes into this dev-docs KB's own
built-in **changelog**, so "update the changelog / release notes for v2" means
writing that version's notes here.

> [!IMPORTANT]
> This is the **developer-docs KB's** changelog (per-version release notes) — NOT
> the standalone **changelog KB** (company product updates / "What's New"). If the
> user wants a company product-update entry, stop and use the `draft-changelog`
> skill instead.

- On a **hidden `draft`** version → safe; just write it.
- On a **public** version (`beta`/`stable`/`deprecated`) → it updates the live
  release-notes page immediately. **Confirm with the user first**, then retry with
  `confirmLiveUpdate: true`.
- Author the notes in Markdown (same blocks as pages). A good release note groups
  changes (New, Improved, Fixed, Breaking) grounded in the diff/PR.

## Step 8 — Report back

For each artifact, report what it is, the version it went into, whether it is a
hidden **draft** or a **live** change (a new/renamed section, a published-page
rename/re-home, a published-spec refresh, or a public-version's release notes), and
the returned `adminUrl` / `manageUrl` deep-link. Remind the user that pages and new
versions are awaiting their review/publish in HelpBento — the live site is unchanged
until they publish, except for any live change you explicitly confirmed.
