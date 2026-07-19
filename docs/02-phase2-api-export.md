# Phase 2 — Headless API & Standalone Export

**Status:** ✎ Placeholder — to be planned in full immediately after Phase 1 completes.
**Parent:** [`ROADMAP.md`](./ROADMAP.md)

> This document is a **deliberate stub**. It captures the agreed direction and open questions so the next planning pass has a starting point. It is **not** an implementation spec yet. We will expand it into the same depth as [`01-phase1-features.md`](./01-phase1-features.md) once Phase 1 ships.

---

## Goal (one sentence)

Take a rig built in the browser tool and **deploy it as a standalone, controllable API instance** that can be hosted and driven remotely — turning Anime2.5DRig from a client-only poser into a runtime you can put behind a network.

---

## Why now / why this order

- Phase 1 adds the **authoring richness** (visemes, gestures, timeline) that a remote API actually needs to be worth driving. Shipping the API first would expose a too-thin control surface.
- Phase 1 also forces a clean separation between **rig data** (the `buildRig` output), **param state** (`P`/`cur`/`tgt`), and **rendering** (the WebGL block in `tick`). That separation is exactly what Phase 2 needs to lift the runtime out of the DOM.

---

## Anticipated workstreams (to be refined during planning)

These are the current best guesses. Each will get its own checklist + acceptance block when we plan for real.

### C1 — Extract a headless rig runtime
- Pull the WebGL `tick`/`deform`/`fadeAlpha`/spring logic out of `index.html` into a framework-agnostic core module (the `runtime`).
- Core takes `{rig, params, dt}` and emits either (a) a rendered frame to a bound canvas, or (b) a deformed mesh + layer list for off-screen rendering.
- Must run in Node (with `headless-gl` or a software WebGL) and in the browser unchanged.

### C2 — Control API (HTTP + WebSocket)
- **HTTP:** load model, set/get params, trigger presets/gestures, health check.
- **WebSocket:** stream param updates at framerate; subscribe to frame events.
- Schema should reuse Phase 1's `P` shape verbatim — no translation layer.
- Auth: bearer token (v1), API keys (v1.1).

### C3 — Frame capture / streaming
- On-demand PNG/WebM capture per request or as a streamed MJPEG/WebRTC feed.
- Reuses B4's capture primitives but server-side.
- Decision needed: render-on-server (needs headless WebGL) vs render-on-client-with-server-coordination.

### C4 — Export formats
- **Standalone HTML bundle:** single self-contained `.html` with the rig + a minimal control panel (no build step). For sharing a character.
- **Docker image:** `docker run anime25rig --model ./my.psd --port 8080`.
- **Serverless function:** AWS Lambda / Cloudflare Workers shape for per-request frame rendering (constrained but cheap).

### C5 — Multi-instance orchestration
- Run N rigs behind one gateway; route by instance id.
- Session affinity for WebSocket streams.
- Limits + quota per instance.

### C6 — SDK / client library
- Thin TypeScript client wrapping the WS+HTTP surface so consumers don't hand-roll JSON.

---

## Open questions to resolve during Phase 2 planning

1. **Render location.** Server-side rendering needs a real WebGL in Node (`headless-gl`, `swc-shader-translator`, or a Chromium headless). Each has cost/complexity tradeoffs. Do we commit to server-render, or ship a "control plane only" API where the client still renders?
2. **Stateful vs stateless.** WebSocket implies sticky sessions. Can the HTTP-only path be truly stateless (params-in, frame-out), and is that enough for v1?
3. **Model loading.** Does the server store uploaded PSDs (and where — disk, S3, in-memory)? Or does the client upload the rig-definition JSON produced by `buildRig`, skipping PSD parsing server-side? The latter is cleaner and reuses Phase 1 work.
4. **Licensing/attribution.** Hosting models as a service may have different implications than client-side use. Re-confirm the `see-through` + sample-PSD attribution story (per existing README) before any hosted demo.
5. **Concurrency model.** One rig per process? Worker pool? How does the spring-physics state survive across requests without a long-lived WS connection?
6. **Scope of "controllable."** Is the API driving the same param space as the UI (full `P`), or a reduced "director" surface (presets, gestures, emotion vectors)? The latter is far more stable across versions.

---

## Planning checklist (when we start Phase 2)

- [ ] Re-read this doc and `01-phase1-features.md` to confirm what actually shipped (line refs will have moved).
- [ ] Resolve open questions 1–6 above with the user.
- [ ] For each resolved answer, write the workstream spec (C1–C6) to the same depth as Phase 1's A/B blocks.
- [ ] Add per-workstream checklists + acceptance criteria.
- [ ] Update `ROADMAP.md` status table.
- [ ] Decide Phase 2 implementation order and capture it here.

---

*Last touched: created alongside Phase 1 plan. Next edit: full expansion after Phase 1 completion.*
