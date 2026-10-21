# Stage 7 — sequence and camera logic (TZ «Для кино/раскадровки»)

All logic lives in the `Continuity` IIFE (pure, tested in Node); UI hooks are thin. Not run in a browser.

## 1. Shot types (ECU … ELS)
- Auto label from the main object's size on screen. The object is treated as a full-length figure; `ratio` = its larger dimension in frame heights, `max(height, width·aspect·0.5)` so a wide object filling the frame width is not read as a wide shot.
- Thresholds (`SHOT_TYPES`): ELS < 0.15 · LS < 0.6 · MLS < 1.2 · MS < 1.8 · MCU < 2.6 · CU < 4.5 · ECU above. They are starting values (TZ gives no numbers) — one table to tune.
- Uses `Evaluator.screenItems` (analytic, from the frame's own overrides), so it works for any frame, not only the current one, and costs no render.
- Shown as a gold badge on the filmstrip thumbnail (tooltip: name and ratio) and as «План: …» in the camera panel. No main object or main object off-screen → no label.
