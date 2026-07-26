---
name: ploxs-presentations-test
description: Create, inspect, and edit designed Google Slides presentations through the Ploxs TEST MCP server, including authoring slide HTML yourself from a Ploxs style config. Use when the user asks to make a presentation or slide deck, apply a brand style to slides, design slides by hand as HTML frames, list or inspect their Ploxs/Google Slides decks, or edit slides.
---

# Ploxs Presentations (Test)

Ploxs turns notes, URLs, document text, and tabular data into designed Google Slides
presentations, edits those decks slide-by-slide, and — on this test server — converts
slide HTML **you** author into a deck in the user's Google Drive.

All work happens through the Ploxs MCP tools on the isolated test deployment
(server `ploxs-test` at `https://test.ploxs.com/mcp`). This is a testing environment:
accounts, credits, styles, and decks are separate from production `ploxs.com`. Never
mix the two in one conversation.

## Before anything else

1. Call `get_account_status`. Check:
   - `google.driveConnected` — must be true before creating or editing Google Slides.
     If not, send the user to the `links.settings` / `links.googleDrive` URL from the
     response and wait for them to connect.
   - `billing` / `modelTiers` — if credits are insufficient, calls fail with an
     entitlement error; surface the message and stop.
2. Everything is asynchronous. Creation returns a `jobId`; edits return a `task`.
   Wait with `wait_for_presentation` (jobs) or `wait_for_presentation_edit` (tasks)
   before a dependent change or before telling the user it's done.

## Settle the topic before you build

A style request is not a content request. "Make me a deck in Amazon's style" says
nothing about what the slides say. If the subject is genuinely unspecified, ask one
short question and stop — do not guess a topic and burn a deck job on it.

Once the topic is known, decide the slide count yourself unless the user fixed one.
Let the content decide: one idea per slide, and a slide for every idea that carries
its own evidence.

## Two ways to build a deck — choose deliberately

**A. Ploxs generates the slides** (`create_presentation`) — the default when the user
hands over raw material and wants Ploxs to do the design: notes to expand, URLs to
extract, CSVs to chart, generated illustrations, or "just make me a deck about X".

**B. You author the slides as HTML frames** (`create_presentation_from_html`) — use
this when the design or content should come from you:

- The user wants specific layouts, structures, or wording you already produced.
- The content is in your context: a codebase, a report you wrote, analysis you ran,
  file contents, benchmark tables, architecture summaries.
