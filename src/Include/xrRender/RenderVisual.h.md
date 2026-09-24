# src/Include/xrRender/RenderVisual.h

> The root of the renderer's model hierarchy as the rest of the engine sees it: an opaque drawable with bounds and a type tag.

**Needs** — [`xrEngine/vis_common.h`](../../xrEngine/vis_common.h.md) · [`Kinematics.h`](Kinematics.h.md) · [`KinematicsAnimated.h`](KinematicsAnimated.h.md) · [`ParticleCustom.h`](ParticleCustom.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`Kinematics.h`](Kinematics.h.md) · [`ParticleCustom.h`](ParticleCustom.h.md) · [`RenderDetailModel.h`](RenderDetailModel.h.md) · [`FBasicVisual.h`](../../Layers/xrRender/FBasicVisual.h.md) · [`LightTrack.h`](../../Layers/xrRender/LightTrack.h.md) · [`R_Backend_hemi.cpp`](../../Layers/xrRender/R_Backend_hemi.cpp.md) · [`R_Backend_hemi.h`](../../Layers/xrRender/R_Backend_hemi.h.md) · [`r__dsgraph_structure.h`](../../Layers/xrRender/r__dsgraph_structure.h.md) · [`r__pixel_calculator.h`](../../Layers/xrRender/r__pixel_calculator.h.md) · [`r__sector.h`](../../Layers/xrRender/r__sector.h.md) · [`CustomRocket.cpp`](../../xrGame/CustomRocket.cpp.md) · [`ZoneVisual.cpp`](../../xrGame/ZoneVisual.cpp.md) · [`base_client_classes_script.cpp`](../../xrGame/base_client_classes_script.cpp.md) · [`physic_item.cpp`](../../xrGame/physic_item.cpp.md)
**Tier floor** — T2: the interface is four queries; what stops it going higher is the visibility record it exposes by reference, which the renderer writes every frame from inside the culling pass.

## Purpose

Every drawable thing the engine owns — a wall segment, a tree, a crate, a creature, a particle group, a level-of-detail impostor — is one of these. The engine holds it, positions it, and hands it back to the renderer to draw; it never learns what is inside. That single decision is what makes the graphics device a pluggable seam rather than a dependency: **the engine's models are handles, and only the renderer can create, duplicate or destroy one.**

The type is deliberately almost empty. Everything specific lives behind a downcast query.

## State

```text
RECORD RenderVisual
  vis_data : VisData        # owned by the visual, written by the renderer's culling pass
  type     : int            # the model-type code from the model file format
```

```text
RECORD VisData              # see xrEngine/vis_common.h
  sphere        : Sphere    # world-space bounding sphere, current pose
  box           : Box       # world-space bounding box, current pose
  markers       : list<int> # one per render context; the frame this visual was last enqueued
  accept_frame  : int       # the frame it was last admitted to the main render
  hom_frame     : int       # the frame its next occlusion test is scheduled for
  hom_tested    : int       # the frame its last occlusion test ran
  shader_data   : ObjectShaderData ref  # per-instance material parameters
```

**Invariants** — the bounds are in *world* space and are only as fresh as the last skeleton solve or transform update; a caller that needs them exact must force that first. The per-context markers exist because one visual can be enqueued into several render passes in a frame (main view, each shadow map, the reflection pass) and each pass needs its own "already added" flag; a single flag would drop the model from every pass but the first.

## `IRenderVisual` — what an implementor must provide

### `visibility_data`

**Contract** — returns the visual's visibility record *by reference*, mutable. The renderer's culling pass stamps its markers into it; the engine reads the bounds from it to decide whether an object is worth simulating in detail. Never absent.

**Notes** — Handing out a mutable reference to internal state is the interface's one real compromise, and it is a deliberate one: the alternative is a getter and a setter per field, called once per visual per pass per frame. A rebuild may keep the aliasing or move the per-frame markers out of the model entirely into the culling pass's own scratch table, which is cleaner and costs an indirection.

### `type`

**Contract** — the model-type code this visual was built from: plain mesh, hierarchy, progressive mesh, animated skeleton, rigid skeleton, level-of-detail impostor, tree, particle effect, particle group, and a few more. Read from the model file's header and never changed.

**Notes** — This is the shipped file format's own enumeration leaking through the interface, and it leaks for a real reason: the game branches on it in a handful of places (is this thing a tree? is it a particle group?) where a downcast would be heavier. A rebuild can replace it with capability queries everywhere except where the *file format's* code is what is being reported, which is the model loader's business alone.

### `sub_model(index)`

**Contract** — a child visual of a composite model, or nothing. Hierarchical and level-of-detail models are trees of visuals; this exposes one level of that tree. The default answer is nothing, so a leaf model need not implement it.

### `as_skeleton`, `as_animated_skeleton`, `as_particles`

**Contract** — the three capability queries. Each returns the same object under a richer interface if it supports it, and nothing otherwise. All default to nothing.

**Notes** — This is the engine's whole type system for models. There is no shared base beyond this one, no registry, no dynamic type query: an implementor overrides the queries it can answer. It is crude and it is also exactly the right shape for a seam, because it lets the renderer keep every concrete model type private while the engine asks only the three questions it actually has code for. A rebuild should keep the *set* of questions — has a skeleton, plays animations, is a particle emitter — and is free to express them as interface tests, capability flags, or a tagged union.

## Ownership and lifetime — the rule that governs the whole family

An implementor is created only by the renderer's model factory, from a model path or a pre-opened stream, and destroyed only by the renderer's model deleter. The engine holds the handle, keeps it across frames, may duplicate it (which shares the immutable mesh and skeleton data and copies only the per-instance state), and must release it before the renderer's level teardown runs. Nothing outside the renderer may allocate one, and nothing may keep one past a level unload.

The corollary matters more than it looks: **the immutable half of a model — mesh, bone hierarchy, motion banks — is shared between every instance and reference-counted inside the renderer, while the mutable half — bone instances, blend pool, visibility record — is per instance.** Duplication is cheap precisely because of that split, and the engine duplicates freely (every corpse, every dropped item). A rebuild that copies a model wholesale on duplicate will load a level and then run out of memory.
