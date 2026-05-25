# SG Retirement Planner

A Singapore-focused retirement planning tool. Models CPF (OW + AW + age-band rates + tiered interest + SA closure at 55 + CPF LIFE drawdown), 3-phase FIRE targeting, Monte Carlo simulation with multiple withdrawal strategies, glidepath portfolio modelling, and lifetime net-worth projection. Single HTML file — no build step, no backend, no dependencies beyond Chart.js (CDN).

🔗 **Live:** https://weih-gh.github.io/sg-retirement-planner/

---

## Features

### Planning
- Single & couple modes (combined or separate finances)
- 3-Phase Present Value FIRE target (Bridge / Mid-gap / Longevity) with CPF LIFE + OA-at-55 offsets
- Live status tiles: FIRE Number, Years to FIRE, Portfolio at Retirement, Monthly Withdrawal
- Net-worth-over-a-lifetime chart with median Monte Carlo trajectory + horizontal FIRE-target reference
- 13 milestones (Income / FIRE / CPF) with progress badges
- Derived hints: "save $X/mo more, OR retire N yr later"

### CPF engine (2026 rates)
- OW ceiling (S$8,000/mo from 2026) + AW ceiling (S$102,000 − annual OW)
- Age-banded employee + employer contribution rates (7 bands)
- Tiered extra interest: +1% on first $60k (<55), +2%/+1% on first/next $30k (55+), OA portion capped at $20k
- BHS cap on MA with overflow to SA (pre-55) or RA (post-55)
- SA dissolution at 55, RA top-up from SA → OA up to FRS (or BRS if pledging property)
- OA lump withdrawal at 55 + auto-transfer of post-55 OA contributions
- RA depletion from 65 via CPF LIFE annual payouts (no interest credit post-65 — pooled)
- Housing-via-CPF: OA debited monthly for mortgage, shortfall (depletion / post-tenure) auto-routed to cash expenses

### Simulation
- **Monte Carlo** — 1000-trial default with per-phase blended returns from your allocation
- **Block Bootstrap** — resamples random N-year blocks from 1970–2024 multi-asset history
- **Historical Replay** — walks every possible start year of 1970–2024
- **US Long Historical** — 97-year US-only series (1928–2024)
- 6 withdrawal strategies: Fixed Expenses, Fixed % (SWR), Guyton-Klinger Guardrails, Bucket, Rising Equity Glidepath, CAPE-adjusted, Floor + Upside
- Safety buffer multiplier with auto-target ("solve for 95% success rate")
- Stress test toggle: suppress CPF help (OA + CPF LIFE) for pure-portfolio drawdown

### Portfolio
- 6 asset classes (US/Intl equities, intl bonds, SG equities, commodities, cash) — universe configurable via `data/assetClasses.csv`
- Three allocation strategies: **Hold steady** (one allocation for life), **De-risk** (configurable glidepath: P2 equity reduction, P3 equity reduction), **Custom per phase**
- Blended real/nominal return + σ per allocation, auto-computed from `assetClasses.csv`

### Privacy & sharing
- **All data stays in your browser** (`localStorage` + URL hash)
- **Shareable URLs** — click Share to copy a self-contained link encoding your full plan. Recipient sees your inputs without any server involvement.
- Auto-save with 600ms debounce
- Export inputs as JSON, import from JSON, CSV download of year-by-year CPF projection

---

## Running locally

```bash
# Simplest — open the file
open index.html   # macOS / Linux
start index.html  # Windows

# Or run a local server (recommended so the CSV fetches work cleanly)
python3 -m http.server 8080
# → http://localhost:8080
```

---

## Hosting on GitHub Pages

Already deployed at https://weih-gh.github.io/sg-retirement-planner/.

To re-enable from scratch:
1. Repo settings → **Pages** → Source: **Deploy from a branch** → `main` / `/` (root)
2. Wait ~1 minute. Site goes live at `https://<user>.github.io/<repo>/`.
3. Every push to `main` redeploys automatically.

---

## Releasing a new version

1. Bump `VERSION` in `index.html` (search `const VERSION =`).
2. Bump `version.json` at the repo root to match.
3. Add an entry to `CHANGELOG.md`.
4. Commit + push to `main`.
5. GitHub Pages redeploys in ~1 minute. Users see an "Update available" banner on their next visit (fetched from `version.json`, compared to bundled VERSION).

### Patch / minor / major

- **Patch** (e.g. 1.1.0 → 1.1.1) — bug fix or UI tweak, no schema change.
- **Minor** (e.g. 1.1.0 → 1.2.0) — new feature, backwards compatible.
- **Major** (e.g. 1.1.0 → 2.0.0) — breaking schema change. Bump `STATE_VERSION` and add a migration in `migrateState()`. Old saved states + old shared links keep working via the migration.

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the full release + migration playbook.

---

## How shareable URLs work

Click **Share** → the current state is base64-encoded into the URL hash:

```
https://weih-gh.github.io/sg-retirement-planner/#s=eyJtb2RlIjoic2luZ2x...
```

- No backend, no database — the link IS the data.
- Hash fragments are never sent to servers (browser-side only).
- Recipient opens the link → state loads into their browser as a starting point. They can play with it without affecting your data.
- For very large plans (≥ ~1,800 chars encoded), consider using **Export** instead of Share — produces a JSON file you can email/attach.

---

## Data sources

Five CSV files in [`data/`](./data/) drive the historical simulations and asset-class assumptions:

| File | Coverage |
|---|---|
| `assetClasses.csv` | 6 asset classes with forward-looking return / σ / category |
| `series.csv` | 1970–2024 annual nominal returns for all 6 asset classes |
| `usLongSeries.csv` | 1928–2024 annual nominal returns for 3 simplified buckets |
| `cpiUS.csv` | 1928–2024 US CPI |
| `cpiSG.csv` | 1970–2024 SG CPI |

See [`data/README.md`](./data/README.md) for schemas, source provenance, and update workflow.

---

## Project structure

```
sg-retirement-planner/
├── index.html              # Entire app (HTML + CSS + JS)
├── version.json            # Drives the in-app "update available" banner
├── data/
│   ├── README.md           # CSV schemas + source provenance
│   ├── assetClasses.csv
│   ├── series.csv
│   ├── usLongSeries.csv
│   ├── cpiUS.csv
│   └── cpiSG.csv
├── README.md               # This file
├── CHANGELOG.md            # Version history
├── CONTRIBUTING.md         # Release process + state migration playbook
└── LICENSE                 # MIT
```

---

## Contributing

Bug reports, methodology corrections, and PRs welcome via [Issues](https://github.com/weih-gh/sg-retirement-planner/issues). See [CONTRIBUTING.md](./CONTRIBUTING.md) for the development workflow.

---

## Disclaimer

For illustrative and planning purposes only — not financial advice.
CPF figures use 2026 published rates. Forward-looking return assumptions are estimates, not guarantees.
Consult a licensed financial adviser before acting on these projections.
