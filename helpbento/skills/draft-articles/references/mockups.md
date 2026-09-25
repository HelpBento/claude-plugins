# Generating UI mockups for article images

This guide tells you WHEN you may hand-draw an SVG mockup of the product you are
documenting, HOW to draw one in the product's own style, and how to host it in
HelpBento with `upload_image` so it can be embedded in an article, a
developer-docs page, or a changelog entry.

## What you're making

A clean SVG that looks like a real screen of the app being documented, drawn from
that app's own design tokens (colors, radii, font) and its real labels. You write
the SVG markup yourself, then pass it as plain text to `upload_image`, which
sanitizes it, stores it in the article's (or changelog entry's) media folder, and
returns a short URL to embed.

## Tell the user: a mockup is only as good as the model drawing it

A mockup is not a screenshot. You, the AI model in this session, draw it by hand
as SVG, so its accuracy and polish depend entirely on how capable that model is.
Before drawing any, make sure the user knows this: say it when you offer or
confirm mockups (e.g. "These are SVG mockups I draw myself rather than
screenshots, so they're only as good as the model you're running; you can review
or replace them in the HelpBento editor"), and repeat it in your final report
next to the list of mockups you made.

- Mockups need a frontier-tier model (Claude Opus-class or your provider's most
  capable model). A smaller or faster model's mockups come out visibly off: tell
  the user so, and default to text-only articles unless they still want mockups.
- Always offer a text-only option. Articles render fine with no images.

## The faithfulness gate: check BEFORE deciding to draw

A mockup depicts a real screen of the app, so you may only draw one when ALL of
these are true:

1. You can read the app's source code in this session, and you have READ the
   screen's actual template/markup (the component HTML/JSX/template file for
   that route). Inferring the screen from its route name, the feature name, or
   general knowledge of what such screens usually look like does not pass. A chat
   with no access to the codebase never passes: offer to upload a real
   screenshot instead.
2. You can list, verbatim from that template, the real labels you will draw: nav
   items, button text, field placeholders, column headers, status names.
3. You found the app's design tokens (Step A), or you are deliberately using the
   neutral default below and will say so.

If any is false, draw nothing: leave the image out and tell the user which gate
failed. A feature card with no image renders fine; a mockup of a screen that
doesn't exist as drawn misleads every reader. The same rule applies element by
element while drawing: never fill a gap with an invented control or label; leave
it out.

## When to draw one

Only when the user wants visuals and the screen passes the gate:

- **Feature cards:** when an article documents an app feature and carries a
  ` ```feature ` card, draw the feature's main screen for the card's `image:`.
- **Instructional steps:** when a "how to do X" step is clearer shown, add an
  inline image of that exact screen.
- If the app has both a light and a dark mode and the user hasn't said which,
  ask once; draw every mockup in a run in the same mode.

## Step A: Derive the visual style from the codebase

Before drawing, mine the target repo so the mockup looks like *their* product, not
a generic wireframe:

1. Find design tokens — search the codebase, in order:
   - CSS custom properties / SCSS: `**/_variables.scss`, `**/tokens.*`,
     `**/theme.*`, any `:root { --… }`.
   - Tailwind: `tailwind.config.*` (`theme.extend.colors`, `borderRadius`,
     `fontFamily`).
   - CSS-in-JS / design-system package: a `theme` object, a `tokens` export.
2. Extract and note: brand/primary color, background, surface/card, text, muted
   text, border, the radius scale, font family, shadow style.
   **Theme mode:** extract the values for the mode the user chose. Dark values
   usually live in an override block — `[data-theme="dark"]`, `.dark`,
   `data-mode`, or `@media (prefers-color-scheme: dark)` — layered over the
   `:root` defaults; "App default" means the plain `:root` values with no
   override applied. Take every color from ONE mode — a light background with
   dark-mode text (or vice versa) reads as broken.
3. Identify the screen you're depicting from the component/route the article
   documents (you already read this code). Capture REAL labels: nav items, button
   text, field placeholders, status names.
4. If you cannot find a design system, use the neutral default below and SAY SO in
   your report to the user.

Neutral default (ONLY when no tokens are found):
bg `#ffffff`, surface `#f7f7f8`, text `#1a1a1e`, muted `#6b6e76`,
border `#e5e5e9`, primary `#4f46e5`, radius 8/12px, font system sans.

