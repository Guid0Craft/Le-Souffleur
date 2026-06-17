# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Le Souffleur** ("the prompter") is a French-language web app for memorizing theater scripts. You paste a script, pick which character you play, and the app hides a tunable percentage of *your* lines' words behind tappable blanks so you can rehearse. Tap a blank to reveal the word.

There is **no build system, no framework, no dependency manifest, and no tests**. The entire app is two files:

- `index.html` — the whole application: markup, inline CSS (`:root` design tokens + dark/light themes), and one big IIFE of vanilla ES (`"use strict"`). No bundler, no transpile step.
- `config.js` — Supabase credentials only (`window.SUPABASE_URL`, `window.SUPABASE_ANON_KEY`). Loaded via `<script src="config.js">` before the app script. Intentionally separate so updating `index.html` never clobbers the user's keys. The `anon` key here is public by design (security is enforced by Supabase RLS); never put the `service_role` key in it.

Supabase JS is pulled from a CDN (`@supabase/supabase-js@2`); there is no local copy.

## Running / developing

Open `index.html` directly in a browser, or serve the folder statically (e.g. `python -m http.server`) and visit it. Static serving is closer to production. There is nothing to compile and no watch step — edit and reload.

To "lint", just load it and check the browser console.

## Architecture

It's a single-IIFE state machine with five views (`auth`, `library`, `editor`, `role`, `study`) toggled by `show(name)`, which swaps the `.active` class and reconfigures the topbar. Module-level `let` variables hold all state (`library`, `current`, `blocks`, `thresholds`, `revealed`, etc.).

**Script parsing** (`parse`) is the heart of the app. Plain text is turned into typed blocks (`spacer`, `heading`, `dida`, `speaker`, `line`) by line-level heuristics:
- A line in ALL CAPS, ≤4 words, not ending in sentence punctuation → a **character name** (`speaker`), and becomes the speaker for following lines.
- A line wrapped in parentheses → a **didascalie** (stage direction), shown in italics, never blanked.
- ALL CAPS matching `HEADING_RE` (ACTE / SCÈNE / TABLEAU / PROLOGUE / ÉPILOGUE) → a **heading**.
- Everything else → a **line** attributed to the current speaker.

Caps detection (`isAllCaps`) and word matching are Unicode-aware (`\p{L}`) and French-locale aware — preserve this when touching parsing.

**Blanking model:** for the chosen character's lines, words are tokenized (`splitParens` → `tokenize`, separating leading/core/trailing punctuation). Each blankable word gets a stable random `threshold` in `thresholds[]`, assigned once by `freshThresholds()`. A word is hidden when `threshold < hidePct/100`. This means changing the hide-percent slider re-thresholds nothing — it just moves the cutoff, so words reveal/hide consistently. "Mélanger" (shuffle) calls `freshThresholds()` to redraw. `revealed` (a Set of word indices) and `revealAll` override hiding. `renderScript()` rebuilds the DOM from `blocks` + this state on every change.

**Persistence is dual-layer:**
- Local: `localStorage` under `STORE_KEY = "souffleur_v1"` (plus separate keys for sort direction and theme). Always written via `saveLibrary()`.
- Cloud (optional): Supabase `texts` table, gated by `cloudActive`/`currentUser`. Every local mutation is mirrored through `upsertRemote` / `deleteRemote`. The slider's cloud sync is debounced (`syncTimer`, 800ms).

Local and remote text shapes differ: the app uses `hidePct` (camelCase); the DB row uses `hide_pct` and adds `user_id`/`updated_at`. Convert with `rowFromText` / `textFromRow` at the boundary — don't leak DB field names into app state.

**Auth & startup:** if `config.js` provided keys and the Supabase CDN loaded, the app creates a client and shows the auth view; `onAuthStateChange` drives `onLogin`/`onLogout`. On first login with an empty cloud, local texts are pushed up as a migration (`onLogin`). Otherwise (no keys, or "Continuer sans compte"), `localMode()` runs purely on `localStorage`. Always keep this local-only fallback working — the app must function with no network and no `config.js`.

## Conventions

- UI text, comments, and error messages are in **French**. Match that.
- Vanilla DOM only — no JSX, no template engine. Build nodes with `document.createElement` and use the local `el`/`add` helpers; `esc()` any user text that goes into `innerHTML`.
- Bump `STORE_KEY` / migrate carefully if you change the persisted text shape, or you'll strand users' saved scripts.
