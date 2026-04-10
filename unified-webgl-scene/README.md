# Single-Canvas Multi-Layout 3D in React

How I replaced two independent Three.js renderers with one unified scene that spring-interpolates 8 models between a carousel ring and a DOM-aligned bento grid — on a single WebGL context.

## The problem

My homepage shows 8 project cards, each with a rotating 3D model. It also has a carousel mode where those same 8 models orbit in a ring. Two views, same models, two completely separate rendering systems:

```
BEFORE

┌─────────────────────────────────────────────┐
│  Carousel3D                                 │
│  ┌───────────────────────────────────┐      │
│  │  WebGL Context #1                 │      │
│  │  1 renderer, 1 scene, 1 RAF      │      │
│  │  8 models in a ring              │      │
│  │  drag-to-spin interaction        │      │
│  └───────────────────────────────────┘      │
│                                             │
│  HomeCarouselGrid                           │
│  ┌───────────────────────────────────┐      │
│  │  WebGL Context #2                 │      │
│  │  1 renderer, 8 scenes, 8 cameras │      │
│  │  scissor viewports per card      │      │
│  │  no transitions possible         │      │
│  └───────────────────────────────────┘      │
└─────────────────────────────────────────────┘
```

This had three issues:

1. **Two WebGL contexts.** Browsers cap active contexts at ~8-16. Every context costs GPU memory. Two renderers meant two RAF loops fighting for frame budget.

2. **No transitions.** Switching between carousel and grid was a hard cut — mount one, unmount the other. The models couldn't fly between layouts because they lived in different scenes.

3. **Duplicate state.** Both systems built the same 8 models from the same builders with the same accents. Two sets of lighting. Two resize handlers. Two cleanup paths.

## Constraints

- **Static export.** The site is a Next.js `output: "export"` deployed to Cloudflare Pages. No server components, no streaming. Everything runs client-side.
- **DOM layout owns positioning.** The bento grid is CSS Grid with responsive breakpoints. 3D models need to appear *inside* those cards, but the cards are plain DOM elements with text, links, and other chrome layered on top.
- **One canvas, full viewport.** The canvas is `position: fixed; inset: 0` so it can render both the carousel ring (centered in 3D space) and models aligned to arbitrary DOM rects.

## Approach: Unproject, don't scissor

The old HomeCarouselGrid used **scissor viewports** — one canvas, but clipped rendering into each card's bounding rect with a per-card camera:

```
Old: scissor approach
┌──────────────────────────────────────┐
│ Canvas (full viewport)               │
│  ┌─────────┐  ┌─────────┐           │
│  │ scissor  │  │ scissor  │          │
│  │ region 1 │  │ region 2 │   ...    │
│  │ cam + scene│ cam + scene│         │
│  └─────────┘  └─────────┘           │
│  Per-card: setScissor → setViewport  │
│            → render → repeat         │
└──────────────────────────────────────┘
```

This works but makes transitions impossible. Each model lives in its own isolated scene with its own camera. You can't animate a model from one scissor region to another.

The unified approach flips this: **one scene, one camera, one render call.** Models are placed in world space. In grid mode, their world positions are computed by unprojecting each card's screen-space center back into 3D:

```
New: unified scene
┌──────────────────────────────────────┐
│ Canvas (full viewport)               │
│                                      │
│  All 8 models + earth in ONE scene   │
│  ONE camera, ONE render call         │
│                                      │
│  Grid mode:                          │
│    DOM card rect → NDC → unproject   │
│    → world position on depth plane   │
│                                      │
│  Carousel mode:                      │
│    Models on ring at fixed angles    │
│                                      │
│  Transition:                         │
│    Spring-lerp between both layouts  │
└──────────────────────────────────────┘
```

## Key code

### 1. Slot registration

Each bento card renders an invisible `<div>` that registers itself with the scene via React Context:

```tsx
export function UnifiedSlot({ index, slotKey, className }: UnifiedSlotProps) {
  const ctx = useContext(UnifiedSceneContext);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!ctx || !ref.current) return;
    return ctx.registerSlot(slotKey, index, ref.current);
  }, [ctx, slotKey, index]);

  return <div ref={ref} className={className} aria-hidden="true" />;
}
```

The provider stores a `Map<string, { index: number; el: HTMLDivElement }>`. On every animation frame, the loop reads `getBoundingClientRect()` from each registered element to find where models should go.

### 2. Screen-to-world unprojection

The core trick. Given a card's screen-space bounding rect, find the world-space position where the model should sit:

