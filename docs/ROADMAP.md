# Anime2.5DRig — Roadmap

This document is the master index. Every implementation pass should start here, then drill into the relevant phase plan. The goal: **zero drift, zero missed details** — every checkbox below has a corresponding detailed spec with exact integration points in the code.

---

## Phases at a glance

| Phase | Status | What | Doc |
|---|---|---|---|
| **Phase 1** | ☐ Planned | Natural-movement + authoring features (6 features + 4 idle animations) | [`01-phase1-features.md`](./01-phase1-features.md) |
| **Phase 2** | ✎ To be planned | Headless API + standalone export (host rigs as controllable API instances) | [`02-phase2-api-export.md`](./02-phase2-api-export.md) |

---

## How to run an implementation pass

1. Open the phase doc.
2. Pick a workstream (each is independently shippable).
3. Follow the checklist top-to-bottom. **Do not skip the "Acceptance" block** — it is the definition of done.
4. Update the status emoji in this table and the workstream's status line when complete.

Status legend: ☐ not started · ☐ partial · ☑ done · ✎ in progress

---

## Phase 1 — workstreams (detailed in `01-phase1-features.md`)

### A. Natural-movement idle animations (do these FIRST — smallest, biggest payoff)

- ☐ **A1 — Micro-saccades + gaze targeting**
- ☐ **A2 — Blink variation (double / half / asymmetric)**
- ☐ **A3 — Breath-coupled postural sway + sighs**
- ☐ **A4 — Head-motion-coupled secondary action (hair settle + brow micro)**

### B. Core authoring features

- ☐ **B1 — Viseme set (a/i/u/e/o) for proper lip-sync**
- ☐ **B2 — Preset save/load (localStorage) + lightweight keyframe timeline**
- ☐ **B3 — Gesture system for arms/hands**
- ☐ **B4 — Scene/video export (WebM + PNG sequence)**
- ☐ **B5 — PSD layer-group (folder) support + multi-layer auto-grouping**
- ☐ **B6 — Hair/body self-collision + directional wind**

---

## Phase 2 — preview (will be planned in full after Phase 1)

Goal: take a rig (built in the browser tool) and **deploy it as a standalone, controllable API instance** that can be hosted and driven remotely.

Anticipated workstreams (subject to change during Phase 2 planning):

- Headless rig runtime extracted from `index.html` into a reusable core module.
- HTTP/WebSocket control API: load model, stream params, drive animations, capture frames.
- Export formats: standalone HTML bundle, Docker image, serverless function.
- Auth, rate-limiting, multi-instance orchestration.

**👉 Detailed plan to be written in `02-phase2-api-export.md` immediately after Phase 1 completes.**

---

## Conventions for all implementation passes

- **No silent rewrites.** If a planned integration point has moved since the doc was written, stop and update the doc's line refs before proceeding.
- **One workstream per commit boundary** where practical; commit message prefix `[A1]`, `[B3]`, etc.
- **Preserve the existing visual identity** (monochrome + deep-red theme, dark panel). New UI matches existing `.sec` / `.row` / `.tg` / `.btn` classes unless a workstream explicitly adds a new component.
- **No new external runtime deps** unless a workstream calls it out and justifies it. Currently: `ag-psd.min.js` (vendored) + MediaPipe (lazy CDN). Phase 2 may add a server runtime (Node/Bun) — that's the one planned exception.
- **Client-side first.** Phase 1 adds zero server requirements. Everything still runs from `file://`-compatible static hosting (camera/mic still require https/localhost per existing `README.md` note).
