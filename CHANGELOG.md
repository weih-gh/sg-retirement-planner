# Changelog

All notable changes to the SG Retirement Planner are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/):
- **MAJOR** — breaking schema change (old shared URLs may need migration)
- **MINOR** — new feature, backwards compatible
- **PATCH** — bug fix or UI improvement

---

## [1.1.2] — 2026-05-26

### Fixed
- **Definitions & Methodology mobile responsiveness** — page no longer requires horizontal scrolling on phone widths. Added `min-width: 0` to `.def-content` so the grid track stops inflating to fit oversized children. Definition-list entries (`.def-kv`) now stack term-above-definition vertically on screens ≤ 700px. Formulas keep their inner horizontal scrollbar without pushing the page layout.

### Changed
- **Glidepath card** rewritten to reflect the three named Portfolio strategies (Hold steady / De-risk / Custom per phase). Removed stale references to "Simple mode" and "Advanced allocation mode" which no longer exist.
- **Glidepath methodology** renamed from "Simple-mode glidepath" to "Glidepath strategy". Notes that `eqFactor` is now user-tunable via the P2 / P3 equity-reduction sliders (defaults 25% / 45%), rather than the previous hard-coded 0.75 / 0.55.
- **Withdrawal Strategies methodology** expanded from 6 to 7 entries — adds the missing **Fixed Expenses (Inflation Adjusted)** strategy (the default) and splits it from **Fixed Withdrawal % (SWR)**. Each entry now also notes whether it uses the Simulation SWR.
- **CPF LIFE payout methodology** now describes the post-65 RA depletion model (RA decrements by annual payout each year, no interest credit — matches the engine's `runningRa1Post65` logic added in v1.1.0).
- **RSTU & MA top-ups card** updated — removed the inaccurate "not exposed in UI" claim. Voluntary top-up fields are wired through the state and tax-relief estimate displays when set.

---

## [1.1.1] — 2026-05-25

### Changed
- **Share-link compression**: URL hashes now use LZ-string (≈70% smaller than the previous base64 encoding). A realistic couple-mode + Separate-finances + Advanced-allocation plan fits comfortably under the ~2,000-char limit that some email clients and chat apps truncate at. New compressed format uses the `#s2=` hash prefix.
- **Backward compatible**: existing `#s=` share links from v1.0.0 / v1.1.0 continue to work indefinitely — the decoder tries the new compressed format first, then falls back to the legacy base64 path. No action required from anyone holding an old shared URL.
- **Graceful degradation**: if the LZ-string CDN fails to load (e.g. offline), the encoder falls back to legacy base64 so Share still produces a working (longer) URL.

### Added
- LZ-string library loaded from jsDelivr CDN (~3 KB, MIT-licensed, sync API).

---

## [1.1.0] — 2026-05-25

### Added
- **Net worth over a lifetime chart** (Results section) — median Monte Carlo trajectory of investable portfolio + CPF (excl. MA) with a dashed retirement-age marker and horizontal FIRE-target reference line. Updates with debounced 400-trial MC.
- **Glidepath portfolio strategy** with three named modes: **Hold steady** (flat allocation), **De-risk** (configurable P2 / P3 equity reductions), **Custom per phase**. Replaces the implicit auto-glidepath that previously activated in Simple mode.
- **Lifetime chart info tooltip** explaining methodology (median of 400 MC trials).
- **CPF FRS / ERS at 55 milestones** — projected RA at 55 vs FRS/ERS escalated to year-of-55, with binary "Will be achieved / Not achieved" status. Previously compared today's combined CPF (including MA) to today's FRS, which silently double-counted MA and ignored escalation.
- **Footer** with version, MIT licence link, GitHub source link, and disclaimer.
- **Update-available banner** — on load, fetches `version.json` from the deployed site and surfaces a dismissible refresh notice when the bundled VERSION is stale.
- **Cache-Control meta tag** (1-hour TTL) so the HTML file picks up updates promptly.
- **`data/` folder** with the five historical-data CSVs (`assetClasses.csv`, `series.csv`, `usLongSeries.csv`, `cpiUS.csv`, `cpiSG.csv`) and a README documenting schemas + source provenance.
- **`version.json`** at repo root drives the update banner.
- **MIT LICENSE** file.

### Changed
- **FIRE status banner** redesigned. Two-card layout when both partners on track (single "You're on track for Financial Independence — $X surplus"), per-person rows when statuses differ. Surplus / shortfall shown as a right-aligned pill.
- **FIRE Number status tile** filled green (matches the Age-locked reference card from the inspiration screenshot), SWR % removed from subtitle, status tiles centred horizontally instead of stretched.
- **Safety Buffer card** — `<h3>` moved inside `.fire-card` so the title gets the compact uppercase styling that matches the Planner's FIRE Target card.
- **Simulation SWR slider** moved out of the global simulation card layout and into the per-strategy lever block for strategies that actually consume it (Fixed Withdrawal %, Guyton-Klinger). Strategies that don't use it no longer carry the stale "Adjust the Simulation SWR above" footer.
- **Data sources status row** repositioned to sit below the Monte Carlo fan chart instead of between mode info and strategy info.
- **Years-to-FIRE tile** on the Deeper Dive tab now uses the YoY scan when buffer = 1.0× so it matches the Planner tab exactly; switches to the continuous-contribution closed-form only for buffer > 1.0× (matching the "work longer past fireAge" framing).
- **Default portfolio allocations** zeroed out — new users start with empty allocations instead of an opinionated 60/40 default.
- **Definitions tab** — CPF extra interest description rewritten to match the actual implementation (was wrongly listing "+1% / +0.5%" tiers).
- **Header tab order** preserved; methodology pages re-organised with Investing first, clickable section headers, and renamed "FIRE / LeanFIRE / FatFIRE" → "FIRE Definitions".
- **N1 / N2 / N3** renamed to **P1 / P2 / P3** for visual consistency with the rest of the UI.
- **CPF breakdown table** gained an OA Withdrawals column (housing payments via OA, with shortfall tracking when OA depletes mid-mortgage).
- **Housing & Mortgage section** moved above Household Expenses for a more intuitive input order.
- **Expenses card** banner — "Exclude housing if paid via CPF OA" callout when the housing-via-CPF toggle is on, with live-patched payment amount.
- **Income card** AW (Additional Wages) routed correctly into monthly contributions; OW ceiling derived dynamically from the latest published year so future ceilings auto-flow without code changes.

### Fixed
- **Post-65 RA depletion** — previously held the RA balance flat at `raAt65` for the rest of life (CPF LIFE income appeared from nowhere). Now correctly subtracts `cpfLifeAnnual` from RA each year from age 65 onward, with no interest credit on the residual balance (matches the actual policy where post-65 interest goes to the risk pool).
- **OA double-count at age 55** — `oaAt55` was captured but `state.oa` was never zeroed, so the post-55 auto-transfer logic re-injected the same balance (plus interest) back into the portfolio. Net worth was overstated by approximately one `oaAt55` for ages 55+.
- **Mortgage-via-CPF shortfall** — when OA depleted before the mortgage tenure ended (or post-65 when CPF tracking stops), the engine silently stopped deducting the mortgage instead of routing the unpaid portion to cash expenses. Now correctly adds the deflated nominal shortfall back to yearly expenses for the affected years.
- **Mortgage tenure honored in CPF engine** — OA was previously debited for the housing payment through age 65 regardless of remaining tenure. Capped to actual remaining tenure, with the post-55 auto-transfer no longer reserving a housing buffer once the mortgage is paid off.
- **Combined-couple FIRE banner** read different `m.projected` values than the Portfolio at Retirement tile, leading to "$828k surplus" in the banner but "$588k above target" in the tile for the same scenario. Both now derive from the YoY single source of truth.
- **Retirement horizon display** showed total years from currentAge (including accumulation) instead of retirement-only years (`lifeExp − fireAge + 1`).
- **`renderResults` couple modes** — separate-couple branch now also renders the lifetime chart.
- **Mobile responsiveness** — status tile grid, hero action buttons, share/export overflow, info tooltip tap-to-toggle, iOS-16px input zoom fix.

### Developer
- **`DATA_SOURCES.repo`** populated to `weih-gh/sg-retirement-planner` so the DataLoader fetches CSVs from this repo's `data/` folder by default. Embedded copies in `index.html` remain as offline fallback.
- **`VERSION` constant** drives the header pill, footer text, and update-banner comparison.
- **`CONTRIBUTING.md`** added with release workflow and state-migration playbook.

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