```typescript
function screenToWorld(rect: DOMRect): THREE.Vector3 {
  const cx = rect.left + rect.width / 2;
  const cy = rect.top + rect.height / 2;

  // Screen → NDC (Normalized Device Coordinates, -1 to 1)
  const ndcX = (cx / window.innerWidth) * 2 - 1;
  const ndcY = -(cy / window.innerHeight) * 2 + 1;

  // NDC → world ray
  _v.set(ndcX, ndcY, 0.5).unproject(camera);
  const dir = _v.sub(camera.position).normalize();

  // Intersect ray with depth plane at Z=0
  const t = (0 - camera.position.z) / dir.z;
  return camera.position.clone().add(dir.clone().multiplyScalar(t));
}
```

The `0.5` depth value is arbitrary — `unproject` casts a ray from the camera through that NDC point, then we solve for where that ray hits the `Z=0` plane. The model ends up at the world position that projects back to the card's center on screen.

This runs every frame, so models track their cards through scrolling, resizing, and layout changes automatically.

### 3. Dual-scale model config

Each model needs different parameters for each layout:

```typescript
const MODELS: ModelConfig[] = [
  {
    build: buildPathosFolder,
    carouselScale: 0.62,  // small, in ring
    gridScale: 4.2,       // large, filling card
    gridRotY: 0.2,        // camera-facing angle
    spinSpeed: 0.18,      // slow spin in grid mode
  },
  // ... 7 more
];
```

Carousel scale is ~0.5-0.7 (models share a tight ring). Grid scale is ~3-5 (models fill their card area). The scale difference is ~6-8x.

### 4. Spring interpolation

A single `modeProgress` value (0 = carousel, 1 = grid) drives everything:

```typescript
// Spring-lerp toward target each frame
const target = modeRef.current === "grid" ? 1 : 0;
const nextMp = mp + (target - mp) * Math.min(1, SPRING * dt);

// Per-model interpolation
wrapper.position.lerpVectors(carouselPos, gridPos, nextMp);

const gridRotY = cfg.gridRotY + now * 0.001 * cfg.spinSpeed;
wrapper.rotation.y = carouselRotY * (1 - nextMp) + gridRotY * nextMp;

const s = cfg.carouselScale * (1 - nextMp) + cfg.gridScale * nextMp;
model.scale.setScalar(s);
```

Every model's position, rotation, and scale are interpolated between their carousel layout values and their grid layout values. The spring constant (`SPRING = 4.5`) controls how snappy the transition feels.

The earth object gets the same treatment — centered and large in carousel mode, small and offset to the hero area in grid mode:

```typescript
earth.position.lerpVectors(
  new THREE.Vector3(0, -0.1, 0),    // carousel: centered
  new THREE.Vector3(3, 2.5, -2),    // grid: hero area
  nextMp,
);
earth.scale.setScalar(0.9 - nextMp * 0.4);
```

### 5. Carousel interaction gating

Drag-to-spin only works in carousel mode. The ring's rotation fades out as `modeProgress` approaches 1:

```typescript
ring.rotation.y = ringAngle * (1 - nextMp);
canvas.style.pointerEvents = nextMp > 0.5 ? "none" : "auto";
```

In grid mode, the canvas is pointer-transparent so users can click the DOM cards beneath it.

## Architecture comparison

```
                    Old (scissor)          New (unified)
─────────────────────────────────────────────────────────
WebGL contexts      2                      1
Scenes              1 + 8 per-card         1
Cameras             1 + 8 per-card         1
Render calls/frame  8 (scissor loop)       1
Lighting setups     2 × (ambient + 2 dir)  1 × (ambient + 2 dir)
Mode transitions    hard cut               spring interpolation
Position mapping    per-card camera         screenToWorld() unproject
RAF loops           2                      1
```

## Results and tradeoffs

**What worked:**
- Smooth transitions. Models fly from ring positions to card positions with interpolated scale and rotation. Looks like one cohesive system instead of two bolted-on renderers.
- Lower GPU overhead. One context, one lighting rig, one render call.
- Scroll tracking for free. `getBoundingClientRect()` every frame means models stay locked to their cards during scroll, resize, and responsive breakpoint changes.

**Tradeoffs accepted:**
- `getBoundingClientRect()` on 8 elements every frame. This triggers layout reads but in practice costs <0.1ms on modern browsers. Would be a problem at 50+ slots.
- One shared camera means models in the grid corners are viewed at slight angles vs. the old per-card camera that always looked straight at each model. Barely noticeable in practice.
- The `Z=0` depth plane assumption means all grid models sit at the same depth. If cards overlapped vertically, models would z-fight. The bento grid doesn't have this issue.

**What I'd do differently:**
- Add per-model stagger to the spring transition so models fly out in sequence instead of all at once.
- Consider `IntersectionObserver` to skip `getBoundingClientRect()` for off-screen cards instead of checking `rect.width > 1`.
- The carousel interaction code (drag physics, velocity damping) could be extracted into a reusable hook since it's independent of the rendering system.

## Running it

This is part of [deutschmark.online](https://deutschmark.online). Toggle the carousel/grid view in the top-right corner of the homepage to see the transition. The source is at `components/UnifiedScene.tsx`.
