# `src/Layers/xrRender/blenders/` — the material templates

Every surface the engine draws is drawn by one of the templates in this directory. The
parent chapter ([`../README.md`](../README.md)) establishes what a *blender* is — a
parameterized material template loaded from data, which compiles to a concrete pass list,
and whose g-buffer-writing members mark stencil under a high-bit-preserving mask that the
lighting resolve tests. This chapter is the catalogue: eighty-odd templates, what each one
draws, and the four contracts they all obey.

Read it as a data-format chapter, not as rendering code. The interesting decisions here
are not "how is a wet road shaded" — that lives in the shipped shader source, which the
engine never writes — but *which pass list does a material with these textures and these
flags produce, on this renderer generation, for this phase of the frame*. The templates
are pure functions from (parameters, context) to a pass list, and they are the only place
where the shipped material library meets the frame graph.

## Where it sits

This is a sub-directory of chapter 18, and it rests on exactly three things from its
parent: the template base and its identity record ([`../Blender.cpp`](../Blender.cpp.md)),
the recorder that a template writes its pass list into
([`../Blender_Recorder.cpp`](../Blender_Recorder.cpp.md)), and the frozen list of class
identifiers ([`../Blender_CLSID.h`](../Blender_CLSID.h.md)). It reaches nothing else. It
reaches the graphics device not at all: a template names programs, textures, samplers and
state, and the recorder resolves those names through the resource registry.

Chapters 19–21 consume it. Each backend owns a table from class identifier to template,
and **the tables differ** — the same shipped material names `"LM      "` and gets the
forward lightmap stack on the old renderer and a single g-buffer write on the modern ones.
Several files here exist in two spellings, one per generation, and the build selects which
one is compiled; those pairs are noted in the tables below.

## Load-bearing ideas, named once

**A template's interface is four things.** A template declares (1) a set of named, typed,
editor-visible parameters — an alpha reference, a texture name, a blend mode, a
tessellation selector; (2) three capability answers, asked before compilation, saying
whether this kind of surface may take a detail texture, a baked lightmap, or a
height-displaced parallax layer; (3) a versioned save/load pair for its parameter block;
and (4) the compile entry point. Nothing else is public, and a template may not reach past
the recorder to touch anything.

**The parameter block is a wire format, and it is frozen.** A material in the shipped
library is an eight-byte class identifier, an identity record, a version number, and a
run of bytes that only the named template knows how to read. The version belongs to the
*template*, not to the file: each one carries its own, bumps it when it gains a parameter,
and its loader must still accept every earlier layout it ever shipped. A loader that reads
the current layout from an older record consumes bytes belonging to the next material and
corrupts the rest of the library — there is no length prefix to resynchronize on. Several
templates in this directory are on their second or third version for exactly this reason.

The blocks are written as *tagged property lists*: a marker, then the value, and for an
enumerated parameter also the full list of `(value, label)` pairs so an authoring tool can
present a menu without knowing the enumeration. Those labels are dead weight to a running
game, which reads only the selected index — but they occupy bytes in the shipped library,
so a rebuild that writes a material must write them too.

**The naming contract.** A material names its template by an eight-character tag,
space-padded to exactly eight, packed into one integer. The set of tags is the frozen list
in [`../Blender_CLSID.h`](../Blender_CLSID.h.md). **A rebuild must register the same
identifiers, spelled and padded the same way, or the shipped material library will not
load.** This is not a style point: a tag that differs by one trailing space matches
nothing, the material resolves to no template, and the level fails. Two consequences
follow:

- Some templates exist *only* to answer a tag. Their composition is dead — no art in any
  shipped level still uses them — but the tag is in the list, the library contains records
  that name it, and something must consume those bytes. The lightmap-plus-environment
  template is the clearest case.
- The mapping is **per backend**, and a backend is allowed to answer a tag with *nothing*.
  On the deferred renderers, the screen-desaturation, blur, shadow-projection and
  forward-light tags all resolve to nothing, because the deferred frame graph does those
  jobs itself. A material naming one of those tags simply produces no passes, and the
  draw stream skips it.

