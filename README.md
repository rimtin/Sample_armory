# Spoolara Armory — prototype

A browser-based "virtual armory" where a cosplay / 3D-printing enthusiast builds
an avatar from their **real body measurements**, imports character/armor **`.stl`
parts**, and the app **auto-fits** each part to the right region of the body so it
prints to fit a real person. Fitted parts re-export as print-ready STLs in mm.

This is a working prototype of the core experience in the brief, built to drop
into a larger application.

---

## Run it

```bash
npm install
npm run dev      # opens http://localhost:5173
```

Then:

1. The avatar stands on the lit platform. Enter measurements on the left — the
   avatar rescales by height and parts re-anchor live.
2. **Import STL** (button) or **drag `.stl` files onto the window**. Samples are
   in the provided `STL Files/` archive.
3. A part auto-detects its body region and fits. Adjust **Tight / Snug / Loose**,
   pick colour / style, or open the gizmo (**Move / Rotate / Scale**, or keys
   **G / R / S**) and the **nudge / rotate** number fields for fine control.
4. Click a part in the scene or the list to select it. Click the avatar to drop a
   **coordinate pick** marker and "Move selected here".
5. **Export this part (mm)** for one piece, or **Export assembly** for all parts
   posed together. Output is binary STL in millimetres, re-centred at the origin.

`npm run build` produces a static `dist/` you can host anywhere.

---

## Core requirements — status

**1. Interactive 3D armory** — React Three Fiber scene with orbit / pan / zoom, a
lit platform with a glowing ring, back wall and emissive gear niches, soft
shadows, ACES tone-mapping, and the reference image as backdrop. The platform
tracks the avatar's feet so the figure always stands on it.

**2. Avatar from measurements** — loads the provided rigged `humanoid.glb`,
scales it so its **true rendered height** matches the height input, and reports
its real bounding box + bone positions to the fitter. 12 measurements drive part
sizing; height drives the body.

**3. Import + fit + export STL**
- Import via button or drag-and-drop, multi-file; rendered with selectable colour
  and style (matte / metallic / glossy / emissive — visual only).
- **Auto region detection** from filename **and folder path** — 13/13 correct on
  the sample set (helmet, torso, back, belt, both arms with L/R sides, legs),
  overridable via dropdown.
- **Auto-fit**: hybrid sizing — limbs/torso size by girth (circumference →
  diameter), helmet/boot by length — to a target derived from the user's
  measurement × a Tight/Snug/Loose clearance, then anchored to the body region.
- **Manual control**: transform gizmo, size %, XYZ nudge (cm), XYZ rotate (°),
  flips, plus a click-to-place coordinate picker.
- **Live fitted-size read-out** in cm so the user can confirm the fit.
- **Export** bakes the fit into the geometry, converts to millimetres, re-centres
  at origin, and downloads — per-part or as a combined assembly.

---

## The interesting problem 1 — fitting STLs with no reliable scale

STL files carry no units and no consistent origin. The samples confirm it: the
helmet's bounding box sits ~9,650 units off-origin, and parts span from a few
hundred to tens of thousands of units. So raw STL size/position tell us nothing.

Approach (`src/utils/fitting.js`):

1. **Re-centre** each STL on its bounding-box centre at import.
2. Compute the region's **target real-world size (cm)** from measurements;
   circumferences become a diameter (`circ / π`) since most armor wraps the body.
3. Apply the **fit allowance** (Tight 2% / Snug 6% / Loose 12% clearance).
4. **Uniformly scale** so the part's dominant fit dimension hits that target —
   uniform only, so parts never distort.
5. **Anchor** to the body region.

Verified end-to-end: a 57 cm head auto-fits to a 16.8 cm-wide Snug helmet that
seats on the avatar's head, and exports as a 168 × 212 × 192 mm STL — matching the
on-screen read-out.

## The interesting problem 2 — anchoring on a skinned avatar

The provided avatar is a **SkinnedMesh**, and its skeleton's bind pose sits in a
different space than the deformed mesh the GPU actually draws — so neither the
rest-pose bounding box nor raw bone world-positions line up with what you see.

The fix (`src/components/Avatar.jsx`): compute the avatar's **true rendered
bounds** by running each vertex through `applyBoneTransform` (the same skinning
the GPU does) and unioning the results. Parts then anchor to that real box via
anatomical ratios (head ≈ 0.95 of height, chest ≈ 0.74, etc.), so they seat on
the visible body. Bone vectors are still used for limb orientation.

---

## Project structure

```
src/
  components/
    Armory.jsx            Lit platform, glowing ring, walls, niches
    Avatar.jsx            Rigged avatar; true skinned-bounds + bone reporting
    SceneParts.jsx        Part mesh, transform gizmo, coordinate pick marker
    MeasurementsPanel.jsx Live measurement inputs
    PartEditor.jsx        Region / fit / colour / style / manual / export
  utils/
    bodyModel.js          Measurement schema, regions, fit presets, region detect
    fitting.js            Hybrid sizing + anatomical anchoring
    exportStl.js          Bake transform → print-ready binary STL (mm)
  App.jsx                 State, drag-drop, scene framing, wiring
public/
  models/humanoid.glb     Provided rigged avatar
  backgrounds/armory-bg.png Provided reference image (backdrop)
```

Fitting/detection/export are pure functions; the scene is plain R3F components —
both lift cleanly into a larger app.

---

## Decisions, assumptions, trade-offs

- **Height scales the avatar; other measurements drive part sizing**, not body
  reshaping. Believable non-uniform reshaping of one static skinned mesh isn't
  reliable; a production version would use a parametric body (SMPL-style) so the
  mesh reflects every measurement. Main known limitation.
- **Anatomical-ratio anchoring** off the true skinned box (above) is robust across
  poses and heights; per-part offset is always available via the gizmo/nudge.
- **Filename + folder region detection**, always overridable.
- **Uniform scale only** for auto-fit, to preserve printability and proportions.
- Scoped out for time: collision/penetration checks against the body, automatic
  L/R mirroring of a single imported part, and saved loadouts.

---

## How AI was used

AI was a pair-programmer throughout: scaffolding the R3F/Vite app, drafting CSS,
and working through the fitting math. Where it helped: boilerplate and the
STL/export plumbing. Where it needed correction: the **skinned-mesh anchoring** —
the naive approaches (trust the bounding box, or trust bone positions) both fail
on this model because its bind pose and rendered mesh occupy different spaces;
getting parts to seat correctly required computing true skinned bounds via
`applyBoneTransform`, verified by importing samples and checking placement on
screen.

---

## Tech

React + Vite, React Three Fiber + drei, Three.js (GLTFLoader, STLLoader,
STLExporter, TransformControls, OrbitControls). No backend — fully client-side.
