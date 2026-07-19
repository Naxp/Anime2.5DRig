# Phase 1 — Natural-Movement & Authoring Features

**Status:** ☐ Planned (not started)
**Scope:** 6 core features (B-series) + 4 default idle animations (A-series)
**Rule:** Implement A-series **first** (smallest code, highest perceived-life gain), then B-series in dependency order.

All line references are against the current `main`. **If a referenced line/function has moved, update this doc before continuing.**

---

## File map (where work lands)

| File | Role | Phase 1 changes |
|---|---|---|
| `index.html` | UI + WebGL runtime (single file) | Most A-series + B2/B3/B4 changes |
| `lib/rigger.js` | Rig builder, pure typed-array | B1 (visemes), B5 (folder support), B6 (collision anchors) |
| `lib/genericparts.js` | Built-in close-diff fallback | B1: add viseme mouth base images (parallel to existing `P.mouth`) |
| `lib/ag-psd.min.js` | Vendored PSD parser | **Do not touch.** |

---

## Key anchor points in current code

Confirmed against `main`:

| Symbol | Location | Used by |
|---|---|---|
| `const P` (param defaults) | `index.html:383` | all workstreams that add params |
| `const DEFAULTS` | `index.html:388` | reset + B2 |
| `const auto` (toggle flags) | `index.html:392` | A1–A4, B2, B3 |
| `const sliders` (id→key map) | `index.html:394` | new params auto-wire through here |
| `const presets` | `index.html:415` | B2 |
| `function applyRig` | `index.html:248` | B1, B5 (rig data shapes) |
| `function fadeAlpha` | `index.html:636` | B1 (new fade types) |
| `function deform` | `index.html:645` | A4, B1, B6 |
| `function tick` | `index.html:772` | A1–A4, B3 (main per-frame hook) |
| `if(auto.idle&&!camLive)` | `index.html:788` | A1, A3 (gaze + postural sway hook in here) |
| `if(auto.rand&&!camLive)` | `index.html:794` | A1 (replace uniform noise) |
| `if(auto.blink)` | `index.html:810` | A2 (rewrite blink state machine) |
| `for(const k in cur)` (smoothing) | `index.html:823` | A4 (head-velocity tracking) |
| `e.breath=`, `e.breathHead=` | `index.html:827-828` | A3 |
| `const headDX` | `index.html:831` | A4, B6 (wind + settle) |
| `var SLOTS` | `lib/rigger.js:24` | B1 (add viseme slots) |
| `function normName` | `lib/rigger.js:14` | B1 (viseme aliases), B5 (group recrusion) |
| `function buildRig` | `lib/rigger.js:374` | B5 |
| `(psd.children || []).filter(c => c.imageData)` | `lib/rigger.js:378` and `:529` | B5 (recurse into groups) |
| `function detectStrands` | `lib/rigger.js:290` | B6 (tip positions for collision) |

---

# A-series — Default idle animations

**Shared goal:** make the character feel alive with **no** input driver. All four compose with existing mouse/cam/mic paths (which already `Math.max`/`Math.min` over `tgt`). Add a global toggle row.

## A1 — Micro-saccades + gaze targeting

**Problem:** In idle, eyes are dead-still; `auto.rand` only adds uniform drift noise to `eyeX/eyeY` (`index.html:794`). Real gaze is **fixate → saccade → fixate**, not drift.

### Spec
- Replace the uniform-noise eye contribution in the `auto.rand` block with a **saccade state machine**.
- New state object (declare near `index.html:631` next to `rnd`):
  ```js
  let gaze = {
    ex: 0, ey: 0,            // current held target (-1..1)
    nextSaccade: 0,          // timestamp of next jump
    dwell: 0,                // current dwell duration
    jitterPhase: Math.random()*100  // for micro-jitter
  };
  ```
- Per-frame (inside `auto.idle && !camLive` block, `index.html:788`):
  1. If `now > gaze.nextSaccade`: pick new target within an elliptical ROI (closer to center than the existing `rnd.ex/ey` range — use radius ~0.5 weighted toward center), set `dwell = 600 + Math.random()*2400` ms, set `nextSaccade = now + dwell`.
  2. Apply target **instantly** to a separate `tgt.eyeX/eyeY` accumulator (saccade = ~20ms jump; the existing `cur += (tgt-cur)*dt*14` smoothing at `:823` will smear it ~70ms which is acceptable). *Option B (preferred for realism):* add a tiny 20ms fast-track by overriding `dt*14` → `dt*60` for eye channels only during the saccade window.
  3. Add **micro-saccade jitter** always-on: ~2–3 Hz sinusoid at amplitude 0.012, plus a small Gaussian noise at amplitude 0.004. This never stops, even mid-dwell.
