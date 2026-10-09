# Progressive Lens Comparator

A free, browser-based teaching model of a progressive addition lens. Enter the glasses a patient is wearing and the prescription just measured, choose what they are looking at, and both lenses are mapped side by side: where each is clear, where it blurs, what the eye has to do to compensate, and what the wearer would see.

Created by **Dr Robert Burke**, optometrist, Calgary Vision Centre. Part of the [Visual Systems Lab](https://calgaryvisioncentre.com/lab).

## What it does

- **Two prescriptions, both eyes**, with sphere, cylinder, axis and add. Cylinder and axis are modelled with power vectors, so an axis change alone produces blur. Step buttons change any value by 0.25 D.
- **Viewing distance:** distance, TV, computer, reading, phone, or any distance in cm. The lens needs the distance prescription plus the demand of that distance, less the accommodation the patient can comfortably supply (Hofstetter amplitude, selectable usable fraction). "Add needed" is shown as an output.
- **Three lens maps** for each lens and eye: blur strength for the chosen distance, the design's unwanted astigmatism, and mean power relative to what the eye needs. A profile strip plots power along the corridor against the need.
- **Fit and gaze simulator:** fitting height and centration sliders move the lens on the face; gaze sliders (or dragging on any map) move the pupil through the lens. "Find clear view" walks the pupil to the nearest posture that clears the chosen distance and tilts a head silhouette by the chin lift that takes.
- **What the patient sees:** a letter chart blurred by the residual error at the pupil, worse eye, with an approximate Snellen acuity.
- **Whole-lens comparison:** clear, usable and no-strain shares of the lens area, current vs new.
- **Lens designs:** generic hard, average and soft families, and two illustrative premium families with selectable corridor length and emphasis.
- **Shareable state:** the address bar encodes every setting. "Copy link" and "Save image" (a labelled 1200×630 PNG) are in the header.

## The model, honestly

This is a teaching model, not ray tracing of a real lens file.

- Geometry is in millimetres: a 60 mm lens, fitting cross 7.2 mm above the geometric centre, 2.5 mm near inset, corridors of 11, 14 or 17 mm from the fitting cross to the full add, 25 mm from the lens to the eye's centre of rotation (1 mm of gaze drop is about 2.3° of chin lift).
- The add rises along the corridor as a smooth logistic curve whose midpoint shifts with the design emphasis.
- Unwanted astigmatism grows with distance from the corridor centre (which follows the inset), is strongest beside the corridor and in the near periphery, and scales with the add and the design's hardness. The local mean power drops by half of it.
- Blur strength is the power-vector length of the residual error after accommodation has covered what it can: spherical shortfall or excess, the prescription's cylinder mismatch, and the lens' unwanted cylinder (root-sum-square). Up to 0.30 D counts as clear (depth of focus).
- Accommodation is only used for a shortfall of plus, never for an excess.
- The named designs are illustrative, not measured manufacturer data.

Not modelled: vertex distance and pantoscopic tilt, prism and Prentice-rule effects, lens thickness, chromatic effects, and the real asymmetry of manufacturer designs. The acuity estimate is an empirical conversion from blur strength and pupil size.

## Running it locally

Plain HTML, CSS and JavaScript in one file, no build step and no third-party requests. Open `index.html` directly, or serve the folder:

```
python3 -m http.server
```

## Embedding

The page supports `?embed=1&parent=<host page URL>`. In embed mode it hides the header subtitle, the About section and the footer, reports its height to the host with `postMessage({ type: 'pal-height', height })`, and mirrors its `#hash` to the host with `{ type: 'pal-hash', hash }` so shared links point at the host page.

```html
<div id="pal-box"></div>
<script>
  (function () {
    var APP = "https://drburke-droid.github.io/progressives/";
    var box = document.getElementById("pal-box");
    var iframe = document.createElement("iframe");
    iframe.src = APP + "?embed=1&parent=" + encodeURIComponent(location.origin + location.pathname) + location.hash;
    iframe.title = "Progressive Lens Comparator by Dr Robert Burke";
    iframe.scrolling = "no";
    iframe.style.cssText = "display:block;width:100%;height:1400px;border:0;border-radius:12px;overflow:hidden;";
    iframe.setAttribute("allow", "clipboard-write; web-share");
    box.appendChild(iframe);
    window.addEventListener("message", function (e) {
      if (!e.data) return;
      if (e.data.type === "pal-height") iframe.style.height = e.data.height + "px";
      if (e.data.type === "pal-hash") history.replaceState(null, "", e.data.hash);
    });
  })();
</script>
```

## Files

| File | Purpose |
|---|---|
| `index.html` | The whole application: page, styles, model, drawing, deep links, image export |
| `AUDIT.md` | Audit of the original version, what was fixed and what remains |

## Licence and disclaimer

© Dr Robert Burke. All rights reserved unless a licence file says otherwise.

Educational use only. This comparator illustrates how progressive lenses work. It does not examine anyone's eyes, does not replace a refraction or a dispensing measurement, and is not a substitute for an eye exam.
