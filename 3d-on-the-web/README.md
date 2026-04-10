# Building 3D Objects for the Web with Three.js

What I learned building eight procedural models for my homepage: how I structured the builders, where the PS1-inspired look helped, where it hurt, and which performance decisions actually mattered.

## Why I went procedural

My homepage has eight project cards, each with its own rotating 3D object: a manila folder, VCR, traffic cone, camera, microphone, PDA, desk calendar, and a jury-rigged streaming toolkit hub. None of them are imported `.glb` or `.gltf` assets. Every model is built from Three.js primitives in code.

That was not the original plan. I started with one folder model because I wanted a very specific worn, hand-built look and could not find an asset that matched it. After a few more models, the approach became a system.

The upsides were real:

- No asset loading or asset pipeline
- Full control over materials and silhouette
- Procedural textures that can inherit an accent color from the card they sit on

The tradeoff is authoring cost. A simple model is still a few hundred lines of geometry and texture code. The hub model ended up much larger than that.

## The model pattern

After the first few models I settled on a consistent builder shape: base form first, then subcomponents, then surface detail.

```typescript
export function buildVCRPlayer(accent: THREE.Color): THREE.Object3D {
  const g = new THREE.Group();

  const body = new THREE.Mesh(
    new THREE.BoxGeometry(1.1, 0.18, 0.65),
    psTex(makeBodyTexture(), { roughness: 0.92 }),
  );
  g.add(body);

  // display, tape slot, buttons, labels, screws

  return g;
}
```

That structure kept the models readable. It also made it easy to share conventions across all eight builders, especially around accent handling and material tiers.

## Material system

Most of the rendering look came from two material factories:

```typescript
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

`flatShading: true` does most of the aesthetic work. It keeps surfaces faceted and readable without needing dense geometry. From there I mainly varied `roughness`, `metalness`, and occasional `emissive` use:

| Roughness | Metalness | Typical use |
|---|---|---|
| `0.85-1.0` | `0.0` | cardboard, plastic, matte paint |
| `0.7-0.9` | `0.05-0.2` | weathered metal, older hardware |
| `0.3-0.6` | `0.4-0.6` | trim, brushed metal, cleaner accents |
| `0.0-0.2` | `0.7-0.9` | polished surfaces and highlights |

The microphone used the widest range. The grille needed to read as metallic, while the sign and indicator lights relied on `emissive` rather than real lights.

## Procedural textures with canvas

Every label, screen, sticker, stain, and panel treatment comes from a `<canvas>` that gets turned into a Three.js texture. No image assets.

The base pattern is simple noise plus a cheap edge-darkening pass:

```typescript
function makeNoisyTexture(w, h, baseR, baseG, baseB, opts) {
  const ctx = canvas.getContext("2d");
  const img = ctx.createImageData(w, h);

  for (let i = 0; i < img.data.length; i += 4) {
    const px = (i / 4) % w;
    const py = Math.floor(i / 4 / w);
    const n = (Math.random() - 0.5) * opts.noise;

    const ex = Math.min(px, w - px) / (w * 0.15);
    const ey = Math.min(py, h - py) / (h * 0.15);
    const ao = 0.55 + 0.45 * Math.min(1, Math.min(ex, ey));

    img.data[i] = (baseR + n) * ao;
    img.data[i + 1] = (baseG + n) * ao;
    img.data[i + 2] = (baseB + n) * ao;
    img.data[i + 3] = 255;
  }

  ctx.putImageData(img, 0, 0);
}
```

It is not physically accurate ambient occlusion, but it is cheap and it works well for flat panels. On top of that base I layered text, scratches, stains, gradients, doodles, and labels with plain Canvas 2D operations.

That let each model carry its own personality. The folder interior has notebook-style sketches. The VCR body has worn labels and panel marks. The hub front panel has stenciled text, rust, scuffs, and sticker damage built up in layers.

One practical detail: I disabled mipmap generation on these canvas textures. In this scene they are usually rendered near authored size, so skipping mipmaps saved memory and upload work without hurting the look. It does not halve memory usage, but it does avoid the extra mip chain overhead.

## Dialing the PS1 look back

The original look pushed much harder into PS1-style instability: screen-space vertex snapping, nearest-neighbor filtering, and a geometry distortion pass I called `jankify`.

The vertex shader snapped projected positions to a coarse grid:

```glsl
vec4 mvPosition = modelViewMatrix * vec4(transformed, 1.0);
gl_Position = projectionMatrix * mvPosition;

