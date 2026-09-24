# src/Include/xrRender

> The contract between the engine and its renderer. Every file here is a pure interface with no implementation; together they are what makes the graphics device a *pluggable* seam instead of a hard dependency.

## What this module is responsible for

This directory holds no code. It holds the set of interfaces that the engine, the game, the user interface and the editors are allowed to know about the renderer — and, running the other way, the set of obligations a renderer backend must discharge to be usable. Two backends ship against it: an OpenGL one that runs on every platform, and a Direct3D 11 one that runs on Windows. Neither appears anywhere in this directory, and nothing outside the renderer names either of them.

It is chapter 4 of [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order). It rests on almost nothing — the core types, the vector and matrix layer, and the shared animation data model — and everything from the engine upwards rests on it. A rebuilder designing their own renderer reads this chapter and the [graphics device seam](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device), and may ignore the two backend chapters entirely.

## The shape of the boundary: a scene and a camera, not draw calls

The single most important thing to understand before reading any twin here is where the line is drawn, because it is drawn much higher than most engine/renderer splits.

**The engine does not tell the renderer what to draw. It tells the renderer what exists, and then asks for a frame.**

A frame runs like this:

```text
FUNCTION render_one_frame()
  render.begin()                      # acquire the back buffer, clear, start the device frame

  render.calculate()
    # The renderer walks its OWN scene structure: the level's sector and portal
    # graph, clipped against the camera frustum, admitting sectors whose
    # portal-clipped frustum is non-empty. For each dynamic object it reaches,
    # it calls back into the engine:
    #
    #     object.renderable_render(context, root)
    #       -> render.add_visual(context, root, visual, transform)
    #
    # The object's only job in that callback is to hand over its visual and its
    # world transform, possibly several of them (a creature plus its weapon plus
    # the weapon's attachments). It performs no culling and issues no draws.
    #
    # The renderer then has a complete scene and does everything else itself:
    # occlusion testing, shadow-caster selection, light gathering, material
    # sorting, level-of-detail choice, skinning, and the whole pass structure.

  render.render()                     # execute the frame: every pass, in the renderer's order
  render.render_menu()                # the 2D layers, through the UI vertex sink
  render.end()                        # present
```

The consequences of drawing the line here are the whole design:

- **The engine owns no graphics state.** There is no "bind this texture", no "set this blend mode", no draw call anywhere outside the renderer and the two narrow immediate interfaces ([`UIRender.h`](UIRender.h.md) and [`DrawUtils.h`](DrawUtils.h.md)) that exist for two-dimensional and debug drawing.
- **The renderer owns visibility.** Sectors, portals, occlusion queries and the software occluder pass are all inside it. The engine's objects carry a [visibility record](RenderVisual.h.md) the renderer writes into, and that is the extent of their involvement.
- **The renderer owns the frame's structure.** Deferred or forward, how many shadow cascades, whether there is a depth pre-pass — none of that reaches the engine. This is why the two shipped backends can differ in generation (an older forward path and a newer deferred one) behind one interface.
- **The interface is therefore wide, not deep.** It is a *frame graph* boundary: the engine hands over a scene and a camera and gets back a presented frame. A draw-call boundary would be narrower and would put every one of the decisions above on the wrong side.

Where this leaves a rebuilder: the seam is worth preserving exactly as it is. It is the reason a renderer can be replaced without touching the game, and it is unusually clean for an engine of this vintage.

## The load-bearing ideas, named once

**A "shader" here is a material pass description, not a program.** Throughout this directory and the whole engine, a shader is a named entry in the game's material data listing passes, blend and depth state, texture bindings and sampler settings. The device programs it eventually compiles are a separate, lower thing. Interfaces here that create a "shader" from a name are resolving material data.

**Every model is an opaque handle.** The engine can create one by name, duplicate it, destroy it, position it, and ask it three capability questions ([has a skeleton](Kinematics.h.md), [plays animations](KinematicsAnimated.h.md), [is a particle emitter](ParticleCustom.h.md)). It can never see inside one. See [`RenderVisual.h`](RenderVisual.h.md).

