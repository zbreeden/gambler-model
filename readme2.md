# Create updated README.md and index.html for The Gambler (NFL-focused with Signal broadcast)
#
# - Preserves GTM/dataLayer pattern from current index.html
# - Adds an NFL "Top-3 Edges" renderer reading data/broadcasts/gambler/latest.json
# - README now documents NFL workflow, broadcast schema, and GitHub Action
#
# Files will be saved to /mnt/data so Zach can download and commit.


from pathlib import Path

root = Path("/mnt/data")
root.mkdir(parents=True, exist_ok=True)

readme = r"""# FourTwenty • The Gambler (NFL)

> Probabilistic market scanner for **NFL totals**, using a Monte Carlo layer to estimate **Over/Under** hit rates and broadcast a daily/weekly **Top-3 Edges** to **The Signal**.

## Purpose
Turn weekly NFL slates into actionable, auditable picks. The Gambler estimates outcome probabilities (e.g., Over 45.5 hits 57%) and compares them with market implied probabilities to flag value (edge).

## What’s inside
- **Model (NFL)**: pace → plays; efficiency (EPA/play, success rate) → points/play; weather/roof multiplier; Negative-Binomial–style variance.
- **Simulation**: Monte Carlo to produce distributions for team points and totals → Over%/Under%.
- **Broadcasts**: JSON payloads written to `data/broadcasts/gambler/` for the **Signal** dashboard widget.
- **Automations**: GitHub Action to run **Thu AM** (TNF) and **Sun AM** (main slate) or manually via workflow dispatch.

## Repo layout (excerpt)
