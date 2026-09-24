# `src/Layers/xrRender` — the renderer core

Chapter 18 of [`SYSTEM-REQUIREMENTS.md`](../../../SYSTEM-REQUIREMENTS.md#7-build-order).

This is the largest chapter in the recipe and the one the most pages depend on. It holds
everything about rendering that is **not** a graphics API and **not** a frame graph: how a
frame's visible set is discovered, what a material is and how one is compiled out of
shipped data, what a model is and how each kind of model turns itself into a draw, how
textures are named and paired, where lights and shadow budgets live, and the single
redundancy-filtering stream every draw in the engine passes through.

Both shipped backends compile this source. The deferred frame graph that sits on top of it
is [chapter 19](../xrRender_R2/README.md); the two fillings of the
[graphics device seam](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) are chapters
20 and 21. Nothing here knows which backend it is running in: the boundary is a handful of
type aliases and a capability record, and the pages say where.

## Where it sits

It rests on the core layer (virtual filesystem, configuration, strings, threads,
allocator), the math layer, the collision database (whose frustum type is used more by the
visibility walk than by collision), the material system, the particle simulator, and the
interfaces in [`src/Include/xrRender`](../../Include/xrRender/README.md) — which is the
contract this chapter fills. It reaches the engine for the device, the console and the
object registry; that edge is one of the declared cycles in §7 and is broken at the
interface.

Three subdirectories belong to this chapter:
[`blenders/`](blenders/README.md) — the material templates themselves;
[`Debug/`](Debug/README.md); [`Utils/`](Utils/README.md).

---

## The ideas you need before the twins make sense

Every one of these is stated here once so that three hundred pages can stay terse. When a
twin says "the coverage estimate" or "the frozen naming contract", this is what it means.

### A frame is five acts, and each runs in a *render context*

A context is one independent traversal: its own camera, its own frustum, its own visibility
options, its own set of buckets, its own command list. The main view is one context; each
sun cascade, the rain shadow map and each shadow-map batch are others. They are prepared
during *calculate*, before any drawing, and joined at the point in the frame graph where
their output is first needed. Several contexts can be walked in parallel; this is why every
per-visual "already seen this frame" marker is indexed by context, not global.

1. **Calculate** — cameras, thresholds and light gathering; contexts are started.
2. **Collect** — the visibility walk files every drawable leaf into a bucket.
3. **Sort** — buckets are ordered, by cost for opaque geometry and by correctness for
   everything else.
4. **Drain** — buckets are emptied into the command list as draws.
5. **Present** — the frame graph's post chain and the swap.

Acts 2–4 are this chapter. Act 1's light gathering is this chapter; its frame graph is
chapter 19.

### Visibility is three filters in a fixed order

A rebuild must apply them in this order, because each is cheap only because its
predecessor already ran.

**Topology.** A level is authored as **sectors** (rooms) joined by **portals** (convex
polygons in a plane). The walk starts in the camera's sector and recurses: for each portal
out of the current sector, clip the current frustum against the portal's polygon; if the
result is non-empty, enter the neighbour with that narrower frustum and a narrower screen
rectangle. A sector reached by two routes is entered twice with two different frusta, which
is correct and is why the walk is not a visited-set traversal. Portals that are still open
at the recursion limit are filled with ambient rather than left as holes. Every triangle in
the collision database names its sector, which is how a point in the world is resolved to a
sector at all — by casting a ray and asking what it hits.

**Occlusion.** The level ships a small set of authored **occluder** surfaces. Once per
frame they are software-rasterized into a low-resolution depth image, reduced to a maximum
pyramid, and then any bounding box can be rejected in a few comparisons. It is
conservative — it never rejects something visible — and it is scheduled: a fixed slice of
triangles is rasterized per frame and objects are tested on a rotating schedule, because
neither side is affordable at full rate.

**Coverage.** Apparent size, estimated as `radius / distance²`. This is not an area and not
a solid angle; it is the cheap monotone stand-in for one, and **every threshold in the
engine is calibrated against this exact expression** — the discard cutoff, the
level-of-detail switch band, the continuous detail ramp, the "stop sorting, it does not
matter" cutoff, a shadowing light's shadow-map allocation. A rebuild that substitutes a
truer measure must recalibrate all of them together.

A fourth filter, hardware **occlusion queries**, is not part of collection: it is used
after the depth buffer exists, to decide whether a *light* is worth accumulating. Query
objects are pooled and handed out in a strict allocate/collect pattern so that a small fixed
set is reused frame after frame, and a query's answer is read a frame or more later — never
in the frame that issued it, which would stall.

### The draw stream: buckets, the sort key, and what "batch" means

Collection does not draw. It files each drawable leaf into a **bucket** keyed by the
material pass it will be drawn with, inside a slot indexed by the pass's position in its
material (pass 0, pass 1, …). Four bucket shapes exist, distinguished by what a queued item
has to remember: static geometry (coverage and the visual — the world transform is
identity), dynamic geometry (coverage, owner, visual and a *copy* of the transform, because
the owner may move before the bucket is drained), dynamic geometry carrying its own
material element, and imposters.

Draining sorts twice:

- **Passes within a slot** are ordered by a *content* equality first — two distinct pass
  records that resolve to the same programs, textures and state compare equal, so the sort
  leaves them adjacent and the command list's redundancy filter drops the change entirely —
  and then by the largest coverage in the bucket, so the biggest thing is drawn first.
- **Items within a pass** are ordered by coverage descending, so the depth buffer rejects
  the most pixels earliest. Above a configurable coverage this sort is skipped as not worth
  its cost.

That is the whole sort key: *(pass slot, pass content, coverage)*. Everything that is not
opaque geometry is ordered by correctness instead and has its own bucket — transparency
back to front by distance, decals after the surfaces they sit on, the first-person weapon
layer inside its own projection and near-plane range applied once around the whole bucket.
Distortion is a *separate* registration rather than an alternative one: a refracting
surface is filed twice, once into the distortion buffer and once normally.

"Batching" in this engine means two different things and the pages distinguish them: geometry
that shares one immutable buffer and is drawn as index ranges (the level's static world,
models in the pool), and geometry generated per frame into a **dynamic stream** — an append
ring for vertices and one for indices, written until full, then wrapped with a discard so
the device never waits on the pointer that just passed. Sprites, particles, decals, grass on
the software path, software-skinned meshes, debug lines and all UI geometry share those two
rings, and one shared immutable index buffer supplies the quad pattern every sprite in the
engine draws through.

### A material is compiled from data, and the compilation is frozen

The word **shader** in this project usually means *a compiled material*, not a GPU program.
The chain is:

```text
material name + texture names
  -> blender (a parameterized template, identified by a four-character class id)
  -> compile(element)  ->  Shader
                             element[0..5]   # one per render mode / pass phase
                               pass[0..n]    # each a bound set of:
                                 programs (one per shader stage)
                                 state block
                                 vertex declaration
                                 texture list
                                 constant list
                                 animated matrix list
```

A **blender** is a template with named, typed, editor-visible parameters whose byte layout
inside the shipped material library is a **wire format** — see
[`blenders/README.md`](blenders/README.md). `Compile` is the act of turning
(parameters, requested element, current quality options, device capabilities) into a
concrete pass list. The requested element is a selector, which is why one blender serves
the g-buffer pass, the shadow pass, the old forward path and the distortion pass rather
than there being four blenders.

Three things about this are frozen because shipped data must load:

- **The class identifiers.** A shipped material file names its blender by a fixed
  identifier. Register the same identifiers or the data does not load.
- **The parameter blocks.** Each blender's saved parameters are versioned and are read
  back verbatim from the shipped library.
- **The named constant vocabulary.** A shipped shader program gets its per-frame values by
  *spelling a name*. The set of names an engine answers to — camera and transform products,
  time, weather and fog terms, wind, per-object lighting, screen size, texture sizes — is a
  published surface, and a program that asks for a name nobody fills gets nothing.

Bindings are resolved through the compiled programs' own reflection data: the engine asks
each stage which named constants and which texture slots it actually declared, merges the
stages into one table per shader, and binds only what is there. A constant's destination is
encoded as a single integer covering every stage and register it occupies, so one write
reaches all of them.

Compiled programs are **cached on disk, keyed by the source and the macro set** — the same
source compiled with different quality macros is a different cache entry. This is required,
not an optimization: the shipped shader corpus is large enough that a cold compile is a
visible load-time cost.

**Everything is interned.** One registry owns every texture, material, compiled pass, state
block, program, vertex declaration, render target and animator in the process, under one of
two rules: *intern by name* (two requests for `wall_01` are one texture) or *intern by
value* (two passes that resolve to the same device state are one pass). This is what makes
the content-equality sort above meaningful, and it is what the device-lost bracket walks
when the graphics device is torn down and rebuilt.

### The stencil contract

This one crosses chapter boundaries and is easy to lose.

**Every template that writes the g-buffer marks stencil with reference value 1, comparison
always, read mask all bits, and write mask `0x7f`.** The write mask is the point: bit 7 is
reserved, so marking coverage cannot disturb it. The lighting resolve in chapter 19 tests
"stencil ≥ 1" to restrict itself to pixels geometry actually covered, and the multisample
edge pass sets bit 7 on pixels whose samples disagree so that later lighting can run once
per pixel in the interior and once per sample only on edges. A rebuilder who uses a full
write mask here gets correct-looking output that falls apart the moment multisampling is
enabled.

The light-volume stencil counter in chapter 19 lives in the same eight bits, which is why
it advances by two per light and is only reset when it would run out of range — and why
that range is seven bits, not eight, when multisampling is on.

### The model types

Everything drawable is a **visual**: a type tag, a bounding volume, a material, and a way
to emit a draw. The types, and what each one exists to solve:

| Visual | The problem it solves |
|---|---|
| static | A fixed mesh, a slice of the level's shared buffers — optionally with a position-only twin so depth-only passes cost half the bandwidth |
| hierarchy | A model that draws nothing and holds children that do |
| progressive | Level of detail *without swapping meshes*: vertices ordered most-important-first and indices grouped per level, so choosing a detail level is choosing a draw range — no popping, no second copy |
| imposter | Eight prebaked billboards around an object, one per compass direction; the two facing the camera most nearly are cross-faded. Replaces the real geometry below a coverage threshold, with a band where both are drawn |
| skinned | A mesh bound to a skeleton, in eight vertex layouts (one to four bone influences × quantized or full precision), deformed on the device or on the processor depending on what the load-time decision found |
| tree | Wind-animated vegetation: vertices quantized into a fixed tile, lighting arriving as a scale-and-bias pair, and wind delivered as a handful of constants shared by every tree in the frame |
| particle effect / group | A playing particle instance that is also a visual, so the visibility walk sees it; and a timeline that starts and stops several of them at named times |
| detail (grass) | Not a visual at all — see below |

A **model pool** keeps one loaded copy of each model file and hands out cheap clones: a
clone shares its original's geometry and owns only what must differ (its skeleton pose, its
per-instance constants). Dead clones go on a free list rather than being reloaded. This is
the reason model load time does not scale with the number of objects wearing a model.