**Class identifier zero means "never named by data."** Roughly a quarter of the templates
here are not materials at all: they are the deferred frame graph's own passes — light
accumulation, the deferred resolve, bloom, luminance reduction, ambient occlusion, the
multisample edge marker, the wet-surface chain — written as templates so that they go
through the same recorder, the same program-name resolution and the same state interning
as an art material. They are constructed directly by the render-target setup, never looked
up by tag, and they carry identifier zero to say so. A rebuild may express these as plain
pass descriptions instead; what it may not do is give them a tag, because then a hostile
or mistaken material could name one.

**`Compile` is a selector, not a method per pass.** One template compiles differently
depending on which *element* of the frame graph is asking. The element is an index handed
to compile, and the sixth-slot layout is frozen because the shipped material scripts fill
slots by index and the draw code selects by index:

| Element | Forward generation | Deferred generations |
|---|---|---|
| 0 | normal, high detail | g-buffer fill, high detail (parallax and detail textures live) |
| 1 | normal, low detail | the same surface without the expensive extras, or the forward detour |
| 2 | additive pass: point light | shadow-map generation |
| 3 | additive pass: spot light | shadow-map generation |
| 4 | lighting for models | directional shadow-map generation |
| 5 | unused | unused |

This is why there is one template per *kind of surface* and not one per pass. A wall is one
material; the frame asks it for element 0 when filling the g-buffer, element 2 when filling
a shadow atlas, and gets two different pass lists from the same parameters. **A template
that does not answer an element emits nothing, and that is meaningful** — it is how a
blended surface says "I cast no shadow" and how a forward-only material says "skip me in
the deferred pass."

The screen-space and internal templates reuse the element index for their own enumerations
entirely: a light-accumulation template's elements are its shadowing modes (fill,
unshadowed, shadowed-scaled, shadowed-fullsize, translucent-masked); a stencil-mask
template's elements are which light shape it is bounding; a bloom template's elements are
the five steps of the separable blur. The index is a small integer whose meaning belongs to
the template, and only the world- and model-surface family shares the table above.

**Level-of-detail is a parameter of the context, not of the material.** Whether detail
texturing is on, whether the device has tessellation, whether multisampling resolves
cut-outs through alpha-to-coverage or through a hard alpha test, which renderer generation
is running — all of these arrive as facts about the context, and every template branches on
them inline. The result is that the same material produces a different, cheaper pass list
on weaker hardware, with no second material in the data.

**The forward detour.** The single most-repeated decision in this directory: a g-buffer
stores one opaque sample per pixel, so a surface that is genuinely translucent, or that the
artist marked as needing painter's-order sorting, cannot live there. Every deferred surface
template asks the same question first — *can this be written into the g-buffer at all?* —
and detours to a forward-blended, approximately-lit pass when the answer is no. The test is
the same everywhere: a material that uses its alpha channel with a *low* cut threshold
wanted fade, not cut-out, and a material that asked for strict back-to-front ordering is by
definition not resolvable by a depth buffer.

**Texture names decide the shader.** For the deferred surfaces, a pass is not chosen by a
flag but derived from the *names* of the textures a material happens to reference — a
matching bump map, a detail map, a height channel each add a suffix to the program name the
pass asks for. This is the shared builder in [`uber_deffer.cpp`](uber_deffer.cpp.md), and it
is the reason the shipped art can gain a bump map by adding a file with the right name and
no material edit. It also means the set of shader programs a rebuild must be able to
resolve is a product of naming conventions, not a list.

---

## World surfaces — the forward generation

The oldest renderer's fillings. Baked lightmaps, fixed-function texture stages, and an
additive pass per dynamic light. Where a file is marked *forward only*, the build excludes
it from the modern backends and a different file answers the same tag there.