## Step B: Draw the SVG (house style)

- Wrap the screen in app **window chrome** (a title/URL bar with three small dots)
  so it reads as a real screenshot.
- Use the derived palette. **Hairline** (1px) borders, the app's real radii, and
  the brand color reserved for the ACTIVE/primary element (active nav, primary
  button, the text cursor) — never as a flat page background.
- Generous whitespace. Show a representative slice, not every pixel.
- When the article is instructional, show the ONE signature interaction the step
  is about (an open menu, a selected row, a filled field) — not a dead screen.
- Keep label text short enough to fit its container (rough budget: ~7px per
  character at 14px). Truncate with … rather than overflow.
- Set an explicit `viewBox` at ~16:10 (e.g. `0 0 1600 1000`) so the feature-card
  frame does not letterbox it.
- Fonts: an SVG inside an `<img>` cannot load the page's web fonts, so set
  `font-family` to a stack that NAMES the app font with system fallbacks, e.g.
  `font-family="Nunito, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif"`.
  Text renders in the fallback — fine for most mockups. Only for high-fidelity
  instructional shots where exact type matters, embed a subset font as a base64
  `@font-face` or convert text to `<path>` (heavier; off by default).

## Step C: Fidelity by purpose

- **Decorative / spotlight** (a top-of-article feature card setting the vibe):
  more abstract and brand-forward. Abstraction means showing LESS — crop to a
  representative slice, drop secondary labels, simplify shapes. Every element
  you DO show must exist on the real screen; a decorative purpose is never a
  license to invent controls, labels, or layout.
- **Instructional / demonstrative** ("here's how to do X"): faithful
  near-screenshot — accurate layout, REAL labels, and the actual control the
  reader must act on.

## Step D: Check it, then upload it

1. **Faithful:** every labeled element in the SVG (nav item, button, field,
   heading, status) traces back to the template file you read. Delete any
   element you cannot point to in the code, or, if that guts the drawing, drop
   the mockup entirely.
2. **Well-formed:** ONE `<svg>` root with a `viewBox`, valid XML (close every
   tag, escape `&` as `&amp;` and `<` as `&lt;` in text). Keep it lean: a mockup
   is usually 5-20 KB; the hard limit is 1 MB.
3. **Look at it** (best-effort): if your environment can render SVG to PNG
   (`rsvg-convert`, `cairosvg`, `qlmanage`, a headless browser), render it and
   look at the result before uploading. Fix clipped or overflowing text.
4. **Upload:** call `upload_image` with the SVG markup as plain text in `svg`,
   plus exactly one target:
   - `{ articleId, svg }` for an article or developer-docs page;
   - `{ changelogEntryId, svg }` for a changelog entry.

   The target must already exist. For a NEW article: create the draft first
   (text only, no image lines), upload each mockup against the returned
   `articleId`, then call `update_article` with the full Markdown including the
   image URLs.
5. **Read the result.** It returns `{ path, url, removed?, notice }`.
   - `removed` lists anything the sanitizer stripped: scripts, event handlers,
     `foreignObject`, external links or `url(…)`s. Only local `#fragment`
     references and embedded `data:` images/fonts survive. If the drawing
     relied on something removed, fix the SVG and upload again.
   - `notice` is the reminder that the mockup is model-drawn. Relay it to the
     user in your report.
   - An error means the SVG was rejected (not well-formed, no `viewBox`, too
     large). Fix it and retry, or leave the image out.
6. **Fail-safe:** if you cannot get a clean upload, embed NO image rather than
   a broken one, and say so in your report.

## Step E: Embed it

