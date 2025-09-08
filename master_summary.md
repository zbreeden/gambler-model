# The Gambler → Signal Broadcast Pack (Top‑3 Edges)

This pack wires a daily **Top‑3 edges** broadcast from **The Gambler** into your **Signal** dashboard. It assumes you’ll later swap in real data sources; for now it runs with a toy Monte Carlo core that you can enhance.

---

## 0) Repo layout (suggested)

```
signal-model/
├─ data/
│  └─ broadcasts/
│     └─ gambler/
│        ├─ top3_2025-09-08.json
│        └─ latest.json  ← symlink or copy of the most recent
├─ scripts/
│  └─ gambler_scan.py
├─ seeds/
│  ├─ tags.yml
│  ├─ glossary.yml
│  └─ modules.yml
├─ web/
│  ├─ gambler_top3.js
│  └─ components.html  (optional demo block)
└─ .github/workflows/gambler-scan.yml
```

---

## 1) Seeds

### `seeds/tags.yml` (append)

```yml
- key: gambler_top3
  label: "Gambler: Top‑3 Edges"
  description: "Daily broadcast of three highest‑EV opportunities from The Gambler model vs market totals."
  kind: broadcast
  gloss_ref: gambler
  deprecated: false

- key: betting_edge
  label: "Betting Edge"
  description: "Difference between model probability and market implied probability, used to rank opportunities."
  kind: metric
  gloss_ref: edge
  deprecated: false
```

### `seeds/glossary.yml` (append)

```yml
- key: gambler
  term: "The Gambler"
  definition: >
    Portfolio module that estimates outcome probabilities (e.g., Over/Under hit rates) using a simulation layer
    and flags edges vs. market lines. Outputs ranked picks for broadcasts.
  examples:
    - "The Gambler broadcasted Top‑3 Edges at 09:30 ET."
  see_also: [signal, edge, simulation, monte_carlo]

- key: edge
  term: "Edge"
  definition: >
    The advantage of the model over the market, often measured as the difference between model probability
    and the market's implied probability at offered odds.
  examples:
    - "Model Over% is 57% vs implied 52.4% → +4.6% edge."
  see_also: [expected_value, probability]

- key: monte_carlo
  term: "Monte Carlo Simulation"
  definition: >
    A method that generates many random outcomes from assumed distributions to approximate probabilities,
    expected values, and uncertainty.
  examples:
    - "Simulated 50k games to estimate Over probability."
  see_also: [simulation]
```

### `seeds/modules.yml` (append or ensure)

```yml
- key: the_gambler
  label: "The Gambler"
  orbit: Delivery & Insight
  emoji: "🎲"
  status: active
  description: "Probabilistic market scanner (totals/spreads) with Monte Carlo simulation, broadcasting daily Top‑3 edges."
```

---

## 2) Broadcast schema

### `data/broadcasts/gambler/top3_YYYY-MM-DD.json` (example)

```json
{
  "broadcast_key": "gambler_top3",
  "date": "2025-09-08",
  "sport": "MLB",
  "generated_at": "2025-09-08T09:30:00-04:00",
  "source": "the_gambler",
  "items": [
    {
      "rank": 1,
      "game_id": "CHC@CIN",
      "start_et": "2025-09-08T18:40:00-04:00",
      "venue": "Great American Ball Park",
      "away": "CHC",
      "home": "CIN",
      "proj_total": 10.1,
      "market_total": 9.0,
      "over_prob": 0.57,
      "under_prob": 0.43,
      "edge_pct": 0.046,          
      "suggested_side": "Over",
      "note": "Wind out; hitter‑friendly park; bullpen fatigue last 3d."
    },
    {
      "rank": 2,
      "game_id": "LAD@SF",
      "proj_total": 7.2,
      "market_total": 7.5,
      "over_prob": 0.45,
      "under_prob": 0.55,
      "edge_pct": 0.036,
      "suggested_side": "Under",
      "note": "Marine layer; strong SP matchup."
    },
    {
      "rank": 3,
      "game_id": "NYY@TB",
      "proj_total": 8.3,
      "market_total": 8.0,
      "over_prob": 0.52,
      "under_prob": 0.48,
      "edge_pct": 0.018,
      "suggested_side": "Over",
      "note": "Neutral park; slight model lean."
    }
  ]
}
```

> `edge_pct` = `(model_prob_for_side − implied_prob_for_side)`; you can keep it generic for now by assuming –110 juice (\~0.524 implied) until you ingest real odds.

Optionally, maintain `data/broadcasts/gambler/latest.json` as a copy of the most recent file for the front end.

