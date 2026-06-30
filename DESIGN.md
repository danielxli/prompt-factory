# Prompt Factory — Design & Technical Notes

The canonical reference for how the game is built and balanced. Player-facing instructions live in [`README.md`](README.md); the original product spec is `prompt-factory-product-spec.md` (kept local, not in the public repo).

> **One file.** The entire game — engine, UI, styling, save system — is `index.html`. No build step, no dependencies, no server. It runs from `file://` or any static host.

---

## 1. Architecture

All logic is in one inline `<script>`, organized top-to-bottom:

| Section | Responsibility |
|---|---|
| `C` (config) | Every balance constant. Tuning = editing this object + the tables below. |
| `GENS`, `UPGRADES`, `CAPITAL`, `AUTOS`, `EVENTS`, `BT_TYPES`, `ERA_STORY` | Data tables. |
| `freshState()` / `freshRuntime()` | `S` = the entire serializable sim state. `R` = transient runtime (combo, buy-mode, dirty flags) — never saved. |
| selectors | `rates()`, `opsEconomics()`, `passiveCashPerSec()`, `qualityFactor()`, `researchGain()`, etc. Pure reads of `S`. |
| actions | `actScrape/actClean/actStep/actStartRun/actDeploy/actServe/actEval/actStabilize/actRaise/actCapital/buyGen/buyUpgrade/actPrestige`. The **only** things that mutate `S` outside `tick`. |
| `tick(dt)` | Fixed per-frame sim step (rates, hype decay, passive cash, run auto-progress, instability, buffs, events, stars, era check). |
| juice | `pop/toast/shake/sfx/vibrate`, the Breakthrough spawner, the canvas loss-plot. |
| `render()` | Reads `S`/`R` → DOM. Never mutates sim state. Heavy lists re-render only when a `structHash()` changes; everything else updates at ~10 Hz. |
| save / migrate / offline | localStorage, versioned migration chain, closed-form offline catch-up. |
| `frame()` | rAF loop: `tick` (dt clamped to 0.25s) → `drawPlot` (60fps) → `render` (10Hz) → autosave (2s). |

**Quirks worth knowing**
- `rates` is a `function` declaration reassigned once to `ratesBoosted` (wraps the original to layer event/frenzy multipliers). `_origRates` holds the unwrapped version.
- `window.__pf` is a deliberate debug handle exposing state + functions so the engine can be driven headlessly (see §6). Harmless in production.
- UI never writes `S` directly — only through actions/tick (simulation-integrity discipline from the spec).

---

## 2. Core loop & systems

1. **Scrape data** (tap) → **Start training run** → **tap "Run gradient step"** to descend the live **loss curve** → a model is born.
2. **Deploy** a model → **Serve** for cash (manual early; passive via Inference cluster / Operations later).
3. Spend cash on **generators** (×1/×10/Max bulk buy) that automate each verb, and on one-time **algorithmic upgrades** (multipliers).
4. Hit a **wall** → **Next Generation** (prestige) banks Research for permanent multipliers → replay faster, further.

**The active layer (what keeps it fun moment-to-moment)**
- **Run boost:** tapping gradient steps during *any* run (even an auto-run) banks `run.boost` (≤ `BOOST_CAP`), multiplying that model's capability by `1 + boost` (up to ×2.5) and speeding the run. Idle = baseline models; tap = much bigger ones. The hero verb never becomes obsolete.
- **Breakthroughs (golden cookies):** a tappable ✦ token (`spawnBreakthrough`) drifts in every ~30–70s for a variable reward — **Frenzy** (`prod` buff ×7), **Overclock** (`run` buff ×4), or a cash **windfall**. Buffs are entries in `S.buffs = [{type,mult,t}]`, applied via `frenzyMult()` / `runSpeedMult()` and expired in `tick`.
- **Instability:** mid-run spike opens a ~3s **Stabilize** window. Catching it banks bonus boost + cash; missing it decays progress (divergence). Automated late by Auto-recovery.
- **Crits / combo:** `CRIT_P` 0.13, 5–8×, gold flash + shake + sound + haptic. Combo builds to ×3.2 on fast taps, decays when idle.

