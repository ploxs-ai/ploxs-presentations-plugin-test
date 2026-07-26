---
name: ploxs-presentations-test
description: Create and edit designed Google Slides decks through the Ploxs TEST MCP server (test.ploxs.com), including authoring the slide HTML yourself. Use when the user asks to make or edit a presentation or slide deck, apply a brand style to slides, design slides as HTML frames, or list/inspect their Ploxs decks.
---

# Ploxs Presentations (Test)

Test server `ploxs-test` at `https://test.ploxs.com/mcp`. Accounts, credits, styles and
decks are separate from production `ploxs.com` — never mix them in one conversation.

Everything is async: creation returns a `jobId`, edits return a `task`. Wait with
`wait_for_presentation` / `wait_for_presentation_edit` before a dependent step or before
telling the user it's done.

## Start

1. `get_account_status` — `google.driveConnected` must be true. If not, give the user
   the `links.googleDrive` URL and stop. On insufficient credits, surface the message
   and stop.
2. Style: `list_style_configs`, then pass `style_config_id`. A `deck:` style's name
   describes the deck it came from, **not its colors** — read its `colorPalette` hexes
   before trusting it. Pass inline `style_config` only for branding the user gives you
   that isn't saved yet (check it with `validate_style_config`). `auto_style: true` is a
   last resort, and never for authored frames.
3. If the topic is genuinely unspecified, ask one short question and stop. Don't guess a
   topic and burn a deck job on it.

## Which mode

**`create_presentation`** — the user hands over raw material and wants Ploxs to write
and design it: `markdown`, `urls` (≤10), `file_texts` (≤20), `csv_sources`, plus optional
`instructions` and `slide_count` (1–60). Idempotent ~10 min; daily caps apply.

**`create_presentation_from_html`** — *you* write the slides. Use it when the content is
already yours: analysis you just ran, a codebase, a report you wrote, exact tables and
numbers that must survive intact, or layouts the user specified. Frames convert exactly
as authored — no regeneration, no restyling. Say which mode you're using.

## Authoring HTML frames

**Call `get_html_frame_spec` with your style, then start writing frames immediately.**

That one response is your whole brief: stage size and safe area, the palette and fonts,
the type scale, the binding HTML contract, the Chart.js protocol with a copy-ready
widget, and any overlay exclusion zones. Follow it literally; don't re-derive tokens or
invent colors, fonts and sizes.

Write **every frame in one pass, back-to-back in a single turn.** No outline document, no
design memo, no turn-per-slide. Each frame is the inner HTML of one slide, and frame 1
becomes slide 1. With subagents, give each worker one frame plus the spec's tokens — send
the full contract and `charts` block only to a worker whose slide has a chart.

Hold these constant as you write:

- sizes copied from `briefs.layout` — not a fresh scale per slide
- one small class vocabulary reused in every frame (`.wrap`, `.lede`, `.card`, `.stat`,
  `.note`, `.icon`). Class names auto-suffix per frame so they cannot collide;
  per-frame prefixes like `.f1-card` only bloat the CSS
- **no `font-family` anywhere** — the deck already applies the heading font to `h1`–`h6`
  and the body font to everything else
- numbers only from the material you were given. Anything projected, guided, estimated or
  dated gets labelled on the slide itself (`2026 (guided)`, `est.`, `as of 2021`) — a
  caveat in chat doesn't travel with the deck

**Charts:** follow `spec.charts` and copy its widget's plumbing verbatim — unique
`<canvas id>`, its loader, `dataset.initialized`, `animation: false`. Anything less
exports as an empty box. If the data doesn't need a chart, build it as markup instead;
that converts to editable Slides shapes rather than a flat image.

**Submit:** `create_presentation_from_html` with `frames` (`{name, html}` in slide order),
`title` (plain text — not HTML-escaped), and the same style you gave the spec. Then
`wait_for_presentation`, hand over `editUrl`/`viewUrl`, and keep the `deckRef`.

**Validate when charts are involved.** `validate_html_frames` is free and takes all
frames in one call; chart failures are silent and cost a whole job, so check those first.
For pure-markup decks go straight to submit — it returns the same warnings, and errors
queue nothing. Warnings quote what they matched: if you can't find that thing in your
frame, it isn't about your frame.

## Changing a deck that already exists

Conversion happens **once**, at the initial export. `create_presentation_from_html`
cannot update a deck — calling it again builds a *second* deck and a second Drive file,
and the user's existing link goes stale.

Every later change goes through the Ploxs edit tools on the `deckRef`, which run Ploxs'
AI against the live file and edit it in place:

- `edit_slide` — rewrite/redesign one slide from a `prompt`
- `add_slides` — insert (`insertion_placement` `before`/`after` needs
  `anchor_slide_number`; default `end`)
- `add_image_to_slide` / `add_infographic_to_slide` — generated image or chart
- `update_presentation` — batch of 1–20 of the above

Fetch `get_presentation_outline` first: slide numbers are 1-based against the live deck,
and inside a batch each number resolves as of that step. Poll
`wait_for_presentation_edit(task_id)` before a dependent edit; 3 tasks active max.

The two asset tools **add** by default and keep what's already there. When the user wants
to change, refresh or swap the existing image/chart, confirm and pass
`replace_existing_asset: true` with `redesign_slide: true`.

Decks Ploxs hasn't seen: `connect_presentation` with a Slides URL or ID. If it returns
`ready: false`, relay the `actionUrl` and retry after the user approves; a new deck also
needs a style. `list_presentations` lists known ones.

## Errors

Returned as text with `(code: …, retryable: …)`.

- `google_not_linked` / `google_scope_upgrade_required` — relay the link, wait
- `invalid_html_frames` — names the frames and rules; fix and resubmit, never retry
  unchanged
- `style_config_required` — pass the style the frames were authored against
- `slide_not_found` — the message has the real slide count; re-fetch the outline
- `active_job_limit` / `rate_limited` — wait for running work, then retry
- entitlement/credit errors — report to the user, don't retry
- `html_frames_disabled`, or the frame tools are missing — this deployment doesn't have
  the HTML frame surface; use `create_presentation` and say so

Never invent a `deckRef` or a slide number — source them from the tools.