Two load-time mesh optimizations run here and are worth naming because they change the
*data*, not the drawing: triangles are reordered to exploit the hardware's post-transform
vertex cache (a cache model is simulated to score candidate orderings), and vertices are
renumbered so the vertex buffer is walked front to back.

**Detail objects** — grass and debris — are a separate system because they are a *field*,
not a set of objects. The level ships a 2-metre grid; each cell names up to four models and
carries a quantized density map plus baked lighting in sixteen bytes. A sliding square
window of decompressed cells follows the camera, rotated a row at a time and refilled at a
fixed budget per frame, nearest first. Decompression dithers the density to decide where a
plant goes, ray-casts down to find the ground, and derives yaw and scale from a seed made
of the cell's own coordinates — so the same cell always produces the same grass.

**Wallmarks** (decals) are cut out of the collision geometry itself: the surface triangles
under the impact are clipped to the decal's projection, batched by material, and faded out
on a timer. A skinned variant exists that follows a posed skeleton.

### Texture naming and the two conventions that must be reproduced exactly

A texture is a *name*, and what that name denotes is decided at load: a still image, a
video stream, or a timed sequence of stills. The binding function is chosen once so the
draw path never branches on which it was. How many top mip levels are dropped is a quality
setting applied at load, not a sampler state.

