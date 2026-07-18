# Encodr plugin for Claude Code

Generate high-quality [Encodr](https://encodr.app) study decks straight from Claude Code. This plugin bundles:

- **`encodr-card-gen` skill** - researches your material, drafts a mixed-type deck (qa, mc, tf, cloze, typed, explain, sequence), QA-checks every card, then creates the deck.
- **Encodr MCP server** - the tools that actually create decks in your account (`create_deck`, `add_cards`, `list_decks`, `get_weak_cards`, and more).

## Install

In Claude Code:

```
/plugin marketplace add dhruvhanda15-dev/encodr-plugin
/plugin install encodr@encodr
```

On first use you'll be asked to sign in to your own Encodr account (OAuth). Your decks stay private to you.

> Note: skills and plugins run in the Claude Code CLI and IDE integrations only - not in the claude.ai web app or desktop app. To use Encodr from claude.ai, add the MCP server there as a connector: `https://mcp.encodr.app/mcp`.

## Use

Just ask, for example:

- "Make Encodr cards for the Krebs cycle"
- "Turn these lecture notes into a study deck" (paste or point to a file)
- "Build a 30-card deck for AWS SAA networking"

The skill handles research, card-type mixing, QA, and deck creation, then hands you the deck's share link.

## What's in here

```
.claude-plugin/
  plugin.json         plugin manifest
  marketplace.json    makes this repo installable as its own marketplace
skills/
  encodr-card-gen/
    SKILL.md          the card-generation skill
.mcp.json             wires up the Encodr MCP server on install
```