| File | Role |
|---|---|
| [`BlenderDefault.cpp`](BlenderDefault.cpp.md) · [`BlenderDefault.h`](BlenderDefault.h.md) | The workhorse: base texture modulated by a baked lightmap, plus the additive point- and spot-light passes |
| [`Blender_default_aref.cpp`](Blender_default_aref.cpp.md) · [`Blender_default_aref.h`](Blender_default_aref.h.md) | The same, alpha-tested, for cut-out geometry that still takes a lightmap |
| [`Blender_Vertex.cpp`](Blender_Vertex.cpp.md) · [`Blender_Vertex.h`](Blender_Vertex.h.md) | Geometry with no lightmap: lighting comes from the vertices themselves |
| [`Blender_Vertex_aref.cpp`](Blender_Vertex_aref.cpp.md) · [`Blender_Vertex_aref.h`](Blender_Vertex_aref.h.md) | The same, alpha-tested — grates, chain-link, foliage cards |
| [`Blender_BmmD.cpp`](Blender_BmmD.cpp.md) · [`Blender_BmmD.h`](Blender_BmmD.h.md) | Terrain: a lightmapped base plus a tiled "implicit detail" texture named by the material itself rather than resolved from the texture database *(forward only)* |
| [`Blender_LaEmB.cpp`](Blender_LaEmB.cpp.md) · [`Blender_LaEmB.h`](Blender_LaEmB.h.md) | Lightmap added to an environment reflection, both modulating the base; six hardware-fitting variants. Dead in the shipped data — kept because its tag is in the frozen list |
| [`Blender_Lm(EbB).cpp`](Blender_Lm%28EbB%29.cpp.md) · [`Blender_Lm(EbB).h`](Blender_Lm%28EbB%29.h.md) | A lightmapped surface whose base alpha reveals a reflection underneath — the world counterpart of the reflective model template. The one world template whose single implementation serves every generation, with the generation-specific elements compiled out |

## World surfaces — the deferred generations

The same tags, answered with g-buffer writes. Every one of these marks stencil under the
mask that leaves the multisample edge bit alone, and every one emits a depth-only element
for shadow-map filling.

| File | Role |
|---|---|
| [`blender_deffer_flat.cpp`](blender_deffer_flat.cpp.md) · [`blender_deffer_flat.h`](blender_deffer_flat.h.md) | The lightmapped-diffuse tag reduced to one g-buffer write and one shadow-map write; also answers the vertex-lit tag |
| [`blender_deffer_aref.cpp`](blender_deffer_aref.cpp.md) · [`blender_deffer_aref.h`](blender_deffer_aref.h.md) | The alpha-tested world surface, with two lives: a g-buffer write when it is cut out, a forward-blended pass when it is translucent |
| [`Blender_BmmD_deferred.cpp`](Blender_BmmD_deferred.cpp.md) | Terrain filled for the deferred path: a four-way ground blend driven by a mask texture derived from the base texture's name |
| [`uber_deffer.cpp`](uber_deffer.cpp.md) · [`uber_deffer.h`](uber_deffer.h.md) | **The shared g-buffer pass builder**: derives which shader variant, which textures and which sampler settings a pass needs from the *names* of the textures the material references |

## Dynamic models

| File | Role |
|---|---|
| [`Blender_Model.cpp`](Blender_Model.cpp.md) · [`Blender_Model.h`](Blender_Model.h.md) | The forward default for dynamic models: base texture lit by the per-object light projector, optional alpha blending, plus the additive-light and shadow-casting elements |
| [`blender_deffer_model.cpp`](blender_deffer_model.cpp.md) · [`blender_deffer_model.h`](blender_deffer_model.h.md) | The template every animated model in the game is drawn with on the modern backends: the forward-detour decision, the g-buffer write, and the shadow-map variant |
| [`Blender_Model_EbB.cpp`](Blender_Model_EbB.cpp.md) · [`Blender_Model_EbB.h`](Blender_Model_EbB.h.md) | A model with an environment reflection: the base texture's own alpha decides, per texel, how much reflection shows through |
| [`Blender_Model_EbB_deferred.cpp`](Blender_Model_EbB_deferred.cpp.md) | The same tag on the deferred path: the reflection is dropped, the model becomes a g-buffer write, and only the blended case stays forward |

## Procedurally placed geometry

Grass, debris and foliage are not authored objects; they are placed by the engine from a
density map and animated in the vertex stage. Their templates are the only ones whose
vertex program is load-bearing to the *material* rather than to the mesh.