- **Mouse/cam override preserved:** those blocks already set `tgt.eyeX/eyeY` directly (`:782`, `:786`), which is fine — saccades only run in the idle branch.
- Rare-event: 5% chance per saccade of a **smooth-pursuit** dwell (target drifts slowly across the ROI over the dwell instead of holding).

### Checklist
- [ ] Declare `gaze` state object next to `rnd`.
- [ ] Remove the `rnd.ex`/`rnd.ey` lines from the `auto.rand` block (keep `ax/ay/az/bd`).
- [ ] Add saccade scheduler inside the `auto.idle && !camLive` block.
- [ ] Implement jitter term (always-on, low amplitude).
- [ ] (Preferred) Add per-channel smoothing-speed override for eyes during saccade window.
- [ ] Implement 5% smooth-pursuit branch.
- [ ] Verify mouse-tracking still overrides (move mouse, eyes follow; release, saccades resume).
- [ ] Verify cam-tracking still overrides (turn on cam, eyes follow face).

### Acceptance
- Eyes visibly fixate on points and jump between them at a believable cadence.
- At rest there is faint continuous eye jitter visible at 100% zoom.
- Mouse/cam/mic still take precedence when active.

---

## A2 — Blink variation (double / half / asymmetric)

**Problem:** Current blink fires one identical full blink every 1.6–5.4s (`index.html:810`). Mechanical.

### Spec
- Rewrite the blink scheduler as an explicit state machine. Replace the `blinkT`-as-scalar logic with:
  ```js
  // blink.state: 'idle' | 'closing' | 'open' | 'reclose' | 'done'
  let blink = {
    state: 'idle', t: 0, nextAt: now+1800,
    plan: { closures: 1, depth: 1.0, eye: 'both' }  // 'both'|'L'|'R'
  };
  ```
- When scheduling a new blink (current `nextBlink` logic at `:811`), roll for type:
  - **Normal full** (70%): `closures=1, depth=1.0, eye='both'`.
  - **Half blink** (12%): `closures=1, depth=0.45, eye='both'` — common during concentration.
  - **Double blink** (13%): `closures=2, depth=1.0, eye='both'`, second closure ~120ms after first reopen.
  - **One-eye micro** (5%): `closures=1, depth=0.6, eye='L'|'R'` random.
- Use the existing timing envelope (close 80ms, hold ~340ms, open 160ms) but parameterize the **minimum lid value** by `depth`.
- Apply to the correct eye channel only:
  ```js
  if (blink.plan.eye !== 'R') tgt.eyeOpenL = Math.min(tgt.eyeOpenL, v);
  if (blink.plan.eye !== 'L') tgt.eyeOpenR = Math.min(tgt.eyeOpenR, v);
  ```
- Re-arm `nextAt` with the existing 1.6–5.4s distribution, plus the 18% "quick re-blink within 280ms" twist already present at `:811` (keep it).

### Checklist
- [ ] Define `blink` state object replacing the bare `blinkT`/`nextBlink` vars at `index.html:630`.
- [ ] Implement state machine: idle → closing → open → (reclose if closures>1) → done → idle.
- [ ] Wire the 70/12/13/5 probability roll at scheduling time.
- [ ] Parameterize min-lid by `depth`; apply per-eye by `eye`.
- [ ] Preserve the 18% quick re-blink twist.
- [ ] Verify each blink type occurs over a ~2-minute idle (eyeball the distribution).
- [ ] Verify cam/mic blink contributions still compose (Math.min still wins).

### Acceptance
- Over a 2-minute idle, all four blink variants visibly occur.
- No regressions to cam-driven blink (FaceMesh still drives lid closure).

---

## A3 — Breath-coupled postural sway + sighs

**Problem:** Breathing (`index.html:827`) and body lean (`tgt.body` from idle, `:792`) are independent sines. Posture feels detached from breath. No sighs/yawns.