**Mid/late systems**
- **Operations (E3+):** one compute pool split by a slider into Train (capability) vs Serve (cash). Unit economics: `cash/s = (price − cost_per_token) × min(demand, serve_cap)`. Test-time-compute upgrade is the designed margin trap.
- **Capital (E4+):** venture debt / compute credits (non-dilutive, add burn) and cloud prepay (−25% cost/token).
- **Fundraising:** `dilution = amount/(valuation+amount)`; ownership only ever decreases (within a generation). Hype multiplies valuation.
- **Full automation (E6+):** toggles that auto-perform verbs and hide their manual buttons (UI empties as you automate — the spec's inversion).

---

## 3. Progression & the prestige spine

Era is gated by **capability AND research** (`checkEra`): you enter era *i* only when `bestCapability ≥ ERA_TH[i]` **and** `research ≥ ERA_RESEARCH[i]`. Eras are monotonic within a generation; the first deploy is required to leave Era 0.

```
ERA_TH        = [0, 8, 80, 800, 8000, 80000, 800000, 8000000]   // ≈10× per era
ERA_RESEARCH  = [0, 0,  0,   0,   40,   400,   3000,    20000]   // 0 for eras 0–3
```

So eras 0–3 are reachable in generation 1; **Era 4+ requires Research, which only `actPrestige` grants** → the prestige loop is mandatory, not optional.

```
researchGain  = floor(RESEARCH_K × bestCapability^(1/3))   // concave; keyed off capability so it COMPOUNDS across gens
researchMult  = 1 + research × 0.04                        // permanent ×mult to all production AND capability
```

Keying research off **capability reached** (not lifetime compute, which resets and plateaus) is what makes the loop accelerate: more research → bigger models next gen → more research. Grinding the wall longer before resetting earns more — that "when do I reset?" call is the strategic core. A full playthrough is ~12 generations to the Lightcone victory (100 stars).

**Storyline legibility** (added because the narrative was getting lost in 2-second toasts): a persistent **mission bar** (next goal + progress + active-buff chips), a **story card** the first time each era is reached (`ERA_STORY`), a scrollable **Lab log** (`S.log`), and a `researchWalled()` state that turns the mission/prestige UI into a clear call to action. Model **class names track the era/capability tier**, not raw run count.

---

## 4. Balance constants (single source of truth = `C` + tables)

| Constant | Value | Notes |
|---|---|---|
| `STARTER_GPU` | 0.6/s | free compute trickle |
| `SCRAPE_BASE` / `CLEAN_YIELD` / `CLEAN_DATA_COST` | 2.2 / 3 / 4 | clean costs data (anti clean-spam) |
| `STEP_BASE` / `Q_COEF` / `OPT_RATE` | 0.045 / 0.012 / 1.1 | run progress + quality factor |
| `BOOST_PER_TAP` / `BOOST_CAP` | 0.05 / 1.5 | run boost → up to ×2.5 capability |
| `CRIT_P` / mult | 0.13 / 5–8× | |
| combo window/cap/step/decay | 900ms / 3.2 / 0.18 / 2.4·s | |
| `SERVE_BASIS` | 0.8 | cash per serve = cap × this × combo × crit × mults |
| `DEPLOY_HYPE` / `EVAL_HYPE` / `EVAL_CD` / `HYPE_DECAY` | 22 / 10 / 6s / 0.015·s | |
| `GROWTH` | 1.13 | generator cost growth (per-tier overridable) |
| `INSTAB_WINDOW` | 3s | |
| `OFFLINE_RATE` / `OFFLINE_CAP_S` | 0.5 / 12h | |
| `RESEARCH_K` / formula | 5 / `floor(5·cap^⅓)` | prestige currency |
| research mult | `1 + research·0.04` | |
| `BOOST_*`, `ALLOC_P` 1.6 | | allocation-minigame penalty exponent |
| `baseCeiling(n)` | `prev·1.45 + 5`, start 8 | gentle per-model capability growth |
| `runCost(i)` | data `15·1.8^i`, compute `12·2.15^i` | steep compute = the in-gen accumulation gate |
| stars | `√probes · 0.18/s`, win at 100 | galactic finale pacing |

Generators (base cost / cost-growth / output): scraper 12 / 1.13 / +0.8 data·s · curator 30 / +0.5 quality·s · optimizer 80 / 1.15 / auto-steps (gated on first deploy) · inference 220 / 1.14 / passive cash · gpuRack 160 / +2 compute·s · autoEval 340 · autoRecovery 600 / 1.6 / max 5 · cluster 2.2k / +12 · datacenter 28k / +80 · licensed 42k / +14 data · silicon 480k / +500 · synthetic 1.5M / +60·flywheel · gigawatt 9M / +3.5k · planetary 120M / +25k · dyson 4B / +180k · probes 20B / 1.2 / stars.

Upgrades: flashAttention 500 (×1.6 compute) · moe 1.6k (×1.5 cap) · chinchilla 2.4k (allocation hint) · rlhf 9k (×1.6 serve) · distillation 16k (×1.4 serve, −cost/token) · testTime 90k (×1.8 cap, ↑cost/token — the trap).

> Tuning philosophy: balanced *fast* on purpose (perfect play ≈ 15 min to victory; casual check-in play stretches to hours/days via offline + the prestige loop). Era gating is the structural throttle; the economy keeps you always ~minutes from the next thing.

---

## 5. Save / migration / offline
- `localStorage` key `promptfactory_save_v1`; autosave every 2s + on visibility-hidden / pagehide.
- `migrate()` runs a version chain. `SCHEMA = 4`. Saves with `schema_version < 4` predate the prestige redesign → reset to a fresh state (settings preserved), since the progression model changed fundamentally. Transient `buffs` are always cleared on load.
- Offline: elapsed wall-clock (capped 12h) applied at 50% to passive production/cash/stars; manual-only progress does not accrue offline.
- Export/import save (base64) and a full wipe in Settings.

---

## 6. Testing (no framework — headless Node + real Chrome)
- **Syntax:** extract the `<script>` and `node --check`.
- **Balance sims:** stub the DOM (universal Proxy mock element + `window/document/localStorage/performance` and **no-op `setTimeout/clearTimeout`** so the Breakthrough scheduler doesn't keep Node alive), `eval` the real script, drive `window.__pf`. `sim2.js` plays the prestige loop; `sim.js` tests single-generation strategies (`bootstrap`/`measured`/`greedy`) — these validated the AGI-at-51% ownership trade-off and the ~12-generation victory. Render probes inspect the mock DOM after `render()`.
- **Screenshots:** real Chrome `--headless=new --screenshot` at a forced 390px layout (inject `html,body,#app{max-width:390px}` — headless ignores `--window-size` width). The rAF loop prevents clean exit, so run Chrome backgrounded and **hard-kill it after ~14s** once the PNG is written. Always use a unique `--user-data-dir` and `pkill` strays.

(The driver scripts lived in the session scratchpad, not the repo.)

---

## 7. Deployment
- Public repo: `github.com/danielxli/prompt-factory`. Live: **https://danielxli.github.io/prompt-factory/** (GitHub Pages, main branch root).
- Iterate: edit `index.html` → `git commit && git push` → Pages rebuilds in ~30s (HTTPS, so canvas/audio/haptics work on iOS). CDN edge cache can lag a minute; cache-bust to verify.

---

## 8. Known limitations & next levers
- **Pacing is short for a hardcore idle game** (~15 min perfect play). Deliberate; lengthen by widening `ERA_TH`/`ERA_RESEARCH` or steepening `runCost`.
- **Ownership resets each prestige**, so the dilution tension is now within a generation (raise to push further this gen vs. keep ownership for a better end-of-gen stake). `S.bestStake` tracks the meta score. Spec-consistent ("the dream run is AGI at 51%").
- **Not yet built** (from the improvement analysis): threshold-at-50%-affordability reveals (#5), recontextualization brags / milestone fanfare (#9), a richer "While you were away" offline summary (#10). The first 30 seconds and the mid-game waits are the spots most worth punching up next.
- Big numbers use JS doubles (fine to victory; a big-number lib would be needed for deep prestige stacking). `sanitize()` clamps overflow to 1e300 rather than wiping progress.
