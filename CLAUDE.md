# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`index.html` is the entire application: a single, self-contained, offline-first interactive training course ("Db2 for z/OS — From Senior LUW DBA to IBM Z DBA"). There is no build step, no package manager, no server, and no external runtime dependencies — everything (HTML, CSS, JS, and all course content/data) lives in this one file and must keep working when opened directly via `file://`.

There is no test suite, linter, or CI config in this repo. `README.md` is a placeholder (just the repo name).

## Working with this codebase

- **Edit in place.** Almost every change is a targeted `Edit` inside `index.html`. Because the file is one big `<script>` block, always anchor edits on unique surrounding context (e.g. a specific module's closing `});` plus its `deep:`/`sources:` line) rather than generic strings like `deep:"", sources:[]` which repeat dozens of times across modules.
- **No build/run command exists.** To "run" the app, just open `index.html` in a browser, or serve the directory with any static file server (`python3 -m http.server`, etc.) — nothing to install first.
- **Verify changes with a headless browser**, since there's no test suite:
  ```bash
  node -e "new Function(require('fs').readFileSync('index.html','utf8').match(/<script>([\s\S]*?)<\/script>/)[1])"  # quick syntax check
  ```
  For real verification, use Playwright (available via the global `npm root -g` install, Chromium at `/opt/pw-browsers/chromium`) to load the file, drive `location.hash` through routes, and assert `page.on('pageerror'|'console')` stays empty. This is how prior work in this repo was validated — load the page, walk every sidebar link, exercise quizzes/search/JCL lab/simulator, and confirm no console errors.
- Keep the file self-contained: no new `<script src>`/`<link>` to external hosts, no build tooling, no framework. Inline everything.

## Architecture (all inside `index.html`)

The file is organized top to bottom as: `<style>` (design tokens + components) → `<body>` shell → one `<script>` containing state, a structured content data model, and vanilla-JS render functions wired to a hash router.

### 1. State & persistence (~line 279)
A single `STATE` object (theme, learning mode, font/contrast/motion settings, low-energy mode, completed lessons, lesson notes, quiz/diagnostic/final results, bookmarks) is loaded from and saved to `localStorage` under key `db2zos_training_state_v1` via `loadState()`/`saveState()`. `applyTheme()` reflects `STATE` onto `<html data-theme|data-mode|data-contrast|data-motion>` attributes, which CSS selectors key off of (e.g. `html[data-mode="core"] [data-level="dba"] { display:none }` implements the Core/DBA Deep Dive/Expert content gating).

### 2. Course content data model (~line 316–2183)
`COURSE.modules` is an array of 41 module objects (`m00`–`m40`), each built with `COURSE.modules.push({...})` and containing one or more lessons created via the `L(o)` helper (line 319), which fills in a fixed pedagogical shape: `why`, `luw` (the LUW→z/OS bridge: know/changes/why), `concept`, `visual`, `example`, `handsOn`, `trap`, `remember[]`, `check[]` (quiz questions), `deep` (expert-gated content), `sources[]`. **New/edited lesson content must follow this same shape** — the lesson renderer (`renderLesson`) assumes these exact keys and renders them in a fixed 10-part template order.

Reference data lives in flat arrays right after the modules: `GLOSSARY` (line 2186), `DICTIONARY` — the LUW↔z/OS term mapping (line 2236), `SQLCODES` (line 2265), `COMMANDS` (line 2311).

### 3. Render functions + hash router (~line 2328 onward)
No framework — each page is a `render*()` function that sets `#content.innerHTML` and wires up event listeners imperatively. `router()` (line 3193) parses `location.hash` and dispatches to the matching `render*()`:

| Route | Function |
|---|---|
| `#/` | `renderDashboard()` |
| `#/lesson/:modId/:lessonId` | `renderLesson()` |
| `#/diagnostic` | `renderDiagnostic()` |
| `#/glossary`, `#/dictionary`, `#/commands`, `#/sqlcodes` | reference pages |
| `#/map` | `renderMap()` — clickable SVG knowledge map |
| `#/simulator` | `renderSimulator()` — pattern-matched fake z/OS console (`SIM_RESPONSES`) |
| `#/jcllab` | `renderJCLLab()` — JCL repair exercises (`JCL_EXERCISES`) checked by regex |
| `#/final` | `renderFinal()` — pulls `check[]` questions from specific modules per category (`buildFinalQuestions()`) into a scored competency report |
| `#/checklist` | `renderChecklist()` |

`renderSidebar()` (line 2366) builds nav from `COURSE.modules` grouped by `level` (`foundation`/`core`/`deep`/`lab`/`capstone`) plus static links to the reference tools. All interactive labs (simulator, JCL lab) are explicitly UI-labeled as simulated/not connected to a real system — preserve that labeling in any related changes.

### 4. Conventions to preserve
- All simulated/educational environments must stay clearly marked as simulated (this was a hard requirement from the original spec).
- Every lesson's `luw` bridge should avoid forcing false LUW↔z/OS equivalences (e.g., data sharing ≠ pureScale) — this is a recurring, deliberate theme through the content and the `DICTIONARY` data.
- Version-sensitive Db2/z/OS claims should be flagged as version-specific or hedged with "verify against IBM documentation," consistent with existing lesson content.
- Respect the strict CSP-like self-containment: no external fonts/scripts/images/CDNs.
