# src/Layers/xrRender_R2/r2_rendertarget_phase_PP.cpp

> The film post-process: one full-screen pass applying blur, desaturation, double vision,
> animated grain, a colour base and an optional colour grading — all driven by the game
> layer rather than by the renderer.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`r2_types.h`](r2_types.h.md) · [`xrRender/ColorMapManager.h`](../xrRender/ColorMapManager.h.md) · [`xrCore/PostProcess/PPInfo.hpp`](../../xrCore/PostProcess/PPInfo.hpp.md)
**Used by** — [`r4_rendertarget_phase_combine.cpp`](../xrRenderPC_R4/r4_rendertarget_phase_combine.cpp.md)
**Tier floor** — T1: it inspects the bound program's constant table to discover a
texture's dimensions before it can build coordinates.

## Purpose

Everything the *game* does to the image: concussion blur, the desaturation of a wounded
actor, the doubled vision of drunkenness or a head injury, film grain, and the colour
grading that gives each level its palette. The renderer computes none of these values —
the game layer pushes a parameter block every frame — so this file's job is to turn that
block into vertex coordinates and constants, decide whether the pass is worth running at
all, and animate the grain.

The three coordinate builders and the two "is this worth doing" predicates live in
[`r2_rendertarget.cpp`](r2_rendertarget.cpp.md); this page covers the pass itself and the
grain animation it drives.

## State

The parameter block, pushed by the game layer through setters on the render-target object:

```text
RECORD PostParameters
  blur          : real          # extra half-texel displacement of the sample point
  gray          : real          # desaturation, 0..1
  duality_h, duality_v : real   # horizontal and vertical eye separation
  noise         : real          # grain opacity
  noise_scale   : real          # grain size, as a multiplier on the noise texture
  noise_fps     : real          # how often a new grain offset is rolled
  colour_base   : colour        # multiplied in; neutral is mid-grey
  colour_gray   : colour        # the tint desaturation moves toward
  colour_add    : vector        # added in; neutral is zero
  cm_influence, cm_interpolate : real    # colour grading strength and cross-fade
  cm_textures   : two named lookup volumes to interpolate between
```

**Invariants** — neutral is not zero for the multiplicative parameters: the base colour's
neutral is mid-grey on every channel, because the program treats it as a signed
multiplier around the midpoint. Testing against zero instead would leave the pass
permanently on.

## `phase_post_process`

**Contract** — draws one full-screen quad into the back buffer, reading the combined
scene image. Selects one of two program elements depending on whether colour grading is
active. Advances the grain animation as a side effect.

```text
FUNCTION phase_post_process()
  bind the back buffer and its depth
  element = colour grading needed ? the graded element : the plain one

  gray_blend  = 255 * (1 - gray)     as an alpha
  noise_blend = 255 * (1 - noise)    as an alpha
  # the two blend factors ride in the alpha of the two colours the vertex carries,
  # because this vertex format has two colours and no spare scalar channel
  base = colour_base with noise_blend substituted into its alpha
  tint = colour_gray with gray_blend  substituted into its alpha

  duality coordinates  -> two coordinate sets (right eye, left eye)
  noise coordinates    -> one coordinate set, this frame's grain window
  quad: four vertices carrying position, the two colours, and those three
        coordinate sets; offset by the configured screen-shift pair
  constants: the additive brightness, and (grading influence, grading cross-fade)
  draw
```

**Notes** — the screen shift is a pair of settings that displace the whole post-processed
image by a fraction of a pixel. It exists to correct a half-texel misalignment on one
family of drivers and is zero everywhere else.

Packing the desaturation and grain amounts into the alpha channels of two vertex colours,
rather than sending them as constants, is a vertex-format artefact: this quad's format has
two colours and three coordinate pairs and no room left, and adding a constant would have
meant a second constant buffer update per frame on the hardware of the time. A rebuild
should send them as constants and say so.

The colour grading interpolates between *two* lookup volumes with a cross-fade parameter,
which is how the game blends one level's palette into the next across a loading
transition, and how weather-driven grading changes smoothly through a day.

## Grain animation

**Contract** — see `compute_noise_coordinates` in
[`r2_rendertarget.cpp`](r2_rendertarget.cpp.md). The grain's offset is re-rolled on a
fixed schedule derived from the configured rate, not every frame, and the window is sized
so the noise texture tiles across the screen at the configured scale.

**Notes** — reading the noise texture's size out of the *currently bound program's*
constant table, rather than from a known constant, is the one genuinely awkward thing
here: it means the coordinate builder can only run after the program is selected, and it
asserts loudly if the material forgot to bind a noise texture. The reason is that the
grain's texture is named by the material, so its size is data. A rebuild that fixes the
grain texture's size in the engine can delete the lookup.
