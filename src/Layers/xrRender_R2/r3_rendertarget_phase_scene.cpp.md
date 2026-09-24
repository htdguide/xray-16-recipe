# src/Layers/xrRender_R2/r3_rendertarget_phase_scene.cpp

> Brackets the G-buffer fill: what is cleared before it, what is bound during it, and the
> resolve that closes it on hardware that could not write albedo directly.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`r2_types.h`](r2_types.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: multiple-target binding, clears and stencil state.

## Purpose

Three entry points around the frame's largest pass. *Prepare* decides what must be cleared
before anything is drawn — a decision that depends on which post-process effects are
active, because most of them read the position target in regions no geometry covered.
*Begin* binds the G-buffer and sets the stencil protocol. *End* performs the albedo
resolve on the weak path, and is otherwise empty.

## `phase_scene_prepare`

**Contract** — clears the depth buffer, and the position target as well whenever any
effect will read outside the covered region. Called once per frame before the G-buffer
fill. Also resets the "volumetric target was touched" flag.

```text
FUNCTION phase_scene_prepare()
  wants_clear_position =
      advanced post is on
      AND ( soft particles OR depth of field
            OR (light shafts enabled AND this weather's shaft intensity is non-zero)
            OR ambient occlusion )
  IF wants_clear_position
      bind position + the scene depth
      clear position to black
      clear depth (and, under multisampling, also albedo, the accumulator and the
          back depth — because the multisampled and resolved buffers must agree)
  ELSE
      bind the back buffer + the scene depth
      clear depth only
  volumetric target untouched = true
```

**Invariants** — a pixel of the position target that no geometry writes must read as zero,
not as last frame's content. Soft particles fade against the depth they read there; depth
of field and ambient occlusion sample neighbourhoods that cross the silhouette of the
world into empty sky. Without the clear, those effects sample stale geometry and produce
ghosting that follows the camera.

**Notes** — the clear is skipped when none of those effects is on, because clearing a
screen-sized float target is not free and the G-buffer fill overwrites every covered
pixel anyway. The normal and albedo targets are deliberately *not* cleared even when
position is: nothing reads them outside the covered region, since the coverage stencil
gates every consumer.

## `phase_scene_begin`

**Contract** — binds the G-buffer and establishes the stencil protocol for the fill: every
pixel a surface covers gets the value one. Draws front faces only. Four bind variants,
chosen by two independent options.

```text
FUNCTION phase_scene_begin()
  IF the normal target exists
      IF the albedo work-around is on: bind position, normal, accumulator
      ELSE                             bind position, normal, albedo
  ELSE                                  # packed G-buffer: two targets
      IF the albedo work-around is on: bind position, accumulator
      ELSE                             bind position, albedo
  ...with the scene depth in every case
  stencil: always pass, write 1 under mask 0x7f
  cull back faces; colour writes on
```

**Invariants** — the stencil write mask excludes the high bit, so the fill cannot disturb
an edge marking. It is the same mask the emissive pass uses, for the same reason.

**Notes** — binding the *accumulator* as the third target during the fill is the albedo
work-around: on a device that cannot mix target bit depths, all three bound targets must
match, so albedo is written into the float accumulator and moved afterwards. It is safe
because the accumulator carries nothing yet — the clear that matters happens later, at
first light.

## `phase_scene_end`

**Contract** — on the albedo work-around path, copies the accumulator's contents into the
real albedo target through one full-screen quad restricted to covered pixels. Otherwise
does nothing beyond the anisotropy reset.

```text
FUNCTION phase_scene_end()
  IF NOT albedo work-around THEN RETURN
  bind albedo + the scene depth
  no culling; stencil: pass where the stored value is at least 1
  material = the mask description's albedo element
  draw a full-screen quad
```

**Notes** — the mask description is the same object the light-volume stencil marking uses;
it carries an element per job and this is one of them. Routing the copy through it rather
than through a blit is what makes the copy *stencil-restricted*, which matters because the
accumulator must be left black outside the covered region.

## `disable_anisotropy`

**Contract** — a hook, empty on both modern backends, that once lowered the anisotropic
filtering level after the G-buffer fill so that later full-screen passes did not pay for
it. The state it manipulated is now part of a sampler state object set per pass, so the
hook has nothing to do. Kept because the frame graph calls it.
