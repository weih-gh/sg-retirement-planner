# Changelog

All notable changes to the SG Retirement Planner are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/):
- **MAJOR** — breaking schema change (old shared URLs may need migration)
- **MINOR** — new feature, backwards compatible
- **PATCH** — bug fix or UI improvement

---

## [1.4.2] — 2026-05-30

### Fixed
- **"Years to FIRE" tile no longer shows "0 mo / Already FIRE-ready" misleadingly.** When the portfolio has already crossed the planned-retirement FIRE target (e.g. $772k sized for retiring at 53→88) but the user hasn't reached their retirement age yet, the tile now computes and shows the **estimated earliest retirement age** — the first age at which the projected portfolio would genuinely sustain retirement sized for *that* age. Previously, having $800k at age 32 when the age-53 FIRE target is $772k produced "0 mo / Already FIRE-ready", incorrectly implying the user could retire today (they can't — a 32→88 retirement needs a much larger portfolio). The tile now shows e.g. "6 yr · Est. Earliest FIRE · age 38". "Est." is shown because CPF LIFE and OA lump values are held at their planned-retirement-age levels for performance (exact values at earlier retirement ages depend on CPF contribution years, which aren't re-simulated per candidate age).

### Technical
- New helper `_earliestFireYears(R, primaryCurrentAge)` placed alongside `_yearsToFireFromYoy`. Rescans `R.yoy` with a per-candidate-age FIRE target (recomputed via `R.fire.assumptions` with a substituted `FIRE_AGE`). Mortgage end age is correctly held constant; only the post-retirement years-remaining changes. No engine changes.

---

## [1.4.1] — 2026-05-30

### Changed
- **"Simple" label** — the two-phase expense mode button was relabelled from "2-Phase" to "Simple" for clarity.
- **Expense profile toggle styling** — the toggle is now wrapped in a `config-block` container, matching the visual style of the "Planning for" and "Household finances" toggles elsewhere in the app. Fixes the label font (now uppercase/muted, consistent with all other section labels) and the layout (label above toggle, not floating beside it inline).
- **P1 Bridge disabled when FIRE ≥ 55** — if the user's target retirement age is 55 or older there is no Bridge phase (CPF OA is already unlocked at retirement), so the P1 Bridge input is now greyed out with a "N/A — FIRE ≥ 55" hint. The disabled state is live-patched in `refreshComputed` so it updates immediately when the FIRE age slider is moved, without losing input focus on other fields.
- **P1 Bridge hint is dynamic** — the hint text "Age X–54" updates live whenever the user changes their Target retirement age, via the same `refreshComputed` patch path.
- **Removed footnote** — the "All in today's $ — the engine inflates each phase…" explanatory note below the custom-phase grid has been removed. The field labels and hints are self-explanatory; the blurb was noise.

---

## [1.4.0] — 2026-05-30

### Added
- **Custom per-phase expense profile.** The Expenses & Savings card now has a "2-Phase / Custom per phase" toggle. Custom mode replaces the single "Retirement Annual Expenses" field with three phase-specific inputs that mirror the engine's 3-Phase FIRE model:
  - **Pre-FIRE** — today → FIRE age (accumulation years)
  - **P1 Bridge** — FIRE age → 55 (CPF OA unlocks at 55)
  - **P2 Mid-gap** — 55 → 65 (before CPF LIFE)
  - **P3 Longevity** — 65+ (CPF LIFE active)
  
  Lets users model the classic "go-go / slow-go / no-go" spending curve (higher P1 spend for travel/active years, lower P3 for slowing-down years), or a "healthcare bump" pattern (lower P2, higher P3), or any pattern in between. All values entered in today's $ — the engine handles per-year inflation indexing within each phase, matching how the single "Retirement" field has always worked.

- **Switching modes seeds the new fields from the old.** Toggling from 2-Phase → Custom copies `Now → Pre-FIRE` and `Retirement → P1 = P2 = P3` (flat baseline), so users start from their existing intent and tweak from there.

- **Couple + Separate Finances:** each person owns their own per-phase profile (each card has its own toggle). The engine sums per-year expenses across both partners using their respective FIRE ages as the phase boundary clocks. Combined mode keeps a single household-level profile (matching how Combined treats other fields).

### Changed
- **`FIRE.calculateMultiPhaseTarget`** now accepts optional `E_p1`, `E_p2`, `E_p3` overrides. When omitted (default), all three default to `E` and behaviour is identical to v1.3.x.
- **`runMonteCarloSimulation` + `_runHistoricalTrajectory`** accept an optional `expenseProfile` ctx parameter. When provided, the per-year base withdrawal switches across the 55 / 65 phase boundaries; when absent (or in two-phase mode where the profile is flat), the behaviour is identical to v1.3.x.
- **`computeYearlyProjection`** uses per-year expense from the profile; in two-phase mode the profile is flat (preFire = Now, P1=P2=P3 = Retirement) so YoY output is unchanged.
- **LeanFIRE stays as a single number** — it's a stress-test target (a floor lifestyle), and a single figure communicates that intent better than per-phase.

### Notes on withdrawal strategies under Custom mode
The per-year baseline applies cleanly to:
- **Fixed Expenses (Inflation-Adjusted)** — each year withdraws that year's phase baseline. Strongest fit for custom-per-phase.
- **Guyton-Klinger guardrails** — the band recomputes per year against the current baseline; rebalancing the spending target at phase transitions is sensible behaviour.
- **Bucket / Floor+Upside** — same as GK; baseline drives the floor each year.

For **Fixed Withdrawal % (SWR)** the per-phase profile has no effect — that strategy is portfolio-driven (`portfolio × SWR`) and ignores the expense baseline. The CPF LIFE / other-income offset still applies normally on top.

### Backward compatibility
- Old saved states (no `expenseMode` / `expensesPerPhase` fields) default to `"twoPhase"` and behave identically to v1.3.x.
- Shared URLs from v1.0–1.3 continue to work — the deep-merge with `DEFAULT_STATE` fills in the missing fields with sensible defaults.
- The validation harness (V1–V40) is unchanged — all existing tests use the single-E signature, which still works.

---

## [1.3.2] — 2026-05-30

### Fixed
- **"Monthly contribution (pre-FIRE)" was overstated when any cash mortgage existed.** Path A (v1.3.0+) instructs users to exclude the full monthly mortgage from the expense input, so the auto-computed savings rate `take-home − expenses` silently ignored the user's real cash mortgage outflow. Users with mortgages paid in cash were seeing an inflated monthly contribution figure (and a downstream optimistic "years to FIRE" estimate).
- **Fix:** `effectiveMonthlyContrib(slot, pk, cashMortgageMonthly)` now accepts an optional cash-mortgage subtraction. `computeAll` and `renderExpenses` both pass each person's `_housingShares.pXCashMonthly` so the displayed and engine-consumed contribution is "true current monthly savings going to portfolio".
- **MC engines now add the cash mortgage back to pre-FIRE contributions after the loan ends.** Since `monthlyContrib` is the today-snapshot (mortgage active), the MC accumulation loop reclaims that cash flow for years where the mortgage has expired but FIRE hasn't been reached yet. Symmetric with the v1.3.1 retirement-side mortgage handling.
- **Breakdown text under the monthly-contribution pill** now shows the mortgage line item: e.g. `(S$6,000 take-home) − S$2,500 expenses − S$632 cash mortgage`.

### ⚠️ Behaviour change — read this if upgrading from 1.3.1

If you have any cash mortgage burden and were relying on the auto-computed monthly contribution:
- **Headline monthly contribution will drop** by your cash mortgage amount (the engine now matches reality).
- **"Years to FIRE" milestone may extend** slightly (you're actually saving less per month than v1.3.1 thought).
- **MC final-portfolio-at-FIRE distributions will shift down** for users whose mortgage extends into accumulation years.

If your mortgage has already ended (tenure < years until FIRE), the post-mortgage years contribute at the gross rate as before — MC handles this per-year correctly via the add-back logic.

### Note on the manual override

If you set "Monthly contribution" to a manual value (rather than auto), it's used as-is. We trust your number. The auto-mode subtraction only fires when you let the engine compute the contribution.

---

## [1.3.1] — 2026-05-30

### Fixed
- **FIRE number was inconsistent between "100% via CPF OA" and "100% via cash" mortgage splits.** Example: a $600k loan / 25yr / 0% deposit case showed FIRE = $2.15M when paid fully via CPF, but $2.01M when paid fully via cash — the cash case was under-counting by ~$140k because the cash mortgage stream wasn't being added to the retirement-year withdrawal need.
- **Root cause:** v1.3.0's Path A fix only applied to `computeYearlyProjection` (the Year-by-Year table). The 3-Phase PV target (`calculateMultiPhaseTarget`) and the Monte Carlo engines (`runMonteCarloSimulation`, `_runHistoricalTrajectory` → bootstrap / historical replay / US long) all kept using `annualExpRet` alone for retirement withdrawals — silently dropping the cash portion of any mortgage that extended past the FIRE age.
- **Fix:** Both layers now accept `cashMortgageAnnual` + `mortgageActiveYearsFromFire` and add the cash mortgage as a fixed-nominal stream during the active-mortgage years post-FIRE:
  - `calculateMultiPhaseTarget` adds the PV of the cash stream, split across Phases 1 / 2 / 3 by which post-FIRE years the loan covers. (PV at FIRE: `Σ M / (1+r)^t` for `t ∈ [0, N)` clipped to each phase's window.)
  - `runMonteCarloSimulation` and `_runHistoricalTrajectory` add `cashMortgageAnnual` to the gross withdrawal need per year while `(age − fireAge) < mortgageActiveYearsFromFire`.
- Per-person separate-finances mode wires each person's own cash share via `_housingShares.p1CashMonthly` / `p2CashMonthly` and their own `fireAge − currentAge` offset.

### ⚠️ Behaviour change — read this if upgrading from 1.3.0

If your mortgage extends into retirement and any portion is paid via cash, your **FIRE number will increase** and your **Monte Carlo success rate will decrease** — both are now correctly accounting for the cash mortgage burden that was previously missed by the Planner-tab status tiles, milestones, lifetime chart, and Safety Buffer simulation. The Year-by-Year table is unchanged (it was already correct in v1.3.0). If your mortgage ends before your FIRE age, no change.

---

## [1.3.0] — 2026-05-30

### Changed
- **Mortgage is now always tracked separately from the expense input.** Previously, the cash portion of the mortgage (the part *not* funded by CPF OA) fell through the cracks — the engine relied on the user to manually include it in their annual expenses. Subtle and easy to miss; if you slid the CPF % down from 100% to 50%, the cash portion of the mortgage would silently disappear from the model unless you remembered to top up your expense input.
- **New engine behaviour (Path A):**
  - **CPF-funded portion** is auto-debited from CPF OA via the CPF engine (unchanged from v1.2.x).
  - **Deliberate cash portion** — the part the user chose *not* to fund via CPF — is now auto-added to annual expenses by `computeYearlyProjection` for every year the mortgage is active. New `cashMortgageAnnual = (p1CashMonthly + p2CashMonthly) × 12` adds to the year's expenses (deflated to real $).
  - **OA depletion shortfall** (when CPF was supposed to pay but OA ran out) continues to route to cash, same as v1.2.x.
- **Expense input model is now unambiguous: always exclude the full monthly mortgage from your expense fields.** Engine handles the rest. The inline hint copy reflects this:
  - With CPF: *"Exclude monthly mortgage ($1,053/mo) — we auto-debit $421/mo from CPF OA and $632/mo from cash expenses."*
  - Without CPF: *"Exclude monthly mortgage ($1,053/mo) — we auto-debit it from cash expenses."*
- The hint fires for **any active mortgage**, not just when CPF % > 0.

### ⚠️ Behaviour change — read this if upgrading from 1.2.x

If you had previously **included** the cash portion of your mortgage in your expense input (as the 1.2.x hint encouraged), your FIRE projection will now show **higher annual expenses** post-upgrade — the engine is also adding the cash portion. **To restore your previous projection, subtract the cash portion of the mortgage from your expense input.** If you had been excluding the full mortgage already (as some users may have done), no change needed.

### Out of scope (deferred)
- Monte Carlo simulation. The MC engine continues to use the user's `annualExpRet` for retirement withdrawals. If a mortgage extends into retirement years, MC may under-count required withdrawals during the active-mortgage period. This is a known limitation of v1.3.0 — track via the Year-by-Year projection on the Planner tab for accurate per-year cash flow during retirement. Will address in a follow-up.

---

## [1.2.5] — 2026-05-30

### Changed
- **Per-person CPF % sliders disable when their share = 0** (couple mode). If the user drags the mortgage-split slider so that You cover 0%, the "Your share via CPF OA" slider becomes greyed out and non-interactive — its value is moot (0% of nothing is nothing). Same for Partner when their share is 0%. The disabled state toggles live on every split-slider drag via `refreshComputed` patching the `disabled` attribute. Existing global CSS `input[type="range"]:disabled { opacity: 0.45; cursor: not-allowed }` provides the visual feedback.

---

## [1.2.4] — 2026-05-30

### Fixed
- **Expense-CPF hint info icon now consistent with other info buttons** — replaced the CSS-generated Unicode `ⓘ` glyph with the same SVG (circle + vertical stem + small dot) that `_infoIcon()` uses throughout the rest of the app. Same stroke width, same proportions, same `var(--accent-2)` tint.

### Hotfix (carried from 1.2.3 release window)
- Removed a stray `}` left over from an earlier rewrite of `renderHousing` that broke JavaScript parsing in v1.2.3 and prevented the whole page from rendering.

---

## [1.2.3] — 2026-05-30

### Changed
- **Housing CPF controls gated behind a toggle.** The Housing & Mortgage section now starts with a single "Pay mortgage via CPF OA?" checkbox (with an info tooltip matching other toggle patterns in the app). Sliders only appear when the toggle is on — clean default state with no zero-value clutter.
- **Toggle defaults are sensible:** enabling sets P1 to 100% via CPF OA (and P2 to 100% too in couple mode). Disabling zeros all per-person CPF % so the engine treats the mortgage as fully cash-funded. The split (% between P1 and P2) is preserved either way — it's orthogonal to CPF/cash mix.
- Removed the **border-top separator above the split slider** — the new toggle gates the whole CPF block, so a separator isn't needed.
- Removed the **footnote** "Property equity adds to net worth but doesn't fund retirement spending unless you downsize" from the Housing card.

### Fixed
- **"You's CPF OA" in the per-person summary text.** The possessive helper was scoped inside `renderHousing` only, so the live summary patched by `refreshComputed` still used `${p1Name}'s CPF OA` — broken when name was the default "You". Hoisted to a top-level `housingPossessive(name)` helper used by both renderers.
- **Info icon now consistent with other info buttons** — moved from inside an `fSliderPct({ info: … })` (rendered as part of a slider label) to a `_infoIcon()` next to the toggle's label, matching the SVG-circle pattern used by the Safety Buffer's CPF-help toggle and other app-wide info tooltips.

---

## [1.2.2] — 2026-05-29

### Changed
- **Expense-exclusion reminder is now an inline hint, not a banner.** Replaces the standalone info banner that sat above the contribution field. The new hint lives directly under the expense input row with a small ⓘ glyph; text adapts based on inputs:
  - No mortgage configured → hidden (no reminder needed).
  - Mortgage exists, no portion via CPF OA → "Exclude monthly mortgage if paid via CPF OA" (educational).
  - Mortgage exists, paid partially or fully via CPF OA → "Exclude monthly mortgage — currently $X/mo via CPF OA" with per-partner breakdown in couple mode.
- Removed the `#oa-callout-banner` markup and its live-patched amount span. The new hint reuses `computeHousingCpfShares()` as the single source of truth for the dollar amount.

---

## [1.2.1] — 2026-05-29

### Fixed
- **Housing CPF split slider was off-by-100.** The new state fields (`housing.split.p1Pct`, `housing.p1.cpfPct`, `housing.p2.cpfPct`) were initialised as percentages (0–100) but `fSliderPct`'s storage convention is fractions (0–1). After the first slider drag, the units flipped — partner % showed 99% instead of 0% at full slider, and `computeHousingCpfShares()` returned per-person CPF debits that were 100× smaller than intended. Fixed by switching the affected state fields, `syncHousingDerived`, `computeHousingCpfShares`, `reconcileLegacyHousingSplit`, and the V40 validation test to fractions throughout.
- **"You's share" label corrected to "Your share."** Added a `possessive(name)` helper that returns "Your" for the default "You", otherwise `{name}'s`. Other partner names (e.g. "Alex's") read naturally.

### Changed
- **Housing UI cleanup (couple mode).** Removed the redundant "Mortgage contributions" sub-heading (replaced with a thin border-top separator). Removed the "Partner covers the rest" hint that duplicated the live partner-pct hint below the split slider. The per-CPF-slider hints now show live computed dollar amounts (e.g. `$1,500/mo via Your CPF OA · $0/mo cash`) instead of generic copy — anchors the abstract percentages in concrete money.
- **Single mode UI** also gains the live `$/mo via CPF OA · $/mo cash` summary line for consistency.
- All housing summary spans are patched in-place by `refreshComputed` (no card re-render) so slider focus is preserved while dragging.

---

## [1.2.0] — 2026-05-29

### Added
- **Shared mortgage payments (couple co-owners)** — both partners can now contribute to the monthly mortgage via CPF OA. Previously only Person 1's OA was debited; this disadvantaged couples where both names are on the property. New controls in the Housing & Mortgage section (couple mode only):
  - **Mortgage split slider** — % of the monthly payment You cover vs Partner (auto-mirrored to sum to 100%).
  - **Per-person CPF / cash mix slider** — each partner sets what fraction of *their* share is debited from their CPF OA. The remainder comes from cash.
  - Examples: 50/50 split + both 100% CPF = each OA debited for half the mortgage. 70/30 split + You 100% CPF + Partner 0% CPF = your OA covers 70%, partner pays 30% cash.
- **Combined OA-callout banner** in Expenses card — shows the total CPF deduction across both partners with a per-person breakdown when both contribute.
- **V40 validation test** — verifies the per-person split math via `computeHousingCpfShares()` for a 60/40 split scenario.

### Changed
- `cpfInputsFor(pk)` now reads per-person `housingDeductionAnnual` from the new state model. Both partners' OA shortfalls (when OA depletes mid-tenure or post-65) automatically route to their own cash via the existing shortfall logic — no new engine code needed.
- `computeYearlyProjection` sums both partners' OA housing shortfalls into the year's cash expenses.
- New helper `computeHousingCpfShares()` — single source of truth for housing CPF mechanics, used by `cpfInputsFor`, `computeYearlyProjection`, `renderExpenses`'s OA callout, and the Housing UI.
- New helper `syncHousingDerived()` — keeps `split.p2Pct = 100 − split.p1Pct` and updates the legacy `payMortgageViaCpf` flag on every recompute (for JSON export back-compat).

### Backward compatibility
- Legacy `housing.payMortgageViaCpf` boolean is preserved in state and round-trips through JSON export / share-link import / localStorage. Old planner versions reading a new snapshot will see the legacy flag accurately reflect "any CPF contribution".
- `reconcileLegacyHousingSplit()` migrates old saved state on load: `payMortgageViaCpf: true` → P1 covers 100% via CPF (preserving the current behaviour exactly).
- Existing single-mode users see no UI change beyond the binary "Monthly payment via CPF OA?" checkbox becoming a 0–100% slider.

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
