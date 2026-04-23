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
