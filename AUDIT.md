# Progressive Lens Analyzer: Audit

Scope: `index.html` as committed in the first commit on this branch. The page was loaded in headless Chromium (1500 px desktop and 390 px phone viewports), timed, screenshotted, and the optics model was extracted into a Node harness and probed numerically. No JavaScript runtime errors or console warnings occur on load or on any control change.

## Summary verdict

The tool loads cleanly, every control triggers a re-render, and the core chairside stories do come through: adding +0.25 DS at distance visibly lifts every sample point including near, and an over-minused current pair shows the distance-clear band sitting below the fitting cross, which is exactly the "chin up to find the plus" picture. Three things undermine it as designed today:

1. The nasal inset is applied in the temporal direction, so the near zone sits on the wrong side of the lens for both eyes.
2. "Copy New Rx to Current" triggers nine full renders and freezes the page for about 6 seconds.
3. The "OC fit" control only moves the drawn crosshair. The lens model, heatmaps and metrics do not change.

Everything else is either a labelling inconsistency, a modelling choice worth revisiting, or an opportunity to make the demo more captivating.

## Status (third commit: rebuild as the Progressive Lens Comparator)

The page was rebuilt in the Visual Systems Lab style used by the Interactive Gaze Simulator: dark panel theme, topbar with Copy link / Save image / About, deep links in the hash, embed mode with height reporting, an About section with "The model, honestly", JSON-LD and social metadata. Everything from the earlier status list carries over. New in this version:

