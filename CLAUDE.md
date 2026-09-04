# shade-finder

A skin tone measurement tool for **tonalist** (working name), a research-phase cosmetics brand. Single HTML file, no build step, no dependencies, no backend.

## What it is for

The brand's thesis is that complexion products fail olive and South Asian skin tones because of crude *undertone* handling, not insufficient shade depth. The mission is undertone accuracy, not shade count.

This tool exists to collect real undertone readings from real people before any product exists. It is a research instrument and a data collection mechanism, not a shade quiz. Every design decision should follow from that: show the numbers, report uncertainty honestly, and never return a confident answer the measurement doesn't support.

It may also become a portfolio case study, so the diagnostics and the reasoning behind the method should stay visible and legible rather than being hidden.

## How it is deployed

- Hosted on GitHub Pages from `main`, root folder, served as `index.html`.
- **Commit directly to `main`.** Pages only rebuilds from `main`, so a pull request leaves the live site unchanged.
- The camera requires a secure context. It works on the Pages URL. It does not work from `file://`, from a plain-http local server, or inside any in-app browser (Instagram, WhatsApp, etc). Bugs of the form "camera won't start" are almost always this, not the code.

## Hard constraints — do not break these

1. **One file.** Everything stays in `index.html`. No bundler, no npm, no framework, no build step. It has to be droppable onto any static host.
2. **No network calls.** No analytics, no CDN scripts beyond the Google Fonts link, no uploading images anywhere. Face images never leave the device. This is a privacy promise made in the UI copy and it must stay true.
3. **No image is ever stored or transmitted.** Only averaged RGB values per zone. Sampling happens on an offscreen canvas and the frame is discarded.
4. **Flash timing stays slow.** Each flash holds for ~700–1000ms. Never speed the sequence up into strobe territory (nothing above ~2Hz). This is a photosensitivity safety limit, not a preference.
5. **Calibration lives in the URL hash**, not localStorage, so it survives a reload and can be shared as a link. Keep it that way.
6. **Readings are session-only and exported as CSV.** There is deliberately no backend yet.
7. **Bump the build stamp in the `.subline` on every edit** (`build YYYY-MM-DD.N · research`, where N increments within a day). There's no build step or cache-busting, so this is the only visible signal on the deployed page that a given push actually landed — don't skip it.

## The measurement pipeline

In order, as implemented in `takeReading()` and `compute()`:

1. **Camera runs on its own auto pipeline — deliberately not locked.** An earlier version tried `applyConstraints` with `whiteBalanceMode: manual` / `exposureMode: manual` to stop auto white balance from silently correcting the exact thing being measured. On phone cameras this destabilised the feed instead: the picture would darken dramatically mid-session, recovering only when the app was backgrounded and foregrounded (the browser tearing down and re-acquiring the camera, dropping the forced override). It was also the likely source of visibly wrong colour output downstream. So the lock attempt was removed; the camera's actual mode (`exposureMode`/`whiteBalanceMode` from `getSettings()`) is reported honestly in diagnostics instead of forced. A 1px canvas blur is the only frame processing applied, purely to knock down sensor noise before averaging — no sharpening, no tone/colour mapping.
2. **Eight full-screen flashes**, sampled from the live video: `ambient` (black), white, grey, red, green, blue, `warm` (warm-tinted near-white, `#FFE8B8`), skin reference. `warm` was originally `#FFC832`, a much dimmer, more saturated amber — once it became the actual tone-reading flash (see below), that dimness (only ~78% max luma, and just 50/255 on blue) was starving the reading of light on top of the existing "screen is weak at a distance" problem, so it was brightened toward full luminance while staying visibly warm-tinted. **Two disjoint roles, and this distinction is the core design principle of the whole pipeline: black/white/grey/red/green/blue are calibration-only — known colours whose entire job is characterising how this specific camera+screen pair distorts a known colour into whatever the sensor reports. `warm` is the one and only flash the actual skin-tone reading is taken under, run through the correction the calibration flashes derive. Calibration flashes are never read for tone; `warm` is never used to derive the correction.** (`skinref`, `#C68642`, is still sampled and shown in diagnostics as an extra data point, but used for neither — see the note under calibration gains.)

   This split exists because two earlier attempts collapsed the two roles into one flash (reading tone straight off `white`, uncorrected or under-corrected) and both came out with a persistent blue-grey cast that no amount of per-flash gain tuning fixed — because the miscalibrated flash was also the one being measured, there was nothing independent to check it against.
