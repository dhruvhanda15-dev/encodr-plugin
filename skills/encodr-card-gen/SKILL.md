---
name: encodr-card-gen
description: Use when a user wants to generate high-quality Encodr study cards/decks from a topic, syllabus, PDF, notes, or textbook via the Encodr MCP server. Researches the material, drafts a mixed-type deck using every card format correctly, QA-checks it, then creates the deck. Triggers on "make Encodr cards", "study deck for X", "turn these notes into flashcards".
---

# Encodr Card Generator

## Overview

Turns any source material (a topic, a syllabus, pasted notes, a PDF, a chapter) into a well-built Encodr deck through the Encodr MCP server. The goal is not "many cards" - it is cards that make someone actually learn the material: one idea per card, mixed formats matched to the cognitive job, and every card factually verified.

The Encodr MCP tools are the only way cards reach the app. Do not write cards to a file and call it done - the deliverable is a created deck and its `share_url`.

## The MCP tools you will use

- `list_decks` - find existing decks / parent deck ids before creating or appending.
- `get_deck` - read a deck's existing cards (avoid duplicating them when appending).
- `create_deck` - make a new deck. Params: `name`, `cards[]`, optional `description`, optional `parent_deck_id` (to nest under a subject). Returns a `share_url` - always give it to the user.
- `add_cards` - append `cards[]` to an existing `deck_id` the user owns.
- `get_weak_cards` - the user's most-missed cards; use when they ask for *remedial* / *review* cards.
- `get_progress` - streak/xp/activity; only for context if asked.

`cards[]` accepts 1-200 entries per call. For big decks, create with the first batch, then `add_cards` in further batches of up to 200.

## The 7 card types - pick the right tool for the job

Never make a deck of only one type. Match the type to what the learner must *do*:

| Type | Use it for | Required fields |
|------|-----------|-----------------|
| `qa` | A fact with one clean free-recall answer | `prompt`, `answer` |
| `mc` | Discriminating between confusable options; when good distractors exist | `prompt`, `options[]` (>=2), `correctIndex` (0-based) |
| `tf` | A single testable assertion; catching a common misconception | `prompt` (the statement), `isTrue` |
| `cloze` | A key term/number inside a sentence you want recalled in context | `clozeText` (must contain `{{...}}`), `answer` |
| `typed` | Exact terms/spelling/formulas where typing it matters | `prompt`, `answer`, optional `acceptable[]` (alt spellings) |
| `explain` | A concept worth explaining in the learner's own words | `prompt`, `answer` (model answer), `rubricPoints[]` (>=1 discrete things a good answer hits) |
| `sequence` | An ordered process, timeline, or pipeline | `prompt`, `steps[]` (>=2, in correct order) |

Rough healthy mix for a general deck: ~35% qa/typed (recall), ~25% mc/tf (discrimination), ~15% cloze, ~15% explain, ~10% sequence - shift toward `explain`/`sequence` for conceptual subjects, toward `qa`/`typed`/`cloze` for vocab/facts.

### Enrichment fields (use them - they make cards teach, not just test)

- `mnemonic` - a memory aid, for anything arbitrary/hard to recall.
- `whyTrue` - a one-line rationale for the answer (great on `qa`, `tf`, `mc`).
- `distractorRationales[]` (mc only) - one entry per option saying why each is right/wrong. Add these on any non-trivial `mc`.
- `tier` - `recall` | `concept` | `theory`. Optional (inferred from type), but set it deliberately: `recall` = facts, `concept` = understanding/application, `theory` = synthesis/why. A good deck spans all three, not just `recall`.

## Card-writing rules (enforce these in QA)

1. **One idea per card.** If the answer has "and", it is probably two cards.
2. **Answers are short and unambiguous.** `qa`/`typed` answers should be a word to a short phrase, not a paragraph. Put extra detail in `whyTrue`, not the answer.
3. **Questions stand alone.** No "as mentioned above" / "in this chapter". A card is seen in isolation months later.
4. **`mc` distractors are plausible and parallel** - same length/category as the answer, not obvious throwaways. 3-4 options. Vary `correctIndex`; never make it always the same slot.
5. **`tf` is ~50/50 across the deck** and each false statement targets a real misconception, not a nonsense claim.
6. **`cloze` blanks the load-bearing term**, not a trivial word. One blank unless multiple are tightly linked.
7. **`explain` rubricPoints are discrete and tickable** - each is one distinct thing a good answer says (3-5 is ideal), not a full sentence of prose.
8. **`sequence` steps are atomic and correctly ordered**, each a short imperative/noun phrase.
9. **No duplicate coverage** - two cards must not test the identical fact from the same angle. (Rephrasing the same fact as different *types* is fine and good.)
10. **Faithful to the source.** Never invent facts, dates, or numbers. If a claim can't be verified, cut the card.

## Naming conventions