Two authoring conventions in the shipped art are frozen and are the most common source of
"it loads but looks wrong":

- **Detail textures are named by the base texture, never by the material.** A side
  database maps a *base* texture entry to its detail partner, the partner's tiling scale,
  and whether the detail layer modulates colour, perturbs the normal, or both. This is why
  a material must be able to say which of its textures is "the base" before anything else
  about it can be decided, and why a blender that cannot say so cannot be detailed at all.
  When the detail layer's bump half cannot be used — the hardware or the quality setting
  says no — it is **promoted to a diffuse detail rather than dropped**, because dropping it
  makes affected surfaces visibly flat next to unaffected ones.
- **Bump maps have an engine-specific channel packing,** including a companion map, and
  when a bump map is missing one is *synthesized from the diffuse texture* into exactly
  that packing. The same side database carries each texture's surface material and its
  parallax setting.

### Lights, and how shadow budget is spent

A light is a shape in space plus an entry in the spatial database, so the visibility walk
finds lights the same way it finds geometry. The level ships a baked static set; the sun is
picked out of it by identity and is repositioned every frame to wherever the weather system
says the sun is, five hundred metres behind the camera. A shadowing point light is split
into six cone lights, one per cube face.

Each frame the visible lights are gathered into a package and split into three buckets, the
ordering rule being that a light whose occlusion query has not answered yet is drawn last
and the rest are drawn brightest first. For each shadowing spot light, one decision is made
per frame: **how much of the shadow atlas it deserves** (from its coverage) and **what
camera it is rendered from**. A per-light cache learns, one candidate per frame, which
visuals inside the light's volume cast no visible shadow, and skips them for as long as the
light does not move.

Two more lighting records live here. **Indirect bounce lights** are produced by shooting
photons from a light into the static world, keeping the strongest hits as virtual lights and
normalizing their total energy to a configured budget. And every dynamic object carries a
**light track**: an estimate of how lit it is, built from a small number of sky rays per
frame into a twenty-six-direction sphere, a sun ray every few frames, and per-light
visibility that rises fast and falls slow — all accumulated into a six-faced ambient cube,
and all on a schedule that drops to twice a minute for anything standing still.

### The `dx*` files are port fillings, not features

Roughly twenty files here named `dx…Render` are this chapter's side of an interface the
engine declares in [`src/Include/xrRender`](../../Include/xrRender/README.md): fonts, the
sky and weather, lens flares, rain, lightning, the UI vertex sink, the debug overlay, the
profiler graph, the collision debug view, video sequence items, decal variant sets. The
engine never learns how any of them are drawn. One factory object turns each engine-side
request for a "renderer companion" into a concrete instance and frees it back into this
module's allocator — which matters, because the two sides may be separate binaries with
separate heaps.

---

## The files

### The frame: visibility, collection, ordering

