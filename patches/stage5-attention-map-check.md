# Stage 5 — attention map check (TZ 9.4, spectral residual)

Checked in Node (pure logic extracted from the HTML) and fixed what did not hold. **Not run in a real browser**; the focus threshold (smoothstep 0.3–0.75) is not calibrated on real rendered scenes.

## What was verified
- `FFT.fft1d/fft2d` equal a naive DFT and round-trip (error < 1e-6).
- `FFT.spectralResidual` follows TZ 9.4: gray → FFT → log amplitude → minus smoothed → inverse FFT with the original phase → square → blur. A small bright object on a flat ground gets ≥ 70 % of the attention mass inside its (padded) box; an odd-one-out in a regular texture is the peak; a global contrast change keeps the map (corr 1.000). Note: the map lights up object **edges**, not the pixel at the object's centre, so tests measure mass, not argmax.

## Fixed
1. **Phantom spike on flat/empty frames.** Zero residual → zero phase → inverse FFT gives a delta in the top-left corner; an empty frame got "clarity" 0.73. Flat input now returns an all-zero map (clarity neutral 0.5).
2. **S_focus leaked.** The map is blurred, so mass spills outside the object's tight box: a single perfectly framed object had focus fraction 0.25 → score 0. The box is now padded by the blur width (`FOCUS_PAD` u 0.05, v 0.09): 0.85 → score 1.0; with an equal rival 0.43 → 0.19.
3. **Saliency and `FrameRenderer.renderFrame` ignored the frame's own object state.** They rendered the live scene, so Improve candidates with object edits (stored only in `frame.overrides`) were scored by focus/clarity as if nothing moved, and the before/after ghost picture did not show object edits (arrows did). New `SceneManager.withFrameState(frame, fn)` shows the frame's objects, runs `fn`, restores position/rotation/scale/visibility/colour (also on throw); no-op when the frame has no overrides.
4. **Wrong heat data in the overlay.** "Тепловая (оценка)" drew `AnalyzerLite.heatData` (size heuristics), the computed saliency (`lastEval.evalOverlay.heatmapData`) was ignored. It now draws the whole attention map as a heat image (transparent → yellow → red); falls back to the peak blobs.

## Not done (TZ extras)
- The separate "Тепловая карта" checkbox (`ovHeat`, general overlays) still uses the `AnalyzerLite` heuristic.
- `Evaluator` still contains an unused duplicate FFT / `spectralResidual` / `renderSaliencyGray`; only `FFT.spectralResidual` is used.
- Offscreen saliency render rounds width/height to powers of two (aspect slightly squashed for non-16:9 frames).

## Follow-up: Lab colour contrast and sharpness bonus (TZ 9.4 extras)

Map = `0.65 · spectral residual + 0.35 · colour contrast`, optionally × sharpness, × centre bias (0.6–1.0), normalised to max 1 (an all-zero map stays zero).

- **Colour contrast.** sRGB → CIELAB (D65); ΔE from every pixel to the frame's mean colour; ΔE ≤ 1 (just-noticeable) = 0, ΔE ≥ 50 = 1; lightly blurred. The renderer's RGB is now kept (the map used grey only). Checked: white = (100,0,0), sRGB red = (53.24, 80.09, 67.20).
- **Luminance gain.** Spectral residual is contrast-invariant: a frame whose brightness differs by one or two levels produced a "confident" map out of noise and drowned the colour term. Its weight is now scaled by `clamp(range/0.1, 0, 1)`, so an equal-luminance object is found through colour (mass in its box 0.62 vs ~0.17 before).
- **Sharpness bonus (DoF).** Blur circle on the sensor from TZ 8.4: `b = f²/(N(s−f))·|z−s|/z` (background z→∞). Weight = 1 inside the permissible circle `c`, then `1/(1 + b/c − 1)`; a fully blurred point keeps 45 % of its attention (`SHARP_BASE`). Cross-check: at the TZ example's near/far limits (85 mm, f/1.8, 3 m → 2936/3067 mm) `b = c = 0.03 mm`. Depth comes from the analytic ID/depth map (object-centre distance to the camera, not along the axis; 64×H grid sampled for the 64×32 map).
- **Switch.** `frame.lens.dof` (TZ's `lens.dof: bool`, default off, as TZ says evaluation runs on the sharp picture unless asked) — checkbox «Резкость влияет на карту внимания» in the camera panel. Aperture and focus distance come from the frame; Improve does not change them. The focus criterion's explanation says when sharpness was applied.
- Tests: Node (`Lab`, blur at TZ limits, weights, depth→sharpness map, mix, flat frame, wiring) and in-app. Not run in a browser; the 0.65/0.35 mix, `SHARP_BASE`, `COLOR_DE_FULL` and `LUMA_RANGE_FULL` are reasonable defaults, not calibrated on real renders.

## Follow-up: symmetry per TZ 9.3 and the TZ section 14 audit

- **Audit of TZ 14 pure-function tests in Node** (extracted from the HTML): thirds point > 0.95 ✓ (1.000), centre ≈ 0.29 ✓ (0.291), FF 50 mm 3:2 FOV 39.60 / 26.99 / 46.79 ✓, FF 85 mm f/1.8 s=3 m → 2.936 / 3.067 m ✓, vignette FF 24 mm 0.304 ✓; dolly-zoom and Improve limits were already covered by in-app tests and the Node harnesses. The one gap was symmetry.
- **Symmetry was a proxy**: `gauss(|cx − 0.5|, 0.18)` of the main object's centre. Now `Evaluator.s_mirrorSymmetry` compares the left and right halves of the frame's luminance (TZ 9.3), taken from the same offscreen render as the attention map (objects use unlit `MeshBasicMaterial`, so lighting does not skew it). The proxy remains the fallback when there is no GPU render.
- **Normalisation.** The TZ formula `1 − mean|L − L'|` is ≈ 1 for any sparse frame (small objects on black), so it cannot tell a shifted object from a centred one. Used: `S = 1 − Σ|L − L'| / Σ(L + L')` after subtracting the background level and a light blur (≈3 % width). Results: one object or a mirrored pair on the axis → 1.00; shifted by 2 / 5 / 10 % of the width → 0.80 / 0.40 / 0.04; mirror images score equally; an empty frame → neutral 0.5 (flagged `empty`, falls back to the proxy). Profile «Симметрия» (weight 0.30) and Improve now see the whole scene, not one object.
- Not run in a browser.
