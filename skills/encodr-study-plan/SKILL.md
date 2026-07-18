---
name: encodr-study-plan
description: Use when a user wants to review their Encodr progress and get a study plan - reads their weak cards, streak, and deck coverage via the Encodr MCP server, diagnoses what they're missing (both cards they keep failing AND topics their decks never cover for the class), then compiles a prioritized, dated study plan and offers to generate remedial cards. Triggers on "review my Encodr cards", "what am I missing", "what should I study next", "make me a study plan", "am I ready for my exam".
---

# Encodr Study Plan

## Overview

Diagnoses where a learner actually stands in Encodr and turns it into a concrete plan: what to review, in what order, on what days, and which gaps to close before an exam. It looks at two kinds of "missed":

1. **Failed recall** - cards the learner keeps getting wrong (from `get_weak_cards`).
2. **Coverage gaps** - topics the *class/syllabus* requires that the learner's decks never test at all. These are invisible to the app because you can't miss a card that doesn't exist. Finding these is the highest-value thing this skill does.

The deliverable is a written plan the learner can follow, plus an offer to create the remedial cards that fill the gaps (handing off to the `encodr-card-gen` skill).

## The MCP tools you will use

- `get_progress` - streak, xp, last 30 days of activity, overall totals. Reveals consistency and momentum.
- `get_weak_cards` (limit up to 50) - the most-missed cards, with recent misses/lapses. This is "everything you missed".
- `list_decks` - all decks (owned/forked/system) with card counts. Maps which classes/subjects exist.
- `get_deck` - full card list for a deck. Use to see *what a deck actually covers* so you can compare against the syllabus and find coverage gaps.
- `add_cards` / `create_deck` - only when the user approves generating remedial cards (delegate the card-writing to `encodr-card-gen`).

Read-only tools first. Never create or modify cards without explicit approval.

## Step 0 - Scope

Ask briefly (skip anything already given):
- **Which class/subject or deck** is this plan for? (If they have one clear active deck, infer it and confirm.)
- **The goal**: an exam on a date? a quiz? general mastery? A date turns the plan into a dated schedule.
- **The syllabus/scope** for coverage-gap analysis - a topic list, syllabus link, textbook + chapters, or "the standard curriculum for X". Without this, you can only analyze failed cards, not missing topics. Push gently for it; it's what makes the plan complete.
- **Time available** - minutes/day and days/week - so the schedule is realistic.

## Step 1 - Pull the data

Call `get_progress`, `list_decks`, and `get_weak_cards` (limit 50). Then `get_deck` on the target deck(s). Hold the raw results; don't dump them at the user.

## Step 2 - Diagnose

Analyze across three lenses:

**A. Consistency (from progress).** Streak, active days in last 30, cards/day trend. Flag if they've lapsed, cram in bursts, or study nothing for stretches. Consistency problems change the plan more than content gaps do.

**B. Failed recall (from weak cards).** Cluster the weak cards by theme, not one by one. Look for patterns: "every miss is a date", "all the enzyme cards", "confusing X vs Y". Each cluster is a weak *area*, not just weak cards. Note lapse counts - a card lapsed 5 times needs a different fix (better mnemonic, reformatted card, or a prerequisite gap) than one missed once.

**C. Coverage gaps (from deck contents vs syllabus).** This is the part the app can't do. Spawn one or more `Explore` / `general-purpose` agents (or `deep-research` for an unfamiliar curriculum) to build the *expected topic list* for the class, then diff it against the topics the user's cards actually cover (from `get_deck`). Output the concrete topics with zero or thin card coverage. Weight by exam importance where known. If the user gave an explicit syllabus, use that as ground truth instead of researching.

## Step 3 - Compile the plan

Produce a written plan with these parts:

1. **Where you stand** - 3-5 honest sentences: consistency, strongest areas, the 2-3 weakest clusters, and the biggest coverage gaps. No sugar-coating; this is a diagnosis.
2. **Priority order** - rank what to work on by *impact x urgency*. Rule of thumb: fix high-frequency exam topics that are also weak first; a rarely-tested topic you've already mastered is last. Coverage gaps on core topics usually outrank polishing cards you miss occasionally.
3. **A dated schedule** (if there's an exam date) or a **weekly cadence** (if not). Respect spaced repetition: interleave decks rather than blocking one topic for days, put new/remedial material earlier so it gets more review cycles before the exam, and reserve the final days for weak-card review + full mixed practice, not new content. Fit it to their stated minutes/day.
4. **Gap-closing actions** - for each coverage gap, name what to add (e.g. "12 cards on the electron transport chain, mixed types"). For each weak cluster, prescribe the fix: re-review, reformat the card, add a mnemonic, or add prerequisite cards.
5. **A checkpoint** - a simple readiness signal, e.g. "you're on track when weak-card count is under N and every syllabus topic has cards you pass twice in a row."

Keep it skimmable - headers, short bullets, real numbers from their data. Cite actual deck names and card counts so it's obviously *theirs*, not generic advice.

## Step 4 - Offer to act

Offer, don't auto-do:
- **Generate remedial cards** for the coverage gaps and repeated lapses - hand off to the `encodr-card-gen` skill, which will research, mix types, QA, and create/append via MCP. Nest new decks under the class's parent deck.
- **Reformat chronically-lapsed cards** - propose better versions (add `mnemonic`/`whyTrue`, split two-answer cards, convert an ambiguous `qa` into `mc` with real distractors) and, on approval, add the improved versions.
- **A focused review deck** - optionally compile the weak cards' topics into one "exam crunch" deck.

Only touch the account after the user says yes.

## Principles

- **Diagnose from data, not vibes.** Every claim in the plan should trace to a number you pulled (a lapse count, active-days figure, a topic missing from `get_deck`).
- **Coverage gaps > card polish.** A topic with no cards is a bigger risk than a card missed twice. Lead with gaps.
- **Respect spacing.** Never recommend cramming one topic for days straight; interleave and space toward the exam.
- **Honest over encouraging.** If they're behind, say so and show the path. A plan that flatters is useless.
- **Read-only until approved.** Analysis never writes to the account; card creation always waits for a yes.

## Anti-patterns

- Listing every weak card individually instead of clustering into weak *areas*.
- A plan that ignores the syllabus and only reshuffles existing cards - it will miss the topics that aren't there.
- Generic advice ("study consistently, review often") with no reference to their actual decks, streak, or exam date.
- Auto-creating cards or decks before the user approves.
- Recommending a study order that blocks one topic for days and violates spaced repetition.