- **Deck name:** `Subject - Topic` (e.g. `Biology - Cellular Respiration`, `Spanish - Preterite Verbs`, `AWS SAA - VPC & Networking`). Title case, no trailing punctuation, <=200 chars. If it's a chapter/exam, include it: `Anatomy - Ch. 4: The Skeletal System`.
- **Nesting:** if a subject deck already exists (check `list_decks`), pass its id as `parent_deck_id` so the new deck nests under it. Offer to nest when the user is clearly building a course.
- **Description:** one line - scope + source, e.g. "Krebs cycle, ETC, and ATP yield; from Campbell Ch. 9."
- **Card counts:** default 20-30 cards for a focused topic. Ask before generating 100+. Prefer several tight decks over one giant deck.

## Workflow

For anything beyond a trivial handful of cards, run this as a multi-agent pipeline so research and QA are done by fresh eyes, not the same pass that wrote the cards.

### Step 0 - Scope (ask, briefly)
Confirm: the **source** (topic / pasted text / file path / URL), the **exam or goal** (drives difficulty and tier mix), rough **card count**, and whether to **create new** vs **append** to an existing deck. If they gave a file/URL, read/fetch it first. Don't ask for anything they already provided.

### Step 1 - Research (spawn agents in parallel)
Dispatch one or more `Explore`/`general-purpose` agents (or `deep-research` for unfamiliar/broad topics) to build a verified fact sheet BEFORE writing cards. In one message, spawn agents split by subtopic. Each returns: key facts, definitions, exact numbers/dates, common misconceptions (gold for `tf`/`mc` distractors), and any ordered processes (gold for `sequence`). If the source is user-supplied text, extraction replaces web research - but still verify anything that looks like a claim you're unsure of.

### Step 2 - Draft
From the fact sheet, write the `cards[]` array. Deliberately mix types per the table above, spread tiers, and add `mnemonic`/`whyTrue`/`distractorRationales` where they help. Organize cards in a sensible teaching order (foundational → advanced).

### Step 3 - QA (spawn a fresh reviewer agent)
Spawn a separate `general-purpose` agent whose only job is to audit the drafted JSON against the "Card-writing rules" above and the source facts. It must flag: factual errors, duplicate/overlapping cards, ambiguous answers, weak `mc` distractors, `correctIndex` off-by-ones, `cloze` missing `{{}}`, single-type monotony, and any card whose answer is really two cards. Fix everything it flags. For high-stakes material (medical, legal, certification), run a second independent verifier and only keep cards both accept.

### Step 4 - Create
Call `create_deck` (or `add_cards`) with the cleaned array. Batch >200 cards across calls. Report back: deck name, card count, the type breakdown, and the **`share_url`**.

### Step 5 - Offer follow-ups
Offer: more cards on subtopics, a harder "exam-mode" pass, or - if they've been studying - `get_weak_cards` to generate targeted remedial cards for what they keep missing.

## Quick reference - minimal valid cards

```json
[
  { "type": "qa", "prompt": "What enzyme unwinds DNA at the replication fork?", "answer": "Helicase", "tier": "recall", "whyTrue": "Helicase breaks hydrogen bonds between base pairs." },
  { "type": "mc", "prompt": "Which molecule is the primary energy currency of the cell?", "options": ["ATP", "ADP", "NADH", "Glucose"], "correctIndex": 0, "distractorRationales": ["Correct - hydrolysis of its terminal phosphate releases usable energy.", "ADP is the lower-energy product, not the currency.", "NADH is an electron carrier, not the direct currency.", "Glucose stores energy but must be broken down first."] },
  { "type": "tf", "prompt": "Glycolysis requires oxygen to proceed.", "isTrue": false, "whyTrue": "Glycolysis is anaerobic; it occurs in the cytoplasm without O2." },
  { "type": "cloze", "clozeText": "The Krebs cycle takes place in the {{mitochondrial matrix}}.", "answer": "mitochondrial matrix" },
  { "type": "typed", "prompt": "Net ATP yield from glycolysis (per glucose)?", "answer": "2", "acceptable": ["two", "2 ATP"] },
  { "type": "explain", "prompt": "Explain why the electron transport chain needs oxygen.", "answer": "O2 is the final electron acceptor; without it electrons back up, NADH can't be reoxidized, and the chain halts.", "rubricPoints": ["O2 is the final electron acceptor", "electrons/protons back up without it", "NADH cannot be reoxidized", "chain and ATP synthesis stop"], "tier": "concept" },
  { "type": "sequence", "prompt": "Order the stages of aerobic respiration.", "steps": ["Glycolysis", "Pyruvate oxidation", "Krebs cycle", "Electron transport chain"] }
]
```

## Anti-patterns

- One deck of 60 `qa` cards. Mix types.
- Answers that are paragraphs. Move detail to `whyTrue`.
- `mc` with one real option and three jokes. Distractors must tempt.
- Inventing numbers/dates to fill a card. Verify or cut.
- Writing cards to a `.md` file and stopping. The card only exists once an Encodr MCP tool creates it.
- Skipping QA on high-stakes material.