- Feature card: put the returned `url` on the `image:` line of the
  ` ```feature ` block:

  ````
  ```feature
  icon: layers
  eyebrow: Knowledge base
  title: The Article Editor
  image: https://storage.googleapis.com/help-bento-public/companies/…/articles/…/….svg

  A distraction-free, block-based editor for writing help articles.
  ```
  ````

- Inline instructional step: `![alt](url)` on its OWN line:

  ```
  ![The article editor with the slash menu open](https://storage.googleapis.com/help-bento-public/…/….svg)
  ```

Always write meaningful `alt` text: it is read by screen readers and shown if the
image fails to load.

## Report back

For each mockup, tell the user which screen it shows, the fidelity used
(decorative or instructional), and whether the style came from the repo's tokens
or the neutral default. Remind them the mockups were drawn by the model, are only
as good as that model, and should be checked against the real screens in the
HelpBento editor before publishing.

## Worked example

This SVG was drawn for HelpBento's own article editor, using the real tokens
from HelpBento's `src/styles/_variables.scss` — brand violet `#8b5cf6`, warm-paper `#fafaf8`,
hairline `#e2e2de`, 10–12px radii, Nunito. ~8 KB of SVG.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1600 1000" font-family="Nunito, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif">
  <defs>
    <clipPath id="win"><rect x="0" y="0" width="1600" height="1000" rx="16"/></clipPath>
    <filter id="pop" x="-30%" y="-30%" width="160%" height="160%">
      <feDropShadow dx="0" dy="10" stdDeviation="18" flood-color="#14151a" flood-opacity="0.14"/>
    </filter>
  </defs>
  <g clip-path="url(#win)">
    <rect x="0" y="0" width="1600" height="1000" fill="#ffffff"/>
    <rect x="0" y="0" width="1600" height="56" fill="#fafaf8"/>
    <line x1="0" y1="56" x2="1600" y2="56" stroke="#e2e2de" stroke-width="1"/>
    <circle cx="30" cy="28" r="6" fill="#dcdcd6"/>
    <circle cx="54" cy="28" r="6" fill="#dcdcd6"/>
    <circle cx="78" cy="28" r="6" fill="#dcdcd6"/>
    <rect x="600" y="15" width="400" height="26" rx="13" fill="#f2f2ee"/>
    <text x="800" y="32" text-anchor="middle" font-size="13" fill="#9a9ca2">app.helpbento.com/admin/articles/new</text>
    <rect x="0" y="56" width="300" height="944" fill="#fafaf8"/>
    <line x1="300" y1="56" x2="300" y2="1000" stroke="#e2e2de" stroke-width="1"/>
    <rect x="24" y="82" width="32" height="32" rx="9" fill="#8b5cf6"/>
    <rect x="30" y="88" width="8.5" height="8.5" rx="2" fill="#ffffff" opacity="0.95"/>
    <rect x="41.5" y="88" width="8.5" height="8.5" rx="2" fill="#ffffff" opacity="0.7"/>
    <rect x="30" y="99.5" width="8.5" height="8.5" rx="2" fill="#ffffff" opacity="0.7"/>
    <rect x="41.5" y="99.5" width="8.5" height="8.5" rx="2" fill="#ffffff" opacity="0.95"/>
    <text x="68" y="105" font-size="17" font-weight="700" fill="#14151a">HelpBento</text>
    <g>
      <rect x="30" y="161" width="18" height="18" rx="3" fill="none" stroke="#9a9ca2" stroke-width="1.6"/>
      <line x1="30" y1="170" x2="48" y2="170" stroke="#9a9ca2" stroke-width="1.6"/>
      <line x1="39" y1="161" x2="39" y2="179" stroke="#9a9ca2" stroke-width="1.6"/>
      <text x="60" y="179" font-size="14.5" fill="#6b6e76">Dashboard</text>
    </g>
    <rect x="16" y="200" width="268" height="46" rx="8" fill="#8b5cf6" fill-opacity="0.10"/>
    <rect x="30" y="211" width="15" height="18" rx="2.5" fill="none" stroke="#6d28d9" stroke-width="1.6"/>
    <line x1="34" y1="217" x2="41" y2="217" stroke="#6d28d9" stroke-width="1.6"/>
    <line x1="34" y1="221" x2="41" y2="221" stroke="#6d28d9" stroke-width="1.6"/>
    <text x="60" y="229" font-size="14.5" font-weight="600" fill="#6d28d9">Articles</text>
    <g>
      <circle cx="33" cy="271" r="2" fill="#9a9ca2"/><line x1="40" y1="271" x2="48" y2="271" stroke="#9a9ca2" stroke-width="1.6"/>
      <circle cx="33" cy="279" r="2" fill="#9a9ca2"/><line x1="40" y1="279" x2="48" y2="279" stroke="#9a9ca2" stroke-width="1.6"/>
      <circle cx="33" cy="287" r="2" fill="#9a9ca2"/><line x1="40" y1="287" x2="48" y2="287" stroke="#9a9ca2" stroke-width="1.6"/>
      <text x="60" y="279" font-size="14.5" fill="#6b6e76">Categories</text>
    </g>
    <g>
      <line x1="32" y1="329" x2="32" y2="319" stroke="#9a9ca2" stroke-width="2.4"/>
      <line x1="39" y1="329" x2="39" y2="313" stroke="#9a9ca2" stroke-width="2.4"/>
      <line x1="46" y1="329" x2="46" y2="323" stroke="#9a9ca2" stroke-width="2.4"/>
      <text x="60" y="329" font-size="14.5" fill="#6b6e76">Analytics</text>
    </g>
    <g>
      <line x1="30" y1="373" x2="48" y2="373" stroke="#9a9ca2" stroke-width="1.6"/>
      <line x1="30" y1="385" x2="48" y2="385" stroke="#9a9ca2" stroke-width="1.6"/>
      <circle cx="41" cy="373" r="3" fill="#fafaf8" stroke="#9a9ca2" stroke-width="1.6"/>
      <circle cx="36" cy="385" r="3" fill="#fafaf8" stroke="#9a9ca2" stroke-width="1.6"/>
      <text x="60" y="379" font-size="14.5" fill="#6b6e76">Settings</text>
    </g>
    <line x1="16" y1="928" x2="284" y2="928" stroke="#ecece8" stroke-width="1"/>
    <circle cx="38" cy="958" r="15" fill="#8b5cf6" fill-opacity="0.14"/>
    <text x="38" y="963" text-anchor="middle" font-size="12.5" font-weight="700" fill="#6d28d9">AL</text>
    <text x="62" y="954" font-size="13.5" font-weight="600" fill="#14151a">Ada Lovelace</text>
    <text x="62" y="970" font-size="11.5" fill="#9a9ca2">Admin</text>
    <line x1="300" y1="118" x2="1600" y2="118" stroke="#ecece8" stroke-width="1"/>
    <text x="332" y="93" font-size="14" fill="#9a9ca2">Articles</text>
    <text x="398" y="93" font-size="14" fill="#cfcfca">/</text>
    <text x="412" y="93" font-size="14" fill="#6b6e76">New article</text>
    <text x="1150" y="93" font-size="13" fill="#9a9ca2">Syncing…</text>
    <circle cx="1132" cy="88" r="4" fill="none" stroke="#b9a4f3" stroke-width="2" stroke-dasharray="14 6"/>
    <rect x="1230" y="71" width="160" height="34" rx="8" fill="#ffffff" stroke="#e2e2de" stroke-width="1"/>
    <text x="1246" y="93" font-size="13.5" fill="#9a9ca2">Select category</text>
    <path d="M1372 86 l5 5 l5 -5" fill="none" stroke="#9a9ca2" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/>
    <rect x="1402" y="71" width="104" height="34" rx="8" fill="#ffffff" stroke="#e2e2de" stroke-width="1"/>
    <circle cx="1420" cy="88" r="4" fill="#b6b8bd"/>
    <text x="1432" y="93" font-size="13.5" font-weight="600" fill="#14151a">Draft</text>
    <path d="M1486 86 l5 5 l5 -5" fill="none" stroke="#9a9ca2" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/>
    <circle cx="1530" cy="88" r="9" fill="none" stroke="#9a9ca2" stroke-width="1.6"/>
    <circle cx="1530" cy="88" r="3" fill="#9a9ca2"/>
    <circle cx="1568" cy="88" r="10" fill="none" stroke="#9a9ca2" stroke-width="1.6"/>
    <path d="M1568 82 v6 l4 3" fill="none" stroke="#9a9ca2" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/>
    <text x="556" y="286" font-size="22" fill="#cfcfca">+</text>
    <circle cx="540" cy="276" r="1.6" fill="#cfcfca"/><circle cx="546" cy="276" r="1.6" fill="#cfcfca"/>
    <circle cx="540" cy="282" r="1.6" fill="#cfcfca"/><circle cx="546" cy="282" r="1.6" fill="#cfcfca"/>
    <circle cx="540" cy="288" r="1.6" fill="#cfcfca"/><circle cx="546" cy="288" r="1.6" fill="#cfcfca"/>
    <text x="590" y="214" font-size="40" font-weight="650" fill="#b3b5ba">Untitled</text>
    <text x="590" y="290" font-size="19" fill="#14151a">/</text>
    <rect x="600" y="272" width="2" height="24" fill="#8b5cf6"/>
    <g filter="url(#pop)">
      <rect x="590" y="312" width="356" height="252" rx="10" fill="#ffffff" stroke="#e2e2de" stroke-width="1"/>
    </g>
    <text x="608" y="338" font-size="11" letter-spacing="0.6" font-weight="700" fill="#9a9ca2">BASIC</text>
    <rect x="600" y="348" width="336" height="46" rx="8" fill="#8b5cf6" fill-opacity="0.10"/>
    <rect x="610" y="357" width="28" height="28" rx="6" fill="#ffffff" stroke="#e2e2de"/>
    <text x="624" y="376" text-anchor="middle" font-size="13" font-weight="700" fill="#6b6e76">T</text>
    <text x="650" y="369" font-size="14" font-weight="600" fill="#14151a">Text</text>
    <text x="650" y="385" font-size="11.5" fill="#9a9ca2">Plain paragraph</text>
    <rect x="610" y="405" width="28" height="28" rx="6" fill="#ffffff" stroke="#e2e2de"/>
    <text x="624" y="424" text-anchor="middle" font-size="11.5" font-weight="700" fill="#6b6e76">H1</text>
    <text x="650" y="417" font-size="14" fill="#14151a">Heading 1</text>
    <text x="650" y="433" font-size="11.5" fill="#9a9ca2">Large section heading</text>
    <rect x="610" y="453" width="28" height="28" rx="6" fill="#ffffff" stroke="#e2e2de"/>
    <text x="624" y="472" text-anchor="middle" font-size="11.5" font-weight="700" fill="#6b6e76">H2</text>
    <text x="650" y="465" font-size="14" fill="#14151a">Heading 2</text>
    <text x="650" y="481" font-size="11.5" fill="#9a9ca2">Medium section heading</text>
    <rect x="610" y="501" width="28" height="28" rx="6" fill="#ffffff" stroke="#e2e2de"/>
    <circle cx="619" cy="515" r="2" fill="#6b6e76"/><line x1="625" y1="515" x2="632" y2="515" stroke="#6b6e76" stroke-width="1.6"/>
    <circle cx="619" cy="521" r="2" fill="#6b6e76"/><line x1="625" y1="521" x2="632" y2="521" stroke="#6b6e76" stroke-width="1.6"/>
    <text x="650" y="513" font-size="14" fill="#14151a">Bulleted list</text>
    <text x="650" y="529" font-size="11.5" fill="#9a9ca2">Unordered list</text>
  </g>
  <rect x="0.5" y="0.5" width="1599" height="999" rx="16" fill="none" stroke="#e2e2de" stroke-width="1"/>
</svg>
```

Token mapping used:
- `--primary #8b5cf6` → active-nav tint (10%), brand mark, text cursor, slash-item highlight; `#6d28d9` for active text/icon.
- `--background #fafaf8` → sidebar + chrome bar; `--surface-raised #ffffff` → editor canvas, popover, pills.
- `--text #14151a` / `--text-muted #6b6e76` → labels; placeholder via the muted↔surface mix (`#b3b5ba`).
- `--border #e2e2de` / `--border-muted #ecece8` → hairlines.
- radii 10–12px → cards/popover; 6–8px → pills/controls.
