# Anime2.5DRig

Demo: https://852wa.github.io/Anime2.5DRig/

A 2.5D avatar tool that automatically rigs and animates a separated-part PSD just by dropping it onto the browser.
It automates the setup (mesh division, deformation, physics configuration) that was previously done manually.
No installation required — all processing is client-side.

## Demo / Usage

1. Publish this repository on GitHub Pages (Settings → Pages → Branch: main / root), or run locally:
   ```
   python -m http.server 8000
   # → http://localhost:8000
   ```
2. Open the page, drop a separated-part PSD (or click "Load sample.psd").
3. Auto-rigging runs and the avatar animates immediately with idle motion, blinking, lip-sync, and hair physics.

> Camera tracking (MediaPipe FaceMesh) and mic lip-sync only work over https or localhost (due to browser permission restrictions). Drop-and-play works even when opened with `file://`.

## What happens automatically

- Accepts [see-through](https://github.com/shitagaki-lab/see-through) output PSDs directly (`mouth`→`mouth_open` auto-rename)
- **Synthesizes generic close-diff parts when closed-eye/closed-mouth diffs are missing** (scaled and positioned to match anchors, auto-adjusted to eyelash/mouth color; fine-tune position and angle via dedicated sliders)
  - Prefers `eye_close.psd` (both eyes on one layer, spaced apart) / `mouth_close.psd` from the repository root. Falls back to built-in data.
- Low-alpha noise removal (connected component filtering)
- **Automatic left/right separation** of eyes, eyebrows, eyelashes, and closed-eye parts (determined by connected component centroids)
- **Automatic anchor detection** for eyelid position, iris center, mouth, neck pivot, etc.
- **Automatic hair strand detection** (peak detection of hair-tip contour, up to 6 strands per layer)
- **Pseudo-3D head rotation** via per-layer depth assignment (parallax + shear)
- Per-strand double spring physics (**stiff at root, fluffy at tip**), chest bounce, breathing
- **Cross-fade** for closed-eye/closed-mouth diffs, iris stencil clipping (confined to eye whites)

## Layer Naming Convention

Layer names (Japanese "のコピー" and full-width characters are auto-normalized):

| Layer Name | Content | Required | Notes |
|---|---|---|---|
| `face` | Face base | Required | Anchor reference. This one is mandatory |
| `eyewhite` | Eye whites (both sides) | Optional | Auto-separated left/right |
| `irides` | Irises (both sides) | Optional | Target for gaze movement and iris scaling |
| `eyelash` | Eyelashes (open eye) | Optional | |
| `eye_close` | Closed eyes | Optional | Cross-fades during blinking |
| `eyebrow` | Eyebrows (both sides) | Optional | Target for angle and vertical control |
| `mouth_open` | Open mouth | Optional | Opens the jaw by degree |
| `mouth_close` | Closed mouth | Optional | |
| `nose` | Nose | | |
| `ears` | Ears | | |
| `earwear` | Ear accessories | | |
| `neck` | Neck | | Top follows the head |
| `topwear` | Upper body clothing | | Subject to breathing and chest bounce |
| `bottomwear` | Lower body clothing | | |
| `handwear` | Arms/hands | | Target for arm height control |
| `headwear` | Hat, headband, etc. | | |
| `front hair` | Front hair | | Strand physics + 3-block manipulation |
| `back hair` | Back hair | | Strand physics |

- **Splitting hair across multiple layers**: Use `front hair_1`, `front hair_2`, `back hair_1`, etc. with `_number` suffixes. Each layer becomes an independent strand group with its own physics (strand count is auto-determined from layer width).
- Layers with non-convention names are still loaded (position-based estimation of head/torso, with follow behavior only).
- Layer groups (folders) are now supported (the rig recurses into folder children). Flatten still works the same.
- **Neck and torso**: Using see-through output as-is can make it difficult to resolve the neck-torso depth relationship, which may cause artifacts at the boundary during animation. If this happens, merging the neck into the torso layer (no neck layer, with the neck included in topwear) tends to work better.
- Square canvas is recommended (tested with 768×768 to 2048×2048).

## Features

Expression presets (smile/surprise/squint/wink left/right), independent left/right eye openness, eyebrow angle (independent + symmetric), gaze, iris scale, eye/mouth "close-ease" thresholds, front hair 3-block manipulation with **dedicated front-hair sway and softness**, arm height/position, chest bounce (adjustable intensity and position), body tilt, idle/random motion, random lip-sync, mic lip-sync, mouse tracking, **webcam tracking** (head XYZ, left/right blink, mouth, gaze), background switching (transparent/green screen).

## Roadmap

Work is tracked in [`docs/`](./docs):

- **[Phase 1 — Natural-movement & authoring features](./docs/01-phase1-features.md)** — 4 default idle animations (saccades, blink variation, breath-coupled sway + sighs, motion-coupled secondary action) + 6 core features (visemes, preset/timeline, gestures, video export, folder support, hair collision/wind).
- **[Phase 2 — Headless API & standalone export](./docs/02-phase2-api-export.md)** — host rigs as controllable API instances. *Planning stub; will be expanded after Phase 1.*

See [`docs/ROADMAP.md`](./docs/ROADMAP.md) for the master index and status.

## Structure

```
index.html      App body (UI + WebGL runtime)
lib/rigger.js   Auto-rig generation (pure TypedArray implementation, testable in Node)
lib/ag-psd.min.js  PSD parser (ag-psd, MIT)
lib/genericparts.js  Generic closed-eye/closed-mouth diff parts (built-in fallback)
eye_close.psd    Source image for closed-eye diff (optional, replaceable)
mouth_close.psd  Source image for closed-mouth diff (optional, replaceable)
sample.psd       Sample model (place your own)
```

The runtime uses WebGL1 (mesh warp + stencil). External communication is limited to MediaPipe CDN loading when camera tracking is enabled.

## Known Limitations

- The app does not decompose single illustrations into parts. Use [see-through official demo (HuggingFace Space)](https://huggingface.co/spaces/24yearsold/see-through-demo) to split and drop the resulting PSD (post-processing is fully automatic).
- Mouth opening is a simplified diff-switching + deformation expression (smoother with more intermediate diffs).
- Depth is based on a name-based fixed table (layer order respects the original PSD).

## License

MIT (bundled ag-psd is also MIT, MediaPipe references Apache-2.0 via CDN).

For single-illustration layer decomposition, usage of [shitagaki-lab/see-through](https://github.com/shitagaki-lab/see-through) (Apache-2.0, SIGGRAPH 2026) is assumed. This tool is an independent third-party tool responsible for post-processing and rigging of that project's output PSDs, and does not include see-through's code or models.
**The copyright of sample PSD illustrations belongs to their respective authors.** When distributing your own models, use your own copyrighted material.
