# 🎲 FourTwenty • The Gambler

**Risk desk for scenarios, expected value, and probability simulations.**  
The Gambler explores how odds, variance, and trade-offs play out across bets and choices.  
It doubles as a **GTM/GA4 sandbox**, pushing PII-free events into `dataLayer` for live testing.

---

## 🌟 What’s inside
- **Purpose**  
  Provide a themed playground for experimenting with probabilities (coin flips, dice, parlays) and mapping those events to analytics.
- **Artifacts**  
  - `/playground` → UX Playground for custom event triggers  
  - `/latest.json` → broadcast of current “top plays” with origin metadata  
  - `/artifacts` → screenshots, GIFs, sample dashboards  
  - `/gtm` → container exports  
  - `/ga4` → debug notes, views
- **Telemetry**  
  Unified **GTM container (`GTM-WVC4SNLB`)** across constellation. Events verified in GA4 DebugView.

---

## 🚀 Quick start
1. Clone repo and serve locally:  
   ```bash
   python3 -m http.server 5500
Then open /index.html or /playground/ux_playground.html.
2. Attach GTM Preview and watch events stream into Tag Assistant.
3. Validate in GA4 DebugView — look for events such as:
bet_sim (with bet_type + result)
cta_click
form_submit
nav_search
virtual page_view

📊 Highlights
Probability Console → Coin flip, dice roll, parlay simulator (results pushed into dataLayer).
Broadcast Integration → latest.json publishes The Gambler’s current signals.
Analytics Demo → Fire sample events (cta_click, form_submit, etc.) to test GTM/GA4 flows.
BI Views (future) → Expected value charts, variance dashboards, win/loss streak analysis.

🗂 Related modules
🚀 The Launch – foundation and entry point
🫀 The Archive – heart and integrity keeper
📡 The Signal – system nerves, broadcasting signals

⚖️ License MIT — free to use, adapt, and extend.