| File | Role |
|---|---|
| [`r__sector.h`](r__sector.h.md) · [`r__sector.cpp`](r__sector.cpp.md) | The level's visibility topology: sectors, portal polygons, the two-way wiring, and the traverser's shape |
| [`r__sector_traversal.cpp`](r__sector_traversal.cpp.md) | The visibility walk: recurse through every portal whose polygon survives the frustum, clipping frustum and screen rectangle at each step |
| [`r__sector_detect.cpp`](r__sector_detect.cpp.md) | Resolving a point to a sector by casting a ray and asking what it hits |
| [`HOM.h`](HOM.h.md) · [`HOM.cpp`](HOM.cpp.md) | The software occlusion map: authored occluders rasterized into a depth hierarchy, with the per-triangle and per-object skip schedules |
| [`occRasterizer.h`](occRasterizer.h.md) · [`occRasterizer.cpp`](occRasterizer.cpp.md) | The occlusion buffer's dimensions, fixed-point depth encoding, maximum-pyramid reduction and rectangle test |
| [`occRasterizer_core.cpp`](occRasterizer_core.cpp.md) | The software scan converter: spans widened half a pixel and inter-triangle gaps bridged, so the map stays conservative |
| [`r__occlusion.h`](r__occlusion.h.md) · [`r__occlusion.cpp`](r__occlusion.cpp.md) | The hardware occlusion-query pool and the strict allocation pattern its callers must honour |
| [`QueryHelper.h`](QueryHelper.h.md) | The five-call shim that hides two very different device query models behind one shape |
| [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md) | One render context: its options, its topology, its buckets, its command list |
| [`r__dsgraph_types.h`](r__dsgraph_types.h.md) | The bucket types: what a queued draw remembers, and how queues are keyed |
| [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) | **Collection**: the three filters, the coverage estimate and its thresholds, and the funnel every drawable leaf passes through |
| [`r__dsgraph_render.cpp`](r__dsgraph_render.cpp.md) | **Draining**: the two sorts, the continuous detail parameter, and each special bucket's required order |
| [`r__dsgraph_render_lods.cpp`](r__dsgraph_render_lods.cpp.md) | Drawing the distant world as imposters, cross-faded between the two prebaked views nearest the camera |
| [`r__pixel_calculator.h`](r__pixel_calculator.h.md) · [`r__pixel_calculator.cpp`](r__pixel_calculator.cpp.md) | An offline tool that measures a model's actual pixel coverage from six directions |
| [`r__sync_point.h`](r__sync_point.h.md) · [`r__sync_point.cpp`](r__sync_point.cpp.md) | Frame pacing: a rotating fence set so the loop waits on work submitted N frames ago, not on the frame just submitted |

### The command list and the geometry streams

| File | Role |
|---|---|
| [`R_Backend.h`](R_Backend.h.md) | **The render command list**: a redundancy-filtering mirror of the whole device state, through which every draw in the engine passes |
| [`R_Backend.cpp`](R_Backend.cpp.md) | The shared quad index buffer — the one immutable index pattern every sprite, particle and screen quad draws through |
| [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md) · [`R_Backend_Runtime.cpp`](R_Backend_Runtime.cpp.md) | The list's lifecycle, the setters too small to be worth a call, dropping every cached assumption, and installing a whole pass's texture list |
| [`R_Backend_xform.h`](R_Backend_xform.h.md) · [`R_Backend_xform.cpp`](R_Backend_xform.cpp.md) | The transform cache: three set matrices, four derived products, recomputed the moment an input moves |
| [`R_Backend_hemi.h`](R_Backend_hemi.h.md) · [`R_Backend_hemi.cpp`](R_Backend_hemi.cpp.md) | The per-object lighting channel: the ambient cube, the surface material selector, three free-form data slots |
| [`R_Backend_tree.h`](R_Backend_tree.h.md) · [`R_Backend_tree.cpp`](R_Backend_tree.cpp.md) | The wind channel: the eight named constants that bend a tree and unpack its quantized vertices |
| [`R_Backend_DBG.cpp`](R_Backend_DBG.cpp.md) | Immediate-mode debug geometry, and the overdraw visualisation |
| [`R_DStreams.h`](R_DStreams.h.md) · [`R_DStreams.cpp`](R_DStreams.cpp.md) | The two per-frame scratch rings every piece of generated geometry is written into |
| [`BufferUtils.h`](BufferUtils.h.md) | Vertex and index buffer ownership: four buffer kinds by host-copy and append-vs-discard, plus the layout description helpers |
| [`FVF.h`](FVF.h.md) | The six vertex layouts the engine's own immediate drawing uses, and the two conventions that make them work on both backends |
| [`r_constants.h`](r_constants.h.md) · [`r_constants.cpp`](r_constants.cpp.md) | The reflected shader interface: what a constant is, and the single integer that encodes every destination it occupies |
| [`r_constants_cache.h`](r_constants_cache.h.md) | Selects the backend's implementation of the constant write cache |
| [`stats_manager.h`](stats_manager.h.md) · [`stats_manager.cpp`](stats_manager.cpp.md) | Running totals of device memory by purpose and pool |

### Materials: templates, compilation, the resource registry

