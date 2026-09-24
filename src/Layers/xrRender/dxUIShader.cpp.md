# src/Layers/xrRender/dxUIShader.cpp

> A named material plus the texture it samples, for callers — the UI, the debug overlay, decals — that need a material handle but must not know what a material is.

**Needs** — [`Include/xrRender/UIShader.h`](../../Include/xrRender/UIShader.h.md) · [`dxUIShader.h`](dxUIShader.h.md) · [`Shader.h`](Shader.h.md) · [`r_constants.h`](r_constants.h.md) · [`Include/xrRender/ImGuiRender.h`](../../Include/xrRender/ImGuiRender.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Debug overlay UI](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`dxUIShader.h`](dxUIShader.h.md)
**Tier floor** — T1: to answer "what are this material's pixel dimensions" it walks the material's pass list and its reflected constant table down to a bound texture.

## Purpose

Chapter 4's material handle is deliberately opaque: the UI names a material and a texture, gets something back, and passes it to the vertex sink. This file is the filling. Three of its four jobs are one-line delegations to the resource manager; the fourth — resolving the handle down to *the* texture, and from there to a size and to a handle the debug overlay can draw — is the only algorithm, and it exists because callers need to lay out around an image whose dimensions only the loaded texture knows.

## State

```text
RECORD UIMaterialHandle
  material            : optional<Material>   # shared reference into the resource manager
  base_sampler_name   : text                 # frozen: the conventional base-colour sampler
```

Invariants:

- The material is a shared reference. Creating adds a use, destroying drops one; the resource manager owns the compiled thing. Two handles naming the same material and texture refer to the same object, which is what makes equality meaningful.
- The base sampler name is a constant of the engine, not of the handle. It is stored per object only so that a rebuild can see it is a *name in the shipped shader sources* and therefore frozen.

## `create`

**Contract** — Resolve a material name and an optional texture name through the resource manager and take a reference. Blocks: the material may have to be parsed, its shaders compiled or fetched from the compiled-blob cache, and the texture loaded from disk. A name that does not resolve produces an empty handle rather than a failure, because the UI routinely names optional decoration.

## `destroy`

**Contract** — Drop the reference. Does not free the material unless this was the last user.

## `initialised`

**Contract** — Whether a material was resolved. Callers test this before drawing; an unresolved handle must not reach the vertex sink.

## `equal`

**Contract** — Two handles are equal when they hold the same material reference. This is *identity*, not equivalence: two independently created handles naming the same material and texture compare equal because the resource manager returns the same object for the same name pair. The UI uses this to avoid redundant material changes between widgets.

## `base_texture`

**Contract** — Resolve the handle to the texture its base-colour sampler is bound to, or nothing. Never blocks; reads only already-loaded state.

```text
FUNCTION base_texture() -> optional<Texture>
  IF material is empty THEN RETURN none
  # The UI never uses a material's level-of-detail variants or its later passes:
  # by convention a UI material is one element with one pass. Reaching for the
  # first of each is therefore not an approximation, it is the convention.
  pass = material.elements[0].passes[0]
  IF pass has no texture list THEN RETURN none
  IF pass.textures is empty   THEN RETURN none
  binding = pass.constants.lookup(base_sampler_name)
  slot    = binding is present ? binding.sampler_index : 0
  RETURN pass.textures[slot].texture
```

**Notes** — The fallback to slot zero when the material declares no base-colour sampler covers materials authored before the naming convention existed; the shipped data contains both kinds, so the fallback is load-bearing, not defensive.

## `base_texture_resolution`

**Contract** — Report the base texture's pixel width and height, and whether there was one. Yields zeroes when there is none. This is what lets a widget size itself to its image — an icon, a map tile, a portrait — without the layout code loading anything.

## `debug_overlay_texture_id`

**Contract** — Package the base texture as the pair the immediate-mode debug toolkit wants: an opaque per-backend texture identifier plus its dimensions as real numbers. Yields an empty pair when there is no texture. This is the one point where the [debug overlay seam](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui) reaches an engine texture, and it goes through the *UI material* handle rather than a texture handle precisely so that debug tooling can name things the same way the game's own screens do.

## `copy`

**Contract** — Adopt another handle's contents, taking a reference to the same material.
