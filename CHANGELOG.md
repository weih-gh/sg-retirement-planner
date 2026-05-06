# Changelog

All notable changes to the SG Retirement Planner are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/):
- **MAJOR** — breaking schema change (old shared URLs may need migration)
- **MINOR** — new feature, backwards compatible
- **PATCH** — bug fix or UI improvement

---

## [1.0.0] — 2026-04-27 🎉 Initial release

### Core planner
- Single and couple planning mode
- Combined or separate household finances for couples
- Three-phase unified projection for combined couples with different retirement ages:
  - Phase 1: Both working → contributions from both incomes
  - Phase 2: One retired, one working → working partner subsidises household
  - Phase 3: Both retired → full drawdown with CPF LIFE offsets

### CPF engine
- 2026 MOM age-banded contribution rates (employee + employer)
- S$8,000/month ordinary wage ceiling + Additional Wage (AW) ceiling (S$102,000 − OW)
- Age-banded OA/SA/MA allocation ratios (7 bands)
- CPF interest accrues monthly on lowest balance, credited annually on 31 December
- SA closes at 55 — excess SA → OA, excess OA → investable portfolio
- Retirement Account (RA) created at 55 from SA first, then OA top-up
- BRS/FRS/ERS selection with editable BRS (default S$110,000)
- CPF LIFE annual payout locked from RA balance at exact age 65

### Projections
- Monthly compounding for all portfolio, CPF, mortgage and income calculations
- First-year savings pro-rated from current calendar month
- Real (inflation-adjusted) returns using Fisher equation
- Cash buffer drawdown strategy in retirement (3-month minimum, 12-month top-up target)
- Proper mortgage amortisation from user-supplied rate, balance and tenure
- Employee CPF correctly deducted from gross income before savings calculation

### Analysis tabs
- **Scenario analysis** — overlay a second drawdown against base projection
- **Monte Carlo** — 200/500/1000 simulations, log-normal returns, p10–p90 bands, survival probability
- **Formulas & methodology** — full documentation of every calculation

### FIRE status
- Per-household FIRE banner with earliest FIRE age (monthly precision via interpolation)
- Per-person banners for couples with different retirement ages
- 5 key metrics: FIRE number, years to FIRE, portfolio at retirement, monthly withdrawal at SWR, CPF LIFE payout

### Milestones
- LeanFIRE, CoastFIRE, Full FIRE, Half FIRE
- 1× income saved, 5× expenses saved, investment income vs expenses
- CPF FRS and ERS progress bars

### App features
- Shareable URLs — full plan state base64-encoded in URL hash, no server required
- Version tracking — version badge in header, update notification banner
- Schema migration framework for future version upgrades
- Auto-save to localStorage (600ms debounce)
- Export inputs as JSON, import from JSON
- Download year-by-year projection as CSV
- Collapsible sections with state persistence
- Dark mode — manual toggle + system preference detection, persisted to localStorage
- Mobile responsive — 2-column grid collapse, scrollable tabs, touch-friendly inputs