| File | Role |
|---|---|
| [`Blender_detail_still.cpp`](Blender_detail_still.cpp.md) · [`Blender_detail_still.h`](Blender_detail_still.h.md) | The detail-object layer — grass and debris — drawn with a vertex program that either sways it in the wind or holds it still |
| [`Blender_detail_still_deferred.cpp`](Blender_detail_still_deferred.cpp.md) | The same, deferred, where the cut-out edges resolve against multisample coverage instead of against a hard alpha test |
| [`Blender_tree.cpp`](Blender_tree.cpp.md) · [`Blender_tree.h`](Blender_tree.h.md) | Trees and bushes: an alpha-tested surface bent by a wind vertex program, with one flag that turns the wind off and repurposes the template as a distant-object impostor |
| [`Blender_tree_deferred.cpp`](Blender_tree_deferred.cpp.md) | Foliage deferred: a g-buffer write with an optional alpha-to-coverage prepass, and a shadow-map element with its own four vertex programs |
| [`Blender_Particle.cpp`](Blender_Particle.cpp.md) · [`Blender_Particle.h`](Blender_Particle.h.md) | The particle material: one texture, one of six named blend modes, and nothing else |
| [`Blender_Particle_deferred.cpp`](Blender_Particle_deferred.cpp.md) | The same six modes plus soft-particle depth fading, and a shadow-map element that turns every mode into a darkening pass |

## Screen-space and legacy effects

Tags the forward renderer implements and the deferred renderers deliberately answer with
nothing, because the deferred frame graph owns those jobs itself. The general-purpose
template is the exception and is alive on every backend.

| File | Role |
|---|---|
| [`Blender_Screen_SET.cpp`](Blender_Screen_SET.cpp.md) · [`Blender_Screen_SET.h`](Blender_Screen_SET.h.md) | **The general-purpose material**: every aspect of the pipeline state is a parameter. The shipped data uses it for screen overlays, user-interface art, decals and anything with no template of its own |
| [`Blender_Screen_GRAY.cpp`](Blender_Screen_GRAY.cpp.md) · [`Blender_Screen_GRAY.h`](Blender_Screen_GRAY.h.md) | Desaturation as a full-screen pass, computed on fixed-function hardware with a dot product against a luminance constant |
| [`Blender_Blur.cpp`](Blender_Blur.cpp.md) · [`Blender_Blur.h`](Blender_Blur.h.md) | A two-tap screen blur: two copies of the frame, each scaled by a constant, added |
| [`Blender_Shadow_Texture.cpp`](Blender_Shadow_Texture.cpp.md) · [`Blender_Shadow_Texture.h`](Blender_Shadow_Texture.h.md) | The material a model wears while its silhouette is rendered into a shadow texture: pure black, no texture, no depth |
| [`Blender_Shadow_World.cpp`](Blender_Shadow_World.cpp.md) · [`Blender_Shadow_World.h`](Blender_Shadow_World.h.md) | Projecting an already-rendered silhouette back onto the world by multiplying the frame where it is dark |
| [`blender_light.cpp`](blender_light.cpp.md) · [`blender_light.h`](blender_light.h.md) | The forward renderer's additive light: a two-stage fixed-function stack multiplying a 2D attenuation map by a 1D falloff by the light's colour |

## The deferred frame graph's own passes

Identifier zero — never named by data, constructed directly by the render-target setup.
These are the passes of chapter 19 expressed in the same vocabulary as an art material, so
that they intern their state and resolve their program names the same way.