| File | Role |
|---|---|
| [`blenders/`](blenders/README.md) | The material templates themselves — one directory, its own page |
| [`Blender.h`](Blender.h.md) · [`Blender.cpp`](Blender.cpp.md) | The base behaviour every template shares: identity record, sort priority, strict back-to-front, and the rule that only the active backend may make one |
| [`Blender_CLSID.h`](Blender_CLSID.h.md) | **The frozen set of template identifiers the shipped game data names** |
| [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`Blender_Recorder.cpp`](Blender_Recorder.cpp.md) | **The material compiler**: the scratch context a template emits passes into, and the detail-texture convention |
| [`Blender_Recorder_R2.cpp`](Blender_Recorder_R2.cpp.md) | The programmable half: a pass is a pair of named programs, and every texture binding is resolved through their reflection data |
| [`Blender_Recorder_StandartBinding.cpp`](Blender_Recorder_StandartBinding.cpp.md) | **The published constant-name vocabulary**: every name a shipped shader may spell, and the producer behind each |
| [`tss.h`](tss.h.md) | The recording front end a pass writes its state through, including the old fixed-function combiner's argument rules |
| [`tss_def.h`](tss_def.h.md) · [`tss_def.cpp`](tss_def.cpp.md) | The recorded state list: a flat accumulated set, then its translation into device state objects |
| [`Shader.h`](Shader.h.md) · [`Shader.cpp`](Shader.cpp.md) | The compiled material tree — shader, element, pass, bound lists — and the equality rules that let identical ones be shared |
| [`ShaderResourceTraits.h`](ShaderResourceTraits.h.md) | The per-stage compilation table: source name, profile, entry point, creation call — and the one create-or-reuse routine that drives all six |
| [`ResourceManager.h`](ResourceManager.h.md) | **The single resource registry**: everything interned by name or by value, so identical things are one thing |
| [`ResourceManager.cpp`](ResourceManager.cpp.md) | Material name plus texture names to a shader: six element variants, interned |
| [`ResourceManager_Loader.cpp`](ResourceManager_Loader.cpp.md) | Reading the shipped material library — animator definitions and template parameter blocks — and tearing it down |
| [`ResourceManager_Resources.cpp`](ResourceManager_Resources.cpp.md) | The interning primitives: one create/delete pair per resource kind |
| [`ResourceManager_Reset.cpp`](ResourceManager_Reset.cpp.md) | The device-lost bracket: what every device-owned resource releases, and the order it is all rebuilt in |
| [`ResourceManager_Scripting.cpp`](ResourceManager_Scripting.cpp.md) | Material definitions that ship as scripts: the declarative surface, and how a named function becomes a compiled pass list |
| [`SH_Atomic.h`](SH_Atomic.h.md) · [`SH_Atomic.cpp`](SH_Atomic.cpp.md) | The indivisible device resources a pass is built from, and the rule that each unregisters itself as it dies |
| [`SH_Constant.h`](SH_Constant.h.md) · [`SH_Constant.cpp`](SH_Constant.cpp.md) | An animated colour constant: four waveforms, evaluated at most once per frame |
| [`SH_Matrix.h`](SH_Matrix.h.md) · [`SH_Matrix.cpp`](SH_Matrix.cpp.md) | An animated texture-coordinate matrix: scroll, rotate, scale, and the two camera-derived reflection projections |
| [`SH_RT.h`](SH_RT.h.md) | A render target: named, interned, device-owned, with the lifecycle every such surface shares |
| [`SH_Texture.h`](SH_Texture.h.md) · [`SH_Texture.cpp`](SH_Texture.cpp.md) | A texture as the renderer sees it: a name, whatever kind of source it denotes, and a binding function chosen once |

### Textures and their authoring conventions

| File | Role |
|---|---|
| [`Texture.cpp`](Texture.cpp.md) | Name to device texture: where the file is looked for, how many top mips are dropped, and how a missing bump map is synthesized in the exact packing the shipped shaders expect |
| [`TextureDescrManager.h`](TextureDescrManager.h.md) · [`TextureDescrManager.cpp`](TextureDescrManager.cpp.md) | **The texture-description database**: which detail texture, bump map, surface material and parallax setting belong to a base texture |
| [`ETextureParams.h`](ETextureParams.h.md) · [`ETextureParams.cpp`](ETextureParams.cpp.md) | The per-texture authoring sidecar that ships beside the image and says what the texture *means* |
| [`ColorMapManager.h`](ColorMapManager.h.md) · [`ColorMapManager.cpp`](ColorMapManager.cpp.md) | The two colour-grading lookup slots, and the rule that a grading texture never leaves memory |

### Models

