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
- Lab colour contrast and sharpness (depth of field) bonus for the map; centre bias is applied (0.6–1.0).
- The separate "Тепловая карта" checkbox (`ovHeat`, general overlays) still uses the `AnalyzerLite` heuristic.
- `Evaluator` still contains an unused duplicate FFT / `spectralResidual` / `renderSaliencyGray`; only `FFT.spectralResidual` is used.
- Offscreen saliency render rounds width/height to powers of two (aspect slightly squashed for non-16:9 frames).
