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
| `skills/ploxs-presentations-test/SKILL.md` | The skill: both deck modes, the HTML frame contract, error handling |

This is a testing build; behaviour and tool names can change without notice. Issues and
feedback: privacy@ploxs.com.

## License

MIT — see [LICENSE](./LICENSE).
