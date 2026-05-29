# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**MyTurn** — a Duolingo-style web trainer for learning Jersey (Channel Islands) civics: voting, the States Assembly, parishes, youth services, candidacy, Crown-Dependency status, courts, and practical civic action.

It is a fork of the Python-teaching `duoPython` project (Indent), with all course content swapped for Jersey civics. The architecture is unchanged: zero-dependency, zero-build static **PWA**, no framework or bundler. Everything is hand-written vanilla HTML/CSS/JS.

> Note: localStorage keys keep the historical `pylingo_*` prefix inherited from the upstream. They are inert strings — nothing about the app is Python-specific anymore — and renaming them would wipe any pilot users' progress. Leave them.

## Running & developing

There is no build step — edit files and reload. But you **must serve over HTTP**, not `file://`, or the service worker won't register and PWA behavior breaks:

```sh
python3 -m http.server 8765   # then open http://localhost:8765
```

A `.claude/launch.json` is set up so `preview_start` with name `myturn` will launch this server.

There are no automated tests or lint — verification is manual in a browser.

## Where everything lives

Almost the entire app is **`index.html`**: `<style>` block, then one big `<script>`. Other files are small and supporting:

- `sw.js` — service worker (offline cache + update banner). Cache version is `myturn-v1`.
- `manifest.json` — PWA manifest
- `mascot.svg`, `icon-*.png`, `apple-touch-icon.png` — branding assets (still the original Indent mascot — not Jersey-themed yet)

## Architecture (the parts that span files / functions)

**Screens, not routes.** Full-screen `<div>`s; `showScreen(id)` hides all and shows one. No router, no URL state. Home screen is `mapScreen`.

**Single `state` object → localStorage.** `state = { xp, completed, streak, lastActive }`. `loadState()`/`saveState()` persist to `STORAGE_KEY` (`pylingo_v2`). Sound and arcade high score under separate keys.

**Course content is data, in `TOPICS`.** 8 topics, each with `levels` of `exercises`.

- **4 core topics** (4 levels each): `vote`, `states`, `parishes`, `youth`
- **4 advanced topics** (3 levels each, `advanced: true`): `candidate`, `crown`, `courts`, `action`

Advanced topics stay locked until L2 is cleared in at least `ADVANCED_UNLOCK_THRESHOLD` (2) core topics — see `topicUnlocked()` / `coreL2Cleared()`. **To add or change lessons, edit the `TOPICS` data — no code changes needed.**

**Exercise type system.** Each exercise has a `type` that drives both rendering and grading:
- `mc` — multiple choice (`renderMC`); `codeChoices: true` renders choices in monospace
- `fill` — fill the blank, using `___` in `code` (`renderFill`)
- `tokens` — assemble an answer from shuffled token chips (`renderBuild`)
- `order` — reorder shuffled lines (`renderBuild`, `isLines` branch)

Grading lives in `isCorrect()`. New exercise types must touch `renderExercise`, `isCorrect`, `correctText`, and `hasResponse`.

**Three play modes share the exercise renderers** but have separate session/flow logic:
- **Lessons** — the map flow. `startLesson` → per-question check → `finishLesson`, with hearts budget. Awards XP and marks `completed`.
- **Arcade** (`openArcade`) — timed survival run pulling real exercises from `TOPICS` via `arcadePool()`. Lives, countdown timer, combos, tier-based question pools (`ARCADE_TIER_L2/L3`).
- **Practice** (`openPractice`) — endless, sampled questions from civics fact banks. `PRACTICE_GENERATORS` maps each topic to small generator functions; most use `makeFactGen(POOL)` which picks one fact at random from a per-topic array and shuffles choices. A generator is only eligible once a level has been **completed** in the matching `TOPICS` track.

All three funnel into a shared `session` object and the same exercise DOM.

**Service worker / updates.** `sw.js` is **network-first for HTML** and **cache-first for static assets**. Content changes ship on reload. Only bump `CACHE_VERSION` in `sw.js` to force a clean cache flush / show the in-app "Update available" banner.

## Sourcing facts

Course content was drawn from official Jersey sources (gov.je, statesassembly.je / statesassembly.gov.je, vote.je, courts.je). Facts reflect the **post-7 June 2026 States composition**: 9 Senators + 28 Deputies + 12 Connétables = 49 elected Members. Before adding new questions, verify against an official source — and avoid topics flagged as uncertain (e.g. specific named Youth Service programmes, existence of a formal Youth Parliament, candidate minimum age — these were intentionally left out of the initial content).