| File | Role |
|---|---|
| [`FBasicVisual.h`](FBasicVisual.h.md) · [`FBasicVisual.cpp`](FBasicVisual.cpp.md) | What every drawable model is, the header every model file starts with, and the rule that a duplicated model shares its original's geometry |
| [`FVisual.h`](FVisual.h.md) · [`FVisual.cpp`](FVisual.cpp.md) | The plain static model, with its optional position-only twin for depth-only passes |
| [`FHierrarhyVisual.h`](FHierrarhyVisual.h.md) · [`FHierrarhyVisual.cpp`](FHierrarhyVisual.cpp.md) | The container model, and the ownership question that distinguishes its two load forms |
| [`FProgressive.h`](FProgressive.h.md) · [`FProgressive.cpp`](FProgressive.cpp.md) | Level of detail without swapping meshes: choosing a detail level is choosing a draw range |
| [`FLOD.h`](FLOD.h.md) · [`FLOD.cpp`](FLOD.cpp.md) | The far-distance imposter: eight authored billboards and the coverage factor that switches to them |
| [`FSkinned.h`](FSkinned.h.md) · [`FSkinned.cpp`](FSkinned.cpp.md) | The skinned model: repacking to the device layout its bone count calls for, and the queries that need a creature's triangles where the animation just put them |
| [`FSkinnedTypes.h`](FSkinnedTypes.h.md) | The eight skinned vertex layouts, and the packing that hides bone indices and weights in the tangent frame's spare channels |
| [`FTreeVisual.h`](FTreeVisual.h.md) · [`FTreeVisual.cpp`](FTreeVisual.cpp.md) | The wind-animated model: quantized vertices, lighting as scale and bias, wind as four shared constants |
| [`ModelPool.h`](ModelPool.h.md) · [`ModelPool.cpp`](ModelPool.cpp.md) | The model cache: one loaded copy per file, cheap clones, and a free list so a dead clone is reused |
| [`IRenderDetailModel.h`](IRenderDetailModel.h.md) | The renderer-side shape of one grass or debris model and its single operation |

### Skeletons and animation

| File | Role |
|---|---|
| [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`SkeletonCustom.cpp`](SkeletonCustom.cpp.md) | Loading a skeleton from the shipped model file, plus the two things a posed skeleton does for the game: skinned decals and bone-accurate picking |
| [`SkeletonRigid.cpp`](SkeletonRigid.cpp.md) | The pose solve: the two-level rate limit, the parent-before-child walk, and the periodic bounding-volume rebuild |
| [`SkeletonAnimated.h`](SkeletonAnimated.h.md) · [`SkeletonAnimated.cpp`](SkeletonAnimated.cpp.md) | The animation player: motion banks, a fixed blend pool across body parts and channels, and the sample-dequantize-mix step |
| [`Animation.h`](Animation.h.md) · [`Animation.cpp`](Animation.cpp.md) | The four mixing channels and the fixed rule that says how each combines: channels 0 and 1 replace, 2 and 3 layer |
| [`AnimationKeyCalculate.h`](AnimationKeyCalculate.h.md) | The per-bone animation mathematics: quantized key to pose, several poses to one, and the channel fold |
| [`KinematicAnimatedDefs.h`](KinematicAnimatedDefs.h.md) | The four fixed capacities of the animation blender, and why each is the number it is |
| [`KinematicsAddBoneTransform.hpp`](KinematicsAddBoneTransform.hpp.md) | A standing per-bone offset the game layer pins on top of whatever the animation produced |
| [`SkeletonX.h`](SkeletonX.h.md) · [`SkeletonX.cpp`](SkeletonX.cpp.md) | The skinned sub-mesh: the load-time decision of who deforms it, and both paths |
| [`SkeletonXSkinXW.h`](SkeletonXSkinXW.h.md) | The software skinning entry points, one per influence count |
| [`SkeletonXSkinXW_CPP.cpp`](SkeletonXSkinXW_CPP.cpp.md) | Software skinning, portable form, writing straight into device-mapped memory |
| [`SkeletonXSkinXW_SSE.cpp`](SkeletonXSkinXW_SSE.cpp.md) | The same routines as 4-wide float work, and the byte-layout assumptions that makes |
| [`SkeletonXVertRender.h`](SkeletonXVertRender.h.md) | The layout software skinning writes, and its one decision: tangent frames do not survive the processor path |

### Load-time mesh optimization

| File | Role |
|---|---|
| [`VertexCache.h`](VertexCache.h.md) · [`VertexCache.cpp`](VertexCache.cpp.md) | A model of the hardware's post-transform vertex cache, used to score candidate triangle orderings |
| [`NvTriStrip.h`](NvTriStrip.h.md) · [`NvTriStrip.cpp`](NvTriStrip.cpp.md) | The stripifier's front door: four tuning knobs, triangle list to cache-ordered groups, and the vertex renumbering |
| [`NvTriStripObjects.h`](NvTriStripObjects.h.md) · [`NvTriStripObjects.cpp`](NvTriStripObjects.cpp.md) | The stripifier itself: adjacency, competitive strip growth, chop to cache size, order for reuse, flatten with winding preserved |
| [`xrStripify.h`](xrStripify.h.md) · [`xrStripify.cpp`](xrStripify.cpp.md) | The engine's wrapper: what callers ask for and what the reordered index stream guarantees |

### Detail objects — the grass layer