- B2 done: prescriptions are power vectors (M, J0, J45). Cylinder and axis mismatch contribute to blur.
- B3 done: viewing distance is chosen in cm (distance, TV, computer, reading, phone, custom). The need is the new distance prescription plus the dioptric demand, less the usable accommodation. "Add needed" is an output.
- B5 done: unwanted astigmatism is modelled explicitly, shown as its own map, and the local mean power drops by half of it. The old peripheral and Minkwitz weighting inputs are gone.
- B7 done: binocular mismatch uses signed residuals, computed on the full circle and displayed through the frame cut.
- D2 done: "What the patient sees" blurs a letter chart by the residual blur at the pupil, worse eye, with an approximate acuity.
- D4 done: a power profile strip beside each lens pair.
- D6 done: "Save image" exports a labelled 1200 x 630 PNG. "Copy link" shares the full state.
- D7: single-hue sequential amber ramp on the dark surface for magnitude maps and a blue / grey / red diverging ramp for the signed power map, following the data-visualization guidance (one hue for magnitude, two hues with a neutral midpoint for polarity). The pupil can be dragged on any map.
- D8: advanced inputs are gone or folded into the design panel; the six readout pills became one sentence per panel.
- Defaults: current -1.50 DS add +2.00, new -1.00 DS add +2.25, age 62 (no accommodation left on Hofstetter's minimum), 0.30 D depth of focus counted as clear.

Still open: B6 (brand naming is kept, with an "illustrative" note), the design parameters remain the author's estimates, and vertex distance, pantoscopic tilt and prism are not modelled.

## Status (second commit)

Applied in `index.html`:

- A1 to A11 all fixed: inset is nasal for both eyes, OS sample labels flip, Copy button renders once, Enter blurs instead of double rendering, strain thresholds and names match the legend, over-plus is reported as blur above 0.125 D, legend ticks are linear, sample labels no longer collide, reference lines are the fitting cross, half add and near reference point, dead code removed, phone layout no longer scrolls sideways, inputs use numeric keypads and labels are associated.
- Section C: each map is computed once at 360 x 360 into typed arrays and reused for drawing, metrics and binocular disparity. Rendering runs through requestAnimationFrame. Measured render time dropped from 650 ms to about 100 to 180 ms.
- B1: geometry is now in millimetres (60 mm lens, fitting cross 7.2 mm above centre, 2.5 mm inset, corridors 11 / 14 / 17 mm, 25 mm to the centre of rotation). The Minkwitz proxy is rescaled so a 14 mm corridor reproduces the original tuning.
- B4: sample points, strain zones and map reference lines are derived from the ramp (fitting cross, 50 percent add, 100 percent add).
- B7: binocular disparity uses signed residuals.
- B8: the No Strain card says "Turn on Allow accommodation" when accommodation is off.
- B9: defaults are the over-minus demo (current -1.50, new -1.00, add 2.00, age 55).
- D1 and A5: new Fit and Gaze Simulator. Fitting height and PD error shift the lens design relative to the eye for both lenses. Vertical and horizontal gaze move the pupil marker on every map, with the local power printed beside it. "Find clear view" animates the pupil to the nearest posture that clears the active target and the head silhouette tilts by the computed angle.
- D3: +0.25 / -0.25 step buttons on every sphere and add field.

Still open: B2 (power-vector cylinder), B3 (working-distance targets), B5 (true Minkwitz astigmatism map), B6 (brand naming), D2 (blurred letter preview), D4 to D8.

## A. Defects (behaviour differs from the stated design)

### A1. Nasal inset goes temporal (model bug)
`applyNasalShift` (line 1381) uses `dir = -1` for OD and shifts the penalty centre to `+0.08` for OD. The frame path and the sample-point labels define OD temporal as `+x`. Numerically, at `y = 0.50` with a -1.00 / add 2.00 OD lens, the nasal sample point reads +0.52 D while the temporal one reads +0.79 D, and the zero-penalty plateau spans `x` from -0.05 to +0.21. The near zone is therefore inset temporally for OD and, by symmetry, for OS. Fix: flip the sign in `applyNasalShift` (OD `+1`, OS `-1`) or, better, subtract the inset from `x` so the plateau is centred at the inset.

### A2. "Temp" and "Nas" sample labels do not flip for OS (display bug)
`SAMPLE_POINTS` (lines 1285 to 1286) hard-codes Temp at `+0.45` and Nas at `-0.45`. The frame path mirrors for OS (`tDir = -1`), so on the OS diagram the point labelled "Temp" is actually nasal. The OD and OS sample canvases render identically today, which hides A1 from the viewer. Fix: pass `eye` to the labeller and swap names when `eye === 'OS'`.

### A3. "Copy New Rx to Current" renders nine times (performance bug)
`setInputValue` (line 1893) dispatches a bubbling `change` event for each of eight fields, and every input already has a `change` listener bound to `render`. The click handler then calls `render()` once more. Measured: 36 heatmap draws, 9 renders, 5.7 s of frozen UI. Fix: set `.value` without dispatching events, then call `render()` once.

### A4. Enter key renders twice
The `keyup` Enter handler (line 2009) renders, then the `change` event fires on blur and renders again. Minor, but each render costs 650 ms on this hardware. Fix: drop the keyup handler or debounce `render`.

### A5. "OC fit" does not affect the optics
`ocFitDeltaY` (line 1210) only offsets `pupilYn`, which is consumed solely by `drawGuides`. The lens function `palLocalSE` never receives it. Switching OC fit leaves all three comparison metrics unchanged (verified: 2.9 / 0.8 / 30.9 percent before and after). `pupilYN` (line 1206) also ignores both of its arguments and always returns -0.10, so the "pupil y -0.10" readout is a constant. Either wire the offset into the lens model as a vertical shift of the progression relative to the pupil, or remove the control until it does something.

### A6. Strain thresholds disagree with their own legend
Code: `STRAIN_NONE_MAX = 0.19` (line 1020). Legend (line 729): "No strain, ratio < 0.10" and "Mild, 0.10 to 0.24". The comment inside `calculateMetrics` also says 0.10. The top band is named "Beyond reserve" in code and "Extreme" in the legend (line 733). Pick one set of numbers and names.

### A7. Over-plus up to 1.00 D is reported as "No strain" with no blur warning
`buildZoneStrainHTML` (line 1678) only says "No strain but blurry" when the lens is more than `OVERPLUS_BLUR = 1.00` D too plus. A zone that is +0.75 D over target reads simply "No strain", which a patient will hear as "fine". That zone is red on the heatmap right beside it. Suggest reporting any over-plus of 0.25 D or more as "Blurred by +x.xx D (no accommodative strain)".

### A8. Legend tick marks are not on the scale
The six tick labels (lines 938 and 989) are spread evenly with `justify-content: space-between`, but the bar is linear from 0 to 2.00 D. The 0.25 and 0.50 labels sit at 20 percent and 40 percent of the bar when they should be at 12.5 percent and 25 percent. Use evenly spaced values (0, 0.4, 0.8, 1.2, 1.6, 2.0) or absolutely position the labels.

### A9. Sample-point labels collide
The "Above FC" name (drawn 18 px below its point) and the FC value (drawn 24 px above its point) overlap, because the two points are 28 px apart on a 520 px canvas. The FC value is unreadable in the screenshot. Increase the vertical gap between those points or place labels to the side.

### A10. Reference lines on the map are decorative
`drawGuides` draws horizontal lines at 22 percent and 72 percent of canvas height (lines 1269 and 1272). Neither matches the fitting cross (`y = -0.24`, which is 38 percent), the 50 percent-add point, or the near reference point used by the model. Tie them to the model (FC, 50 percent add, 100 percent add) and label them.

### A11. Minor code hygiene
- `totalKey` (line 1788) is unused.
- `readRx` normalises axis but axis and cylinder are collapsed to spherical equivalent and axis is never used again. The Axis input is dead UI (see B2).
- `drawBinoDisparity` uses the OD frame mask for both eyes (line 1566), so the OS edge pixels are slightly wrong.
- `.strain .line` border uses a leftover purple `rgba(102,126,234,…)` (line 313) from a previous palette.
- Phone viewport scrolls horizontally by about 7 px because fixed 100 px inputs plus gaps overflow 390 px (line 137).
- `canvas:hover { transform: scale(1.02) }` (line 233) causes sticky zoom on iPad after a tap.
- Inputs are `type="text"` with no `inputmode`, so an iPad shows the full keyboard instead of a numeric pad chairside.
- Labels are not associated with inputs (`for` attribute), so tapping a label does not focus the field and screen readers get nothing.

## B. Modelling concerns (works as coded, but the physics is weaker than the UI implies)

### B1. The spatial coordinate system has no millimetres
All geometry is in a normalised unit circle. The add ramp reaches 49 percent at `y = 0.18` and 98 percent at `y = 0.50`, i.e. 0.74 units below the fitting cross. If the circle is a 70 mm blank, that is a 26 mm corridor; real corridors are 11 to 18 mm, fitting cross is 4 mm above the OC, near reference point is 14 to 18 mm below FC, inset is about 2.5 mm. Because nothing is in millimetres, the "short / medium / long corridor" choice cannot be checked against the frame B-measurement, and the chin-up demo cannot say "the patient must look 8 mm below the cross". Recommendation: define the lens in mm (blank diameter, FC height, corridor length, inset), derive the normalised coordinates from those, and let the frame cut be a frame A/B/DBL entry.

### B2. Cylinder is reduced to spherical equivalent
A 0.00 / -1.50 x 180 eye and a -0.75 DS eye are treated identically. For a tool meant to show patients why an Rx change matters, the cleanest upgrade is power vectors: compute the current-minus-new difference as (M, J0, J45) and use blur strength `sqrt(M² + J0² + J45²)` as the residual. That makes the Cyl and Axis fields meaningful with about 15 lines of code and correctly shows that an axis change alone produces blur.

### B3. Accommodation is double-counted against the near target
`targetSE` for "near" is `SE + add`. The add was prescribed on the assumption the patient uses roughly half their amplitude, and then `residualErrorMagnitude` lets them spend another reserve on top. Consider defining the targets by working distance (infinity, 60 cm, 40 cm) so that demand is `100 / cm`, then applying the lens power and the usable amplitude once. That also lets the tool show the add as an output ("this patient's add should be about 2.00") instead of an input.

### B4. The "Comp" sample point and the "Middle" strain zone sit in the near zone
Both are at `y = 0.35`, where the model delivers 89 percent of the add. The point the model treats as intermediate (50 percent add) is at `y = 0.18`. Move them, or better, derive the three zone positions from the ramp (0 / 50 / 100 percent add) so they track corridor length and emphasis.

### B5. The heatmap is a "worst of three proxies", not a dioptric quantity
`mag = max(residual, 2.4 × midline deviation, 2.2 × Minkwitz proxy)` for the generic designs. The midline deviation is the same penalty term already inside `residual`, re-weighted. The map is labelled "0.00 D to 2.00 D", which overstates its physical meaning. Honest options: label it "predicted blur index (D-equivalent)", or compute real Minkwitz astigmatism as the lateral derivative of the mean power (`dP/dx`, which the model can differentiate numerically) and show it as a separate astigmatism map like manufacturer plots.

### B6. Design presets are hand-tuned and unverifiable
Nikon SeeMax Ultimate and Hoya MyStyle 3 share near-identical numbers, and the generic hard/soft/average differ by a few percent. The five corridor-width measurements at `y = 0.50` are 0.82, 0.85, 0.88, 1.09, 1.10. That is fine as "premium vs generic", but naming brands invites patients to compare them literally. Consider renaming to tiers ("Standard", "Advanced", "Premium individualised") or adding a footnote that these are illustrative, not manufacturer data.

### B7. Binocular disparity compares unsigned magnitudes
`|magOD - magOS|` treats +0.50 D over-plus in one eye and 0.50 D under-minus in the other as zero disparity. Use the signed residual, and consider showing anisometropic *prism* difference at the near reference point (Prentice rule), which is the clinically felt consequence.

### B8. "No Strain Area" is computed even when accommodation is off
The strain panels say "Accommodation not applied" but the metric card still reports a percentage derived from the Hofstetter reserve. Hide or grey the card when the checkbox is off.

### B9. Defaults are an implausible patient
Current -1.00 with add 2.00, new plano with add 2.00. Start with the demo you run most, for example current -1.50 (over-minused) vs new -1.00, add 2.00, age 55, accommodation on. The first screen then tells the story without any typing.

## C. Performance

| Measurement (headless Chromium, this container) | Value |
|---|---|
| Initial load and render | 1.7 s |
| One `render()` | 650 ms |
| `render()` with binocular maps | 1.0 s |
| "Copy New Rx to Current" | 5.7 s (9 renders) |

Causes and fixes, in order of payoff:
- `calculateMetrics` re-walks all four 720 × 720 maps, recomputing exactly what the heatmap loop just computed. Compute each map once into a `Float32Array`, then both draw and count from it. Roughly halves render time.
- `peripheralMidlineDeviation` calls `fnLensPower(0, y)` per pixel; it is constant per row. Hoist it. Another 25 percent.
- Render at 360 px and let CSS upscale (the canvases are already displayed at `width: 100%`). Four times fewer pixels with no visible loss at chairside viewing distance.
- Debounce `render` by about 100 ms and run it in `requestAnimationFrame`, so typing "-2.25" does not render on every intermediate state.
- With those, a full render should be under 100 ms and the sliders proposed below can be live.

## D. Opportunities to make it novel, captivating, and more clinically useful

### D1. A "gaze" control that moves the pupil down the lens (highest value)
This is the feature your chin-up explanation needs. A single slider (or dragging the crosshair) for vertical gaze in mm below the fitting cross, with:
- a live readout of the local lens power under the pupil and the residual error for the chosen target,
- an auto-computed "to see clearly at distance, this patient must look N mm below the cross, about X degrees of chin lift" using about 25 mm from lens to centre of rotation (1 mm is about 2.3 degrees),
- an animated head silhouette that tilts by that angle.
For an over-minused patient, the pupil walks down into the green band and the silhouette's chin comes up. That is the moment the patient understands.

### D2. A "what the patient sees" simulated chart
Render a Snellen or text sample blurred by the residual dioptres at the pupil position (a Gaussian blur radius proportional to blur strength, scaled for a 3 mm pupil). Show it side by side for current vs new. Patients relate to a blurred letter instantly, far more than to a colour map.

### D3. Rx step buttons
"+0.25 / -0.25" buttons beside each sphere field. One tap, immediate re-render, and the near sample values shift with it. This is the demo for "more plus at distance helps near too".

### D4. Vertical power profile strip
Next to each heatmap, a thin plot of power along the midline (y on the vertical axis, dioptres on the horizontal) with the target as a vertical line. It explains the corridor in one glance, shows the 50 percent-add point, and makes the current-vs-new overlay trivial.

### D5. Real-world object overlay
Instead of only "Distance / Computer / Near" buttons, let the user pick a working distance in cm (or tap presets like phone 33 cm, monitor 65 cm, dashboard 80 cm, TV 3 m) and compute demand from it. Patients recognise their own tasks.

### D6. Shareable snapshot
A "Save image" button that composites the two heatmaps, the Rx table, and the metrics into one PNG for the chart or for the patient to take home. `canvas.toBlob` makes this a few lines.

### D7. Visual polish
- The dark teal gradient and glass cards are attractive. The heatmap palette, however, is the one thing patients look at, and green/yellow/red is not colour-blind safe (about 8 percent of male patients). A viridis-style or blue-to-orange diverging palette keeps the message for everyone.
- Give the frame a slight shadow and a 1 px highlight so it reads as a lens, and dim the area outside the frame rather than leaving it white, so the eye is drawn to the lens.
- Animate transitions between current and new (cross-fade the heatmaps over 300 ms) when a value changes. It makes the "what changed" visible without the viewer searching.
- The header subtitle "Advanced optical analysis and prescription comparison tool" is generic. Something like "See how a prescription change moves clear vision around a progressive lens" tells the patient what they are about to see.

### D8. Reduce clinician cognitive load
- Collapse "Map Appearance and Peripheral Modeling" into an "Advanced" drawer that is clearly for you, not the patient.
- Replace the six readout pills with one sentence summary ("New Rx gives 50.6 percent usable area at distance, up from 2.9 percent").
- Show the active target distance inside each heatmap title so screenshots are self-explanatory.

## E. Suggested fix order

1. A1, A2 (inset and labels), A3, A4 (render cascade), A6, A7, A8 (legend consistency). Small, mechanical, no design decisions.
2. Section C performance refactor. Enables everything interactive.
3. A5 and D1 together: wire vertical offset into the model and expose it as the gaze slider with the chin-lift readout.
4. B1 and B2: millimetre geometry and power-vector residuals. These are the two changes that make the model defensible to a colleague.
5. D2 to D8 as time allows.