### Spec
- **Couple posture to breath:** add a slow postural layer where head **leads** and body **follows with a lag** (mirror the existing head-lags-chest pattern at `:828`). New derived params:
  ```js
  e.postureLead   = 0.5 + 0.5*Math.sin(t*2*Math.PI/6.2);          // ~6.2s postural cycle
  e.postureFollow = 0.5 + 0.5*Math.sin(t*2*Math.PI/6.2 - 0.9);    // body lags ~0.9 rad
  ```
  Feed `e.postureFollow` into `tgt.body` (in addition to existing idle term). Feed `e.postureLead` into a new small head `angleZ` bias (amplitude ~0.04) — head initiates the lean.
- **Sigh/yawn scheduler:** new state object:
  ```js
  let sigh = { nextAt: now + 20000 + Math.random()*20000, t: -1, dur: 0 };
  ```
  When `now > sigh.nextAt`: start a sigh — a single deepened breath that, over `dur = 1600ms`:
  - Amplifies `e.breath` locally (multiply by up to 1.8 at peak).
  - Briefly raises `tgt.brow` by ~0.25 at peak.
  - Drops `tgt.angleY` by ~0.08 (chin dips).
  - Nudges `tgt.mouthOpen` to ~0.15 (a soft inhale/exhale mouth part).
  - Schedules next sigh 20–60s later.
- Mix sigh contributions in via `Math.max`/additive on `tgt.*` so they layer over other drivers.
- Add a **"Sighs" toggle** in the Auto panel (parallel to `tgIdle`), default ON.

### Checklist
- [ ] Add `postureLead`/`postureFollow` derivation in the breath block.
- [ ] Wire `postureFollow` into `tgt.body`; `postureLead` into `tgt.angleZ`.
- [ ] Declare `sigh` state object; add scheduler.
- [ ] Implement sigh envelope (1600ms, peak around 40%).
- [ ] Mix sigh into `tgt.breath/brow/angleY/mouthOpen` (use local multipliers, not global rewrites).
- [ ] Add `tgSigh` toggle in the Auto section; add to `auto` object and toggle wiring (`index.html:409`).
- [ ] Add `sigh: true` to `auto` defaults.
- [ ] Verify a sigh visibly fires within ~30s when toggled on.
- [ ] Verify turning "Sighs" off suppresses them while keeping normal breath.

### Acceptance
- Body lean visibly lags head lean on a slow cycle.
- Sighs occur ~every 20–60s and visibly deepen the breath + shift brows/mouth/jaw.

---

## A4 — Head-motion-coupled secondary action (hair settle + brow micro)

**Problem:** Strands track `headDX` directly (`index.html:831`); when the head reverses direction, hair instantly follows with no settle. Brows never react to head motion.

### Spec
- **Track head velocity:** keep a rolling sample of head yaw/lean. Add near `headDX` declaration:
  ```js
  // headDX history for velocity/zero-crossing detection
  if (!tick.headDXPrev) tick.headDXPrev = headDX;
  const headVel = headDX - tick.headDXPrev;
  if (!tick.headVelPrev) tick.headVelPrev = headVel;
  const zeroCross = (tick.headVelPrev * headVel) < 0;   // direction reversal
  tick.headDXPrev = headDX; tick.headVelPrev = headVel;
  ```
  *(If you dislike function-property storage, hoist these into a `let` block above `tick`.)*
- **Hair settle oscillation:** on `zeroCross`, inject a decaying sinusoid into each strand spring's **target** (not the spring state — let the existing spring absorb it):
  ```js
  if (!L.settleEnv) L.settleEnv = { amp: 0, phase: 0, t: 0 };
  if (zeroCross) { L.settleEnv.amp = Math.min(1.2, L.settleEnv.amp + Math.abs(headVel)*0.6); L.settleEnv.phase = 0; }
  L.settleEnv.t += dt;
  const settleOsc = L.settleEnv.amp * Math.cos(L.settleEnv.t * 18) * Math.exp(-L.settleEnv.t * 1.6);
  L.settleEnv.amp *= Math.exp(-dt * 1.6);
  // add settleOsc into the per-strand txv below
  ```
  `settleOsc` feeds the `txv` used by both stiff and soft springs (`:836`, `:840`).
- **Brow micro-reaction:** on upward head pitch (i.e. `headVel` indicating chin rising), do a tiny brow lift — add `Math.max(0, headVel) * 2.0` to `tgt.brow` (cap contribution at ~0.1).
- **Mouth-corner-on-blink micro:** when a blink's lid value crosses below 0.3 (closing), nudge `tgt.mouthForm` by +0.04 (subtle smile-twitch). Reset when blink reopens.

