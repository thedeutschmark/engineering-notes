# Building 3D Objects for the Web with Three.js

*Procedural modeling, PS1-era shading, and canvas-based textures for a homepage carousel.*

I built a homepage where each project card carries its own small rotating 3D object. None of them are imported assets. They are all built directly in code with Three.js primitives, lightweight materials, and procedural textures.

That was not supposed to become a system. It started as one object, then turned into a pattern.

---

## Why I went procedural

The original reason was simple: I wanted a specific look and could not find assets that matched it.

Once I had a few models working, the upside of doing it procedurally became clearer:

- no external asset pipeline
- direct control over silhouette and materials
- easy color variation per card
- a consistent visual language across unrelated objects

The downside is obvious too. This is slower to author than dropping in ready-made assets, and every object becomes part of the codebase instead of part of an asset folder.

That tradeoff was worth it for this project because the models are small, stylized, and tightly coupled to the site identity.

## The modeling pattern that held up

The useful pattern was not sophisticated. I stopped thinking in terms of "modeling" and started thinking in terms of **assembly**.

Each object is built in layers:

- a **base form** that establishes the silhouette
- **secondary shapes** that make the object recognizable
- **surface treatment** that gives it age, texture, and personality

That kept the builders readable and made it easier to share conventions across the whole set without turning the code into one giant generic model factory.

---

## How a model is actually built

Here is the VCR, which is representative of the pattern.

**Base form.** A BoxGeometry at the right proportions. Before it becomes a mesh, the geometry goes through a vertex perturbation pass that adds subtle random displacement to every vertex. That is what makes the object feel hand-built instead of CAD-clean. The amount is small enough to preserve the silhouette but large enough to break the machine-perfect edges.

**Material.** Every material in the system uses the same factory: flat-shaded, fully matte, double-sided. The factory also injects a **vertex shader modification** that snaps screen-space positions to a coarse grid, creating the polygonal shimmer that defined PS1-era rendering. The grid precision is tuned to be gentle enough to read clearly but coarse enough to feel analog. It also gets coarser with distance from the camera, which keeps close-up objects legible while distant ones shimmer more.

**Procedural texture.** This is where most of the character lives. The VCR body texture is a canvas element drawn entirely in code:

- base fill in deep charcoal
- thousands of tiny random rectangles in faint white to simulate **brushed plastic grain**
- a thin stroke inset for the shell seam
- a green monospace clock display
- button regions with slight color variation

The VHS tapes each get their own label texture with different colors, brand text, and simulated wear marks. **Per-face material assignment** maps the label to the top face of the tape geometry and dark plastic to every other face.

**Assembly.** The tapes are grouped with idle positions (scattered casually) and open positions (fanned out in a row). The builder stores both position sets and a tick function in the group's userData. The carousel animation system calls the tick function each frame, and the tape positions interpolate between idle and open states with staggered easing.

The other objects follow the same pattern with different specifics. The microphone uses tapered cylinder sections for the body, a sphere for the head cap, a canvas texture for the grill mesh pattern, and a TubeGeometry on a CatmullRom curve for the coiled XLR cable. The PDA has a swappable screen texture so it can show different content depending on card state. The folder has a cover that pivots on a hinge with document pages that fan out on open.

---

## The look

The visual direction started with a stronger PS1 influence than the final site ended up keeping.

At first I pushed hard on:

- vertex instability
- coarse filtering
- more aggressive geometric distortion

It was visually interesting, but it crossed the line from "stylized" into "noisy." On mobile especially, the effect started fighting the content instead of supporting it.

The version that survived was the quieter one:

- low-poly forms
- flat-shaded surfaces
- restrained imperfections
- textures that feel worn or handmade rather than clean and synthetic

> The models needed personality, not spectacle.

## Procedural textures mattered more than geometry

The real identity of the objects came less from the meshes and more from the surfaces.

Most of the character lives in **canvas-based texture work**:

- panel labels drawn with `fillText`
- brushed plastic grain from thousands of tiny random rectangles
- edge darkening via procedural ambient occlusion (distance from UV border)
- radial gradients for depth shadows under raised elements
- stickers and doodle marks on the folder interior

That was the difference between "a box that roughly looks like a VCR" and "an object that feels like it has been around."

I liked this approach because it kept the whole aesthetic inside one system. No image assets, no texture atlas, no separate authoring tool. The downside is that canvas drawing code is verbose and hard to preview without running it. But for a set of eight objects, that cost was manageable.

---

## Orientation was harder than modeling

The most annoying problem was not shading or performance. It was **orientation**.

As soon as the same objects had to live in multiple presentation contexts, all the hidden inconsistency around what counted as "front" became visible. A model that looked correct in one layout would feel wrong in another because the surrounding scene assumed a different forward axis.

That is the kind of problem that does not look glamorous in a postmortem, but it matters. A lot of graphics work is really convention management.

If I were starting over, I would lock a stricter orientation convention early and make every presentation context respect it instead of patching per-object offsets later.

## Lighting and performance

The lighting stayed intentionally simple. A warm key, a cool fill, and ambient. No shadows, no environment maps. These objects did not need a complicated rig. They needed enough directionality to read clearly and enough restraint that the materials kept the scene coherent.

The render path uses an offscreen render target that blits to the canvas through a fullscreen quad. That isolates the 3D pass and keeps the upscale filter consistent. Frame delta is capped so animation interpolation does not explode on tab-switch.

Performance ended up depending less on polygon count (these are tiny models) and more on broader scene decisions:

- how many render paths existed
- pixel ratio capped below full retina to avoid overdraw
- canvas textures generated once on init, not per frame
- no mipmaps (unnecessary at this scale, saves memory)

> Renderer architecture matters more than micro-optimizing tiny objects.

---

## What worked

- procedural modeling was the right choice for this scale and style
- flat shading carried more of the visual language than any fancy shader work
- canvas textures gave the objects personality without needing an asset library
- the PS1 vertex snapper at restrained intensity added character without noise
- simplifying the aesthetic improved the site more than pushing the retro effect harder

## What I would change next

- standardize orientation rules much earlier
- reduce repeated texture-building patterns into a smaller shared system
- keep the stylization modular so surface treatment and motion treatment can evolve separately
- keep choosing readability over effect-heavy rendering when the two conflict

## Running it

This work is part of [deutschmark.online](https://deutschmark.online).