**The renderer-side companion pattern.** Most engine objects that need device resources — a font, a weather keyframe, a lens flare element, a lightning bolt definition, a UI element — own a small renderer-side object that holds those resources and does the drawing. Those companions all come from one [factory](RenderFactory.h.md) and are all held through one [ownership policy](FactoryPtr.h.md). That is why almost every interface here declares a `copy` method and a device-create/device-destroy pair: the first is the policy's duplication hook, the second is the device-reset protocol.

**Device create, device destroy, device reset.** Three distinct events, and the companions distinguish them. Create and destroy bracket the device's whole life. A *reset* — a resolution change, a fullscreen toggle — destroys the device-side resources while the engine-side state survives; only [`ImGuiRender.h`](ImGuiRender.h.md) names the reset explicitly, and the others handle it by being destroyed and recreated. A rebuild on an API without a lost-device concept still needs the bracket, because a swap-chain resize invalidates render targets on every API.

**Skeletons are the exception to "opaque".** [`Kinematics.h`](Kinematics.h.md) is a wide interface because the physics layer, the inverse-kinematics layer, the weapon layer, the damage layer and the script layer all need to read and write individual bones. That is an inversion of the rest of the directory and it is unavoidable: the skeleton is where the renderer's data and the simulation's data are the same data.

## Which parts are Direct3D 9 heritage, and what a rebuild should redraw

The original engine shipped in 2007 against a fixed-function-era API, and several of its concepts are still visible in this directory. They are worth naming, because a rebuilder who does not recognize them will faithfully reimplement things that no longer mean anything.

**Pre-transformed vertices.** [`UIRender.h`](UIRender.h.md) offers two vertex kinds, and the first is "already in screen pixels, do not transform". That is the fixed-function transformed-and-lit vertex, which no current API has. It is easy to emulate with an orthographic projection, and a rebuild should do exactly that and keep only one vertex kind — but it must match the pixel-centre convention, or every sharp UI edge blurs by half a pixel.

**The alpha reference.** A per-draw alpha-test cutoff, set as renderer state. Alpha testing as fixed-function state was removed a generation later; it is a comparison in a shader now. Keep the *concept* (UI art relies on cutting, not blending) and move it into the material.

**Ambient colour as renderer state.** [`DebugRender.h`](DebugRender.h.md) sets a global ambient. That was a fixed-function lighting register. In a rebuild it is a constant in a debug material.

**Cull mode as a three-valued global.** Off / clockwise / counter-clockwise, set outside the material. Modern state objects bundle it with the rest of the rasterizer state; the interfaces here set it on its own because it used to be its own register. Fold it into the material and lose two methods.

**Device lost and device reset as a first-class lifecycle.** The renderer interface proper carries a device-state query with a *lost* value and a begin/end reset bracket. Only one API ever had a genuinely lost device. A rebuild keeps the bracket for swap-chain resizes and drops the lost state.

**Renderer "generations" numbered by the original's history.** The mode identifiers a backend advertises are small integers counting the original engine's renderer versions, and the interface still asks a backend which generation it is so that engine code can branch on "at least the deferred one". The numbers are frozen by shipped configuration files (see [`xrRender.h`](xrRender.h.md)) and must be kept; the *branching* on them should not be. A rebuild should replace generation checks with capability checks.

**The occlusion fields in the visibility record.** [`RenderVisual.h`](RenderVisual.h.md) exposes per-visual bookkeeping for a hierarchical occlusion map — a software rasterizer that predates hardware occlusion queries. Both mechanisms are still present. A rebuild picks one.

**Build-configuration-dependent interface shape.** Several interfaces here add or remove methods depending on whether the build is a debug build or an editor build. Because these are interfaces crossing a dynamically loaded module boundary, that changes the binary layout of the call table: **a debug engine cannot load a release renderer.** This is a genuine hazard, not a style point. A rebuild should keep the interface shape constant and gate the debug methods at run time.

