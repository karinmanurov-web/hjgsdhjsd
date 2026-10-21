# Stage 7 — sequence and camera logic (TZ «Для кино/раскадровки»)

All logic lives in the `Continuity` IIFE (pure, tested in Node); UI hooks are thin. Not run in a browser.

## 1. Shot types (ECU … ELS)
- Auto label from the main object's size on screen. The object is treated as a full-length figure; `ratio` = its larger dimension in frame heights, `max(height, width·aspect·0.5)` so a wide object filling the frame width is not read as a wide shot.
- Thresholds (`SHOT_TYPES`): ELS < 0.15 · LS < 0.6 · MLS < 1.2 · MS < 1.8 · MCU < 2.6 · CU < 4.5 · ECU above. They are starting values (TZ gives no numbers) — one table to tune.
- Uses `Evaluator.screenItems` (analytic, from the frame's own overrides), so it works for any frame, not only the current one, and costs no render.
- Shown as a gold badge on the filmstrip thumbnail (tooltip: name and ratio) and as «План: …» in the camera panel. No main object or main object off-screen → no label.

## 2. Rule of 180° (line of action)
- **Line**: through two objects (not guides), on the floor plane (X, Z). Chosen in the new panel «Последовательность» (selects A / B) or automatically: the main object and the nearest other one. Invalid / identical ids fall back to auto. Saved in `project.actionLine {a, b, show}`.
- **Side** of the camera: sign of the 2D cross product (`camera.pos` vs the line), independent of camera height and yaw. Within `LINE_EPS_M` = 0.25 m of the line the shot is **neutral** (side 0): crossing the line *through* an on-the-line shot is legal, a direct side-to-side cut is not.
- **Check**: only between **adjacent** frames. Each frame uses its own object positions (`overrides`), so the line moves with the actors. Coincident actors → no line, no false alarm.
- **UI**: red badge with the number of jumps on the panel title, per-frame list (shot code · side · ⚠ message, click selects the frame), summary line, and a dashed orange line on the floor of the 3D editor (checkbox «Линия в 3D»; it lives in `scene`, not `rootGroup`, so it never appears in frames). Use the top view («Сверху») to see it. Also here: shot-type variety diagram (bars per ECU…ELS) with a hint when a 4+ frame storyboard uses fewer than 3 types.
- Not done from the TZ list: the 30° rule, on-screen direction continuity between frames.