| File | Role |
|---|---|
| [`DetailFormat.h`](DetailFormat.h.md) | **The frozen on-disk format**: a 2-metre grid, four models per cell, quantized density and baked lighting in sixteen bytes |
| [`DetailManager.h`](DetailManager.h.md) · [`DetailManager.cpp`](DetailManager.cpp.md) | The layer's frame: pick visible plants off a background thread, fade by distance, hand three batched lists to a draw path |
| [`DetailManager_CACHE.cpp`](DetailManager_CACHE.cpp.md) | The sliding window of decompressed cells, rotated a row at a time, refilled at a fixed budget per frame |
| [`DetailManager_Decompress.cpp`](DetailManager_Decompress.cpp.md) | Density map to actual plants: dither for placement, ray-cast down for the ground, seed yaw and scale from the cell's own coordinates |
| [`DetailManager_VS.cpp`](DetailManager_VS.cpp.md) | The hardware path's static geometry: each model pre-replicated to fill one draw's constants, each copy stamped with its constant index |
| [`DetailManager_soft.cpp`](DetailManager_soft.cpp.md) | The fallback path: transform every plant on the processor into the shared dynamic stream, in stall-free chunks |
| [`DetailModel.h`](DetailModel.h.md) · [`DetailModel.cpp`](DetailModel.cpp.md) | One grass model: loading it, optimizing its triangle order, and stamping transformed copies into a shared buffer |

### Particles

| File | Role |
|---|---|
| [`PSLibrary.h`](PSLibrary.h.md) · [`PSLibrary.cpp`](PSLibrary.cpp.md) | The particle library: every shipped effect and group definition, loaded once from one packed file and looked up by name |
| [`ParticleEffectDef.h`](ParticleEffectDef.h.md) · [`ParticleEffectDef.cpp`](ParticleEffectDef.cpp.md) | The authored effect definition: its frozen record, the atlas frame convention, and the two behaviours the renderer adds to the simulator's action list |
| [`ParticleEffect.h`](ParticleEffect.h.md) · [`ParticleEffect.cpp`](ParticleEffect.cpp.md) | One playing instance: a fixed 33 ms step, a bounding volume kept current for the visibility walk, and particles as oriented textured quads |
| [`ParticleGroup.h`](ParticleGroup.h.md) · [`ParticleGroup.cpp`](ParticleGroup.cpp.md) | The authored timeline that starts and stops several effects at named times, its runtime, and its merged bounding volume |
| [`dxParticleCustom.h`](dxParticleCustom.h.md) · [`dxParticleCustom.cpp`](dxParticleCustom.cpp.md) | The renderable form of a particle effect: a visual that is also a particle-system instance |

### Lights and shadow budget

| File | Role |
|---|---|
| [`light.h`](light.h.md) · [`light.cpp`](light.cpp.md) | One light: its shape, its spatial-database entry, the transform from unit primitive to volume, and the split of a shadowing point light into six cones |
| [`light_vis.cpp`](light_vis.cpp.md) | Whether a light is worth drawing, by occlusion query — but not every frame, and not when the camera is inside it |
| [`light_smapvis.h`](light_smapvis.h.md) · [`light_smapvis.cpp`](light_smapvis.cpp.md) | The per-light cache of which casters produce no visible shadow, learned one candidate per frame |
| [`light_gi.h`](light_gi.h.md) · [`light_gi.cpp`](light_gi.cpp.md) | Indirect bounce lights: photons into the static world, strongest hits kept, total energy normalized to a budget |
| [`Light_DB.h`](Light_DB.h.md) · [`Light_DB.cpp`](Light_DB.cpp.md) | The level's light set, picking the sun out of it, and putting the sun where the weather says |
| [`Light_Package.h`](Light_Package.h.md) · [`Light_Package.cpp`](Light_Package.cpp.md) | This frame's visible lights in three buckets, ordered so unanswered queries go last and the rest brightest-first |
| [`Light_Render_Direct_ComputeXFS.cpp`](Light_Render_Direct_ComputeXFS.cpp.md) | **The shadow-budget decision**: how much of the atlas a shadowing spot light deserves, and what camera it is rendered from |
| [`Light_Render_Direct.h`](Light_Render_Direct.h.md) · [`Light_Render_Direct.cpp`](Light_Render_Direct.cpp.md) | A vestige: one declared operation with no implementation anywhere |
| [`LightTrack.h`](LightTrack.h.md) · [`LightTrack.cpp`](LightTrack.cpp.md) | The per-object lighting estimate: sky rays, a sun ray, tracked lights, the ambient cube, and the schedule that drops to twice a minute for anything still |
| [`r_sun_cascades.h`](r_sun_cascades.h.md) | The per-cascade record of the sun's shadow: projection, defining rays, size, depth bias |

### Wallmarks (decals)

| File | Role |
|---|---|
| [`WallmarksEngine.h`](WallmarksEngine.h.md) · [`WallmarksEngine.cpp`](WallmarksEngine.cpp.md) | How a decal is cut out of the collision geometry, batched by material, and faded out |
| [`dxWallMarkArray.h`](dxWallMarkArray.h.md) · [`dxWallMarkArray.cpp`](dxWallMarkArray.cpp.md) | A surface material's decal variants, and the uniform random pick from them |

### Device lifecycle, capabilities, console