| File | Role |
|---|---|
| [`blender_light_point.cpp`](blender_light_point.cpp.md) · [`blender_light_point.h`](blender_light_point.h.md) | Adding one omnidirectional light to the accumulator, in five flavours of increasing cost |
| [`blender_light_spot.cpp`](blender_light_spot.cpp.md) · [`blender_light_spot.h`](blender_light_spot.h.md) | Adding one cone light, plus the pass that makes its beam visible in the air |
| [`blender_light_direct.cpp`](blender_light_direct.cpp.md) · [`blender_light_direct.h`](blender_light_direct.h.md) | Adding the sun one shadow cascade at a time, plus the sample-resolved and volumetric variants |
| [`blender_light_direct_cascade.cpp`](blender_light_direct_cascade.cpp.md) · [`blender_light_direct_cascade.h`](blender_light_direct_cascade.h.md) | The sun for the alternate cascade scheme that keeps one shadow map per cascade instead of one shared atlas |
| [`blender_light_reflected.cpp`](blender_light_reflected.cpp.md) · [`blender_light_reflected.h`](blender_light_reflected.h.md) | Adding the one-bounce indirect term |
| [`blender_light_mask.cpp`](blender_light_mask.cpp.md) · [`blender_light_mask.h`](blender_light_mask.h.md) | Writing the stencil masks that bound each light, and copying the accumulator between its temporary and real targets |
| [`blender_light_occq.cpp`](blender_light_occq.cpp.md) · [`blender_light_occq.h`](blender_light_occq.h.md) | The invisible geometry occlusion queries are issued against, and the clearing of the stencil bits they use |
| [`blender_combine.cpp`](blender_combine.cpp.md) · [`blender_combine.h`](blender_combine.h.md) | **The deferred resolve**: reads every g-buffer channel plus the accumulated lighting and produces a lit image, in four variants compositing bloom and distortion on top |
| [`blender_bloom_build.cpp`](blender_bloom_build.cpp.md) · [`blender_bloom_build.h`](blender_bloom_build.h.md) | The bloom chain, whose five elements are the five steps of a separable blur into a half-resolution target, plus the final composite |
| [`blender_luminance.cpp`](blender_luminance.cpp.md) · [`blender_luminance.h`](blender_luminance.h.md) | The three-step reduction of the frame to one average-luminance texel, smoothed against the previous frame's answer |
| [`blender_ssao.cpp`](blender_ssao.cpp.md) · [`blender_ssao.h`](blender_ssao.h.md) | Screen-space ambient occlusion and the half-resolution depth target it reads |
| [`dx11HDAOCSBlender.cpp`](dx11HDAOCSBlender.cpp.md) · [`dx11HDAOCSBlender.h`](dx11HDAOCSBlender.h.md) | The two compute-shader ambient-occlusion passes |
| [`dx11MSAABlender.cpp`](dx11MSAABlender.cpp.md) · [`dx11MSAABlender.h`](dx11MSAABlender.h.md) | Finding the frame's geometric edges and marking them in the high stencil bit, so later passes shade edges per sample and everything else once |
| [`dx11MinMaxSMBlender.cpp`](dx11MinMaxSMBlender.cpp.md) · [`dx11MinMaxSMBlender.h`](dx11MinMaxSMBlender.h.md) | Building the min/max acceleration texture over the sun's shadow map |
| [`dx11RainBlender.cpp`](dx11RainBlender.cpp.md) · [`dx11RainBlender.h`](dx11RainBlender.h.md) | The four-step wet-surface effect: decide what the rain reaches, perturb those normals with animated water, raise their gloss |
| [`glMSAABlender.cpp`](glMSAABlender.cpp.md) | The edge-marking pass as the other backend binds it — same template, alternate compilation |
| [`glMinMaxSMBlender.cpp`](glMinMaxSMBlender.cpp.md) | The min/max reduction, alternate compilation |
| [`glRainBlender.cpp`](glRainBlender.cpp.md) | The wet-surface passes, alternate compilation |

## Authoring tools only

Registered by every backend, named by no shipped material. They exist because the level
editor draws its own overlays through the same material system as the game.

| File | Role |
|---|---|
| [`Blender_Editor_Wire.cpp`](Blender_Editor_Wire.cpp.md) · [`Blender_Editor_Wire.h`](Blender_Editor_Wire.h.md) | The flat-coloured line material wireframes and gizmos are drawn with |
| [`Blender_Editor_Selection.cpp`](Blender_Editor_Selection.cpp.md) · [`Blender_Editor_Selection.h`](Blender_Editor_Selection.h.md) | The translucent highlight drawn over a selected object |

---

A rebuild that wants only to *run* the shipped games needs the world-surface, model,
procedural-geometry and frame-graph families, plus the general-purpose template. It still
needs a loader for every parameter block in the frozen tag list, including the dead ones,
because the bytes are in the library whether anything draws them or not. The two editor
templates can be dropped with the authoring tools.

See [`SYSTEM-REQUIREMENTS.md` §5 Shaders](../../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
for why the programs these templates name are data and not code, and
[Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) for the
state vocabulary a pass is expressed in.
