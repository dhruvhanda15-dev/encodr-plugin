# Encodr plugin for Claude Code

Generate and review [Encodr](https://encodr.app) study decks straight from Claude Code. This plugin bundles:

- **`encodr-card-gen` skill** - researches your material, drafts a mixed-type deck (qa, mc, tf, cloze, typed, explain, sequence), QA-checks every card, then creates the deck.
- **`encodr-study-plan` skill** - reads your progress, weak cards, and deck coverage, diagnoses what you're missing (cards you keep failing *and* class topics your decks never cover), then compiles a prioritized, dated study plan and offers to fill the gaps.
- **Encodr MCP server** - the tools that actually read and create decks in your account (`create_deck`, `add_cards`, `list_decks`, `get_deck`, `get_weak_cards`, `get_progress`).

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
- "Review my Encodr cards and make me a study plan for my bio exam on the 30th"
- "What am I missing before my quiz?"

The card-gen skill handles research, card-type mixing, QA, and deck creation. The study-plan skill reads your account, finds gaps, and writes a dated plan - then hands off to card-gen to fill the gaps.

## What's in here

```
.claude-plugin/
  plugin.json         plugin manifest
  marketplace.json    makes this repo installable as its own marketplace
skills/
  encodr-card-gen/
    SKILL.md          the card-generation skill
  encodr-study-plan/
    SKILL.md          the review + study-plan skill
.mcp.json             wires up the Encodr MCP server on install
```