---

## 3) Python: `scripts/gambler_scan.py` (starter)

```python
#!/usr/bin/env python3
import json, math, os, datetime as dt
from pathlib import Path
import numpy as np

ET = dt.timezone(dt.timedelta(hours=-4))  # America/New_York (fixed for EDT demo)
TODAY = dt.datetime.now(ET).date().isoformat()

# ---- 1) Load or build slate (replace with your real pipeline) ----
slate = [
    {"game_id":"CHC@CIN","start_et":f"{TODAY}T18:40:00-04:00","venue":"Great American Ball Park","away":"CHC","home":"CIN",
     "off_idx_away":1.02,"off_idx_home":0.98,"pitch_idx_home":0.97,"pitch_idx_away":1.05,"park":1.08,"weather":1.03,
     "market_total":9.0, "market_odds_over":-110, "market_odds_under":-110},
    {"game_id":"NYY@TB","start_et":f"{TODAY}T19:10:00-04:00","venue":"Tropicana Field","away":"NYY","home":"TB",
     "off_idx_away":1.05,"off_idx_home":0.99,"pitch_idx_home":0.96,"pitch_idx_away":0.98,"park":0.94,"weather":1.00,
     "market_total":8.0, "market_odds_over":-115, "market_odds_under":-105},
    {"game_id":"LAD@SF","start_et":f"{TODAY}T21:45:00-04:00","venue":"Oracle Park","away":"LAD","home":"SF",
     "off_idx_away":1.06,"off_idx_home":0.97,"pitch_idx_home":0.95,"pitch_idx_away":1.02,"park":0.92,"weather":0.98,
     "market_total":7.5, "market_odds_over":-110, "market_odds_under":-110},
]

LEAGUE_RPG = 4.6
rng = np.random.default_rng(7)

def lam(off, opp_pitch, park, weather, base=LEAGUE_RPG):
    return max(0.5, min(base * off * opp_pitch * park * weather, 10.0))

def implied_prob(american_odds: int) -> float:
    return (abs(american_odds) / (abs(american_odds) + 100)) if american_odds < 0 else (100 / (american_odds + 100))

rows = []
N = 50000
for g in slate:
    la = lam(g["off_idx_away"], g["pitch_idx_home"], g["park"], g["weather"])  # away scoring rate
    lh = lam(g["off_idx_home"], g["pitch_idx_away"], g["park"], g["weather"])  # home scoring rate
    sim_a = rng.poisson(la, size=N)
    sim_h = rng.poisson(lh, size=N)
    sim_tot = sim_a + sim_h

    proj_total = float(sim_tot.mean())
    over_prob = float((sim_tot > g["market_total"]).mean())
    under_prob = 1.0 - over_prob

    # pick side w/ bigger model prob vs the corresponding implied
    imp_over = implied_prob(g["market_odds_over"])
    imp_under = implied_prob(g["market_odds_under"])
    edge_over = over_prob - imp_over
    edge_under = under_prob - imp_under
    if edge_over >= edge_under:
        side, model_p, imp_p, edge = "Over", over_prob, imp_over, edge_over
    else:
        side, model_p, imp_p, edge = "Under", under_prob, imp_under, edge_under

    rows.append({
        "game_id": g["game_id"],
        "start_et": g["start_et"],
        "venue": g["venue"],
        "away": g["away"],
        "home": g["home"],
        "proj_total": round(proj_total, 2),
        "market_total": g["market_total"],
        "over_prob": round(over_prob, 4),
        "under_prob": round(under_prob, 4),
        "edge_pct": round(edge, 4),
        "suggested_side": side
    })

# rank & top‑3
rows.sort(key=lambda r: r["edge_pct"], reverse=True)
for i, r in enumerate(rows, 1):
    r["rank"] = i

payload = {
    "broadcast_key": "gambler_top3",
    "date": TODAY,
    "sport": "MLB",
    "generated_at": dt.datetime.now(ET).isoformat(),
    "source": "the_gambler",
    "items": rows[:3]
}

out_dir = Path("data/broadcasts/gambler")
out_dir.mkdir(parents=True, exist_ok=True)
with open(out_dir / f"top3_{TODAY}.json", "w") as f:
    json.dump(payload, f, indent=2)
with open(out_dir / "latest.json", "w") as f:
    json.dump(payload, f, indent=2)
print("Wrote broadcast:", out_dir)
```

> Later, swap the slate stub with your real schedule/odds ingestion.

---

## 4) GitHub Action: `.github/workflows/gambler-scan.yml`

