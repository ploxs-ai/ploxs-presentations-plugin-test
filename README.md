# Ploxs Presentations — Test Plugin

Pre-release build of the Ploxs presentation plugin for Claude. It connects to the
isolated Ploxs **test** server at `https://test.ploxs.com/mcp` and never touches
production `ploxs.com`.

Looking for the stable plugin? Use
[ploxs-presentations-plugin](https://github.com/vipinsanthosh/ploxs-presentations-plugin).

## What it adds over the stable plugin

Two ways to build a deck, and the skill tells Claude when to use which:

- **Ploxs generates the slides** — from notes, URLs, document text, or CSV data, in a
  saved brand style. Same as the stable plugin.
- **Claude authors the slides** *(new)* — Claude reads your Ploxs style config as
  concrete design tokens (stage geometry, palette, type scale, brand rules), writes the
  slide HTML itself, checks it against the conversion contract, and submits it. The
  frames are converted exactly as authored and uploaded to your Google Drive as a
  Google Slides deck.

  Use it when the design or content should come from Claude: precise layouts, content
  already in its context (a codebase, a report, analysis it just ran), tables that must
  survive intact, or a deck that should be reproducible from files in a repo.

Both paths produce a normal Ploxs deck, so the editing tools — rewrite a slide, add
slides, add generated images or infographics — work on either.

### How the authored path behaves

- **Your brand, not Claude's taste.** Claude asks the server for your style as concrete
  tokens — stage size, the seven palette colors, the deck's type scale, your brand rules,
  the fonts already loaded — and builds against those. It doesn't invent colors or fonts.
- **The whole deck at once.** Slides are authored concurrently from one shared design
  kit, which is how Ploxs' own slide painter works. Expect a deck in one pass, not a
  slide per message.
- **Charts are real Chart.js charts**, built only from numbers in your content and
  captured into the deck during conversion. Claude checks them before submitting, because
  a chart that breaks the conversion rules would otherwise arrive as an empty box.
- **Built once, then edited in place.** Conversion happens at the initial export. After
  the deck exists, changes go through Ploxs' editing tools on the same Google Slides
  file — so your link stays valid and you don't collect duplicate decks in Drive.
- **Facts before design.** The skill has Claude pin down every figure — value, period,
  source, and whether it's reported, guided, or an estimate — before writing any slide,
  and label anything that isn't a reported actual on the slide itself rather than
  mentioning it in chat.

## Install

In Claude Desktop or claude.ai:

1. Sidebar → **Customize** → **Plugins** tab.
2. **+** → **Add marketplace** → **Add from a repository**, and paste:

   ```txt
   https://github.com/vipinsanthosh/ploxs-presentations-plugin-test.git
   ```

3. Install **Ploxs Presentations (Test)** from the newly added marketplace.
4. Open the plugin, click **Connect** on the Ploxs connector, and approve the OAuth
   sign-in. No API key is copied into Claude.
5. Start a new chat and ask *"List my Ploxs styles"* to confirm the connection.

In Claude Code:

```txt
/plugin marketplace add vipinsanthosh/ploxs-presentations-plugin-test
/plugin install ploxs-presentations-test@ploxs-test
```

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
