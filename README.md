# SG Retirement Planner

A Singapore-focused retirement planning tool. Single HTML file — no build step, no server, no dependencies beyond Chart.js.

🔗 **Live:** `https://YOUR-USERNAME.github.io/sg-retirement-planner/`

---

## Features

- CPF engine with 2026 rates, AW ceiling, tiered interest, SA closure at 55, CPF LIFE
- Three-phase projection for couples (both working → one retired → both retired)
- Monte Carlo simulation for sequence-of-returns risk
- Scenario analysis with side-by-side drawdown comparison
- Shareable URLs — full plan state encoded in the URL hash, no backend needed
- Dark mode, mobile responsive, auto-save to localStorage

See [CHANGELOG.md](./CHANGELOG.md) for the full feature list.

---

## Running locally

```bash
# Option 1 — open directly
open index.html

# Option 2 — local server (recommended)
python3 -m http.server 8080
# → http://localhost:8080

# Option 3 — Node
npx serve .
```

---

## Hosting on GitHub Pages

### First-time setup

1. Create a new GitHub repository (e.g. `sg-retirement-planner`)
2. Push this folder:

```bash
cd retirement-planner
git init
git add .
git commit -m "feat: v1.0.0 initial release"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/sg-retirement-planner.git
git push -u origin main
```

3. Go to **Settings → Pages → Source → Deploy from branch → main / root**
4. Wait ~60 seconds → live at `https://YOUR-USERNAME.github.io/sg-retirement-planner/`

---

## Releasing a new version

### Patch (bug fix / UI tweak) — e.g. 1.0.0 → 1.0.1

```bash
# 1. Edit index.html — change VERSION constant near top of <script>
const VERSION = '1.0.1';

# 2. Add entry to CHANGELOG.md

# 3. Commit and push
git add index.html CHANGELOG.md
git commit -m "fix: correct CPF interest calculation"
git push
```

GitHub Pages redeploys automatically in ~60 seconds. All users see the update immediately.

### Minor (new feature) — e.g. 1.0.0 → 1.1.0

```bash
const VERSION = '1.1.0';
git commit -m "feat: add rental income field"
```

### Major (breaking schema change) — e.g. 1.0.0 → 2.0.0

A breaking change means old shared URLs have a different data structure and need migration.

```bash
# 1. Bump version
const VERSION = '2.0.0';

# 2. Add migration in migrateState() in index.html
function migrateState(snap, fromVersion) {
  if (!fromVersion || compareVersions(fromVersion, '2.0.0') < 0) {
    snap.people.forEach(p => {
      // Example: rename additionalWages to annualBonus
      if (p.additionalWages !== undefined) {
        p.annualBonus = p.additionalWages;
        delete p.additionalWages;
      }
    });
    snap.schemaVersion = '2.0.0';
  }
  return snap;
}

# 3. Test an old shared URL loads correctly with migration applied
# 4. Commit
git commit -m "feat!: rename income fields (breaking schema v2.0.0)"
```

---

## How shareable URLs work

Clicking **Share** encodes the full plan into the URL hash using base64:

```
https://your-username.github.io/sg-retirement-planner/#v1.0.0/eyJtb2RlIjoic2luZ2xlIi...
```

- **No server required** — data lives entirely in the URL
- **Recipient loads the exact inputs** — nothing is stored server-side
- **Version-aware** — `migrateState()` upgrades old URLs when schema changes
- **Update banner** — users loading an old-version URL see a notification

---

## Project structure

```
sg-retirement-planner/
├── index.html      # Entire application (HTML + CSS + JS, ~1,800 lines)
├── CHANGELOG.md    # Version history
├── README.md       # This file
└── .gitignore
```

---

## Disclaimer

For illustrative and planning purposes only — not financial advice.
CPF figures are estimates based on 2026 published rates.
Consult a licensed financial adviser before making investment or retirement decisions.