**The `copy` method on fifteen interfaces.** An artefact of the value-semantics [ownership policy](FactoryPtr.h.md). A rebuild with a different ownership model deletes all of them.

**`IRenderDetailModel`.** An empty interface with no members and no real users, left behind by a removed call. See [`RenderDetailModel.h`](RenderDetailModel.h.md) — the finding there is that the grass layer has *no* engine-side surface at all, which a rebuild may want to change.

## The twins

| File | Role |
|---|---|
| [`RenderVisual.h`](RenderVisual.h.md) | The opaque model handle every drawable is; bounds, type tag, three capability queries |
| [`Kinematics.h`](Kinematics.h.md) | The skeleton: bones, transforms, visibility mask, the solve, bone-accurate picking |
| [`KinematicsAnimated.h`](KinematicsAnimated.h.md) | Animation playback: motion banks, blends, body parts, channels, track update |
| [`animation_blend.h`](animation_blend.h.md) | One running animation: clock, weight envelope, loop and freeze rules |
| [`animation_motion.h`](animation_motion.h.md) | The (bank, index) handle naming one animation |
| [`RenderFactory.h`](RenderFactory.h.md) | The only place the engine may construct renderer-side companions |
| [`FactoryPtr.h`](FactoryPtr.h.md) | The ownership policy for those companions: create on construct, destroy on destruct |
| [`xrRender.h`](xrRender.h.md) | The single exported entry point of each renderer backend, and the selection sequence |
| [`UIRender.h`](UIRender.h.md) | The immediate-mode vertex sink all two-dimensional drawing goes through |
| [`UIShader.h`](UIShader.h.md) | A material handle: a named pass chain bound to a named texture |
| [`DrawUtils.h`](DrawUtils.h.md) | The shape vocabulary: crosses, boxes, spheres, cones, gizmos, world text |
| [`DebugRender.h`](DebugRender.h.md) | Batched debug lines plus the slice of renderer state overlays may touch |
| [`DebugShader.h`](DebugShader.h.md) | Declares that a debug material is a UI material |
| [`WallMarkArray.h`](WallMarkArray.h.md) | A surface material's decal variants, and the random pick from them |
| [`ParticleCustom.h`](ParticleCustom.h.md) | A running particle effect: play, stop, follow this emitter, is it done |
| [`particles_systems_library_interface.hpp`](particles_systems_library_interface.hpp.md) | Read-only enumeration of the particle catalogue, for the editors |
| [`EnvironmentRender.h`](EnvironmentRender.h.md) | The sky and cloud domes, per-keyframe resources, and the weather blend |
| [`RainRender.h`](RainRender.h.md) | Draw the rain; report the drop's bounding volume back to the simulation |
| [`ThunderboltRender.h`](ThunderboltRender.h.md) | Draw the live lightning bolt |
| [`ThunderboltDescRender.h`](ThunderboltDescRender.h.md) | Own one authored bolt's model |
| [`LensFlareRender.h`](LensFlareRender.h.md) | The sun disc, the flare chain and the screen gradient |
| [`FontRender.h`](FontRender.h.md) | A font's atlas and the draw of one frame's accumulated text |
| [`ImGuiRender.h`](ImGuiRender.h.md) | The draw half of the debug overlay backend |
| [`StatGraphRender.h`](StatGraphRender.h.md) | Draw one scrolling debug plot |
| [`ObjectSpaceRender.h`](ObjectSpaceRender.h.md) | Debug visualization of collision queries, as accumulated spheres |
| [`UISequenceVideoItem.h`](UISequenceVideoItem.h.md) | Video playing into a UI element's texture, and the end-of-video freeze |
| [`RenderDetailModel.h`](RenderDetailModel.h.md) | An empty handle for a detail-object model; a vestige |