### Checklist
- [ ] Hoist `headDXPrev` / `headVelPrev` state above `tick` (or onto `tick`).
- [ ] Compute `zeroCross` each frame after `headDX`.
- [ ] Add `settleEnv` to every layer that has `L.spr` (initialize in `applyRig` at `:301`-ish, or lazily as above).
- [ ] Inject `settleOsc` into `txv` for both stiff and soft springs.
- [ ] Add brow-lift-on-pitch term to `tgt.brow`.
- [ ] Add mouth-corner-on-blink term to `tgt.mouthForm` (gate on blink lid crossing 0.3).
- [ ] Verify hair visibly settles/oscillates after a sharp mouse-driven head turn.
- [ ] Verify brow micro-lifts on upward pitch.
- [ ] Verify no jitter introduced at rest (zeroCross must not fire on noise — require `|headVel| > 1e-3`).

### Acceptance
- After a head reversal, hair visibly oscillates and damps (1–2 visible settle cycles).
- Brows subtly lift on upward pitch; subtle mouth-corner twitch on blink close.
- Zero spurious motion at rest.

---

# B-series — Core authoring features

**Dependency order:** B5 (folders) unlocks cleaner PSDs → B1 (visemes) → B6 (collision) → B3 (gestures) → B2 (timeline) → B4 (export). They're independent enough to reorder, but this sequence minimizes rework.

## B1 — Viseme set (a/i/u/e/o) for proper lip-sync

**Problem:** Mouth is a single `mouthOpen` scalar × one `mouthForm` slider. Mic/TTs both collapse into "jaw drop." No vowel shaping.

### Spec
- **Five viseme diff layers:** `mouth_a`, `mouth_i`, `mouth_u`, `mouth_e`, `mouth_o`. Each is a fade-layer driven by a 0..1 weight. Add slots to `SLOTS` (`lib/rigger.js:24`):
  ```js
  'mouth_a': { depth: 1.085, group: 'head', fade: 'viseme', viseme: 'a' },
  'mouth_i': { depth: 1.085, group: 'head', fade: 'viseme', viseme: 'i' },
  'mouth_u': { depth: 1.085, group: 'head', fade: 'viseme', viseme: 'u' },
  'mouth_e': { depth: 1.085, group: 'head', fade: 'viseme', viseme: 'e' },
  'mouth_o': { depth: 1.085, group: 'head', fade: 'viseme', viseme: 'o' },
  ```
- **Fallback synthesis:** if any viseme layer is missing, synthesize from the existing `genericparts.js` pattern. Add `P.mouth_a`…`P.mouth_o` base images alongside `P.mouth` (`lib/genericparts.js:8`). These can be simple procedural shapes initially (different mouth openings); upgrade to real art later. Wire through `genericOpts()` in `index.html` and the synth path in `buildRig` (`lib/rigger.js:506`).
- **Runtime weights:** add 5 params to `P` (`visemeA`..`visemeO`, default 0). Expose as **5 tiny sliders OR a single "viseme" radio/XY pad** in the Mouth section. Weights are **normalized** (sum to ≤1) before fade.
- **New fade type** in `fadeAlpha` (`index.html:636`):
  ```js
  if (L.fade==='viseme') {
    const w = ({a:e.visemeA,i:e.visemeI,u:e.visemeU,e:e.visemeE,o:e.visemeO})[L.viseme] || 0;
    return smooth(w);   // weight is pre-normalized 0..1
  }
  ```
- **Mic driver upgrade:** replace the single-band energy→`mouthOpen` mapping (`index.html:818`) with a 5-band FFT split → viseme weights (rough vowel estimate via formant bands). Keep `mouthOpen` from total energy. Add a "Vowel mode" toggle: simple (current) vs formant (new).
- **Deformation:** each viseme layer may slightly scale/translate the mouth; add a `deform` branch for `fade==='viseme'` that nudges y by the weight.

