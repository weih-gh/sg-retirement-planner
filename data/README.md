# Data sources

Five CSV files feed the planner's historical simulations and asset-class assumptions. They're loaded at runtime from `raw.githubusercontent.com` by `DataLoader` in `index.html`. When the network fetch fails (offline, repo unreachable, browser cache miss), the embedded copy compiled into `index.html` is used as a fallback — so the app is functional without these files, but live updates are picked up only when they're present and fetchable.

The DataLoader caches each fetched CSV in `localStorage` and reuses it on subsequent visits unless a manual **Refresh** is clicked or the cache key changes.

---

## File inventory

| File | Shape | Purpose |
|---|---|---|
| `assetClasses.csv` | 6 rows × 5 cols | Asset-class universe + forward-looking real-return / stdev assumptions |
| `series.csv` | 55 rows (1970–2024) × 7 cols | Annual real returns per asset class, used by Block Bootstrap / Historical Replay |
| `usLongSeries.csv` | 97 rows (1928–2024) × 4 cols | Annual real returns for 3 simplified asset buckets; used by US Long Historical mode |
| `cpiUS.csv` | 97 rows (1928–2024) × 2 cols | US CPI, deflates US Long Historical returns |
| `cpiSG.csv` | 55 rows (1970–2024) × 2 cols | SG CPI, deflates `series.csv` returns |

---

## File schemas

### `assetClasses.csv`

```
key,label,realReturn,stdDev,category
usEquity,US Equities,0.065,0.165,equity
intlEquity,Intl Equities (ex-US),0.045,0.175,equity
intlBonds,Intl Bonds,0.015,0.075,bond
sgEquity,SG Equities (STI),0.04,0.185,equity
commodities,Commodities,0,0.195,alt
cash,Cash / Short-term,-0.005,0.02,cash
```

- `key` — programmatic identifier (must match column headers in `series.csv`)
- `label` — display name used in the portfolio allocation sliders
- `realReturn` — long-run expected real (CPI-adjusted) return, expressed as decimal
- `stdDev` — annualised standard deviation of real returns
- `category` — one of `equity`, `bond`, `cash`, `alt`. Drives glidepath logic (which categories get shifted away from equity into bonds/cash in later phases).

### `series.csv` (1970–2024)

```
year,usEquity,intlEquity,intlBonds,sgEquity,commodities,cash
1970,0.0356,-0.113,0.1289,0.05,0.1135,0.065
...
```

Annual total returns per asset class. Used by Block Bootstrap (resamples random N-year blocks) and Historical Replay (walks every possible start year). Returns are nominal — `cpiSG.csv` is used to deflate to real terms inside the simulation.

### `usLongSeries.csv` (1928–2024)

```
year,equity,bonds,cash
1928,0.4381,0.0084,0.0308
...
```

Long-history nominal returns for three simplified asset buckets. Used by US Long Historical mode (97 years of rolling windows, includes 1929 crash, stagflation, GFC). When this mode is active, the user's multi-asset weights are collapsed onto these three buckets: all equity weights → `equity`, all bond weights → `bonds`, all cash weights → `cash`.

### `cpiUS.csv` / `cpiSG.csv`

```
year,rate
1970,0.004
1971,0.019
...
```

`rate` = annual CPI inflation, expressed as decimal. Used to convert nominal returns in the series files to real returns inside the simulation engines.

---

## Provenance

| Series | Source |
|---|---|
| US equity (1928–2024) | Damodaran historical data: S&P 500 total return |
| US bonds (1928–2024) | Damodaran historical data: 10-yr Treasury total return |
| US cash (1928–2024) | Damodaran historical data: 3-mo Treasury bill yield |
| Intl equity (1970–2024) | MSCI EAFE Net Total Return USD (MSCI Inc.) |
| Intl bonds (1990+) | Bloomberg Global Aggregate ex-USD |
| Intl bonds (1970–1989) | US 10Y Treasury proxy (approximation — original series unavailable) |
| SG equity (1998+) | STI Total Return Index SGD |
| SG equity (1970–1997) | STI price index + ~3% dividend approximation |
| Commodities (1970–2024) | S&P GSCI Total Return USD |
| US CPI | US BLS Consumer Price Index (urban consumers) |
| SG CPI | Singapore Department of Statistics (SingStat) Consumer Price Index |

Approximated series are clearly noted — they're used because the original underlying indices weren't reported in those years. Replace with primary-source data if available.

---

## Updating the files

When new annual data is published (typically January/February for the prior calendar year):

1. **CPI**: download the latest annual CPI from BLS (US) or SingStat (SG). Append a new row to `cpiUS.csv` / `cpiSG.csv` with the new year + decimal rate.
2. **`series.csv` / `usLongSeries.csv`**: append a new row with the year's total return for each asset class (decimal).
3. **`assetClasses.csv`**: rarely changes — only if forward-looking return / stdev assumptions are revised, or a new asset class is added.
4. Bump `version.json` at the repo root and `VERSION` constant in `index.html` (use the same string).
5. Update `CHANGELOG.md` with a short entry noting which data series were extended.
6. Commit + push to `main`. GitHub Pages redeploys automatically (~1 min). Users see the update banner on their next visit.

---

## Adding a new asset class

If you want to add e.g. REITs as a tracked asset class:

1. Add a row to `assetClasses.csv`: `reits,REITs (Global),0.045,0.190,alt`
2. Add a `reits` column to `series.csv` filled with annual returns for the years available. The DataLoader will warn but not fail if some years are missing (those years treat the new asset as 0%).
3. Bump VERSION + version.json + CHANGELOG.
4. Users will see the new asset class appear in the Portfolio Allocation sliders automatically — the UI is column-driven.

`usLongSeries.csv` is intentionally not extended for new asset classes; it's a 3-bucket simplification for the deepest-history mode only.
