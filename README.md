# Ploxs Presentations — Test Plugin

Isolated test build of the Ploxs presentation plugin for Claude. It connects to the
isolated Ploxs **test** server at `https://test.ploxs.com/mcp` and never touches
production `ploxs.com`.

Looking for the stable plugin? Use
[ploxs-presentations-plugin](https://github.com/ploxs-ai/ploxs-presentations-plugin).

## What it provides

Two ways to build a deck:

- **Ploxs generates the slides** — from notes, URLs, document text, or CSV data, in a
  saved brand style.
- **Claude authors the slides** — Claude reads your Ploxs style config as
  concrete design tokens (stage geometry, palette, type scale, brand rules), writes the
  final slide HTML once, and submits it for validation and conversion. The
  frames are converted exactly as authored and uploaded to your Google Drive as a
  Google Slides deck.

Choose **Ask every time** (the default), **Ploxs creates**, or **Assistant creates** on
the Ploxs MCP setup page. Ask mode always confirms the creator before each new deck; the
other modes route directly. Later edits still go through Ploxs.

Both paths produce a normal Ploxs deck, so the editing tools — rewrite a slide, add
slides, add generated images or infographics — work on either.

### How the authored path behaves

- **Your brand, not Claude's taste.** Claude asks the server for your style as concrete
  tokens — stage size, the seven palette colors, the deck's type scale, your brand rules,
  the fonts already loaded — and builds against those. It doesn't invent colors or fonts.
- **Straight to slides.** Once Claude has the information and your style, it writes the
  whole final deck in one pass — no prototype/validation slide, repeated HTML payload,
  or planning document in between. That's how Ploxs' own slide painter works too.
- **Charts are real Chart.js charts**, built only from numbers in your material and
  captured into the deck during conversion. Creation validates the chart contract before
  it queues a job.
- **Built once, then edited in place.** Generation or conversion creates the initial
  deck once. After it exists, changes go through Ploxs' editing tools on the same Google
  Slides file — so your link stays valid and you don't collect duplicate decks in Drive.
- **Figures stay honest on the slide.** Numbers come from the material you supplied, and
  anything projected, guided, estimated or dated is labelled on the slide itself
  (`2026 (guided)`, `as of 2021`) rather than mentioned in chat, where it wouldn't travel
  with the deck.

## Install

In Claude Desktop or claude.ai:

1. Sidebar → **Customize** → **Plugins** tab.
2. **+** → **Add marketplace** → **Add from a repository**, and paste:

   ```txt
   https://github.com/ploxs-ai/ploxs-presentations-plugin-test.git
   ```

3. Install **Ploxs Presentations (Test)** from the newly added marketplace.
4. Open the plugin, click **Connect** on the Ploxs connector, and approve the OAuth
   sign-in. No API key is copied into Claude.
5. Start a new chat and ask *"List my Ploxs styles"* to confirm the connection.

In Claude Code:

```txt
/plugin marketplace add ploxs-ai/ploxs-presentations-plugin-test
/plugin install ploxs-presentations-test@ploxs-test
```

## Long-running deck builds

Building a deck takes 1–4 minutes, occasionally longer, and Claude waits for it in a
single tool call. The server keeps that call open for up to 7 minutes.

Claude Code enforces its own per-call limit, so give it headroom before a slow build:

```sh
MCP_TOOL_TIMEOUT=450000 claude    # milliseconds
```

Without it, a long build can trip Claude Code's default timeout mid-wait. Nothing is
lost when that happens — the job keeps running on the server, and Claude picks it up by
asking for the job status again — but raising the limit avoids the interruption. In
Claude Desktop and claude.ai the client's own limit applies and isn't configurable;
Claude simply resumes waiting if it's cut short.

## Before it can create decks

Sign in at [test.ploxs.com](https://test.ploxs.com), connect Google Drive in Settings,
and make sure the account has usage available. Ploxs creates the Slides file in your own
Drive, so Drive linking cannot be done headlessly from Claude. Test accounts, styles,
credits, and decks are separate from production.

## Contents

| Path | Purpose |
| --- | --- |
| `.claude-plugin/marketplace.json` | Marketplace entry Claude reads when you add the repo |
| `.claude-plugin/plugin.json` | Plugin manifest |
| `.mcp.json` | MCP server (`https://test.ploxs.com/mcp`, Streamable HTTP + OAuth) |
| `skills/ploxs-presentations-test/SKILL.md` | The skill: both deck modes, the HTML frame contract, charts, error handling |

Reinstall or update the marketplace entry to pick up skill changes — the skill ships in
this repo, while the tools it drives live on the test server.

This is a testing build; behaviour and tool names can change without notice. Issues and
feedback: privacy@ploxs.com.

## License

MIT — see [LICENSE](./LICENSE).

## Contributing

This repository is generated. The plugin, its skill and these files live in the Ploxs
monorepo and are published from there, so edits made directly here are overwritten on the
next release. Please open an issue instead: privacy@ploxs.com.
