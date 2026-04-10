# Building 3D Objects for the Web with Three.js

Everything I learned building 8 procedural models, a PS1-style earth, and a unified rendering pipeline for my homepage — including the mistakes.

## Why procedural

My homepage has 8 project cards. Each one has a rotating 3D object — a manila folder, a VCR, a traffic cone, a camera, a microphone, a PDA, a desk calendar, and a jury-rigged streaming toolkit hub. None of them are imported `.glb` or `.gltf` files. They're all built from Three.js primitives — `BoxGeometry`, `CylinderGeometry`, `SphereGeometry`, shaped and textured in code.

I didn't plan it this way. I started with the folder model, built it out of boxes because I wanted a specific worn-out look I couldn't get from a downloaded asset, and then just kept going. By the time I had 8 models I'd developed a system for it. The approach has real advantages: zero asset loading, total control over materials, and procedural textures that can take an accent color parameter and tint the whole model to match the card it sits in.

The downside is that each model is 200-400 lines of geometry math. The hub model is over 600.

## The model template

After building a few models I settled on a pattern. Every builder is a function that takes an accent color and returns a `THREE.Group`:

```typescript
export function buildVCRPlayer(accent: THREE.Color): THREE.Object3D {
  const g = new THREE.Group();

  // 1. Base body
  const body = new THREE.Mesh(
    new THREE.BoxGeometry(1.1, 0.18, 0.65),
    psTex(makeBodyTexture(), { roughness: 0.92 }),
  );
  g.add(body);

  // 2. Sub-components (display, tape slot, buttons)
  // 3. Detail geometry (screws, labels, vents)
  // 4. Wireframe overlay for edge definition

  return g;
}
```

Every model follows this structure: base shape first, sub-components on top, detail last. The accent color gets passed to wireframe overlays and emissive elements (LEDs, light housings) so each model picks up the color of its project card.

## Material system

Two factory functions handle 90% of materials:

```typescript
// Solid color, PS1 style
function psCol(color: number, opts = {}): THREE.MeshStandardMaterial {
  const m = new THREE.MeshStandardMaterial({
    color,
    roughness: 1,
    metalness: 0,
    flatShading: true,
    side: THREE.DoubleSide,
    ...opts,
  });
  applyPS1Jitter(m);
  return m;
}

// Canvas texture, PS1 style
function psTex(texture: THREE.Texture, opts = {}): THREE.MeshStandardMaterial {
  const m = new THREE.MeshStandardMaterial({
    map: texture,
    roughness: 1,
    metalness: 0,
    flatShading: true,
    side: THREE.DoubleSide,
    ...opts,
  });
  applyPS1Jitter(m);
  return m;
}
```

`flatShading: true` is the backbone of the look. It makes every triangle face render as a flat plane instead of interpolating normals across it, which gives everything that faceted low-poly feel. Paired with `roughness: 1` (fully matte), it reads as PS1-era hardware rendering even before any shader effects.

Materials break into a few semantic tiers:

| Roughness | Metalness | Use case |
|-----------|-----------|----------|
| 0.85-1.0 | 0.0 | Plastic, cardboard, matte paint |
| 0.7-0.9 | 0.05-0.2 | Weathered metal, aged surfaces |
| 0.3-0.6 | 0.4-0.6 | Brushed aluminum, chrome trim |
| 0.0-0.2 | 0.7-0.9 | Polished steel, mirror finish |

The microphone is the most material-diverse model — the chrome grille head uses `metalness: 0.8, roughness: 0.22` while the neon "ON AIR" sign uses `emissive` with additive blending. Most other models stay in the matte range.

## Procedural textures via canvas

Every label, screen, sticker, and surface detail is painted onto a `<canvas>` element that becomes a Three.js texture. No image files.

The base pattern is a noise generator that creates weathered surfaces:

```typescript
function makeNoisyTexture(w, h, baseR, baseG, baseB, opts) {
  const ctx = canvas.getContext("2d");
  const img = ctx.createImageData(w, h);

  for (let i = 0; i < img.data.length; i += 4) {
    const px = (i / 4) % w;
    const py = Math.floor(i / 4 / w);

    // Per-pixel noise (±12 per channel)
    const n = (Math.random() - 0.5) * opts.noise;

    // Edge ambient occlusion
    const ex = Math.min(px, w - px) / (w * 0.15);
    const ey = Math.min(py, h - py) / (h * 0.15);
    const ao = 0.55 + 0.45 * Math.min(1, Math.min(ex, ey));

    img.data[i]     = (baseR + n) * ao;
    img.data[i + 1] = (baseG + n) * ao;
    img.data[i + 2] = (baseB + n) * ao;
    img.data[i + 3] = 255;
  }

  ctx.putImageData(img, 0, 0);
  // Then paint details on top: labels, stains, scratches
}
```