```yaml
name: Gambler Daily Scan
on:
  schedule:
    - cron: '30 13 * * *'   # 09:30 America/New_York ≈ 13:30 UTC during EDT
  workflow_dispatch: {}

jobs:
  run-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install deps
        run: |
          python -m pip install --upgrade pip
          pip install numpy
      - name: Run Gambler scan
        run: |
          python scripts/gambler_scan.py
      - name: Commit broadcast
        run: |
          git config user.name "signals-bot"
          git config user.email "signals-bot@users.noreply.github.com"
          git add data/broadcasts/gambler/*.json
          git commit -m "chore(gambler): top3 broadcast $(date -u +'%Y-%m-%dT%H:%M:%SZ')" || echo "No changes"
          git push
```

> Adjust the cron if you prefer a different broadcast time.

---

## 5) Front-end widget (drop into Signal dashboard)

### `web/gambler_top3.js`

```javascript
async function renderGamblerTop3(elId = 'gambler-top3') {
  const el = document.getElementById(elId);
  if (!el) return;
  try {
    const res = await fetch('./data/broadcasts/gambler/latest.json', { cache: 'no-store' });
    if (!res.ok) throw new Error('fetch failed');
    const { date, sport, items } = await res.json();

    const header = document.createElement('div');
    header.className = 'broadcast-header';
    header.innerHTML = `<h3>🎲 The Gambler — Top‑3 Edges <small>${sport} · ${date}</small></h3>`;

    const list = document.createElement('div');
    list.className = 'broadcast-list';

    items.forEach(it => {
      const edgePct = (it.edge_pct * 100).toFixed(1);
      const card = document.createElement('div');
      card.className = 'edge-card';
      card.innerHTML = `
        <div class="edge-rank">#${it.rank}</div>
        <div class="edge-body">
          <div class="edge-title">${it.away} @ ${it.home}</div>
          <div class="edge-meta">${it.venue} · ${new Date(it.start_et).toLocaleTimeString([], {hour:'2-digit', minute:'2-digit'})}</div>
          <div class="edge-line">Proj Total: <b>${it.proj_total}</b> · Market: <b>${it.market_total}</b></div>
          <div class="edge-signal ${it.suggested_side.toLowerCase()}">Suggested: <b>${it.suggested_side}</b> · Edge: <b>${edgePct}%</b></div>
        </div>`;
      list.appendChild(card);
    });

    el.replaceChildren(header, list);

    // GA4 ping example (optional)
    window.dataLayer = window.dataLayer || [];
    window.dataLayer.push({ event: 'signal_broadcast_viewed', source: 'the_gambler', items: items.length });
  } catch (e) {
    el.textContent = 'No Gambler broadcast available yet.';
  }
}
```

### Minimal styles (optional)

```html
<style>
  .broadcast-header h3 { margin: 0 0 .5rem; }
  .broadcast-list { display:grid; gap:.75rem; }
  .edge-card { display:flex; gap:.75rem; padding:.75rem; border:1px solid #ddd; border-radius:12px; }
  .edge-rank { font-weight:700; font-size:1.25rem; }
  .edge-title { font-weight:600; }
  .edge-signal.over { color:#155e18; }
  .edge-signal.under { color:#5e1515; }
</style>
<div id="gambler-top3"></div>
<script src="./web/gambler_top3.js"></script>
<script>renderGamblerTop3('gambler-top3');</script>
```

Place that HTML block in your Signal dashboard page where you want the broadcast to render.

---

## 6) GA4 / GTM events (optional)

* `signal_broadcast_viewed` — fired on render with `{ source: 'the_gambler', items: 3 }`.
* Add a click handler to `.edge-card` to push `edge_card_clicked` with `{ game_id, side, edge_pct }`.

```javascript
document.addEventListener('click', (e) => {
  const card = e.target.closest('.edge-card');
  if (!card) return;
  window.dataLayer = window.dataLayer || [];
  window.dataLayer.push({ event: 'edge_card_clicked', source: 'the_gambler' });
});
```

---

## 7) Notes & next steps

* Swap the slate stub with your **real schedule + odds ingestion**.
* Add lineup confirmation and bullpen fatigue to your λ calculation.
* Extend to NBA/NFL with sport‑specific features; reuse the same broadcast surface.
* Optionally, add a **confidence badge** based on simulation width (e.g., interquartile range of totals).

That’s it — once committed, your Signal dashboard will surface a daily 🎲 **Top‑3 Edges** broadcast from The Gambler.

---

# NFL Variant — The Gambler → Signal Broadcast (Top‑3 Edges)

