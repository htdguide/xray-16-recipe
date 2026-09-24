# src/Layers/xrRender/dxLensFlareRender.cpp

> The renderer's filling of the lens-flare port: builds one camera-facing quad per flare element, all into one buffer lock, then draws them one material at a time.

**Needs** — [`dxLensFlareRender.h`](dxLensFlareRender.h.md) · [`Include/xrRender/LensFlareRender.h`](../../Include/xrRender/LensFlareRender.h.md) · [`xrEngine/xr_efflensflare.h`](../../xrEngine/xr_efflensflare.h.md) · [`xrEngine/IGame_Persistent.h`](../../xrEngine/IGame_Persistent.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`R_Backend.h`](R_Backend.h.md)
**Used by** — [`dxLensFlareRender.h`](dxLensFlareRender.h.md)
**Tier floor** — T1: it writes a fixed vertex layout into a mapped scratch buffer with an explicit stride.

## Purpose

A lens flare is three unrelated things drawn with the same machinery: the **source** (the bright disc of the sun itself), the **flares** (the chain of ghosts strung along the line from the sun through the screen centre), and the **gradient** (the wide wash that fills the screen when you look near the sun). The flare object in the engine owns all the geometry parameters — where the sun is, where the chain's axis points, how faded each element currently is; this file turns them into quads.

Each element carries its own material, so the draw cannot be a single call. What *can* be shared is the buffer lock: every quad is built in one pass, then drawn in a loop that only changes the material between draws.

## State

```text
RECORD LensFlareRenderer
  geometry : VertexFormat     # position + packed colour + one texture coordinate

RECORD FlareElementRenderer   # one per authored flare element
  material : Material         # created only if the element names a texture
```

**Invariants** — at most **24** quads are built per call. The limit sizes the buffer lock and the material list; a flare chain with more elements than that silently loses the overflow. The number is an authored ceiling, not a device limit.

## `render(flare, draw_source, draw_flares, draw_gradient)`

**Contract** — builds and draws up to 24 quads. Locks the scratch vertex pool once for the maximum and unlocks with the count actually used. The three booleans let the caller draw the source alone (as the sky pass does), or the ghosts and wash alone (as the post-sky pass does).

```text
FUNCTION render(flare, draw_source, draw_flares, draw_gradient)
  light_colour = flare.light_colour
  materials    = empty list (at most 24)
  buffer       = lock_vertex_pool(24 * 4)

  distance = camera_far_plane * 0.75      # every quad is placed at this depth

  IF draw_source AND flare has a source THEN
    half_x = flare.screen_right * (source.radius * distance)
    half_y = flare.screen_up    * (source.radius * distance)
    colour = source.ignores_colour ? opaque white : light_colour
    colour.alpha = colour.alpha * flare.state_blend
    emit_quad(at flare.light_position, half_x, half_y, colour)
    materials.append(source.material)

  IF flare.blend >= epsilon THEN
    IF draw_flares AND flare has a chain THEN
      axis  = normalize(flare.chain_axis)
      right = cross(axis, flare.view_direction)
      FOR EACH element IN flare.chain
        centre = flare.chain_centre + flare.chain_axis * element.position
        half_x = axis  * (element.radius * distance)
        half_y = right * (element.radius * distance)
        colour = light_colour scaled on all four channels by
                 (element.opacity * flare.blend * flare.state_blend)
        emit_quad(at centre, half_x, half_y, colour)
        materials.append(element.material)

    IF draw_gradient AND flare.gradient_value >= epsilon AND flare has a gradient THEN
      half_x = flare.screen_right * (gradient.radius * flare.gradient_value * distance)
      half_y = flare.screen_up    * (gradient.radius * flare.gradient_value * distance)
      colour = light_colour scaled on all four channels by
               (flare.gradient_value * flare.state_blend)
      emit_quad(at flare.light_position, half_x, half_y, colour)
      materials.append(gradient.material)

  unlock_vertex_pool(materials.count * 4)

  set_world_transform(identity)
  FOR i IN 0 .. materials.count - 1
    IF materials[i] exists THEN
      set_material(materials[i])
      draw_triangles(starting at quad i, 4 vertices, 2 triangles)
```

**Invariants**

- Every quad is sized by `radius × (far_plane × 0.75)`. The flare lives in world space at three quarters of the far clip, so its authored radius is an *angular* size that stays constant on screen as the far plane changes with the weather. Placing it at the far plane exactly would clip it.
- The source and the gradient are built from the camera's screen axes, so they always face the camera square-on. The chain elements are built from the **chain axis and its perpendicular** instead, so each ghost is oriented along the sun-to-centre line — that is what makes a flare chain look like a lens artifact rather than a row of stickers.
- The source may ignore the light's colour and use white. That is an authored per-element flag; the sun's own disc is usually white while its ghosts take the light's tint.
- The source's fade multiplies only **alpha**; the chain's and the gradient's multiply **all four channels**. The difference is real: the source fades out by becoming transparent, the ghosts by becoming dim, which is what an additive blend needs.
- Quads are built in the fixed order source, chain, gradient, and the material list is appended in lockstep. Quad *i* is drawn with material *i*; the two lists must never diverge.
- The four vertices per quad follow the same order the shared quad index buffer expects, matching [`dxFontRender.cpp`](dxFontRender.cpp.md).

**Notes** — The whole chain is skipped when the flare's overall blend is below an epsilon, but the *source* is not: the sun's disc is drawn even when the flare effect is fully faded, because it is part of the sky rather than part of the artifact.

## `create_material(shader_name, texture_name)` / `destroy_material()`

**Contract** — creates one flare element's material, **only if the element names a non-empty texture**. An element with no texture leaves the material empty and its quad is skipped at draw time. That is how an authored flare turns off one of its three parts without a flag.

## `on_device_create()` / `on_device_destroy()`

**Contract** — creates and destroys the vertex format, bound to the renderer's scratch vertex pool and the shared quad index buffer.
