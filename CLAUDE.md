# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Singapore-focused FIRE / retirement planner. The **entire app is one file**: `index.html` (~9,200 lines, HTML + CSS + JS). No build step, no package manager, no backend, no framework. The only runtime dependency is Chart.js (CDN) plus a small LZ-string helper (CDN). Hosted on GitHub Pages as a static file.

`data/` holds five CSVs that drive the historical simulations. `version.json` drives the in-app "update available" banner.

## Running & testing

```bash
# Local dev — open directly, or serve so the data/*.csv fetches work:
python3 -m http.server 8080   # → http://localhost:8080
start index.html              # Windows, file:// (CSV fetch skipped, embedded fallback used)
```

There is **no test framework**. The test suite is `runValidation()` (section 13, ~V1–V37+), which runs automatically on every page load and prints PASS/FAIL to the browser console. A failure flags a regression in the CPF / FIRE / portfolio / Monte-Carlo math. **Add a new V-test for any non-trivial calculation you introduce**, and check the console after any change to the math modules.

## Architecture

`index.html`'s `<script>` is organised into 14 numbered sections (search `════` dividers). The key split:

- **Pure-math modules** (sections 1–5): `CONSTANTS`, `Util`, `CPF`, `FIRE`, `Mortgage`, plus `Portfolio` and `Strategies`. IIFE module pattern, plain functions, no DOM access. This is where the financial logic lives.
- **State + persistence** (section 6): `DEFAULT_STATE`, `STATE`, `STATE_VERSION`, `migrateState`, `Storage`.
- **UI layer** (sections 7–11): `setState`/`setStructural` actions, input helpers (`fMoney`, `fNum`, `fSelect`, `fSliderPct`, `fCheck`…), `render*` section renderers, the `refreshComputed()` live-patch layer, and `computeAll()` + results renderers.

Keep this split — math in the modules, DOM only in `render*`/`refresh*`.

### State flow (the non-obvious part)

All mutations go through one of three paths, and **which one you pick determines what re-renders**:

| Function | Persists | Re-runs math | Re-renders cards? | Use when |
|---|---|---|---|---|
| `setState(path, value)` | yes | `recompute()` + `refreshComputed()` | **No** | a plain input changed; derived displays update via live DOM-patching only |
| `setStructural(path, value, ...renderFns)` | yes | + the renderFns you pass | only the ones you pass | a change alters the *shape* of a card (e.g. toggling a mode) |
| `setExpenseMode(basePath, mode)` | yes | calls `renderExpenses()` | the expenses card | the dedicated expense-mode toggle |

- `recompute()` = `syncHousingDerived()` + `computeAll()` + render status bar / results / milestones / yoy / safety buffer. It does **not** re-render the input cards.
- **`refreshComputed()` is the canonical place to make a derived label/hint/attribute react to a `setState` change without re-rendering (and thus without losing input/slider focus).** It patches elements by `id`. When you need a hint or computed span to track another input live, give it a stable `id` and patch it here — don't reach for a full re-render. (Example: the P1-Bridge age hint and disabled state are patched here as FIRE age changes.)
- `renderAll()` (section 11 end) is the full rebuild, called on load / import / reset.

### Domain model: the 3-phase FIRE timeline

The whole planner is structured around CPF-driven age boundaries. Expenses, returns, and drawdown are all modelled per phase:

- **Pre-FIRE**: today → FIRE age
- **P1 Bridge**: FIRE age → 55 (pure portfolio; no CPF access yet)
- **P2 Mid-gap**: 55 → 65 (OA lump-sum unlocked at 55)
- **P3 Longevity**: 65+ (CPF LIFE payouts begin)

If FIRE age ≥ 55, P1 collapses (there is no bridge). Code that touches phases must handle this.

The CPF engine (section 3) models 2026 published rates: OW/AW ceilings, age-banded contribution rates, tiered extra interest, BHS overflow, SA closure at 55, RA top-up to FRS/BRS, OA-at-55 withdrawal, and CPF-LIFE drawdown from 65. Treat rate constants as primary-sourced — cite CPF/MOM/SingStat when changing them.

### Data layer

`DataLoader` (section 0) fetches the five CSVs from `raw.githubusercontent.com` at runtime and caches them in `localStorage`. Each CSV also has an **embedded copy compiled into `index.html`** as offline fallback. **When you change a data series you must update both** the CSV in `data/` *and* the embedded constant in `index.html`. The portfolio UI is column-driven — adding an asset-class column to `assetClasses.csv` + `series.csv` makes a new allocation slider appear automatically. See `data/README.md` for schemas and provenance.

### Persistence & sharing

State lives only in the browser: `localStorage` + the URL hash. `Storage.load()` priority is **URL hash > localStorage > defaults**. Share links encode the full `STATE` into the hash via LZ-string (`#s2=…`); the legacy uncompressed base64 format (`#s=…`) is still decoded for old links — keep it working.

## Versioning & release discipline

Three things must stay in lockstep on every release: the `VERSION` constant in `index.html` (search `const VERSION =`, ~line 1602), `version.json`, and a new `CHANGELOG.md` entry. The update banner compares bundled `VERSION` against the deployed `version.json`.

- **Patch** (x.y.Z) — bug fix / UI tweak / data extension.
- **Minor** (x.Y.0) — new backwards-compatible feature.
- **Major** (X.0.0) — breaking change to saved-state shape: bump `STATE_VERSION` and add a step to `migrateState()` so old localStorage data and old share links still load.

For *additive* schema changes that don't need a hard version bump, follow the `reconcileLegacy*` pattern (e.g. `reconcileLegacyPortfolioStrategy`) — accept missing/renamed fields and fill defaults — rather than bumping `STATE_VERSION`.

`CONTRIBUTING.md` has the full release + migration playbook.

## Conventions specific to this repo

- **Resist adding dependencies or a build step.** One file, vanilla JS, CDN-only. This is a deliberate constraint.
- LF line endings are enforced for all text files via `.gitattributes` (matters on Windows).
- Inline `on*` handlers call functions exposed on `window` at the bottom of section 14 — if you add a function referenced from rendered HTML, expose it there.