### Checklist
- [ ] Add 5 viseme entries to `SLOTS` (`lib/rigger.js:24`).
- [ ] Add `P.mouth_a`..`P.mouth_o` base images to `lib/genericparts.js` (procedural OK for v1).
- [ ] Extend `genericOpts()` (`index.html:352`) and the synth path (`lib/rigger.js:506`) to handle missing visemes.
- [ ] Add `visemeA..O` params to `P` and `sliders`.
- [ ] Add UI (viseme XY pad or 5 sliders) in the Mouth section.
- [ ] Add `fade==='viseme'` branch in `fadeAlpha`.
- [ ] Add `deform` branch for visemes (subtle scale/translate).
- [ ] Implement 5-band FFT → vowel weights in mic path; gate behind "Vowel mode" toggle.
- [ ] Verify all 5 visemes visibly distinct when manually driven.
- [ ] Verify missing visemes auto-synthesize from `genericparts.js`.
- [ ] Verify existing `mouth_open`/`mouth_close` behavior unchanged when all visemes = 0.

### Acceptance
- Five visibly distinct vowel shapes; mic "Vowel mode" produces plausible a/i/u/e/o from speech.
- No regression to legacy single-scalar mouth when visemes are zero.

---

## B2 — Preset save/load (localStorage) + lightweight keyframe timeline

**Problem:** `presets` is hardcoded (`index.html:415`). No way to author, save, or sequence expressions.

### Spec
- **User presets:**
  - New section "My Presets" below Expression Presets.
  - "+ Save current as preset" button → prompts for name → captures a snapshot of all keys in `P` → stores in `localStorage` under `a25r.userPresets` (JSON). Load on startup.
  - Each user preset is a clickable `.btn` that calls `setSlider` for every key (reuse `index.html:406`).
  - Delete button per preset.
- **Timeline (lightweight):**
  - A new collapsible `.sec` "Timeline" at the bottom of the panel.
  - Concept: a list of **clips**. Each clip = ordered list of **keyframes**. Keyframe = `{t_ms, preset|params, ease}`.
  - v1 controls: add keyframe at playhead (captures current `P`), set time, choose ease (linear/in-out/back), delete. Play/pause/stop. Loop toggle.
  - Playback: a new `timeline` state object drives `tgt` directly (highest-priority driver, overrides idle). Interpolate between adjacent keyframes by ease over the time delta.
  - Persistence: clips saved to `localStorage` `a25r.timeline`.
- **Data shape:**
  ```js
  // localStorage schema
  a25r.userPresets = { "My Smile": {eyeOpenL:0,...}, ... }
  a25r.timeline    = {
    clips: [{ name:"wave", fps:30, loop:true, keys:[
      {t:0,   p:{...}, ease:'linear'},
      {t:400, p:{...}, ease:'inOut'},
      ...
    ]}],
    activeClip: 0
  }
  ```

### Checklist
- [ ] Add `loadStore()`/`saveStore()` helpers wrapping `localStorage` with try/catch.
- [ ] Implement "+ Save preset" → snapshot `P` → persist → re-render preset list.
- [ ] Render user presets as `.btn` row; click applies; long-press/✕ deletes.
- [ ] Add Timeline `.sec` (collapsed by default).
- [ ] Implement clip + keyframe data model.
- [ ] Implement playhead + interpolation (linear, in-out, back eases).
- [ ] Wire timeline as highest-priority driver in `tick` (when playing).
- [ ] Persist clips to `localStorage`.
- [ ] Verify save → reload → preset still there.
- [ ] Verify clip playback visibly animates between two keyframes.

### Acceptance
- User can save/load/delete presets across sessions.
- A 2-keyframe clip smoothly animates; easing is visibly different between linear/in-out/back.

---

## B3 — Gesture system for arms/hands

**Problem:** Arms are just `armY` + `armPos` (`index.html:731`). No named poses, no animation.

### Spec
- **Gesture library:** predefined time-sampled curves for `(armYL, armPosL, armYR, armPosR)` over a duration. v1 gestures:
  - **idle** (default): subtle drift on `armPos`.
  - **wave**: right arm raises and oscillates 3× over 1.2s, then lowers.
  - **peace**: both arms to a static V pose, hold.
  - **thinking**: right hand to chin, hold with slight wobble.
  - **point**: right arm extends forward (approx via `armPos`), hold.
- **Independent L/R:** expose `armYL/armYR/armPosL/armPosR` (split current `armY`/`armPos` into per-side). Backward-compat: legacy sliders drive both sides when L/R not separated.
- **Gesture driver:** new highest-priority driver (below timeline, above idle) that overrides `tgt.armY*`/`tgt.armPos*` while a gesture is active.
- **UI:** new "Gestures" `.sec` with buttons per gesture. "idle" returns to default. Optional "mirror" toggle for left-handed variants.
- **Blending:** gesture → idle cross-fade over 300ms on start and end.