- The user iterates on the markup ("make the second slide two columns", "use my exact
  numbers", "keep this table intact").
- The deck must be reproducible from files in a repo.

Frames are converted **exactly as authored** — no regeneration, no restyling, no
model calls. What you submit is what the user gets. Say which mode you are using.

## Choosing a style (do this first in both modes)

Exactly one style source per call — combining them errors (`style_choice_conflict`):

- `style_config_id` — preferred. Call `list_style_configs` (optionally with `query`)
  and pick a saved (`manual`), uploaded (`upload`), or recent-deck (`deck:` prefixed)
  style. Reuse the user's saved brand style whenever one exists.
- `style_config` — inline JSON when the user supplies custom branding. A complete
  config needs `style` (prose description), `colorPalette` with all 7 hex colors
  (primary, secondary, accent, background, surface, text, textMuted), and
  `typographyConfig` (headingFont, bodyFont, googleFontsImport). Check it with
  `validate_style_config` first. Put durable brand rules in `brandGuidelines`
  (dos/donts, colorUsage, typographyUsage, imageryUsage, chartUsage) and personality
  in `customStyleDirective` — those persist into later edits and images.
- `auto_style: true` — last resort, and only for mode A. Say explicitly that Ploxs
  will invent the visual style. Never use it in mode B: frames must be written
  against a style that already exists.

**A `deck:` style's name describes the deck it came from, not how it looks.** A style
listed as "The Amazon Flywheel" turned out to carry a pink "Cherry Blossom" palette.
Before reusing any `deck:` style, read its returned `colorPalette` and check the hexes
actually match what the user is asking for. If they don't, build an inline
`style_config` instead — a name match is not a brand match.

When you pass an inline config, `get_html_frame_spec` echoes back the palette it
accepted. Read those hexes and confirm they are yours before authoring against them.

## Mode A — Ploxs generates the deck

`create_presentation` with the content you have: `markdown` for notes/outline text,
`urls` (max 10), `file_texts` (max 20), `csv_sources` for tabular data. Optional:
`instructions` (tone/audience/emphasis, max 4000 chars), `slide_count` (1–60),
`export_to_google: false` to keep the deck on Ploxs only.

Then `wait_for_presentation` with the returned `job_id` (default timeout 240s; if it
returns `timedOut: true`, wait again rather than assuming failure). On completion give
the user the `editUrl`/`viewUrl` and **keep the `deckRef`** — every edit tool needs it.
Creation is idempotent for ~10 minutes, so retrying a failed call won't double-charge.

Daily creation limits apply (25/day wallet accounts, 100/day subscribers);
`daily_job_limit` is retryable tomorrow, not now.

## Mode B — author HTML frames yourself

The whole job is **content** and **design**. Get the facts right, then pour them into
a fixed design kit. Everything below is arranged to keep those two things separate,
because mixing them is what makes frame authoring slow.

### 1. Get the spec (mandatory)

Call `get_html_frame_spec` with the style you chose (`style_config_id` or
`style_config`). It returns, for that exact style:

- `stage` — the fixed canvas (1280×720) and the safe content area after the deck's
  margins. Design to those numbers.
- `colorPalette`, `typography` — the exact hex values and font families to use. The
  fonts are already loaded by the deck; never import them yourself.
- `briefs.layout` / `briefs.style` / `briefs.brand` — the deck's type scale, margins,
  alignment policy, archetypes, and brand rules.
- `contract` — the binding HTML rules. **Follow it literally**; it is the same
  contract Ploxs' own slide painter works under.
- `overlays` — present only when the style paints persistent overlay images.
  Each mandate lists rectangles (as fractions of the slide) that are reserved on the
  slides it applies to, and which slides those are (every slide, or first/middle/last).
  Keep **all** content — text, images, charts, cards, borders, accents — fully outside
  them: overlays are composited after conversion, so anything underneath is covered.
  Never draw the overlay artwork yourself.
- `charts` — the required Chart.js protocol plus a copy-ready widget. See
  **Charts** below.
- `parallelAuthoring` — the deck-at-once protocol, same as steps 3–4 below.
- `example` — a worked frame showing token usage (and, by omission, showing that
  frames declare no fonts).
- `limits` — max frames and per-frame size.

Do not skip this and guess. Colors, fonts, and type sizes invented by you will look
wrong next to the user's other decks.

### 2. Lock the content before you write any HTML

Frames are cheap to lay out and expensive to re-do when a number is wrong. Separate
the two passes.

**Build a fact table first.** One row per figure that will appear anywhere in the
deck: value, what it measures, the period it covers, the source, and a status of
`actual` / `guidance` / `estimate` / `stale`. Research once, into this table; do not
research per slide.

Then hold the line on three rules:

- **Every number on a slide comes from the table.** If it is not in the table, it does
  not go on a slide. This is also what makes the chart rule ("only numbers present in
  the slide's own content") automatic instead of a thing to police.
- **Anything not `actual` gets labelled on the slide itself**, in the axis label,
  caption, or badge — `2026 (guided)`, `est.`, `as of 2021`. A caveat you deliver in
  chat afterwards does not travel with the deck, and the deck is the artifact.
- **Mixed periods must be visibly distinguished.** A full-year figure and a Q4 figure
  sitting in the same card row will be read as the same basis unless each card says
  which it is.

Then write the outline: one line per slide with its number, name, the facts it
carries, and the archetype from `briefs.layout`. Prose is not an outline — "what this
slide asserts + which rows of the fact table prove it" is.

### 3. Fix the design kit once, then stop designing

The single biggest waste in frame authoring is writing a bespoke stylesheet per
frame. Decide the kit once, before any frame, and reuse it verbatim:

- the px size for each semantic role, **copied from `briefs.layout`** — never
  re-derived per slide;
- the spacing scale;
- which palette role paints what, and what the one accent per slide is;
- the card/panel treatment (surface color, radius, padding).

**Use the same class names in every frame.** Ploxs auto-suffixes class names per
frame, so `.card` in frame 2 and `.card` in frame 7 cannot collide. Per-frame prefixes
like `.f1-card` / `.f2-card` buy nothing and multiply every selector you emit. A small
fixed vocabulary — `.wrap`, `.eyebrow`, `.h`, `.rule`, `.lede`, `.grid`, `.card`,
`.stat`, `.icon`, `.note` — covers most decks. Each frame then differs only in its
markup and a few overrides, not in a whole restated stylesheet.

**Declare no `font-family` at all.** The deck document already loads the style's fonts
and applies the body font to everything plus the heading font to `h1`–`h6`, so a frame
inherits the correct font with zero declarations. Only write `font-family` to deviate
deliberately — monospace inside `<code>`/`<pre>`. Restating the stack per selector is
the single largest source of wasted frame CSS. `spec.typography.note` says the same, and
both spec examples are written this way.

### 4. Author every frame in one pass

Ploxs' own slide painter builds every slide concurrently, held together by the shared
style kit plus the outline — never by showing one slide the previous slide's markup.
Author the same way. Frame-by-frame across turns is slower and no more consistent,
because each turn re-derives decisions that step 3 already made.

- **With no subagent or task tool — which is the common case — one turn is the
  parallel path.** Emit all frames back-to-back in a single message. Do not spend a
  turn per slide, and do not simulate parallelism by splitting the deck across turns.
- **With subagents**, give each worker one frame or a small group. Send them the
  step-2 fact table rows for their slide, the step-3 design kit, the outline, and any
  `overlays.mandates` for their slide position. Send the **full** `contract` and
  `charts` block only to a worker whose slide has a chart; for pure-markup frames the
  distilled kit is enough, and shipping the whole contract to every worker costs more
  than it protects.
- **Reconcile before submitting**: same role → same px size everywhere, one accent
  focal point per slide, consistent panels, no frame depending on another's classes.

If you are working in a repo, write each frame to a file (`slides/01-title.html`, …)
with parallel writes in one message, and open one at 1280×720 to eyeball it. Files
make revisions cheap and give the user something to review.

### 5. Write one frame per slide

Each frame is the **inner HTML of one slide**. Ploxs wraps it in
`<div id="frame-<n>" class="slide">` and centers it on the stage. Non-negotiables
(the full list is in `contract`):

- One top-level element per frame, no `id` attribute on it, no `<!DOCTYPE>`/`<html>`/
  `<head>`/`<body>`.
- All CSS in one inline `<style>` inside the frame, scoped to classes that exist in
  that frame's markup. No `#id` selectors.
- No `<link>`, no `@import`, no `<script src>`, no Tailwind/Bootstrap/CDN utility
  classes. Nothing loads at render time.
- Sizes in px chosen for 1280×720. No `vw`/`vh`/`dvh`, no `@media`, no viewport
  `clamp()`.
- No animation, transition, or `@keyframes` — they are stripped, so anything that
  depends on them renders in its start state.
- Grid tracks use `minmax(0, 1fr)`; text children set `min-width: 0`. Content must
  fit the safe area with zero overflow — overflow shrinks the whole exported slide.
- Don't set `background-color` on the root: the deck already paints
  `colorPalette.background`. Use `colorPalette.surface` for cards and panels.
- Icons only via `__ICON_<keywords>__` placeholders (e.g.
  `<span class="icon">__ICON_shield-check__</span>`), replaced server-side with real
  SVG that inherits `color`. Never hand-draw SVG or use emoji as icons.
- Images: `<img>` with a `data:` URI or a publicly reachable https URL, always
  `object-fit: contain`, never fixed width *and* height together.
- Charts: one inline `<script>` per frame, only for a Chart.js chart, and only
  following the protocol in **Charts** below. Never fabricate data.

### 6. Charts

A chart is the one place a frame runs JavaScript, and it has a **required protocol**.
The reason: a `<canvas>` is opaque to the converter, so chart pixels only reach the deck
through a pre-render pass that captures each canvas by id and swaps in a PNG. Every way
of getting this wrong fails *silently* — the slide exports with an empty box.

`spec.charts` gives you `rules`, the pinned `libraryUrl`, and a complete working
`example`. **Copy the example's plumbing verbatim and change only the data, chart type,
labels, and colors.** Don't write the loader from memory. The four things that must be
right:

1. **Unique `<canvas id="chart-…">`** — capture is keyed by id. No id, no chart.
2. **Load Chart.js inside your inline script** using the example's parallel-safe
   pattern (`typeof Chart === 'undefined'` → append a script element with
   `spec.charts.libraryUrl`). A bare `<script src="…">` tag is rejected by validation.
3. **`ctx.dataset.initialized = 'true'`** before `new Chart(...)` — the readiness signal
   the exporter waits on. Without it the slide burns a 5-second timeout.
4. **`animation: false`** in options, plus `responsive: true` and
   `maintainAspectRatio: true`, or the capture catches the chart half-drawn.

Then: real numbers from the fact table only; palette colors (primary for the main
series, secondary/accent for supporting, textMuted for grid lines, ticks and legend);
14–16px chart fonts so it reads when projected; `aspectRatio` per chart type from
`rules`; the chart as the frame's hero with a short heading above and at most one line
of annotation below; and never restate the chart's numbers as body text beside it.

`aspectRatio` in options is what actually governs the rendered shape. The canvas
`width`/`height` attributes are a fallback for before Chart.js takes over — set them
to the same ratio so nothing jumps, and size the holder with an explicit `max-width`
that gives the chart 400–470px of vertical space. The holder must not set
`overflow: hidden`.

Label the x-axis honestly: a projected or guided period is a different kind of number
from a reported one, and the axis tick is where that belongs (`2026 (guided)`).

`validate_html_frames` reports the protocol failures — `canvas_missing_id`,
`chart_loader_missing`, `chart_without_canvas` and `canvas_without_chart` are errors;
`chart_ready_flag_missing` and `chart_animation_enabled` are warnings you should still
fix. Treat any chart warning as a defect: the chart will not export as intended.

If a slide's data doesn't genuinely need a chart, express it as markup — a stat row, a
comparison table, a labelled bar built from divs — which converts to native, editable
Google Slides shapes instead of a flat image.

### 7. Validate — and know what a warning is worth

One `validate_html_frames` call with **all** the deck's frames — not one call per frame.
It's free: no credits, no job.

- `errors` block submission: fix them and re-validate once.
- `warnings` never block. `create_presentation_from_html` accepts the frames and
  returns the same warning list.

**Validation earns its round trip on chart frames**, where the failures are silent and
cost a whole job. For a deck of pure markup built from a kit you have already
validated once, going straight to submit is reasonable — the create call reports the
same warnings anyway.

**Read a warning against your own markup before acting on it.** Every warning now
quotes the thing it matched — `id_selector` names the offending selector, for instance.
If you cannot find that thing in your frame, the warning is not about your frame: note
it and move on rather than spending a round trip rewriting on spec.

(An earlier version of this server reported `id_selector` on frames that used no id
selector at all — it was matching hex colours written after a space, like
`border: 1px solid #D1D5DB`. That is fixed. If you meet a warning whose condition you
genuinely cannot find, say so plainly instead of guessing at a workaround.)

Fix every genuinely reported frame in one pass — the same mistake almost always
repeats across frames written together.

### 8. Submit

`create_presentation_from_html` with:

- `frames` — array of `{ name, html }` in slide order. Frame 1 becomes slide 1.
  `name` is a short slide title used in the deck outline.
- `title` — deck title, also the Google Slides file name. Pass plain text: write
  `Retail & Cloud`, not `Retail &amp; Cloud`. (The server now decodes escaped entities
  in the title and slide names, so a slip no longer leaks `&amp;` into the Drive file
  name — but don't rely on it.)
- the **same** `style_config_id` / `style_config` you passed to
  `get_html_frame_spec` (required — it drives deck fonts, background, and the
  Slides conversion).
- optional `instructions` — intent/audience notes stored with the deck so later edits
  stay consistent. It does not change the HTML you submitted. Good place for the
  provenance of your fact table, so later edits inherit the same sourcing.
- optional `export_to_google: false` to build the deck without the Drive upload.

Then `wait_for_presentation` with the returned `job_id`, exactly as in mode A. The
completed job returns `editUrl`, `viewUrl`, and a `deckRef` — a frame-authored deck is
a normal Ploxs deck, so every editing tool below works on it in place. **Keep that
`deckRef`: it is the only way to change this deck** (see step 9).

Conversion runs the browser render + native PPTX path, so it is quick (~30–90s) and
consumes no text-generation credits, but it does require Drive linked and a billable
account.

### 9. Revising — frames build a deck, they never update one

**HTML conversion happens exactly once, at the initial export. Every change after that
goes through the Ploxs editing tools on the deck's `deckRef`.**

This is not a preference. `create_presentation_from_html` has no update mode: it queues
a fresh job, which builds a fresh deck and a **new Google Slides file**. Resubmitting
edited frames does not modify the deck the user already has — it leaves them with two
decks in Drive and a link that no longer points at the current one.

So once the deck exists:

- Use `edit_slide` to rewrite or redesign a slide, `add_slides` to insert, and
  `add_image_to_slide` / `add_infographic_to_slide` for assets; batch several with
  `update_presentation`. These run the Ploxs AI harness against the live deck and edit
  it in place, on the same file and the same link.
- Fetch `get_presentation_outline` first so you target the right slide number.
- Keep your frame files as the record of what you authored, but treat them as the
  build input, not a live editing surface.

Only submit frames again when the user genuinely wants a **new** deck — a fresh build
from revised content, or a rebuild they've agreed to before the first one was shared. If
they ask for a small correction and you are tempted to resubmit, say what would happen
to their existing file first.

### HTTP alternative

The same flow exists as plain HTTP on the test server for scripts, using the same
Ploxs API key or OAuth token: `GET /mcp/html-frames/spec?style_config_id=…`,
`POST /mcp/html-frames/validate`, `POST /mcp/html-frames`,
`GET /mcp/html-frames/:jobId`. Prefer the MCP tools in conversation; mention the
routes only if the user wants automation outside this session.

## Working with existing decks

- `list_presentations` — decks known to Ploxs, each with `deckRef`, title, and URLs.
- `connect_presentation` — bind a deck Ploxs hasn't seen, from a Slides URL or ID.
  If it returns `ready: false` with an `actionUrl`, give the user that link (a Google
  file-picker authorization), wait for approval, then call it again. A deck new to
  Ploxs also requires a style (`style_config_id` or `style_config`);
  `style_config_required` means you skipped that.
- `get_presentation_outline` — slide numbers, titles, and text for a bound deck.
  **Always fetch this before targeting a slide number**; numbers are 1-based and
  resolved against the live deck.

## Editing

Each mutation queues a task; poll with `wait_for_presentation_edit(task_id)` before
the next dependent edit. At most 3 edit tasks can be active at once.

- `edit_slide` — rewrite/redesign one slide from a `prompt` (≤4000 chars). Optional
  `layout_archetype`; `use_slide_context` defaults true.
- `add_slides` — insert slides from `content` (≤30k chars). `insertion_placement`
  `before`/`after` requires `anchor_slide_number`; default is `end`.
- `add_image_to_slide` — generated image; optional `prompt`, `art_style`,
  `aspect_ratio` (`16:9|4:3|1:1|9:16|3:4`), `placement`. Give a `prompt` or leave
  `use_slide_context` on.
- `add_infographic_to_slide` — chart/infographic from `data_description`; optional
  `chart_type`, `chart_title`.
- `update_presentation` — batch of 1–20 of the above in one task. Ops run
  sequentially and each `slide_number` resolves against the deck **as it exists at
  that step** (an earlier `add_slides` shifts later numbers). The batch stops at the
  first failure and reports what completed. Prefer this over many single calls.

### Adding vs. replacing (infographics and images)

By default these two tools **add** a new chart/picture and keep any already on the
slide (`replace_existing_asset: false`), so a slide can quietly accumulate two.
Redesign of the composition is on by default (`redesign_slide: true`).

When the user's intent is to **change, update, refresh, or swap** the
infographic/image — not to place a second one — remake the slide: confirm, then pass
`replace_existing_asset: true` with `redesign_slide: true`. A good check-in is: *"This
slide already has an infographic — remake the slide with the new one (replacing the
old), or add it alongside?"* Default to replace-and-remake unless they ask for both.
The same choice applies inside an `update_presentation` batch.

## Error handling

Errors come back as text with `(code: <code>, retryable: <bool>)`.

- `google_not_linked` / `google_scope_upgrade_required` — relay the action link, wait
  for the user.
- `invalid_html_frames` / `html_frames_invalid` — the message lists the offending
  frame numbers and rules; fix the HTML and re-validate. Never retry unchanged.
- `style_config_required` — pass the style the frames were authored against.
- `presentation_not_connected` — run `connect_presentation` first.
- `slide_not_found` — the message includes the real slide count; re-fetch the outline
  and re-target.
- `active_job_limit` / `rate_limited` — wait for running work, then retry.
- `invalid_style_config` — the message lists missing fields; fix and revalidate.
- Non-retryable entitlement/credit errors — report to the user; don't retry.
- `html_frames_disabled` or missing frame tools — the deployment does not have the
  HTML frame surface enabled. Fall back to mode A and say so.

Never invent a `deckRef` or slide number: source them from
`create_presentation`/`create_presentation_from_html`/`connect_presentation`/
`list_presentations` and `get_presentation_outline`.
