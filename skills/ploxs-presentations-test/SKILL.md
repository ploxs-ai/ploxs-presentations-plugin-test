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
- `parallelAuthoring` — how to author the whole deck at once. See **Author the deck
  in parallel** below.
- `example` — a worked frame showing token usage.
- `limits` — max frames and per-frame size.

Do not skip this and guess. Colors, fonts, and type sizes invented by you will look
wrong next to the user's other decks.

### 2. Author the deck in parallel, not one slide at a time

Ploxs' own slide painter builds every slide of a deck **concurrently**, and holds the
deck together purely through the shared style contract plus the deck outline — never by
showing one slide the previous slide's markup. Author the same way. Going frame-by-frame
is slow and no more consistent, because each turn re-derives the type scale from scratch.

1. **Spec once.** One `get_html_frame_spec` call for the whole deck. Never per slide.
2. **Outline first.** One line per slide: number, name, the content it carries, and the
   archetype from `briefs.layout` it uses. This is the shared plan.
3. **Fix the shared decisions before authoring anything**: the px size for each semantic
   role (copy them from `briefs.layout` — don't re-derive per slide), the spacing scale,
   which palette role paints what, the card/panel treatment, and a per-frame class
   prefix (`.f1-…`, `.f2-…`) so no two frames collide.
4. **Fan out.** If you have subagents or parallel tasks, take one frame each (or small
   groups). Otherwise emit the frames back-to-back in a single turn, and if you're
   writing them to files, issue those writes as parallel tool calls in one message.
5. **Give every worker the whole contract, verbatim** — `contract`, `stage`,
   `colorPalette`, `typography`, `briefs`, `charts` if its slide has a chart, and any
   `overlays.mandates` for its slide position — plus the outline and the step-3
   decisions. Workers share no context: whatever you omit gets invented, and invented
   tokens are exactly what makes a deck look stitched together.
6. **Reconcile before submitting**: same role → same px size everywhere, one accent
   focal point per slide, consistent panels, no frame depending on another's classes.

Then one `validate_html_frames` call for all frames, and one
`create_presentation_from_html` for the deck.

### 3. Write one frame per slide

Each frame is the **inner HTML of one slide**. Ploxs wraps it in
`<div id="frame-<n>" class="slide">` and centers it on the stage. Non-negotiables
(the full list is in `contract`):

- One top-level element per frame, no `id` attribute on it, no `<!DOCTYPE>`/`<html>`/
  `<head>`/`<body>`.
- All CSS in one inline `<style>` inside the frame, scoped to classes that exist in
  that frame's markup. No `#id` selectors — class names are auto-suffixed per frame,
  so two frames can both use `.card` safely.
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

Practical workflow in a repo: write each frame to a file (`slides/01-title.html`,
`slides/02-metrics.html`, …), open one in a browser sized to 1280×720 to eyeball it,
then submit the files' contents in order. Keeping frames on disk makes revisions and
re-submissions cheap, and gives the user something to review.

### 4. Charts

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

Then: real numbers from the slide's content only; palette colors (primary for the main
series, secondary/accent for supporting, textMuted for grid lines, ticks and legend);
14–16px chart fonts so it reads when projected; `aspectRatio` per chart type from
`rules`; the chart as the frame's hero with a short heading above and at most one line
of annotation below; and never restate the chart's numbers as body text beside it.

`validate_html_frames` reports all of these — `canvas_missing_id`,
`chart_loader_missing`, `chart_without_canvas` and `canvas_without_chart` are errors;
`chart_ready_flag_missing` and `chart_animation_enabled` are warnings you should still
fix. Treat any chart warning as a defect: the chart will not export as intended.

If a slide's data doesn't genuinely need a chart, express it as markup — a stat row, a
comparison table, a labelled bar built from divs — which converts to native, editable
Google Slides shapes instead of a flat image.

### 5. Validate before submitting

One `validate_html_frames` call with **all** the deck's frames — not one call per frame.
It's free: no credits, no job.

- `errors` block submission: fix them and re-validate once.
- `warnings` are accepted but tell you the converter will strip or ignore something.
  Fix them anyway unless you have a reason; a warning usually means the slide will
  not look the way you intended.

Fix every reported frame in one pass — the same mistake almost always repeats across
frames written in parallel.

### 6. Submit

`create_presentation_from_html` with:

- `frames` — array of `{ name, html }` in slide order. Frame 1 becomes slide 1.
  `name` is a short slide title used in the deck outline.
- `title` — deck title, also the Google Slides file name.
- the **same** `style_config_id` / `style_config` you passed to
  `get_html_frame_spec` (required — it drives deck fonts, background, and the
  Slides conversion).
- optional `instructions` — intent/audience notes stored with the deck so later edits
  stay consistent. It does not change the HTML you submitted.
- optional `export_to_google: false` to build the deck without the Drive upload.

Then `wait_for_presentation` with the returned `job_id`, exactly as in mode A. The
completed job returns `editUrl`, `viewUrl`, and a `deckRef` — a frame-authored deck is
a normal Ploxs deck, so every editing tool below works on it.

Conversion runs the browser render + native PPTX path, so it is quick (~30–90s) and
consumes no text-generation credits, but it does require Drive linked and a billable
account.

### 7. Revising

To change a slide you authored, prefer re-authoring the frame and resubmitting the
deck (cheap, deterministic, and your files stay the source of truth). Use `edit_slide`
on a frame-authored deck when the user explicitly wants Ploxs' AI to redesign a slide.

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
