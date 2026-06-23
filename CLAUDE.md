# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TPS-EMS 韌性急救訓練模擬系統 is a browser-based EMS/CPR training simulator for Taiwan's pre-hospital emergency care education. The entire application is a **single self-contained file**: `index.html` — no build system, no dependencies, no package manager.

## Running the App

Open `index.html` directly in a browser — no server required. For quick local testing with a live-reload workflow:

```sh
python3 -m http.server 8080
# then open http://localhost:8080
```

There are no lint commands, no test suite, and no CI pipeline.

## Architecture

The application is pure HTML + vanilla JS in a single `<script>` block (~900 lines). The overall data flow is:

```
State (S) → applyRhythmDefaults() → advancePhase() → sampleChannels() → Canvas sweep render
                                                                       → updateVitalsUI()
```

### Core objects

| Symbol | Purpose |
|--------|---------|
| `S` | Single global state object: rhythm, vitals, CPR state, pacing, drug list, stats |
| `RHYTHMS` | 14 cardiac rhythm definitions (name, HR, shockable, organized, pulse, wide, etc.) |
| `DRUGS` | 6 ACLS drug definitions with `apply()` closures that mutate `S` |
| `clk` | Wave clock: `master` time, `beatT0`/`atrialT0`/`respT0` phase origins, `rr` interval |
| `channels` | Canvas render targets keyed by canvas ID (`cEcg`, `cPleth`, `cAbp`, `cCo2`, `cCpr`) |
| `lead12` | Separate canvas map for the 12-lead ECG overlay |

### Waveform pipeline

1. `advancePhase(dt)` — advances `clk.master` and rolls over beat/atrial/resp phase origins
2. `sampleChannels()` — calls `ecgMorph`, `plethMorph`, `abpMorph`, `co2Morph` with current phase offset `te = clk.master - clk.beatT0`
3. `frame()` — rAF loop; steps the above `Math.round(PX_PER_SEC * dt)` times per frame, drawing one pixel per step in sweep mode (erase 14px ahead, draw, advance x)
4. `renderCprWave(dt)` — separate pass for the CPR channel using `sin(f * π)` pulse shape

Waveform morphology uses Gaussian summation (`gauss(t, center, width, amplitude)`). Special cases: VF uses multi-sine noise (`vfWave`), asystole adds tiny drift, torsades uses a sinusoidal amplitude envelope.

### CPR engine

`compress()` is called on spacebar / floating button / "按壓一次" button. It records inter-press intervals in a rolling 8-sample buffer (`S.cpr.intervals`), computes rate, randomises depth (4.6–5.8 cm), and drives EtCO₂ up when pulseless. `qualityFactor()` returns 0–1 from rate score (target 100–120/min) and rhythm CV. `updateCprMetrics(dt)` auto-stops CPR if no press for 1.5 s.

### Defibrillation / Cardioversion

`chargeDefib()` / `deliverShock()` implement probabilistic conversion. Shock success probability for shockable rhythms: base 0.45 + 0.15 if energy ≥ 150 J + CPR quality factor + drug boost (epinephrine +0.08, amiodarone +0.14). After ≥2 shocks with successful VF termination, there is a 50% chance of `achieveROSC()` vs. converting to PEA.

### Instructor ↔ Trainee sync

The three roles are **solo**, **trainee**, and **instructor**. When not solo:

- Uses `window.storage` (Claude Code platform API) via `hasStore` guard; polling interval 800 ms
- Storage keys: `tpsems:{code}:cmd` (instructor → trainee) and `tpsems:{code}:stat` (trainee → instructor)
- Commands carry a `seq` counter; `S.seqIn` prevents replaying the same command
- `ctrlPush(kind)` — instructor sends rhythm/vitals state
- `pushStatus()` — trainee sends CPR quality back to instructor
- If `window.storage` is unavailable (plain browser), `S.net` is forced false and the app runs in solo mode silently

### Control deck rendering

`renderDeck()` writes `innerHTML` for the bottom panel based on the active `tab` variable (`resus`, `drug`, `ctrl`). `wireDeck()` re-binds all events after every re-render. The "report" tab opens `#ovRep` overlay instead of setting deck content.

### 12-lead ECG overlay

`build12()` / `loop12()` / `lead12sample()` run a separate independent rAF loop (`raf12`) when the overlay is open. Each lead applies a per-lead amplitude factor (`cfg.f`) and inversion flag to `ecgMorph()`, with special handling for V1/V2 (rS pattern) and regional ST elevation for STEMI.

## Key Conventions

- All user-visible text is Traditional Chinese (zh-Hant-TW); comments are a mix of Chinese and English.
- The `$` / `$$` aliases are local (`document.querySelector` / `querySelectorAll`). Do not confuse with jQuery.
- `clamp(v,a,b)` and `rnd(a,b)` are local utilities; `now()` wraps `performance.now()`.
- State is mutated directly on `S`; there is no immutability or reactive framework.
- `renderDeck()` must be called after any state change that affects the control panel. Vitals UI updates automatically on every animation frame via `updateVitalsUI()`.
- The `window.storage` API is a Claude Code remote-environment feature and is not standard browser localStorage. All network-dependent code is guarded by `hasStore`.