Switching the broadcast to **NFL** reduces data churn (weekly slates) and lets you run the scan a couple of times per week (e.g., **Thu AM** to catch TNF and **Sun AM** for the main slate). Below is a drop‑in NFL pack that mirrors the MLB structure while using football‑specific features.

## 0) Repo layout (same skeleton)

```
signal-model/
├─ data/broadcasts/gambler/
│  ├─ top3_2025-w01.json
│  └─ latest.json
├─ scripts/
│  └─ gambler_scan_nfl.py
├─ web/
│  └─ gambler_top3.js   (unchanged)
└─ .github/workflows/gambler-scan-nfl.yml
```

## 1) Seeds (no change needed)

The existing `gambler_top3`, `gambler`, and `edge` entries still apply. If you want to be explicit, add a sport field in the payload (already supported below).

## 2) Broadcast schema (NFL)

Example file: `data/broadcasts/gambler/top3_2025-w01.json`

```json
{
  "broadcast_key": "gambler_top3",
  "season": 2025,
  "week": 1,
  "sport": "NFL",
  "generated_at": "2025-09-11T09:30:00-04:00",
  "source": "the_gambler",
  "items": [
    {
      "rank": 1,
      "game_id": "CIN@CLE",
      "start_et": "2025-09-14T13:00:00-04:00",
      "venue": "Cleveland Browns Stadium",
      "away": "CIN",
      "home": "CLE",
      "proj_total": 47.8,
      "market_total": 45.5,
      "over_prob": 0.58,
      "under_prob": 0.42,
      "edge_pct": 0.055,
      "suggested_side": "Over",
      "note": "Neutral‑situation pace + PROE up; mild wind."
    }
  ]
}
```

## 3) Modeling notes (NFL)

* **Core signal**: expected **plays** × **efficiency**.

  * Plays driven by **pace** (seconds/play), timeouts, kneel‑downs, and game script.
  * Efficiency proxied by **EPA/play**, **success rate**, red‑zone TD rate, explosive‑play rate.
* **Game total** ≈ (plays\_home × pts/play\_home) + (plays\_away × pts/play\_away).
* **Adjustments**: QB status (starter/backup), OL injuries, defensive pass rush/coverage, **weather** (wind > 15 mph suppresses passing), **roof/dome**.
* **Distribution**: start with **Negative Binomial** or Poisson‑Gamma on team points; Monte Carlo to combine teams and get **Over%/Under%**.

## 4) Python — `scripts/gambler_scan_nfl.py` (starter)