### Checklist
- [ ] Split `armY`→`armYL/armYR`, `armPos`→`armPosL/armPosR` in `P`, `sliders`, `deform` (`index.html:731`).
- [ ] Backward-compat: if old sliders exist, alias to both sides.
- [ ] Define gesture library (curves) as a data object.
- [ ] Implement gesture state machine with 300ms in/out blend.
- [ ] Add Gesture `.sec` with buttons + mirror toggle.
- [ ] Wire gestures into `tick` between timeline and idle drivers.
- [ ] Verify each gesture plays and returns cleanly to idle.

### Acceptance
- All 5 gestures visibly distinct; smooth blend in/out; no snap.

---

## B4 — Scene/video export (WebM + PNG sequence)

**Problem:** Can only export a static cleaned PSD (`btnSavePsd`, `index.html:440`). No way to capture animation.

### Spec
- **WebM export** via `MediaRecorder` on the WebGL canvas:
  - `canvas.captureStream(30)` → `MediaRecorder` with `video/webm;codecs=vp9` (fallback vp8).
  - "Record" button (red dot) toggles recording; stops → downloads `avatar_<timestamp>.webm`.
  - Show recording duration counter.
- **PNG sequence export** (frame-stepped):
  - Pause `requestAnimationFrame(tick)`.
  - Drive a virtual clock: for each frame `f` in `[0, totalFrames)`, set `now = startMs + f*1000/fps`, call `tick(now)` once, then `canvas.toBlob('image/png')` → queue download (zip via a tiny vendored zip writer OR sequential triggers — v1 can do sequential single-file downloads with a warning, or use the browser's `showSaveFilePicker` API where supported for a folder).
  - Resume real `requestAnimationFrame`.
  - UI: "Export PNG sequence" → prompt for duration + fps → progress bar.
- **Coupling with B2 timeline:** if a clip is loaded, PNG export uses the clip duration as default and advances the timeline playhead per frame.
- **Audio:** v1 exports video-only. (Audio-in is out of scope; document as known limitation.)

### Checklist
- [ ] Add `MediaRecorder` path with codec fallback.
- [ ] Record/stop button + duration UI.
- [ ] Verify file plays in VLC + browser.
- [ ] Implement frame-stepped PNG export (virtual clock).
- [ ] Progress bar during export.
- [ ] Wire default duration to active clip length if B2 present.
- [ ] Graceful error if `captureStream` unsupported (Safari < 14).
- [ ] Document audio-less limitation in README.

### Acceptance
- WebM plays correctly at 30fps with current animation.
- PNG sequence renders N frames matching the on-screen animation.

---

## B5 — PSD layer-group (folder) support + multi-layer auto-grouping

**Problem:** README explicitly lists folders as unsupported. `buildRig` only reads top-level `c.imageData` children (`lib/rigger.js:378`, `:529`).

### Spec
- **Recursive child collection:** replace the flat `.filter(c => c.imageData)` in both `buildRig` (`:378`) and `cleanPsdLayers` (`:529`) with a recursive walk:
  ```js
  function collectImageLayers(node, out) {
    (node.children || []).forEach(function(c){
      if (c.imageData) out.push(c);
      else if (c.children) collectImageLayers(c, out);
    });
    return out;
  }
  ```
  Apply group visibility (`c.hidden` and parent `hidden`) — skip hidden subtrees unless a new "include hidden layers" toggle is on.
- **Group → logical strand group:** when a folder contains multiple `front hair_*` children, optionally treat the whole folder as one strand group (configurable). Default: flatten (current behavior, safest).
- **Pass-through blending:** PSD groups can have pass-through blend mode; since the runtime composites per-layer alpha-over, pass-through flattens naturally — no special handling needed. Document this.
- **Cleanup:** after recursing, also fix up `cleanPsdLayers` so denoise+trim runs on nested layers (it already operates per-layer; just feed it the recursive list).
- **Naming:** prefix nested layer names with their group path? **No** — keep raw names so `SLOTS` matching still works. Instead, log a warning if a nested layer name collides with a sibling.

### Checklist
- [ ] Add `collectImageLayers` helper.
- [ ] Replace flat filter at `lib/rigger.js:378` and `:529`.
- [ ] Respect `c.hidden` and parent hidden chain (add "include hidden" toggle default OFF).
- [ ] Add collision warning for duplicate names.
- [ ] Test with a PSD that has folders (author one in Photoshop, or synthesize via ag-psd write).
- [ ] Update README "Layer Naming" + "Known Limitations" (remove the "folders unsupported" line).
- [ ] Verify a flat PSD still works identically.

### Acceptance
- A PSD with nested folders rigs and animates correctly.
- Flat PSDs unchanged.
- README updated.

---

## B6 — Hair/body self-collision + directional wind

**Problem:** Strands are pure springs with hardcoded sinusoidal wind (`index.html:835`); tips pass through face/torso.

### Spec
- **Collision anchors:** reuse `anchors.face`, `anchors.neckPivot`, `anchors.bodyPivot`. Compute a small set of **repulsion disks** (face disk at `FC`, radius ~`faceW*0.55`; neck disk at `NP`, radius ~`faceW*0.3`; chest disk at `CHEST`, radius already computed).
- **Per-strand-tip repulsion:** for each strand spring, after computing `sp.soft.x`, apply a corrective offset away from any disk whose center is within `diskR + tipMargin`:
  ```js
  // pseudo, inside the strand spring loop at index.html:834
  const tipX = S[s].x + sp.soft.x;   // approx tip world-x
  for (const disk of disks) {
    const dx = tipX - disk.cx, dy = S[s].tipY - disk.cy;
    const d = Math.hypot(dx,dy) || 1e-4;
    const pen = (disk.r + 8) - d;
    if (pen > 0) { sp.soft.x += (dx/d) * pen; /* and bias y */ }
  }
  ```
  (Strand `tipY` is already in `S[s]` from `applyRig`. Tip world-x is approximate; v1 uses strand base x + soft offset — good enough.)
- **Directional wind:** replace the hardcoded `wind` sinusoid (`:835`) with:
  ```js
  const windBase = auto.idle ? (1.8*Math.sin(t*0.8+sp.phase) + 1.0*Math.sin(t*1.9+sp.phase*2.3)) : 0;
  const windDir = e.windX;   // new param, -1..1, default 0
  const wind = windBase + windDir * 2.5;
  ```
  Expose `windX` as a slider in the Body & Physics section.
- **Gust scheduler (optional v1.1):** occasional 400ms wind gusts that decay.

### Checklist
- [ ] Build `disks` array once per frame from anchors (face, neck, chest).
- [ ] Add tip-repulsion in the strand spring loop.
- [ ] Add `windX` param + slider.
- [ ] Replace hardcoded wind term with `windBase + windDir*2.5`.
- [ ] Verify long hair no longer intersects the face on sharp turns.
- [ ] Verify `windX` slider visibly biases hair drift.
- [ ] Verify no perf regression (collision is O(strands × disks), trivial).

### Acceptance
- Hair tips deflect around the face/neck instead of clipping.
- Wind slider shifts the resting drift direction.

---

# Cross-cutting concerns (apply to all workstreams)

- **Param naming:** new params go in `P` (`index.html:383`) AND `DEFAULTS` (`:388`) AND `sliders` (`:394`) so reset + slider wiring just works. Use `pXxx` ids following the existing convention.
- **Toggle naming:** new toggles go in `auto` (`:392`) AND get a `.tg` chip AND get wired in the toggle array at `:409`.
- **Localization:** all new UI strings in English (per the recent translation pass). No Japanese in new code.
- **Theme:** reuse `--acc` / `--acc2` / `--line` / `.sec` / `.row` / `.btn` / `.tg`. New components match.
- **Testing:** there is no test harness today. For each workstream, add a manual verification step (the final checklist item). Phase 2 will introduce automation.
- **Backward compat:** every workstream must leave existing behavior intact when its new toggles/params are at defaults. This is verified by the "no regression" acceptance items.

---

# Suggested implementation order

1. **A1** → **A2** → **A3** → **A4**  *(perceived life, fast wins, share the `tick`/`auto` infrastructure)*
2. **B5**  *(unblocks better test PSDs for everything below)*
3. **B1**  *(visemes — biggest realism feature, needs B5's cleaner PSDs to test fully)*
4. **B6**  *(collision + wind — independent of B1)*
5. **B3**  *(gestures — uses the now-per-side arms)*
6. **B2**  *(timeline — benefits from having gestures + visemes as targets)*
7. **B4**  *(export — final, records everything else)*

Each workstream is independently shippable; reorder is fine if dependencies are respected.