| File | Role |
|---|---|
| [`D3DXRenderBase.h`](D3DXRenderBase.h.md) · [`D3DXRenderBase.cpp`](D3DXRenderBase.cpp.md) | The half of the renderer interface every backend shares: device lifetime, the frame bracket across every context, gamma, resources, the parallel context pool |
| [`HWCaps.h`](HWCaps.h.md) | The capability record: what the renderer may assume about the hardware it found |
| [`xrRender_console.h`](xrRender_console.h.md) · [`xrRender_console.cpp`](xrRender_console.cpp.md) | **The renderer's console variables**: every quality, threshold and debug knob, with its range and default — a frozen, user-facing name set |
| [`xr_effgamma.h`](xr_effgamma.h.md) · [`xr_effgamma.cpp`](xr_effgamma.cpp.md) | The gamma / brightness / contrast ramp and where it is applied |

### Debug drawing and the shape toolbox

| File | Role |
|---|---|
| [`D3DUtils.h`](D3DUtils.h.md) · [`D3DUtils.cpp`](D3DUtils.cpp.md) | The shape-drawing toolbox: preloaded primitive meshes, three dynamic layouts, and the several dozen debug shapes built on them |
| [`du_box.h`](du_box.h.md) · [`du_box.cpp`](du_box.cpp.md) | The unit cube, in indexed-solid, indexed-wire and unshared-vertex forms |
| [`du_cone.h`](du_cone.h.md) · [`du_cone.cpp`](du_cone.cpp.md) | The unit cone: apex at the origin, sixteen-sided cap one unit along the axis |
| [`du_cylinder.h`](du_cylinder.h.md) · [`du_cylinder.cpp`](du_cylinder.cpp.md) | The unit cylinder: twelve-sided, capped both ends |
| [`du_sphere.h`](du_sphere.h.md) · [`du_sphere.cpp`](du_sphere.cpp.md) | The unit sphere's two meshes — a subdivided icosahedron for solids, three great circles for wire |
| [`du_sphere_part.h`](du_sphere_part.h.md) · [`du_sphere_part.cpp`](du_sphere_part.cpp.md) | The sphere wedge used to draw a cone of vision or an audible arc as a solid |
| [`dxDebugRender.h`](dxDebugRender.h.md) · [`dxDebugRender.cpp`](dxDebugRender.cpp.md) | The debug-drawing port's filling: a batching line accumulator plus routing |
| [`dxStatGraphRender.h`](dxStatGraphRender.h.md) · [`dxStatGraphRender.cpp`](dxStatGraphRender.cpp.md) | A profiler graph as two untextured batches in screen coordinates |
| [`dxObjectSpaceRender.h`](dxObjectSpaceRender.h.md) · [`dxObjectSpaceRender.cpp`](dxObjectSpaceRender.cpp.md) | The debug view of the collision world |
| [`Debug/`](Debug/README.md) | Debug-only helpers — its own page |
| [`Utils/`](Utils/README.md) | Small shared helpers — its own page |

### Port fillings: the renderer's side of the engine's interfaces

| File | Role |
|---|---|
| [`dxRenderFactory.h`](dxRenderFactory.h.md) · [`dxRenderFactory.cpp`](dxRenderFactory.cpp.md) | The one object that instantiates each "renderer companion" and frees it back into this module's allocator |
| [`dxEnvironmentRender.h`](dxEnvironmentRender.h.md) · [`dxEnvironmentRender.cpp`](dxEnvironmentRender.cpp.md) | Sky and weather: the two-layer cross-faded sky box, the cloud dome, and the blend between two times of day |
| [`dxFontRender.h`](dxFontRender.h.md) · [`dxFontRender.cpp`](dxFontRender.cpp.md) | Queued text lines to quads, key-binding placeholders expanded, in as few batches as the scratch buffer allows |
| [`dxUIRender.h`](dxUIRender.h.md) · [`dxUIRender.cpp`](dxUIRender.cpp.md) | The vertex sink every two-dimensional thing in the game draws through |
| [`dxUIShader.h`](dxUIShader.h.md) · [`dxUIShader.cpp`](dxUIShader.cpp.md) | An opaque material handle for callers that must not know what a material is |
| [`dxUISequenceVideoItem.h`](dxUISequenceVideoItem.h.md) · [`dxUISequenceVideoItem.cpp`](dxUISequenceVideoItem.cpp.md) | Grabbing the bound material's base texture so a scripted sequence can drive its playback |
| [`dxImGuiRender.h`](dxImGuiRender.h.md) · [`dxImGuiRender.cpp`](dxImGuiRender.cpp.md) | The debug-overlay port: hand the toolkit the device and let its own backend draw |
| [`dxLensFlareRender.h`](dxLensFlareRender.h.md) · [`dxLensFlareRender.cpp`](dxLensFlareRender.cpp.md) | One camera-facing quad per flare element, all in one buffer lock, drawn a material at a time |
| [`dxRainRender.h`](dxRainRender.h.md) · [`dxRainRender.cpp`](dxRainRender.cpp.md) | The rain drop population as streak quads, and splashes as instanced copies of one mesh |
| [`dxThunderboltRender.h`](dxThunderboltRender.h.md) · [`dxThunderboltRender.cpp`](dxThunderboltRender.cpp.md) | One lightning strike: the bolt mesh with an animated texture shift, plus two glow quads tracking the flash |
| [`dxThunderboltDescRender.h`](dxThunderboltDescRender.h.md) · [`dxThunderboltDescRender.cpp`](dxThunderboltDescRender.cpp.md) | The one mesh a lightning-bolt description draws with |