The AO calculation is dead simple — pixels near the edge get darker. It's not physically accurate but it sells the illusion of depth on flat surfaces without any shadow mapping. I use this for every body panel, every cassette shell, every box face.

On top of the noise base, each texture paints its own details with standard Canvas 2D:
- **Text** gets a shadow pass (offset 1-2px, low alpha), then the main text, then sometimes a lighter highlight pass
- **Rust and stains** are radial gradients with random placement
- **Scratches** are thin lines with low opacity
- **Coffee rings** are double-ring strokes at different alphas
- **Fabric weave** (duct tape) is horizontal lines with random vertical jitter

The folder model's inside cover has hand-drawn doodles — a Sankey diagram, a radar chart, a stick figure at a desk, networking nodes — all painted with Canvas 2D path operations. It's tedious to write but the result looks like someone actually drew in a notebook.

One important detail: `generateMipmaps: false` on every texture. These textures are rendered at their native resolution and never minified, so mipmaps just waste memory. I use `LinearFilter` for most textures and `NearestFilter` for a few where I want crisper pixel edges (the folder front cover, some labels).

## The PS1 shader: I really liked that effect, until I didn't

The original vision was full PS1. Vertex jitter, nearest-neighbor filtering, the works. The shader replaces Three.js's standard vertex projection with screen-space quantization:

```glsl
vec4 mvPosition = modelViewMatrix * vec4(transformed, 1.0);
gl_Position = projectionMatrix * mvPosition;

float prec = basePrecision * (1.0 + dist * 0.12);
gl_Position.xy /= gl_Position.w;
gl_Position.xy = floor(gl_Position.xy * prec) / prec;
gl_Position.xy *= gl_Position.w;
```

The `floor()` snaps vertex screen positions to a grid. Lower precision = coarser grid = more wobble. The distance term (`1.0 + dist * 0.12`) increases precision for far-away objects so they don't jitter into oblivion.

I also had a geometry-level distortion called `jankify` that randomly displaces vertices:

```typescript
function jankify(geo: THREE.BufferGeometry, amount: number) {
  const pos = geo.attributes.position;
  const effective = amount * JANKIFY_SCALE;
  for (let i = 0; i < pos.count; i++) {
    pos.setX(i, pos.getX(i) + (Math.random() - 0.5) * effective);
    pos.setY(i, pos.getY(i) + (Math.random() - 0.5) * effective);
    pos.setZ(i, pos.getZ(i) + (Math.random() - 0.5) * effective);
  }
  geo.computeVertexNormals();
}
```

Between the shader jitter and `jankify`, the models had this shimmering, hand-built quality. Edges wobbled. Surfaces weren't quite flat. It looked like a PS1 game and I loved it.

Then I looked at it on mobile and it was unreadable. Text on textures was a smeared mess because `NearestFilter` at non-native resolutions just destroys legibility. The jitter that looked charming at 1440p was nauseating on a phone. I cranked the precision down from 140 to 90 trying to find the sweet spot, which made it *more* aggressive and worse.

So I killed it. Set `applyPS1Jitter` to a no-op, `JANKIFY_SCALE` to 0, swapped `NearestFilter` to `LinearFilter`, turned on antialiasing. Went from PS1 to clean modern rendering in one commit.

And then the page looked boring. Flat shading alone wasn't carrying the aesthetic. The models looked like untextured prototypes.

The fix was bringing it back at 25% intensity. `JITTER_PRECISION` went from 90 to 220 (higher = less wobble). `JANKIFY_SCALE` went from 0.4 to 0.1. The distance attenuation stayed. The result is subtle — you feel it more than you see it. Edges have a slight shimmer. Surfaces aren't perfectly smooth. It reads as "retro-inspired" instead of "PS1 emulator."

The evolution:

```
Mar 19  precision 140, raw jankify, NearestFilter    → full PS1
Mar 20  precision 90, NearestFilter                  → even more PS1
Mar 29  precision 90 + distance attenuation          → PS1 but less broken at distance
Apr 9   no-op shader, jankify 0, LinearFilter, AA    → killed it
Apr 10  precision 220, jankify 0.1, LinearFilter, AA → 25% PS1, favoring PS2/GBA style
```

The lesson: the PS1 look comes from hardware limitations that don't translate well to modern screens and DPI. What works is borrowing the *feel* — flat shading, low polygon counts, slightly imperfect geometry — without the actual rendering artifacts. PS2 and GBA got this right: they kept the low-poly aesthetic but added bilinear filtering and antialiasing. That's where I landed.

## The orientation problem

This one cost me hours. Models built for the carousel ring would face the wrong direction in the bento grid, and fixing it for one context would break the other.

The root cause: when you place a model on a carousel ring, each wrapper rotates to face outward:

```typescript
const angle = i * ANGLE_STEP;
wrapper.position.set(
  Math.sin(angle) * RING_RADIUS,
  0,
  Math.cos(angle) * RING_RADIUS,
);
wrapper.rotation.y = -angle; // face outward from center
```

So a model whose "front" is built on the +Z axis will naturally face the viewer as the ring spins. But some models had their front on -Z (they were built "backwards"), so they faced *inward* toward the ring center. The microphone and PDA both had this problem. The fix was adding `Math.PI` to their Y rotation to spin them 180°.

Then the desk calendar broke. Its front *was* on +Z, and the carousel wrapper already handled the outward facing, so adding `Math.PI` double-rotated it. The fix was removing the `Math.PI` I'd just added — one model's fix was another model's bug.

Then I built the bento grid view. Models from the carousel got dropped into a head-on card layout, and their carousel-tuned rotations were meaningless. The camera model's lens pointed off to the side. The calendar showed its back. The PDA was edge-on.

The solution was a per-model `initialRotY` override in the grid config that zeroes out the builder's rotation and applies a bento-specific facing:

```typescript
const MODEL_SPECS = {
  0: { initialRotY: 0.2 },          // folder: slight angle
  3: { initialRotY: Math.PI + 0.25 }, // camera: flip 180° + offset (lens toward viewer)
  6: { initialRotY: Math.PI + 0.1 },  // calendar: flip 180° (front page toward viewer)
  // ...
};

// Applied on registration:
model.rotation.set(0, spec.initialRotY, 0);
```

The models that needed `Math.PI` in the grid were the ones that *didn't* need it in the carousel, and vice versa. There's no universal "front" when the same model lives in two contexts with different camera relationships.

If I were starting over, I'd establish a convention: every model's front face is +Z, period. Then the carousel wrapper and grid wrapper each handle their own rotation independently, and the builder doesn't need to know which context it's in.

## Lighting

Three lights, that's it:

```typescript
scene.add(new THREE.AmbientLight(0x2a2f38, 0.5));

const keyLight = new THREE.DirectionalLight(0xffeedd, 1.6);
keyLight.position.set(5, 3, 5);
scene.add(keyLight);

const fillLight = new THREE.DirectionalLight(0x6a7a9a, 0.45);
fillLight.position.set(-3, 2, -2);
scene.add(fillLight);
```

The ambient is a cool blue-gray at half intensity — just enough to keep shadow sides from going pure black. The key light is warm white (0xffeedd) from the upper-right, strong enough to define the form. The fill is a dim blue from the opposite side that softens the contrast.

This is standard three-point lighting minus the backlight, and it works for everything from the chrome microphone to the cardboard folder. I tried more complex setups — per-model point lights, rim lights for silhouette definition — but flat-shaded low-poly models don't have enough surface detail to benefit. The key+fill+ambient combo lets `flatShading` do the work.

One thing I do per-model is use `emissive` for functional lights. The hub model's studio light has `emissiveIntensity: 0.85` on its accent-colored face. The red status LED is `emissiveIntensity: 1.1`. The microphone's neon sign is `emissiveIntensity: 2.2` with additive blending. These create the illusion of self-illumination without actual light sources, which is way cheaper than point lights and looks better with flat shading anyway.

Fog helps too:

```typescript
scene.fog = new THREE.FogExp2(0x060608, 0.055);
```

Exponential fog at density 0.055 doesn't hide anything in the scene, but it adds atmospheric depth. Models in the back of the carousel ring are slightly hazier than the front ones. It's a PS2-era trick — cheap depth cue without a depth-of-field shader.

## Performance: the things that actually matter

I burned time on the wrong optimizations early. Here's what actually moved the needle:

### Pixel ratio capping

```typescript
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.75));
```

This is the single biggest performance win. A 3x Retina display renders 9x the pixels of a 1x display. Capping at 1.75 means high-DPI screens still look sharp but you're not burning GPU time on pixels nobody can distinguish. The difference between 2x and 3x is invisible at arm's length. The difference in frame time is not.

### One renderer, not two (or eight)

The site went through three rendering architectures:

```
Phase 1: One Carousel3D renderer (ring of models, single canvas)
Phase 2: Carousel3D + HomeCarouselGrid (two renderers, two RAF loops)
Phase 3: UnifiedScene (one renderer, one RAF loop, both layouts)
```

Phase 2 was the worst. HomeCarouselGrid used scissor viewports — one canvas, but `setScissor` + `setViewport` + `render` per card per frame. Eight render calls, eight scene traversals, eight projection matrix updates, plus a separate renderer for the carousel. Browsers cap WebGL contexts at 8-16, and two full renderers with their own RAF loops were fighting for frame budget.

