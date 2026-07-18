---
name: draft-articles
description: >-
  Draft HelpBento knowledge base / help center articles from the current
  codebase and push them to HelpBento as DRAFTS. Use this when the user asks to
  "create help center articles for this feature / change / PR", "write knowledge
  base articles", "document this feature for the help center", "add this to
  HelpBento", or anything similar. Reads the relevant code or git diff, writes the
  article(s) in Markdown grounded in the real code, auto-categorises against the
  company's existing HelpBento categories, and creates drafts for human review.
  Can also revise an EXISTING article — reading its current content and writing
  the edits back as a draft. Never publishes.
allowed-tools:
  - mcp__plugin_helpbento_helpbento-api__list_categories
  - mcp__plugin_helpbento_helpbento-api__list_tags
  - mcp__plugin_helpbento_helpbento-api__list_knowledge_bases
  - mcp__plugin_helpbento_helpbento-api__get_writing_settings
  - mcp__plugin_helpbento_helpbento-api__create_knowledge_base
  - mcp__plugin_helpbento_helpbento-api__update_knowledge_base
  - mcp__plugin_helpbento_helpbento-api__search_articles
  - mcp__plugin_helpbento_helpbento-api__find_articles_for_symbols
  - mcp__plugin_helpbento_helpbento-api__suggest_linked_articles
  - mcp__plugin_helpbento_helpbento-api__list_content_gaps
  - mcp__plugin_helpbento_helpbento-api__create_draft_article
  - mcp__plugin_helpbento_helpbento-api__get_article
  - mcp__plugin_helpbento_helpbento-api__update_article
  - mcp__plugin_helpbento_helpbento-api__archive_article
  - mcp__plugin_helpbento_helpbento-api__unarchive_article
  - mcp__plugin_helpbento_helpbento-api__create_category
  - mcp__plugin_helpbento_helpbento-api__update_category
  - mcp__plugin_helpbento_helpbento-api__upload_image
  - Bash(node:*)
  - Bash(git:*)
  - Bash(base64:*)
  - Bash(tr:*)
  - Bash(wc:*)
  - Bash(xmllint:*)
  - Bash(rsvg-convert:*)
  - Bash(cairosvg:*)
  - Bash(qlmanage:*)
  - Bash(mktemp:*)
  - Write
  - AskUserQuestion
  - Read
  - Grep
  - Glob
---

# Draft HelpBento articles

You turn work in this codebase into HelpBento knowledge base **draft** articles —
creating new ones, or revising existing ones. Everything you write lands as a
DRAFT for a human to review and publish in the HelpBento admin UI. You can never
**publish** or **delete** anything; updates only ever touch an article's draft
version, never its live published content.

You author articles in **Markdown**. Do NOT emit raw editor JSON — author
Markdown; the HelpBento server converts your Markdown to the editor's format on
ingestion.

## How you talk to HelpBento

Use the plugin's **MCP tools**. They're served by HelpBento over an
authenticated connection — the user connects once via a browser login (Claude
Code manages the OAuth tokens), so you never handle any credentials:

- `mcp__plugin_helpbento_helpbento-api__list_categories` — active categories `{ id, name, slug, knowledgeBaseId }`.
- `mcp__plugin_helpbento_helpbento-api__list_tags` — the company's EXISTING tags `{ id, name, color }`. These are the ONLY tags you may apply; you cannot create tags.
- `mcp__plugin_helpbento_helpbento-api__list_knowledge_bases` — `{ id, name, slug, visibility }`.
- `mcp__plugin_helpbento_helpbento-api__get_writing_settings` — `{ aiAssistantEnabled, defaultTone, companyContext }`: the company's AI Article Assistant settings. See **Step 0**.
- `mcp__plugin_helpbento_helpbento-api__search_articles` — args `{ q, mode?, knowledgeBaseId? }`. `mode`: `full-text` (default — title + body), `title` (fast), or `semantic` (concept match via embeddings, e.g. "login" ↔ "auth"). Best-effort, not exhaustive.
- `mcp__plugin_helpbento_helpbento-api__find_articles_for_symbols` — args `{ symbols, knowledgeBaseId? }`; finds articles that MENTION any of the given names (changed endpoints/flags/labels) with a match snippet — for spotting stale docs after a code change.
- `mcp__plugin_helpbento_helpbento-api__suggest_linked_articles` — args `{ articleId, topic?, knowledgeBaseId? }`; suggests existing PUBLISHED articles worth cross-linking, by semantic similarity. See **Cross-linking related articles**.
- `mcp__plugin_helpbento_helpbento-api__create_draft_article` — args `{ title, markdown, categoryId?, knowledgeBaseId?, excerpt?, tags? }`; `tags` are ids/names from `list_tags` (unknown tags are ignored). Returns `{ articleId, versionId, status, adminUrl, ignoredTags }`.
- `mcp__plugin_helpbento_helpbento-api__get_article` — args `{ articleId }`; returns the existing article's body **as Markdown** plus `{ title, slug, status, contentSource, hasUnpublishedDraft, excerpt, tags, categoryId, knowledgeBaseId }`. Use this before updating, so you edit what's actually there.
- `mcp__plugin_helpbento_helpbento-api__update_article` — args `{ articleId, markdown, title?, excerpt?, tags?, categoryId? }`; works on any article (draft **or published**). ALL edits — body AND title/excerpt/tags/categoryId — land as a DRAFT and never publish: on a published article they are held in the draft and only go live when a human clicks Publish, so the public title and URL stay unchanged until then. `tags` are ids/names from `list_tags` (unknown tags are ignored). Returns `{ articleId, versionId, status, adminUrl, ignoredTags }`.
- `mcp__plugin_helpbento_helpbento-api__create_category` — args `{ name, knowledgeBaseId?, description?, parentId? }`; creates a LIVE (but unpublished) category. Returns `{ categoryId, name, manageUrl }`.
- `mcp__plugin_helpbento_helpbento-api__update_category` — args `{ categoryId, name?, description?, parentId? }`; renames/edits a category (a LIVE change). Returns `{ categoryId, manageUrl }`.
- `mcp__plugin_helpbento_helpbento-api__upload_image` — args `{ articleId, contentType, dataBase64 }`; hosts a real screenshot/mockup and returns `{ path, url }` to embed. See **Uploading real images**.

If a tool fails with an authentication error (e.g. the user hasn't connected
yet, or their session expired), tell the user to run `/mcp`, choose **helpbento-api**,
and complete the browser login — then retry. Don't retry blindly.

## Step 0 — Kickoff: settings + run preferences

Settle the run's preferences ONCE, up front, before reading code or writing
anything:

1. **Fetch the company's writing settings** — call `get_writing_settings`.
   When `companyContext` is set it is ALWAYS part of your brief — real product
   names, terminology, audience, and style guidance — even when
   `aiAssistantEnabled` is false (that toggle gates the in-app editor
   assistant, not the company's voice).
2. **Check mockup viability:**
   - **Model floor:** drawing SVG UI mockups is only reliable on
     Opus 4.8-or-stronger models. If you are a smaller/faster model (e.g. a
     Sonnet- or Haiku-class model), mockups are OFF for this run: skip the
     visuals and theme questions, write text-only articles, and tell the user
     that mockups need a stronger model.
   - **Theme modes:** check whether the repo's design tokens define both a
     light and a dark mode — a second token block under `[data-theme=`,
     `.dark`, `data-mode`, or `prefers-color-scheme: dark`. Only when BOTH
     exist does the theme question apply; otherwise use the one mode the app
     has.
3. **Ask ONE `AskUserQuestion` dialog** with the questions that are still
   open — never a drip of separate asks. Skip any question the user's request
   already answered ("no images", "dark mockups", "keep it casual"), and skip
   the dialog entirely if nothing is open.
   - **Visuals** — "Create UI mockup images for the article(s)?" Options:
     `Yes, where they help` (recommended — feature-card art on feature
     articles plus step illustrations where a how-to is clearer shown; every
     mockup is still subject to the faithfulness gate), `You decide per
     article`, `Text only` (no mockups and no feature cards).
   - **Mockup theme** (only when both modes exist and visuals may be drawn) —
     `App default`, `Light`, `Dark`. If they choose Text only, ignore this
     answer.
   - **Voice** — if `defaultTone` or `companyContext` is set:
     `Use my Article Assistant settings` (recommended; name the tone and note
     that company context was found), `Different tone for this run` (follow up
     with the four tones), `Neutral professional`. If neither is set: ask the
     tone directly — `Professional`, `Casual`, `Technical`, `Friendly`.
4. **Apply the answers for the whole run:** visuals governs the feature-card
   decision (Step 1) and **Generating UI mockups**; theme feeds
   `references/mockups.md` → Step A; voice + `companyContext` govern all prose
   you write.

## Step 1 — Understand what to document

- Identify the feature or change the user wants documented. If they mention a
  diff, a branch, or a PR, read it (`git diff`, `git diff main...HEAD`, the PR
  contents, staged changes) and the source files it touches, plus adjacent files
  (routes, config, README fragments) so the article reflects real behaviour.
- Ground every claim in the actual code: real UI labels, endpoint paths, env var
  names, flags, and defaults. Never invent these. If a fact cannot be grounded
  in the repo, omit it or flag it inline for the reviewer with
  `> [!NOTE] Verify: ...`.
- Confirm scope with the user if ambiguous (one overview article vs. several,
  one per provider, etc.).
- **Is this an app feature?** If the article documents a product feature
  (something users enable or use in the app — an inbox, analytics, an
  integration), it can carry a **feature card**: a branded visual block
  (icon + label + title + blurb, with a mockup image) that mirrors the in-app
  feature pitch. Whether to add one follows the **Step 0 visuals answer** —
  `Yes, where they help` → add the card on feature-documenting articles;
  `You decide` → your judgment; `Text only` → no cards. The card's mockup
  image is generated only when the faithfulness gate in **Generating UI
  mockups** passes; if it fails, the card ships without an image. Never put a
  card on conceptual, troubleshooting, or FAQ articles, where a
  marketing-style card would feel out of place. See the feature block in Step 4.

## Step 2 — Fetch the existing taxonomy

- Call `list_knowledge_bases` and `list_categories` to learn the company's
  structure. Each category carries a `knowledgeBaseId`, so group the categories by
  knowledge base — you'll need that grouping in Step 5. Note the returned `id`s;
  you pass them to `create_draft_article`.
- Also call `list_tags` to fetch the company's existing tags. These are the only
  tags you may apply to an article; you never create tags.

## Step 3 — Check for duplicates

- For each article you plan to write, call `search_articles` with
  `{ q: "<title or topic>" }` first. It defaults to **full-text** (title + body);
  for "is there already an article about X?" prefer `mode: "semantic"` to catch
  concept matches worded differently. Best-effort, not exhaustive. If a close
  match exists, surface it to the user and ask whether to proceed or update the
  existing article (see **Updating an existing article**) rather than creating a
  near-duplicate.

## Step 4 — Write each article in Markdown

- Author clear, task-oriented help content in GitHub-Flavored Markdown.
- Do NOT start the body with a top-level `#` heading that repeats the title —
  HelpBento renders the title separately, so it would appear twice. Begin with
  the intro paragraph and use `##` / `###` for section headings (only heading
  levels 1–3 exist; deeper ones clamp to 3).
- Do NOT emit raw editor JSON — author Markdown; HelpBento converts it server-side.

### Blocks HelpBento supports

Write the Markdown on the left; it becomes the block on the right. Stick to these
— Markdown that maps to no block is dropped on import.

- **Headings** — `##` / `###` (levels 1–3).
- **Paragraphs** with inline `**bold**`, `*italic*`, `` `code` ``,
  `[links](url)`, `~~strikethrough~~`, `==highlight==`.
- **Lists** — `-` / `1.`, nesting preserved (ordered + unordered).
- **Task lists** — `- [ ]` / `- [x]` becomes a checklist (flat; not nested).
- **Callouts** — a GitHub-style alert becomes a coloured callout with an icon.
  **Prefer these over plain quotes** for notes/tips/warnings. The five supported
  types (use ONLY these — any other `[!…]` won't become a callout):
  - `> [!NOTE]` — blue / info
  - `> [!TIP]` — green / lightbulb
  - `> [!IMPORTANT]` — purple / star
  - `> [!WARNING]` — yellow / warning triangle
  - `> [!CAUTION]` — red / error

  The text may sit on the marker line (`> [!NOTE] Heads up.`) or wrap onto the
  following `>` lines — both convert to a callout.
- **Quotes** — a plain `>` blockquote (no `[!TYPE]`) → quote. Reserve quotes for
  actual quotations; use a callout for asides.
- **Code** — fenced blocks ```` ```lang ```` → code block. Use a supported
  language tag: `plaintext, bash, json, yaml, javascript, typescript, jsx, tsx,
  markup` (HTML), `css, scss, python, sql, java, go, rust, ruby, php, csharp,
  swift, kotlin, markdown, docker, graphql`.
- **Tables** — GFM tables (header row + `---` separator); the first row is the
  heading.
- **Images** — `![alt](https://…)` on its own line. The URL can be: an
  absolute external URL, a generated `data:image/svg+xml;base64,…` mockup (see
  **Generating UI mockups**), or a URL returned by `upload_image` for a real
  screenshot (see **Uploading real images**).
- **Feature card** — a branded visual representation of an app feature (icon +
  label + title + blurb, with an optional screenshot in a window frame). Use a
  fenced ` ```feature ` block. Only add one when the user confirmed it in Step 1,
  and place it near the top of the article:

  ````
  ```feature
  icon: rocket
  eyebrow: New feature
  title: Smart Inbox
  image: https://cdn.example.com/inbox.png

  Turn unanswered AI chats into real conversations — customers reach a human
  exactly when they need one.
  ```
  ````

  - `icon:` one of `sparkles, rocket, zap, message-square, bell, bar-chart,
    layers, plug, shield, bot, package`. Omit it for the default (`sparkles`);
    use `none` for no icon.
  - `title:` the feature name (plain text). `eyebrow:` a short label above it
    (optional, plain text).
  - `image:` an absolute screenshot/mockup URL, a generated
    `data:image/svg+xml;base64,…` mockup, or a URL from `upload_image` for a
    real screenshot (optional). By default you generate an SVG mockup — see
    **Generating UI mockups** (or **Uploading real images** for a real one).
  - A blank line (or a `---` line) separates the metadata from the description
    blurb, which supports inline `**bold**` / `*italic*` / `` `code` `` /
    `[links](url)`.
- **Divider** — `---` on its own line.

- Keep the title concise and task-oriented; write a one-sentence plain-text
  `excerpt`. For `tags`, call `list_tags` and apply ONLY the existing tags whose
  names fit the article — pass their names or ids. Never invent a tag: unknown
  values are dropped and returned in `ignoredTags`. If nothing fits, omit `tags`.
  Pass the Markdown directly as the `markdown` argument.

## Generating UI mockups

Articles look far better with a visual. Because this skill runs inside each
company's OWN codebase, you can DRAW a mockup of the relevant screen — styled from
that repo's design tokens — and embed it inline as a `data:image/svg+xml` image.
No screenshots, no hosting, no new dependencies. The full craft and the technical
contract are in `references/mockups.md` — **read it before generating one.**

**The faithfulness gate — check BEFORE deciding to draw.** A mockup depicts a
real screen of this app, so you may only draw one when ALL four are true:

0. You are an **Opus 4.8-or-stronger model**. Mockup drawing is not reliable
   below that tier — smaller models (Sonnet-, Haiku-class) skip mockups for
   the whole run and say so (Step 0).
1. You have READ the screen's actual template/markup in this repo **in this
   session** — the component HTML/JSX/template file for that route. Inferring
   the screen from its route name, the feature name, or general knowledge of
   what such screens usually look like does not pass.
2. You can list, verbatim from that template, the real labels you will draw:
   nav items, button text, field placeholders, column headers, status names.
3. You found the repo's design tokens (`references/mockups.md` → Step A), or
   you are deliberately using the documented neutral default and will say so.

If any of the four is false, draw nothing: omit the `image:` line / the
`![alt](…)` and note in your report which gate failed. A card with no image
renders fine; a mockup of a screen that doesn't exist as drawn misleads every
reader. The same rule applies element-by-element while drawing: never fill a
gap with an invented control or label — leave it out.

**When to generate one** (governed by the Step 0 visuals answer, and only for
screens that pass the gate)
- **Feature cards:** when an article carries a feature card (Step 1 / Step 4)
  and the feature's main screen passes the gate, draw a matching mockup and
  put it on the card's `image:` line.
- **Instructional steps:** when a "how to do X" step is clearer shown, add an
  inline `![alt](data:…)` mockup of that exact screen — the gate applies to
  each screen you depict.
- Draw every mockup in the **theme chosen at Step 0** (`references/mockups.md`
  → Step A shows how to pull that mode's token values). Never mix modes across
  one run's mockups.

**Fidelity follows purpose** (see `references/mockups.md` → Step C):
- decorative / spotlight card → more abstract, brand-forward — abstraction
  means showing LESS of the real screen, never inventing what isn't there;
- demonstrating a how-to → faithful near-screenshot with REAL labels.

**The non-negotiable contract** (details in `references/mockups.md` → Step D):
- derive colors / type from the repo's design tokens (or use the neutral default
  and SAY so in your report);
- one `<svg>` with a `viewBox`; embed as a SINGLE-LINE
  `data:image/svg+xml;base64,…` built with `base64 < f.svg | tr -d '\n'`;
- keep the data URI **under 50 KB**;
- **fail-safe:** if you cannot validate the SVG, embed NO image rather than a
  broken one.

Report which mockups you generated, the fidelity used, and whether the style was
derived from the repo or fell back to the default.

## Uploading real images

For an actual screenshot, photo, or diagram (not a UI mockup you can draw as
SVG) — e.g. a screenshot you just took, or an image file already in the repo —
host it in HelpBento with `upload_image` instead of linking to an external URL:

1. **Get the bytes as base64.** For a file on disk, read it as binary and
   base64-encode it (`base64 < screenshot.png | tr -d '\n'`, or read+encode in
   Node). For an in-memory image (e.g. from a screenshot tool), encode that.
2. **Call `upload_image`** with `{ articleId, contentType, dataBase64 }` —
   `articleId` is the article you're drafting/editing (create or read it
   first; the target must already exist). `contentType` is one of
   `image/png`, `image/jpeg`, `image/gif`, `image/webp`; max 10MB.
3. **Embed the returned `url`** in your Markdown: `![alt](url)` for an inline
   image, or as a feature card's `image:` line (Step 4).

The image lives in that article's own media folder and is deleted
automatically if the article is deleted. It cannot be reused on a different
article — upload it again (or reuse the same articleId) if needed elsewhere.

## Cross-linking related articles

Before finishing a draft (new or revised), suggest related articles to link to:

- Call `suggest_linked_articles { articleId, topic?, knowledgeBaseId? }` — the
  `articleId` is the article you just created or are revising. It matches by
  semantic similarity to that article's own title + body against the company's
  PUBLISHED articles (a draft or archived article never matches — the
  underlying index only carries published content), excluding the article
  itself. `topic` is an optional hint (e.g. `"billing"`) to bias the match.
- Returns `[{ id, title, slug, excerpt, adminUrl }]`, nearest first.
- **Review the suggestions yourself** — only use ones that are genuinely
  relevant; ignore weak or off-topic matches. Write your own Markdown
  cross-references using the `slug`, e.g. `[See also: Webhook setup](/webhook-setup)`
  — typically gathered into a short "Related articles" section near the end of
  the body.
- Never treat a suggestion as already linked: it only lands in the article once
  YOU add the Markdown and the draft is created/updated — same human review
  before publish as the rest of the article.

## Step 5 — Choose the knowledge base, then the category

Pick the knowledge base FIRST — categories belong to a KB (a category's
`knowledgeBaseId` tells you which), so the KB choice scopes the category choice.

- **No KBs, or exactly one KB**: just use it (or omit `knowledgeBaseId` to use the
  default). Don't ask the user — proceed.
- **Multiple KBs**: do NOT silently pick one. Work out the best-fit KB for the
  article — match its topic against each KB's name/purpose and the categories that
  live under it — then the best-fit category within that KB. **Present the choice
  to the user with your recommendation first, then the other viable options**, e.g.:

  > Recommended: **Product Docs** → **Getting Started**
  > Other options: **API Reference** → **Authentication** · **Internal Wiki** → **Onboarding**

  Ask them to confirm or pick a different KB/category before you create the draft.
  If they already named a target up front, or tell you to "just use your pick",
  go with the recommendation without re-asking. When creating several articles at
  once, show the recommended KB+category for each and let them confirm/adjust in
  one go.

Then choose the category, within the chosen KB:

- Pick the `categoryId` (one that belongs to that KB) whose name/slug best matches
  the article.
- If the user named a target ("put it under Billing"), resolve it against the
  fetched list and use its `id`.
- If nothing fits, do NOT silently invent a `categoryId`. Either omit it (the
  article lands uncategorised for triage), or — when the user wants a new
  category — create one with `create_category` after confirming the name and KB
  with them (see **Managing categories** below). You still cannot create
  knowledge bases.

Pass both the chosen `knowledgeBaseId` and `categoryId` to `create_draft_article`.

## Step 6 — Create the draft(s)

- For each article, call `create_draft_article` with `{ title, markdown, categoryId?, knowledgeBaseId?, excerpt?, tags? }`.
- It returns `{ articleId, versionId, status, adminUrl }`. The status is always
  `draft`.

## Step 7 — Report back

- For each draft created, report: the title, the chosen category, and the
  returned `adminUrl` deep-link so the user can jump straight to the editor.
- Remind the user these are DRAFTS — nothing is published until they review and
  publish each one in HelpBento. A human reviews before publish, always.

## Updating an existing article

When the user wants to revise an article that already exists (the code changed, a
step is wrong, they want a section added) — **edit it, don't recreate it**. Use a
read-modify-write flow so you preserve the existing wording and structure rather
than overwriting someone's curation.

This works on **published** articles too, not just drafts. Editing a published
article's BODY doesn't change the live page — it creates a draft of your changes
for a human to review and publish; nothing you do ever publishes. Metadata behaves
the same way: on a published article, `title` / `excerpt` / `tags` / `categoryId`
edits are staged on the draft and only go live when a human publishes.

1. **Find it.** `search_articles { q }` to locate the article and get its
   `articleId`. If several match, confirm which one with the user. (If they're
   clearly describing a brand-new topic, create a draft instead — don't shoehorn
   it into an existing article.)
2. **Read the current content.** Call `get_article { articleId }`. It returns the
   body **as Markdown** plus its metadata. Edit *that* Markdown — make the
   smallest change that satisfies the request; keep the rest byte-for-byte. Don't
   regenerate the whole article from scratch.
3. **Check for an in-progress draft.** If `get_article` returns
   `hasUnpublishedDraft: true`, the article already has unpublished draft edits
   (possibly a human's). **Warn the user and confirm before overwriting** — e.g.
   *"This article has an in-progress draft; updating it will replace those
   unpublished changes. Go ahead?"* Only proceed once they say yes. If
   `hasUnpublishedDraft` is false, your edit simply starts a new draft from the
   published content — safe to proceed.
4. **Ground new claims in the code,** exactly as in Step 1 — real labels, paths,
   flags. Reuse the supported blocks from Step 4 (including the feature card, if
   the article is about a feature and the user wants one). If the revision is
   substantial, consider **Cross-linking related articles** too.
5. **Write it back.** Call `update_article { articleId, markdown, title?,
   excerpt?, tags?, categoryId? }` with the full revised body. Pass `title` /
   `tags` / `categoryId` only if they should change; omit them to leave them as
   they are. The edit lands on the article's **draft version** (a new draft is
   created from the published content if there wasn't one) and returns
   `{ articleId, versionId, status: 'draft', adminUrl, ignoredTags }`. On an
   already-PUBLISHED article, `title` / `categoryId` changes are staged on the draft
   too and only go live when a human publishes, so the public title and URL stay
   unchanged until then. `tags` are ids/names from `list_tags`; unknown ones are
   ignored and returned in `ignoredTags`.
6. **Report back** with the `adminUrl`, and remind the user the edit is a draft
   awaiting their review and publish — the live article is unchanged until they
   publish. If `ignoredTags` came back non-empty, tell them which tags were skipped.

## Finding docs that went stale after a code change

When a diff/PR renames an API endpoint, flag, route, env var, or feature label,
existing articles that reference the OLD name silently go stale. To catch them:

1. Pull the changed names from the diff (real symbols only — e.g.
   `/api/v2/users`, `FLAG_BETA_MODE`, a renamed UI label).
2. Call `find_articles_for_symbols { symbols: [...] }`. It returns, per symbol,
   the articles that mention it plus a `matchSnippet` for context.
3. Summarise what you found and **offer to update** the affected articles — for
   each, run the read-modify-write flow in **Updating an existing article** (the
   edits land as drafts, never published). Don't rewrite anything without the
   user's go-ahead; some mentions are legitimately about the old name (e.g. a
   migration guide).

This is discovery only — it never changes anything itself.

## Finding topics to write

When the user asks "what should I write about next?" (or you want to propose
high-value drafts), call `list_content_gaps`. It reads the company's help-center
analytics and returns two ranked signals:

- **`search_no_result`** — terms readers searched for that returned nothing (a
  missing-content signal), with how often and when last seen.
- **`article_not_helpful`** — published articles readers keep marking unhelpful,
  with the unhelpful count / ratio (a quality signal).

Optional args: `{ periodDays? (default 30), minSearchCount? (default 1),
minUnhelpfulCount? (default 2) }`. It's read-only (aggregated counts, no PII).
Use the results to propose new drafts (`create_draft_article`) for the search
gaps, or to revise weak articles (`update_article`) — always with the user's
go-ahead. Present the gaps and let them pick; don't mass-draft unprompted.

## Archiving & restoring articles

When an article is obsolete or superseded, you can retire it — but only when the
user asked or confirmed:

- **Archive:** `archive_article { articleId }` removes it from the live knowledge
  base while **keeping its content and history** (nothing is hard-deleted). Good
  for "this is replaced by <other article>".
- **Restore:** `unarchive_article { articleId }` brings an archived article back
  as a **draft** (a human publishes it again to make it live) — use it to undo an
  accidental archive.

Both are reversible and content-preserving, but archiving does change what
readers see (the article leaves the live KB), so treat it as a deliberate action
and confirm first. You can never publish or hard-delete.

## Managing categories

You can **create** and **rename/edit** categories — but a category is LIVE
structure, not a draft: a new or renamed category takes effect immediately (a
rename even changes the category's public URL). So treat these as deliberate
actions:

- **Only when asked or confirmed.** Never invent, rename, or re-parent a category
  on your own. Create one only when the user wants a home for an article that has
  no good fit, or explicitly asks — and confirm the **name** and the **knowledge
  base** with them first.
- **Create:** `create_category { name, knowledgeBaseId?, description?, parentId? }`.
  It's created **unpublished** (you cannot publish it — a human does that), so a
  new category won't appear on the public help center until they publish it. Use
  the returned `categoryId` as the article's `categoryId`.
- **Edit:** `update_category { categoryId, name?, description?, parentId? }` to
  rename, retitle, or re-nest. Because a rename changes the public URL, say what
  you're about to change and get a yes first.
- **Out of scope:** you can't delete categories, reorder them, or change their
  publish state. Leave those to a human in HelpBento. (You *can* create/update
  knowledge bases — see below.)

## Managing knowledge bases

You can **create** and **update** knowledge bases too — but a KB is LIVE structure
(no draft state) and requires **owner/admin** rights, so treat these as deliberate,
confirmed actions:

- **Create:** `create_knowledge_base { name, type?, slug?, visibility?, description? }`.
  For help-center articles, `type` defaults to `help_center` (omit it). Create one
  only when the user wants a new knowledge base and confirmed its name. Every KB
  counts against the plan's KB quota. Use the returned `knowledgeBaseId` when
  authoring. (For a `developer_docs` or `changelog` KB, prefer the `draft-dev-docs`
  / `draft-changelog` skills.)
- **Update:** `update_knowledge_base { knowledgeBaseId, name?, slug?, visibility?, description? }`.
  A slug change moves the KB's public URL — say what you're changing and get a yes
  first. A KB's **type cannot change**, and you can't delete a KB — those stay with
  a human in HelpBento.
