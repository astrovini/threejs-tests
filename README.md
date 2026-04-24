# Faro Logo — Three.js Scene

A single-file Three.js site that loads the Faro SVG logo, extrudes it into 3D, and exposes an interactive control panel for geometry, material, camera, and size parameters.

Serve the folder (SVGLoader needs HTTP):

```
python3 -m http.server 8000
# then open http://localhost:8000/
```

---

## Current state

Single file: `index.html`. Uses `three@0.160.0` via `<script type="importmap">` (no build step).

### Scene

- `MeshPhysicalMaterial` (clearcoat + emissive capable).
- `AmbientLight(0.6)` + one `DirectionalLight(1.2)` at `(300, 400, 500)`. No HDRI env map, no shadows.
- `PerspectiveCamera(45°, 0, 0, 800)`.
- `OrbitControls` — drag to orbit, scroll to zoom (clamped 200–1800), pan disabled, damped.
- `pivot` group wraps a `logoGroup` (with `scale.y = -1` to flip SVG's Y axis).
- Mouse-tilt layer: pointer position drives `pivot.rotation` with a lerp; can be toggled off (OrbitControls still works either way).

### Controls panel

Draggable + collapsible, persists to `localStorage` via preset slots. All numeric sliders are in **SVG viewBox units** (1158 × 1080) unless otherwise noted.

**Geometry (ExtrudeGeometry):**

| Control | Range | Step | Default | Meaning |
|---|---|---|---|---|
| `depth` | 0–400 | 1 | 120 | Z-extrusion length. |
| `bevelThickness` | 0–80 | 0.5 | 22 | Z-span of the rolled edge. |
| `bevelSize` | 0–40 | 0.5 | 16 | X-Y rollover — how far the bevel eats inward. |
| `bevelOffset` | -20–20 | 0.5 | 0 | Shifts the bevel profile in/out. Negative pulls inward. |
| `bevelSegments` | 1–24 | 1 | 16 | Smoothness of the bevel curve. 1 = chamfer, 16+ = smooth. |
| `curveSegments` | 4–64 | 1 | 48 | Subdivisions along the SVG's bezier curves. |
| `steps` | 1–10 | 1 | 1 | Z-axis subdivisions. Only matters for deformation. |
| `logoSize` | 100–1000 | 10 | 500 | Target logo height in world units. Scales the `pivot`. |

**Material (MeshPhysicalMaterial):**

| Control | Range | Default | Meaning |
|---|---|---|---|
| `metalness` | 0–1 | 0.2 | 0 = dielectric (plastic/wood), 1 = pure metal. |
| `roughness` | 0–1 | 0.4 | 0 = mirror, 1 = fully matte. |
| `clearcoat` | 0–1 | 0 | Strength of a glossy top layer (car-paint effect). |
| `clearcoatRoughness` | 0–1 | 0.2 | Roughness of the clearcoat layer. |
| `emissiveIntensity` | 0–2 | 1 | Brightness of the self-lit color. 0 = no glow. |
| `color` | picker | `#eaebdb` | Base material color. |
| `emissive` | picker | `#000000` | Self-lit glow color (pairs with intensity). |

**Interaction:**

- `Mouse tilt` checkbox — toggles the pointer-driven pivot rotation.

**Utilities:**

- **3 preset slots** — click a filled slot to load. Click **Save to…** to enter save mode, then pick a slot (confirm prompt on overwrite). `Escape` cancels. Presets persist to `localStorage` under key `faro-presets`.
- **Copy Settings** — copies the current `params` object as JSON to the clipboard.
- **Reset** — restores all defaults, re-homes the camera, closes save mode.
- Each control row has a `?` info icon with a hover tooltip.

---

## Q&A — 3D controls in Three.js

> **What other controls are there for 3D? Is this the right way to extrude a 3D object in Three.js? What is a bevel? What's the difference between bevel thickness and bevel size? What's the scale of the values?**

### Is this the right way to extrude 3D in Three.js?

Yes — `SVGLoader` + `ExtrudeGeometry` is the canonical Three.js approach for turning 2D vector paths into 3D geometry. Alternatives exist but aren't better for a logo:

- **`ShapeGeometry`** — flat, no depth. Would lose the 3D feel.
- **`TextGeometry`** — same family as `ExtrudeGeometry` but for typography (expects a typeface JSON).
- **GLTF/GLB import from Blender** — more control (e.g. custom bevel profiles, subsurface), but requires a modeling step outside the browser.
- **Signed distance fields / raymarched shaders** — fancier, but huge complexity jump.

For a logo from SVG, what you have is the idiomatic path.

### What is a "bevel"?

A bevel is the softened edge where two faces meet. Without a bevel, the rim of your extrusion is a sharp 90° corner (imagine a cookie-cutter shape). With a bevel, that corner is replaced by either a flat chamfer or a rounded curve, which catches light and makes the edge feel physical rather than CGI-flat.

### `bevelThickness` vs `bevelSize`

They describe two different directions of the same rolled edge:

- **`bevelThickness`** — how far the bevel extends along the extrusion (Z) axis. *"How much Z-depth does the rounded edge occupy."*
- **`bevelSize`** — how far the bevel eats inward in the X-Y plane (the face). *"How much does the edge roll over onto the front/back face."*
- **`bevelSegments`** — subdivisions along the curve of the bevel. 1 = flat chamfer, 16+ = smooth rounding.

How they relate:

- `thickness == size` → a clean quarter-circle rolloff (pillow-y).
- `thickness > size` → a flatter, chamfer-like edge.
- `thickness < size` → an overhanging, almost mushroom-like edge (usually weird).
- Total depth = `depth + 2 * bevelThickness` (bevels on both front and back).
- The X-Y footprint shrinks by `bevelSize` on every side of the shape before extrusion, then the bevel curves back out, so the outer silhouette ends up matching the original SVG.
- Geometry breaks when `bevelSize` exceeds roughly half the thinnest feature of the shape (the narrow parts self-intersect).

### Scale of the values

All three-space values are in the SVG's local coordinate units — the SVG viewBox is **1158 × 1080**, so the logo is ~1100 units tall before `logoSize` rescales the pivot. Integer counts (`bevelSegments`, `curveSegments`, `steps`) are unitless. PBR material values (`metalness`, `roughness`, `clearcoat*`, `emissiveIntensity`) are unitless 0–1 (except `emissiveIntensity` which goes 0–2).

### What's still not exposed

**Material-side** (all available on the current `MeshPhysicalMaterial`):

- `sheen` — velvety fuzz of cloth.
- `transmission` + `ior` + `thickness` — glass/translucent plastic.
- `iridescence` — soap-bubble / oil-slick color shift.
- `UVGenerator` — custom UV mapping for textures.

**Scene / rendering:**

- **Environment map (HDRI)** — the single biggest visual upgrade for metallic looks. PBR materials look plasticky until you give them something to reflect.
- **Extra lights** — rim light behind for silhouette, fill opposite the key, colored accent.
- **Shadows** — ground plane + `castShadow` / `receiveShadow`.
- **Post-processing** (`EffectComposer`): bloom (glow on bright pixels), SSAO (contact shadows in crevices), tone mapping, vignette, depth-of-field.

**Interaction / animation:**

- Tilt strength and damping sliders (currently hardcoded to `0.25` / `0.15` target × `0.08` lerp).
- Auto-rotate toggle.
- Camera FOV slider.
- Entrance animation (scale-in, stagger paths).

---

## `landing.html` — cinematic flip-reveal scene

Interactive landing page: Faro logo flips on click and reveals download (App Store / Google Play) and nav (Blog / Legal / Contact) cards. Single file, same `three@0.160.0` importmap pattern.

### Rendering gotchas (document so we don't re-solve them)

**1. `EffectComposer` silently disables `antialias: true`.**
The `WebGLRenderer({ antialias: true })` flag only applies to the default framebuffer. Once you route through `EffectComposer`, everything renders to an internal target with 0 samples. Fix: pass your own multisampled target.

```js
const dpr = renderer.getPixelRatio();
const msaaTarget = new THREE.WebGLRenderTarget(
  Math.floor(innerWidth * dpr),
  Math.floor(innerHeight * dpr),
  { samples: 4, type: THREE.HalfFloatType },
);
const composer = new EffectComposer(renderer, msaaTarget);
composer.setPixelRatio(dpr);
composer.setSize(innerWidth, innerHeight);
```

When you pass a custom target, `EffectComposer` uses `target.width/height` **as-is** — it does *not* multiply by pixel ratio like the default path does. Size the target in device pixels (× `dpr`), then call `setPixelRatio(dpr)` so subsequent resizes scale correctly. Otherwise retina renders at half resolution and looks pixelated.

**2. `camera.near` controls depth precision at distance.**
The perspective depth buffer has logarithmic precision — most of its resolution is packed near the near plane. With `near: 0.1` and the scene ~930 units from camera, 1 world unit resolved to ~5 discrete depth values, causing hash-line z-fighting through decals under MSAA. Push `near` as close to scene depth as you can without clipping — camera z=950, content at z≈0 → `near: 100` gives ~22,000× more precision in the range that matters.

**3. `ExtrudeGeometry` bevels extend *outside* `[0, depth]`.**
The front bevel tip sits at `depth + bevelThickness`, not at `depth`. When offsetting a decal forward of the card, clear the bevel tip, not the face. Rule of thumb: `decal.z = depth/2 + bevelThickness + comfortable_margin`. (Download cards use `+15` with bevelThickness=4; nav cards use `+10` with bevelThickness=2.)

**4. `transparent: true` moves materials to the sort-unstable transparent queue.**
The transparent queue is sorted back-to-front every frame by per-mesh camera distance. For a tilting group with base + decal, the sort can flip and the decal can render first — then the base paints over it. Mitigations:
- Copy the logo pattern (opaque material) wherever possible. Opaque queue is stable.
- `depthWrite: false` on a transparent decal is a footgun: if the sort flips, the decal writes no depth and the base overwrites it. Leave `depthWrite: true` (the default) unless you specifically need semi-transparent layering.
- Once sitting on an opaque base, a single transparent decal plane has nothing to sort-fight with.

**5. `UnrealBloomPass` threshold picks up anti-aliased edges.**
A cream-colored icon (≈0.7 post-tonemap) at threshold 0.32 will pop in/out of bloom as sub-pixel edge coverage changes with the slightest camera tilt. Raise `threshold` above icon luminance (we use 0.85 — below the gold emissive pulse peak, above the decal cream) or darken the decal. Tuning bloom values is *not* a fix for z-fighting or sort flicker — diagnose those separately first.

### Icon rendering

Badge icons are drawn via `Path2D` fed SVG path data into a `CanvasTexture` (uploaded once). The Play Store icon uses the official 4-path multi-color mark (red `#EA4335`, yellow `#FBBC04`, green `#34A853`, blue `#4285F4`) — paths mirrored from `faro-web-page/src/components/HeroSection.tsx`. Apple stays monochrome per Apple's brand guidelines for dark backgrounds.

### Interaction state machine

Four states:
- `IDLE` — logo centered, breathing emissive pulse.
- `FLIPPING` — logo flips 180° and flies to top-left nav position; cards and nav cards fade + slide in staggered.
- `REVEALED` — steady state; cards clickable.
- `RETURNING` — reverse of flipping.

The logo is two stacked groups (`logoFull` for the wordmark, `logoIcon` for the icon-only mark) that swap `visible` at 90° into the flip (`cos(flip) < 0`) — hiding the swap behind the edge-on silhouette.