Phase 3 replaced both with one canvas, one scene, one `renderer.render()` call. Models that were in a ring for carousel mode get spring-interpolated to DOM-projected positions for grid mode. One set of models, one lighting rig, one RAF.

### Texture memory

`generateMipmaps: false` on every texture. Canvas-generated textures don't need mipmap chains — they're used at native resolution. This cuts texture memory roughly in half.

### What doesn't matter

- `powerPreference: "low-power"` — I set this on the grid renderer and couldn't measure a difference on any device I tested. Maybe it helps on specific laptop/GPU combos.
- Polygon count — these models are 50-200 triangles each. Even the complex hub model is maybe 400. You'd need thousands of models before geometry becomes the bottleneck.
- Material count — 5-15 materials per model. The real cost is draw calls, and with one scene and no transparency sorting issues, this is fine.

## Z-fighting and polygon offset

Flat labels on flat surfaces z-fight. Always. If you put a `PlaneGeometry` label on a `BoxGeometry` body at the same depth, the GPU flickers between them because floating-point depth values are identical.

The fix is `polygonOffset`:

```typescript
new THREE.MeshStandardMaterial({
  map: labelTexture,
  polygonOffset: true,
  polygonOffsetFactor: -1,
  polygonOffsetUnits: -1,
});
```

Negative values push the label *toward* the camera in depth buffer space without moving it in world space. I use `-1/-1` for standard overlays, `-3/-3` for tape labels on surfaces, and `-4/-4` for stamps on covers (which sit on top of other overlays).

This is one of those things that's obvious in retrospect but I spent a while staring at flickering sticker textures before I figured it out.

## The hub model

The streaming toolkit hub (`JuryRiggedHubModel.ts`) is the most complex model and the one I learned the most from. It's a military-green electrical box with a microphone, camera lens, tape deck, antenna light, and exposed cables bolted onto it. All procedural.

A few specific things I got better at while building it:

**Canvas texture layering for the front panel:**
1. Base gradient (military green with vertical grain from `Math.sin(px * 0.4) * 4`)
2. 220 thin scuff lines at varying opacity
3. 8 rust patches as radial gradients
4. "DM TOOLKIT" stenciled text with a yellow shadow pass, cream main pass, then random eraser dots using `globalCompositeOperation: "destination-out"` to break up the paint
5. Neon-red LIVE sticker painted as a separate element with gradient fill and highlight
6. Radial AO vignette over the whole thing

Each layer builds on the previous one. The order matters — paint drips go *after* the text, eraser dots go *after* the drips, the vignette goes last.

**Cables using Catmull-Rom curves:**

```typescript
const cablePath = new THREE.CatmullRomCurve3([
  new THREE.Vector3(-0.15, -0.08, 0.22),
  new THREE.Vector3(-0.28, -0.18, 0.28),
  new THREE.Vector3(-0.22, -0.30, 0.14),
  new THREE.Vector3(0.08, -0.28, 0.24),
]);
const cableGeo = new THREE.TubeGeometry(cablePath, 24, 0.013, 6, false);
```

Bright red mic cable and bright yellow power cable, swooping between components. The curve control points took a lot of tweaking — cables need to look like they have weight and sag without clipping through geometry. 24 segments is enough to look smooth, 6 radial segments is enough for a tube that thin.

**Sub-component scale for silhouette readability:**

The mic is scaled to 1.5x, the lens to 1.45x, the tape deck to 1.25x. These are deliberately oversized relative to the box because the model is small on screen and needs the major components to read at a glance. The antenna light is 0.85x — it's a secondary detail that shouldn't compete with the hero elements.

## What I'd do differently

**Convention over configuration for model facing.** Every model should build its front on +Z and let the rendering context handle rotation. I wasted real time on the orientation bug because each model had its own idea of which way was forward.

**Fewer texture sizes.** I have canvases ranging from 64x64 to 1024x680. Standardizing on 2-3 sizes (256, 512, 1024) would simplify the system and make texture memory more predictable.

**Separate the aesthetic from the rendering.** The PS1 shader, jankify, flat shading, and wireframe overlays are all independent effects that I tangled together. When I wanted to dial back the PS1 jitter but keep the flat shading, I had to touch every material factory. These should be composable layers, not baked into the material pipeline.

**Abstract the canvas texture painting.** Every model has its own texture functions with similar patterns (noise base → detail layers → AO vignette). A shared texture builder with a layer stack would cut a lot of repetition. I didn't do this because each texture felt unique enough at the time, but looking at 8 models worth of canvas painting code, the patterns are obvious.

## Running it

This is part of [deutschmark.online](https://deutschmark.online). The model builders are in `components/carousel/`, the material system in `components/carousel/utils.ts`, and the unified renderer in `components/UnifiedScene.tsx`.
