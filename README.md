# AGI-dle

A mobile-first incremental/idle game (built from `prompt-factory-product-spec.md`). Run a frontier AI lab from a garage to the stars — and automate yourself out of every job in it.

**Play:** https://danielxli.github.io/AGI-dle/

**The whole game is one self-contained file: [`index.html`](index.html).** No build step, no dependencies, no server. Open it in any browser.

## How to play

The **mission bar** under the crown always tells you the next goal. Broadly:

1. **Tap "Scrape data"** to gather data, then **Start a training run**.
2. **Tap "Run gradient step"** to descend the loss curve — that births GPT-1. Tapping never stops mattering: even once the auto-optimizer runs your training, **tapping a run banks a "training boost"** (up to ×2.5) that makes that model far bigger — so active play always beats idling. Watch for **crits** (gold) and build a **combo** by tapping fast.
   - **Breakthroughs:** a rare ✦ token drifts across the screen every ~30–70s — tap it for **Frenzy (×7 production)**, **Overclock (×4 training speed)**, or a cash **windfall**.
   - **Catch instability:** when a run spikes, hit **Stabilize** in time for bonus capability and cash.
3. **Deploy** the model to earn cash by **Serving requests**.
4. Spend cash on **generators** that do your jobs for you (web scraper → curation → auto-optimizer). Use the **×1 / ×10 / Max** toggle to buy in bulk. Once the auto-optimizer is online, runs train themselves — you've automated your first job.
5. Scale through 7 eras, each introducing a mechanic with a short story card: deploy & instability → allocation minigame & fundraising → the **Operations** train/serve compute split → capital structure & pricing → self-funding → **AGI** → the **Dyson swarm & galactic expansion**.

### The prestige loop (the spine)

Around the Frontier era you hit a **wall**: the next era needs **Research**, which you only get by starting a **Next Generation** — a reset that banks Research for a permanent ×multiplier to all output and capability. Each generation is a faster replay that breaks one era further. It takes several generations to reach **AGI**, and a few more to colonise the galaxy (the **Lightcone** = victory). Grinding the wall a little longer before resetting earns more Research — that "when do I reset?" call is the strategic heart of the game.

The **Lab log** records your whole story; progress saves automatically and accrues while you're away (offline, capped 12h).

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