```python
#!/usr/bin/env python3
import json, math, os, datetime as dt
from pathlib import Path
import numpy as np

ET = dt.timezone(dt.timedelta(hours=-4))
now = dt.datetime.now(ET)
SEASON = 2025
WEEK = 1  # set via env or small helper that maps date→week

# ---- 1) Stub slate (replace with real schedule/odds) ----
slate = [
    {"game_id":"CIN@CLE","start_et":f"{SEASON}-09-14T13:00:00-04:00","venue":"Cleveland Browns Stadium",
     "away":"CIN","home":"CLE",
     # Pace (sec/play, lower=faster), EPA/play, weather multipliers
     "pace_away":27.0, "pace_home":28.5, "epa_away":0.08, "epa_home":0.02,
     "redzone_td_away":0.62, "redzone_td_home":0.58,
     "weather_mult":0.98,  # <1 suppresses scoring (wind/rain)
     "market_total":45.5, "market_odds_over":-110, "market_odds_under":-110},
    {"game_id":"KC@LAC","start_et":f"{SEASON}-09-14T16:25:00-04:00","venue":"SoFi Stadium",
     "away":"KC","home":"LAC",
     "pace_away":26.5, "pace_home":27.5, "epa_away":0.12, "epa_home":0.05,
     "redzone_td_away":0.65, "redzone_td_home":0.60,
     "weather_mult":1.02,  # dome/neutral
     "market_total":49.5, "market_odds_over":-112, "market_odds_under":-108},
]

rng = np.random.default_rng(11)

# Heuristic conversions
def plays_from_pace(seconds_per_play, possession_time=1800):
    # possession_time ~ 30 minutes per team (regulation), rough heuristic before game script
    return max(45, min(int(possession_time / seconds_per_play), 80))

def pts_per_play(epa, rz_td_rate):
    # EPA/play is already expected points delta; add simple redzone finishing boost
    base = 0.40 + 3.0*max(0, epa)  # baseline ~0.4 ppp + EPA influence
    return base * (0.9 + 0.2*rz_td_rate)  # amplify with redzone efficiency

def implied_prob(american_odds: int) -> float:
    return (abs(american_odds) / (abs(american_odds) + 100)) if american_odds < 0 else (100 / (american_odds + 100))

rows = []
N = 50000
for g in slate:
    plays_a = plays_from_pace(g["pace_away"])  # away plays
    plays_h = plays_from_pace(g["pace_home"])  # home plays
    ppp_a = pts_per_play(g["epa_away"], g["redzone_td_away"]) * g["weather_mult"]
    ppp_h = pts_per_play(g["epa_home"], g["redzone_td_home"]) * g["weather_mult"]

    mean_pts_a = plays_a * ppp_a / 100.0
    mean_pts_h = plays_h * ppp_h / 100.0

    # Use Negative Binomial via Poisson‑Gamma mixture: approx with variance inflation k
    k = 1.4  # dispersion (>1 inflates variance vs Poisson)
    lam_a = mean_pts_a
    lam_h = mean_pts_h

    # Sample by Gamma‑Poisson: draw rate then Poisson
    rate_a = rng.gamma(shape=lam_a/k, scale=k, size=N)
    rate_h = rng.gamma(shape=lam_h/k, scale=k, size=N)
    sim_a = rng.poisson(rate_a)
    sim_h = rng.poisson(rate_h)

    sim_tot = sim_a + sim_h
    proj_total = float(sim_tot.mean())
    over_prob = float((sim_tot > g["market_total"]).mean())
    under_prob = 1.0 - over_prob

    imp_over = implied_prob(g["market_odds_over"])
    imp_under = implied_prob(g["market_odds_under"])
    edge_over = over_prob - imp_over
    edge_under = under_prob - imp_under
    if edge_over >= edge_under:
        side, model_p, imp_p, edge = "Over", over_prob, imp_over, edge_over
    else:
        side, model_p, imp_p, edge = "Under", under_prob, imp_under, edge_under

    rows.append({
        "game_id": g["game_id"],
        "start_et": g["start_et"],
        "venue": g["venue"],
        "away": g["away"],
        "home": g["home"],
        "proj_total": round(proj_total, 1),
        "market_total": g["market_total"],
        "over_prob": round(over_prob, 4),
        "under_prob": round(under_prob, 4),
        "edge_pct": round(edge, 4),
        "suggested_side": side
    })

rows.sort(key=lambda r: r["edge_pct"], reverse=True)
for i, r in enumerate(rows, 1):
    r["rank"] = i

payload = {
    "broadcast_key": "gambler_top3",
    "season": SEASON,
    "week": WEEK,
    "sport": "NFL",
    "generated_at": now.isoformat(),
    "source": "the_gambler",
    "items": rows[:3]
}

out_dir = Path("data/broadcasts/gambler")
out_dir.mkdir(parents=True, exist_ok=True)
with open(out_dir / f"top3_{SEASON}-w{WEEK:02d}.json", "w") as f:
    json.dump(payload, f, indent=2)
with open(out_dir / "latest.json", "w") as f:
    json.dump(payload, f, indent=2)
print("Wrote NFL broadcast:", out_dir)
```

## 5) GitHub Action — `.github/workflows/gambler-scan-nfl.yml`

Run **twice weekly** by default; tweak as you like.

```yaml
name: Gambler NFL Scan
on:
  schedule:
    - cron: '30 13 * * 4'   # Thu 09:30 ET (pre‑TNF)
    - cron: '00 13 * * 7'   # Sun 09:00 ET (pre‑main slate)
  workflow_dispatch: {}

jobs:
  run-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install deps
        run: |
          python -m pip install --upgrade pip
          pip install numpy
      - name: Run Gambler NFL scan
        run: |
          python scripts/gambler_scan_nfl.py
      - name: Commit broadcast
        run: |
          git config user.name "signals-bot"
          git config user.email "signals-bot@users.noreply.github.com"
          git add data/broadcasts/gambler/*.json
          git commit -m "chore(gambler-nfl): top3 broadcast $(date -u +'%Y-%m-%dT%H:%M:%SZ')" || echo "No changes"
          git push
```

## 6) Front‑end

No change required — the same `web/gambler_top3.js` reads `latest.json` and renders the Top‑3 with `sport: "NFL"`. You may update the header text if you want to display `season/week` next to the date.

## 7) Next steps

* Replace the slate stub with **real NFL schedule** data and **odds** ingestion.
* Add **QB/OL injury status**, **PROE**, and **wind/roof** in your feature set.
* Consider a **confidence badge** (e.g., IQR of simulated totals) to prioritize sharper edges.
* Extend to **sides** (spreads) using the same framework (simulate score differential and compare to the spread).
