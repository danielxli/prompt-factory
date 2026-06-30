# Prompt Factory

A mobile-first incremental/idle game built from `prompt-factory-product-spec.md`. Run a frontier AI lab from a garage to the stars — and automate yourself out of every job in it.

**The whole game is one self-contained file: [`index.html`](index.html).** No build step, no dependencies, no server. Open it in any browser.

## How to play

1. **Tap "Scrape data"** to gather data, then **Start a training run**.
2. **Tap "Run gradient step"** to descend the loss curve — that births your first model (GPT-1). Watch for **crits** (gold) and build a **combo** by tapping fast.
3. **Deploy** the model to earn cash by **Serving requests**.
4. Spend cash on **generators** that do your jobs for you (web scraper → curation → auto-optimizer). Once the auto-optimizer is online, runs train themselves — you've automated your first job.
5. Scale through 7 eras: deploy & instability → allocation minigame & fundraising → the **Operations** train/serve compute split → capital structure & pricing → self-funding → **AGI** → the **Dyson swarm & galactic expansion**.
6. **Win** by colonising the galaxy (the Lightcone). The dream run: reach AGI while keeping ≥51% ownership by self-funding instead of diluting away.

Progress saves automatically (localStorage) and accrues while you're away (offline progress, capped at 12h).

## Play on mobile

- **Easiest:** host `index.html` anywhere static (or run `python3 -m http.server` and open your machine's LAN IP on your phone), then open the URL in mobile Safari/Chrome → **Share → Add to Home Screen** for a fullscreen, app-like experience.
- **Offline:** AirDrop / iCloud `index.html` to your phone and open it in a browser — it works fully offline from `file://`.

Respects `prefers-reduced-motion`; sound and haptics are optional and toggleable in Settings.

## Design pillars (from the spec)

1. The domain is the mechanic — scaling laws are the real cost curve.
2. The descending **loss curve** is the core feedback; every model is born from it.
3. Every system asks "how much do you gamble?" — runs fail, raises cost ownership, deploying spends your edge.
4. The UI fills as you scale and empties as you automate.

## Developer notes

The simulation engine is exposed on `window.__pf` (a harmless debug handle) so it can be driven headlessly. Two test harnesses live alongside development:

- **Balance sim** — stub the DOM in Node, `eval` the real engine, and play it with `bootstrap` / `measured` / `greedy` strategies to validate pacing and the ownership trade-off. (See the scratchpad `sim.js` used during build.)
- Per-era capability thresholds, run costs, and juice constants are all defined at the top of the script in the `C` config object and the `GENS` / `UPGRADES` tables — balancing is a data change.
