# Stage 6 — «Улучшить»: сила коррекции и охрана замысла (ТЗ 10.1–10.3)

Status: **implemented in the main HTML; logic executed in Node, browser run still pending.**
Target branch: `67pocoyo`, file: `deepseek_html_20261008_fbe883.html`.

## Что было
`ImproveEngine` перебирал фиксированные шаги (камера до 0.875 м, yaw до 12°, фокусное ×1.25 и т.д.) без лимитов
и не проверял, что замысел автора сохранён. Критерии приёмки этапа 6 («смещения не превышают лимитов силы»,
«граф отношений не нарушается») не выполнялись и не проверялись.

## Что сделано
- **Сила** (`project.improveStrength`, селектор «Сила улучшения», в истории Undo): Мягко / Средне / Сильно.

  | | углы yaw/pitch/roll | сдвиг камеры и цели | фокусное | λ |
  |---|---|---|---|---|
  | Мягко | ±2° | 2% расстояния до главного объекта | ±5% | 0.06 |
  | Средне | ±5° | 5% | ±10% | 0.04 |
  | Сильно | ±10° | 10% | ±15% | 0.02 |

  Пол расстояния — 1 м (рядом с объектом лимит не схлопывается в ноль). Фокусное по % — наше решение
  (в ТЗ только «до ~15%» на максимуме); значения стартовые, подлежат калибровке.
- **Лимиты считаются от исходного кадра**, а не от предыдущего шага → накопление правок не обходит лимит.
- **Граф отношений** (`relationGraph` / `graphViolations`): порядок слева-направо и сверху-вниз, порядок по размеру,
  перекрытия и «кто спереди», объект не вышел из кадра, направление взгляда (`facing`). Зона нечувствительности
  1% (позиции), 15% (размер), 5% площади (перекрытие), чтобы почти равные пары не «мерцали».
  Кандидат с нарушением отбрасывается до оценки (дешевле) и учитывается в `rejected.reasons`.
- **Поиск**: до 3 шагов, на каждом — лучший кандидат по `J = Score − λ·D` (D — средняя доля израсходованного
  бюджета по 6 параметрам). Бюджет времени 2.5 с (`truncated`, если не успели).
- `Evaluator.screenItems(frame)` — экранные рамки объектов (вход графа).
- Панель «Улучшить»: описание правки словами (`describeChange`), лимиты, сколько вариантов отклонила охрана и почему.
- Предложение «лучшее» показывается, только если оно > текущего кадра на 0.2 п.п.; три лучших варианта (по J) — всегда.

## Что НЕ сделано (осознанно)
- Правки объектов (сдвиг в плоскости кадра, масштаб ±10%) — оптимизатор по-прежнему двигает только камеру/цель/фокусное.
  Якоря (`anchor`) станут актуальны вместе с этим.
- Контрвес и «пунктирные предложения» (ТЗ 10.3, последний абзац).
- Призрак исходного положения и стрелки сдвигов на кадре.
- Покадровая калибровка λ и порогов.

## Проверки
- В приложении (раннер тестов): лимиты, переход yaw через 0°, граф отношений, и по всем трём силам — все предложения
  в пределах лимитов и без нарушений графа, половина шага тоже в лимитах. **В браузере не запускались.**
- В Node (движок вынут из HTML, камера и сцена синтетические): 56 проверок, все прошли. Прямое сравнение с поиском без охраны:
  без охраны выбираются варианты с потерей перекрытия / выходом объекта из кадра, с охраной — ближайший безопасный.


## Follow-up: частичное применение правок (ТЗ 10.3, шаг 4)
- У варианта, где изменено ≥2 параметров, в панели «Улучшить» появляются галочки: позиция камеры, цель, yaw, pitch,
  roll, фокусное. «Применить» и «Применить половину шага» берут только отмеченное.
- При смене галочек считается итог выбранного набора; если набор нарушает замысел или лимиты — кнопка блокируется
  и пишется причина.
- `ImproveEngine.apply(result, fraction, parts)` теперь **всегда** перепроверяет лимиты и граф отношений для того, что
  реально будет записано (включая «половину шага», которая раньше не проверялась). `previewParts` даёт ту же проверку
  заранее, чтобы объяснить причину отказа, а не показывать «кадр изменился».
- Нужна именно перепроверка: найдена сцена, где полный вариант (позиция + yaw) допустим, а один yaw уводит объект из кадра.
- Проверки: Node — все подмножества параметров всех предложений (97 + 54 половины шага) совпадают с прямой проверкой
  графа, плюс фиксированная сцена-контрпример; в раннере приложения — разбор/слияние частей и подмножества лучшего
  варианта. **В браузере не запускались.**

## Follow-up: ghost «было/стало» (Improve panel)

The ghost itself was already in the repo (commits `0a1ef16`, `0ca75b7`). Changes on top:

- **Letterbox bug fixed.** Arrow coordinates (u,v) are relative to the frame rect, but were mapped onto the whole renderer canvas including the black bars, so arrows were off (e.g. v=0.2 → 120 px instead of 165 px on 800×600 with a 800×450 rect). The ghost canvas is now cropped to `rect` for both layers.
- **Follows the checkboxes.** The ghost redraws on every `.iaPart` toggle for the shown variant; unchecked parts show the original with no arrows.
- **Arrows for all objects**, main object emphasised («было»/«стало» labels); an object that leaves the frame gets a red «ушёл из кадра» marker, and the caption flags design-guard violations.
- **Schematic fallback** (640 px, height from `frame.aspect`, default 16:9) when the frame preview is not open.
- Pure logic `ImproveEngine.ghostMoves` + in-app tests; Node harness covers picture/schematic modes. Not yet run in a real browser.

## Follow-up: object edits in Improve with anchors (TZ 10.1)

- «Улучшить» may now shift an object in the frame plane (right/left/up/down, full and half step) and scale it (±, full and half step). **Anchors (⚓), locked objects and guides are never touched**: they get no limit entry, so `withinLimits` rejects any frame that moves them, even a forged one.
- Limits scale with strength: shift = 3/5/8 % of frame width at the object's depth, scale = ±4/7/10 %. Measured from the original frame, like the camera limits. Deviation cost D gets a seventh term (the most-stretched object).
- Scaling keeps the object's bottom in place (centre rises by half of the height growth), so objects do not sink into the floor.
- Object changes are separate checkboxes in the variant (`obj:<id>`), join partial apply, half step and the guard re-check. `apply` writes into the live objects of the current frame (the current frame has no overrides) and refuses a stale suggestion if the scene was edited by hand.
- Project field `improveObjects` (default true) + checkbox «Двигать и масштабировать объекты»; the result panel lists protected objects («Не трогаю: …»).
- Tested in Node with a synthetic pinhole scene (anchor, lock, guide, movable main/background): limits, no forbidden overrides, guard, partial apply (camera untouched), half step, `objects:false`. Not yet run in a real browser.