float prec = basePrecision * (1.0 + dist * 0.12);
gl_Position.xy /= gl_Position.w;
gl_Position.xy = floor(gl_Position.xy * prec) / prec;
gl_Position.xy *= gl_Position.w;
```

And `jankify` added random displacement directly to geometry:

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

On desktop that looked distinctive. On mobile it made text unreadable and motion noisier than I wanted. The full effect was technically interesting but wrong for the product.

The final version kept the influence and dropped the excess:

```text
Mar 19  heavy jitter + NearestFilter              -> too aggressive
Mar 29  distance attenuation added                -> better, still brittle
Apr 9   jitter removed, LinearFilter, AA enabled  -> clean but bland
Apr 10  subtle jitter restored, gentler geometry  -> final direction
```

What survived was the part users could feel without having to notice: low polygon counts, flat shading, slightly imperfect geometry, and restrained shimmer. That ended up reading more like "retro-inspired" than "hardware artifact simulator," which was the better fit.

## Orientation was harder than modeling

The most annoying bug was not shading or textures. It was forward direction.

Models that looked correct in the carousel would face the wrong way in the bento grid because the two contexts implied different camera relationships. The carousel wrapper rotates each item to face outward:

```typescript
const angle = i * ANGLE_STEP;
wrapper.position.set(
  Math.sin(angle) * RING_RADIUS,
  0,
  Math.cos(angle) * RING_RADIUS,
);
wrapper.rotation.y = -angle;
```

That works cleanly only if every model shares the same definition of "front." Mine did not. Some were built with front on `+Z`, some on `-Z`, and once the same model had to live in both a ring and a head-on grid, those inconsistencies surfaced immediately.

The short-term fix was per-model overrides:

```typescript
const MODEL_SPECS = {
  0: { initialRotY: 0.2 },
  3: { initialRotY: Math.PI + 0.25 },
  6: { initialRotY: Math.PI + 0.1 },
};
```

The real fix would have been a stricter convention from the start: every builder exports a model whose front is `+Z`, and each presentation context owns rotation independently.

## Lighting

The lighting setup stayed simple:

```typescript
scene.add(new THREE.AmbientLight(0x2a2f38, 0.5));

const keyLight = new THREE.DirectionalLight(0xffeedd, 1.6);
keyLight.position.set(5, 3, 5);
scene.add(keyLight);

const fillLight = new THREE.DirectionalLight(0x6a7a9a, 0.45);
fillLight.position.set(-3, 2, -2);
scene.add(fillLight);
```

Ambient plus key plus fill was enough. Flat-shaded low-poly models benefit more from readable directionality than from a complex light rig. For "live" elements such as LEDs and signs, `emissive` produced a better result than adding extra point lights.

I also used light fog:

```typescript
scene.fog = new THREE.FogExp2(0x060608, 0.055);
```

Not enough to obscure anything, just enough to separate front and back of the carousel ring.

## Performance: what actually helped

I spent time optimizing the wrong things early. The wins that mattered were simpler.

### Cap device pixel ratio

```typescript
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.75));
```

This was the clearest performance improvement. Very high-DPI screens can burn a lot of fill rate for marginal visual gain. Capping the pixel ratio preserved sharpness while cutting unnecessary work.

### Use one renderer and one RAF loop

The site went through three rendering setups:

```text
Phase 1: one carousel renderer
Phase 2: carousel renderer + grid renderer
Phase 3: one unified scene for both layouts
```

Phase 2 was the worst. Multiple render passes, multiple viewports, and two animation loops added complexity and frame cost at the same time. The unified scene replaced that with one canvas, one scene, and one render call, with models interpolating between carousel and grid positions.

### Skip unnecessary mipmaps

For this specific texture set, disabling mipmap generation reduced texture overhead and simplified uploads. It was worthwhile, just not as dramatic as I first thought.

### What did not matter much

- Polygon count at this scale
- Small differences in material count
- `powerPreference` tuning, at least on the devices I tested

The scene was not geometry-bound. It was much more sensitive to pixel cost and renderer architecture.

## Z-fighting and polygon offset

Any flat decal sitting exactly on a flat panel will eventually flicker. The reliable fix was `polygonOffset`:

```typescript
new THREE.MeshStandardMaterial({
  map: labelTexture,
  polygonOffset: true,
  polygonOffsetFactor: -1,
  polygonOffsetUnits: -1,
});
```

I used more aggressive values for labels layered on top of other overlays. That solved the problem without physically moving geometry around.

## The hub model

The streaming toolkit hub was the most useful model to build because it forced several techniques to work together:

- Layered canvas painting for the front panel
- Overscaled subcomponents so the silhouette still reads when the model is small
- Cables built from `CatmullRomCurve3` and `TubeGeometry`

```typescript
const cablePath = new THREE.CatmullRomCurve3([
  new THREE.Vector3(-0.15, -0.08, 0.22),
  new THREE.Vector3(-0.28, -0.18, 0.28),
  new THREE.Vector3(-0.22, -0.30, 0.14),
  new THREE.Vector3(0.08, -0.28, 0.24),
]);
const cableGeo = new THREE.TubeGeometry(cablePath, 24, 0.013, 6, false);
```

The scale choices on that model were deliberately not realistic. The mic, lens, and tape deck are oversized because this object had to read immediately at card scale. Realistic proportion mattered less than visual recognition.

## What I would change next

- Standardize model orientation around a single forward axis
- Normalize texture sizes to a smaller set of predictable resolutions
- Separate material styling, geometric distortion, and shader effects into composable layers instead of mixing them together
- Abstract the repeated canvas-painting patterns into a shared layer system

## Running it

This work is part of [deutschmark.online](https://deutschmark.online). The model builders live in `components/carousel/`, the material helpers in `components/carousel/utils.ts`, and the unified renderer in `components/UnifiedScene.tsx`.