3. **Three zones** are averaged per flash: highlight (upper cheek), midtone (jaw into neck), shadow (outer edge). Positioned for a face turned ~45° right with the neck visible, and — since there's no face detection — for a face filling most of the frame close-up, which the "get close" instruction now requires (the screen is too weak a light source at arm's length). The zone coordinates in `ZONES` were re-tuned once against real close-up screenshots after they'd been landing on the eyes instead of cheek/jaw at that framing; still worth re-checking against a real reading's screenshot. The canvas is mirrored so displayed and sampled coordinates match.
4. **Ambient subtraction in linear light.** All values are converted sRGB → linear before the black frame is subtracted. This isolates the screen's contribution from room light and is the core of why this might work at all. Subtracting in gamma space would be wrong.
5. **Calibration gains**, `calibrationGains()`, fit ONE per-channel correction using all five calibration flashes (white, grey, red, green, blue) at once: for each channel, a least-squares scale fit through the origin across the (observed, target) pairs from all five — `gain[c] = Σ(target·observed) / Σ(observed²)` — normalised to a geometric mean of 1 and clamped to [0.5, 2.0]. This replaced two earlier, narrower attempts: one used only the pure R/G/B flashes (a single noisy data point per channel), the other added a second pass off the grey flash alone; neither incorporated all the available reference colours at once, and both left a systematic blue-grey cast in the readings. This is still a diagonal ("von Kries") correction, the same family real camera white-balance algorithms use — it can correct per-channel scale, not full 3×3 cross-channel colour rotation, which was judged not worth the complexity given the shade library is already placeholder data.
6. **Operator calibration** — exposure `k`, hue shift, saturation and lightness scaling — applied in HSL, then converted to LAB.
7. **Matching** uses CIEDE2000 against a 10-shade library, anchored on the **midtone** zone. Jaw-into-neck is the standard foundation reference: away from blush, away from shadow.

## Known limitations — do not paper over these

- **The reading is relative, not absolute.** Without a physical grey card there is no true reference, so exposure is operator-anchored via the admin sliders against a shade verified on real skin. Do not add copy implying lab accuracy.
- **Ambient correction is unproven.** It is implemented but has never been validated across lighting conditions. If it fails, the honest conclusion is that this needs a physical reference card, not more code.
- **Auto white balance and auto exposure run unlocked on every device, by design now.** The channel-gain and ambient-subtraction math is what corrects for this in software; there is no hardware-level lock backing it up. Readings taken under inconsistent or changing lighting mid-sequence are the ones most likely to be wrong.
- **No face detection.** Zones are fixed fractions with an alignment guide. This was chosen over MediaPipe for reliability and file size. If misalignment turns out to be the dominant error source, that trade is worth revisiting.
- **The shade library is invented placeholder data**, not measured pigment values.

## Testing protocol

The instrument's validity is judged on **repeatability**, not on whether the answer looks right — the eye will happily rationalise a wrong result.

1. Three readings in a row, phone braced, no movement between them. Hex values should land within a few points of each other.
2. Repeat in a different room. Large divergence means ambient correction is not holding.
3. Check the diagnostics block for warnings before trusting any reading.

Admin panel: five taps on the wordmark. It holds the calibration sliders, the before/after comparison, the session readings table, CSV export, and the diagnostics dump.

## Reporting a bug to a cloud session

Cloud sessions run headless with no camera. They cannot run the tool or see what the user sees. Any bug report must include the full diagnostics block from the admin panel plus a description of what was observed, or the fix will be guesswork.

## Design rules

The interface is a measuring instrument, not a beauty product.

- **Dark by function**: the screen is the light source, so ambient contribution must stay low between flashes.
- **The only saturated colour in the interface is measured data.** There is no fixed accent colour. Do not introduce one.
- One typeface (Space Grotesk), hierarchy carried by size and weight.
- Tokens live in `:root`. Use them; don't hardcode hex values in components.
- Numbers are shown, not hidden — ΔE, Lab*, raw RGB. The brand's claim is accuracy, so the evidence stays on screen.
- Motion is limited to the flash sequence itself. Nothing else animates.
- `#miniPreview` is shown during the flash sequence only, above the flash overlay, so the user isn't completely blind to their framing for the ~6s the flash covers everything. It doesn't show the whole face — it shows four live cropped, zoomed, mirrored views: one oval crop matching the alignment guide's bounding box, and three small thumbnails cropped to the exact highlight/midtone/shadow sample rectangles. `renderPreviewFrame()` draws these every animation frame from a dedicated low-res mirrored copy of the source (`previewWork`/`pctx`) — deliberately not the `work`/`wctx` canvas `sampleFrame()` uses, since that's being actively written to mid-sequence and sharing it with a concurrent render loop would corrupt whichever ran mid-draw. Kept small on purpose: together the four crops are a small fraction of screen area, so they barely dent the flash colour actually reaching the face.
- Empty and failure states give direction, not mood. The three camera failure messages each point at a different cause; keep them distinct.
