# Contributing

The planner is a single static HTML file (`index.html`) plus five CSV files in `data/`. No build step, no package manager, no test framework — just open the file in a browser.

---

## Local development

```bash
git clone https://github.com/weih-gh/sg-retirement-planner.git
cd sg-retirement-planner

# Option 1 — just open it
start index.html       # Windows
open index.html        # macOS / Linux

# Option 2 — local server (recommended so the data/*.csv fetches work)
python3 -m http.server 8080
# → http://localhost:8080
```

The browser console runs a suite of `runValidation()` self-tests on load (V1–V37+) and prints results to the console. Any failure flags a regression in CPF / portfolio / MC math. Fix or update the failing test before committing.

---

## Release workflow

### Patch — bug fix / UI tweak (1.1.0 → 1.1.1)

1. Edit the fix in `index.html`.
2. Bump `VERSION` constant (search `const VERSION =` near the top).
3. Update `version.json` to match: `{ "version": "1.1.1", "released": "YYYY-MM-DD" }`.
4. Append an entry to `CHANGELOG.md` under a new `## [1.1.1]` heading.
5. Commit + push to `main`:
   ```bash
   git add index.html version.json CHANGELOG.md
   git commit -m "fix: correct OA interest credit at age 55"
   git push
   ```
6. GitHub Pages redeploys automatically (~1 minute). Users see the update banner on their next visit.

### Minor — new feature, no schema change (1.1.0 → 1.2.0)

Same as patch, but bump the second segment of `VERSION` and `version.json`. Use a `feat:` commit prefix.

### Major — breaking schema change (1.1.0 → 2.0.0)

A breaking change means the JSON shape of saved state changes incompatibly. Old `localStorage` data and old shared URLs must be migrated, or they'll fail to load.

1. Bump `VERSION` to `2.0.0`, bump `STATE_VERSION` to the next integer (e.g. 4).
2. Update `version.json` to match.
3. Add a migration function in `index.html` (search for `migrateState`). Follow the existing pattern:
   ```js
   function migrateState(snap) {
     if (!snap.version) snap.version = 1;
     while (snap.version < STATE_VERSION) {
       if (snap.version === 1) { /* v1 → v2 migration */ snap.version = 2; }
       if (snap.version === 2) { /* v2 → v3 migration */ snap.version = 3; }
       if (snap.version === 3) {
         // NEW: v3 → v4 migration
         // e.g. rename field, reshape array, fill defaults
         snap.version = 4;
       }
     }
     return snap;
   }
   ```
4. Test:
   - Open the planner in v1.1.0, save some state.
   - Switch branches to the v2.0.0 branch (or load the new index.html locally).
   - Refresh — confirm the migration runs silently and the state loads correctly.
   - Test with a shared URL from v1.1.0 — same expectation.
5. Append a CHANGELOG entry under `## [2.0.0]` with a **Breaking changes** subsection.
6. Commit with a `feat!:` or `BREAKING CHANGE:` footer per Conventional Commits.

The `reconcileLegacy*` functions in `index.html` (e.g. `reconcileLegacyCpfHelpFlag`, `reconcileLegacyPortfolioStrategy`) are examples of additive migrations that don't bump `STATE_VERSION` — they accept missing/renamed fields and fill defaults. Prefer this pattern when you can avoid a hard schema bump.

---

## Updating historical data

CSV data lives in `data/` and is fetched by `DataLoader` at runtime. The embedded copies inside `index.html` are the offline fallback.

When BLS / SingStat publish a new annual CPI (typically Jan-Feb for the prior year), or when an asset-class total-return series gets a new year:

1. Append the new row to the relevant CSV.
2. **Also** update the corresponding embedded data block in `index.html` (search the constants section for `cpiUS` / `cpiSG` / `series` / `usLongSeries` arrays).
3. Bump `VERSION` (typically patch — data updates don't break anything).
4. Append a CHANGELOG note like "Extended `cpiSG.csv` and `series.csv` through 2025."
5. Commit + push.

See [`data/README.md`](./data/README.md) for source provenance and column schemas.

---

## Coding conventions

- **One file**, no build step. Resist adding bundlers, transpilers, or external dependencies beyond Chart.js (already loaded from CDN).
- **Comments explain *why*, not *what*** — the code is readable; the design decisions aren't.
- **Functions over classes** — the codebase uses module patterns and plain functions.
- **State mutations** go through `setState(path, value)` (debounced auto-save) or `setStructural(path, value, ...rerenders)` (immediate re-render).
- **Pure math** lives in the `FIRE`, `Portfolio`, `CPF`, `Strategies`, `Mortgage` modules. UI rendering lives in `render*` functions. Keep the split.
- **Validation tests** (V1–V37+ in `runValidation`) document expected behavior — add a new test for any non-trivial calculation you introduce.

---

## Reporting issues

File an issue at https://github.com/weih-gh/sg-retirement-planner/issues with:
- What you were doing
- What you expected to see
- What you actually saw
- (Optional) The base64-encoded URL hash from your browser — lets us reproduce your exact state

Methodology corrections (e.g. "OW ceiling for 2027 is X", "CPF LIFE basic plan factor is Y") are especially welcome — please cite the primary source (CPF / MOM / SingStat).
