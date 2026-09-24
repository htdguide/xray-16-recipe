# src/Layers/xrRenderPC_R4/r4_rendertarget_phase_combine.cpp

> The resolve: turns the accumulated lighting and the G-buffer into one visible image, and runs everything that has to happen after lighting and before presentation, in the one order that works.

**Needs** — [`r4_rendertarget.h`](r4_rendertarget.h.md) · [`r4_rendertarget_u_set_rt.cpp`](r4_rendertarget_u_set_rt.cpp.md) · [`r4_rendertarget_phase_hdao.cpp`](r4_rendertarget_phase_hdao.cpp.md) · [`../xrRender_R2/r2_rendertarget.cpp`](../xrRender_R2/r2_rendertarget.cpp.md) · [`../xrRender_R2/r2_rendertarget_phase_bloom.cpp`](../xrRender_R2/r2_rendertarget_phase_bloom.cpp.md) · [`../xrRender_R2/r2_rendertarget_phase_PP.cpp`](../xrRender_R2/r2_rendertarget_phase_PP.cpp.md) · [`../xrRender_R2/r3_rendertarget_phase_ssao.cpp`](../xrRender_R2/r3_rendertarget_phase_ssao.cpp.md) · [`../xrRender/dxEnvironmentRender.h`](../xrRender/dxEnvironmentRender.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it is target juggling — which allocation is being written, which is being sampled, and where the multisample resolve falls between them.

## Purpose

Step 11 of the frame graph. Everything before it produced buffers; this produces the image.
It lives in the backend chapter rather than the shared one because the **target juggling**
differs between fillings: where the multisample resolve falls, which scratch target the
forward pass writes into, and whether the last draw can go straight to the swap chain. The
*order* of the effects is shared design and a rebuilder must keep it; the allocations they
pass through are this filling's business.

The companions `phase_wallmarks` and `phase_combine_volumetric` are here because they bind
the same scratch pair and would otherwise have to duplicate the aliasing rules.

## `phase_combine`

**Contract** — reads the accumulator, the G-buffer, the occlusion scratch and the exposure
pool; writes the back buffer or, more usually, a low-dynamic-range target that the film
post-process then copies to the back buffer. Leaves the exposure pool swapped for next
frame. Runs the forward pass, the heads-up display and the lens flares as sub-steps, so it
is the last point in the frame where game code can draw. Does not present.

```text
FUNCTION combine()
  # --- exposure framing ---
  # two pool entries per GPU; the frame number picks the GPU's pair
  gpu := frame_number MOD gpu_count
  luminance_src  := pool[2*gpu]        # what last frame measured
  luminance_dest := pool[2*gpu + 1]    # what this frame will measure

  # --- ambient occlusion, one of three implementations ---
  IF the compute occlusion program exists
      run the compute occlusion pass
  ELSE IF occlusion runs on downsampled data
      run the depth downsample only        # see Notes
  ELSE IF occlusion is blurred
      run the screen-space occlusion pass

  # --- the deferred resolve ---
  clear the two screen scratch targets ; bind both with the scene depth
  cull nothing ; stencil off
  draw the sky, then the clouds                    # see Invariants
  stencil: admit only pixels the G-buffer covered
  compute the motion-blur matrices                 # see below
  draw one full-screen quad with the combine material

  # --- what deferred cannot express ---
  bind only the first scratch target with the scene depth
  run the forward pass (translucent geometry, particles, faded portals)
  run the post-process user interface
  IF any volumetric light was active this frame THEN composite it

  # --- resolve, then the post chain ---
  IF multisampling is on THEN resolve both scratch twins into their singles
  run bloom                                        # also measures exposure
  IF anything requests distortion
      bind the second scratch ; clear it to the neutral displacement ;
      draw the distortion mask
  bind the final low-dynamic-range target
  draw one full-screen quad with the antialias/motion-blur/depth-of-field material
  draw the lens flares
  run the film post-process

  swap the two pool entries ; unbind both
```

**Invariants**

- **Sky and clouds are drawn before the combine quad, not after.** They write into the same
  scratch the combine quad is about to write, with depth testing but with the stencil
  admitting everything; the combine quad then overwrites every pixel the G-buffer covered.
  What survives is sky exactly where no geometry was. Drawing them afterwards would need a
  stencil test they do not otherwise require, and would put the clouds behind the fog term
  the combine applies. The clouds specifically are drawn without a depth test, because a
  cloud layer tested against the sky dome's depth produces silhouettes at the dome seams.
- **The forward pass writes to one target, the combine wrote to two.** The second scratch
  is the distortion mask's home and must not be touched by translucent geometry; rebinding
  with only the first is how it is protected.
- **The multisample resolve falls between the forward pass and bloom, and nowhere else.**
  Bloom samples the colour image many times at reduced resolution and cannot do that from a
  multisampled allocation; the forward pass must happen before the resolve because it needs
  the multisampled depth to sort against. There is exactly one gap that satisfies both.
- **Bloom invalidates the high-dynamic-range target**, which is why nothing after it may
  read the accumulator.
- **The exposure pool entries are swapped at the very end**, after everything that could
  read the measurement. The swap is the feedback loop: what this frame measured is what the
  next frame is exposed for.

### The lighting constants the combine quad is handed

These are the parameters of the resolve, and they are where the environment's authored
mood reaches the pixel:

- **The inverse view matrix**, so the program can take an eye-space G-buffer sample back to
  world space for the environment lookups.
- **Ambient colour**, doubled, floored at a small positive value in each channel and then
  scaled by the sun's ambient luminance factor. The floor matters: a fully black ambient
  makes unlit surfaces read as holes rather than as shadow, so the engine refuses to go
  below it.
- **Environment (hemispheric) colour**, doubled and scaled by the sun's hemispheric
  luminance factor. This is the sky's contribution, modulated per pixel by the hemisphere
  factor the G-buffer's normal target carries in its fourth channel.
- **Fog colour**.
- **The sun's colour with its specular scalar in the fourth channel, and its direction
  transformed into eye space.** Eye space, because the entire G-buffer is in eye space.
- **Two occlusion scale factors**, both corrected for field of view against a reference of
  67.5 degrees exactly as the screen-space occlusion pass corrects its own: the noise tiling
  (base 2) and the kernel size (base 150). They are recomputed here rather than shared
  because the combine samples the occlusion result and must undo the same scaling.

The doubling of both ambient and environment colour is a fixed factor with no derivation in
the source; it compensates for the authored environment data having been tuned against a
different exposure convention, and it must be reproduced or every level is half as bright
as it should be.

### The motion-blur matrices

```text
# the previous frame's full view-projection is kept in a single persistent slot
m_previous := saved_view_projection composed with this frame's inverse view
m_current  := this frame's projection
saved_view_projection := this frame's full transform      # for next frame
blur_scale := (motion_blur_strength / 2, -motion_blur_strength / 2) / 12
```

**Invariants** — the program reconstructs a pixel's world position from the G-buffer,
projects it with both matrices and blurs along the difference. `m_previous` is built as
*inverse view then last frame's view-projection*, which is the "where was this point on
screen last frame" operator. The vertical component of the scale is negated because screen
vertical and clip vertical run opposite ways. The divisor of 12 is the number of taps the
blur program takes — the scale is per-tap, so changing the tap count without changing the
divisor changes the blur length.

### The final quad's vertex stream

The last draw carries seven texture-coordinate sets per vertex, not one: the centre tap and
four diagonal neighbours at one pixel's offset, then two four-component sets packing the
axis-aligned cross. The taps are in the **vertex** stream because they are affine in screen
position, so interpolating them is free where recomputing them per pixel is not. That is the
edge-detect antialias kernel's footprint, baked into geometry.

Its constants are the edge-detection barrier and weights and the kernel width (all from
configuration, tuned per game), the two motion-blur matrices and the blur scale, and the
depth-of-field focal parameters with a kernel expressed as half a pixel scaled by the
configured kernel size.

Which pass of the combine material runs is chosen on two independent booleans — whether the
antialias filter is enabled, and whether anything asked for distortion this frame — giving
four compiled passes. Under multisampling the same four come from the per-sample variant
instead.

### Multisampling: the three-way shape

Every draw in this file that touches G-buffer pixels has the same shape, and it recurs
throughout the backend:

```text
IF multisampling is off
    one draw, stencil admits covered pixels
ELSE
    one draw with the per-pixel program, stencil admits interior pixels
        # interior = the edge-marking pass did not set the high stencil bit
    IF the optimized multisample path is available
        one draw with the per-sample program, stencil admits edge pixels
        # the program itself loops over the samples
    ELSE
        one draw per sample with that sample's program,
        the device's sample mask restricted to that single sample
    restore the full sample mask
```

The axis a rebuilder must understand: **the interior of every polygon is shaded once, and
only pixels whose samples disagree are shaded per sample.** The two edge paths differ in
whether the loop over samples lives inside the program (one draw, needs a program that can
address individual samples of a multisampled input) or outside it (one draw per sample,
works anywhere, costs a draw and a state change per sample). Everything else about them is
identical, and a filling whose device can do the former should never do the latter.

## `phase_combine_volumetric`

**Contract** — composites the volumetric (light-shaft) accumulator into the two scratch
targets with a single full-screen quad, writing colour channels only. Called from the
combine when any volumetric light was active.

**Notes** — the alpha channel is masked off because the second scratch's alpha is carrying
something else by this point. Light shafts get their own accumulator, rather than sharing
the lighting accumulator, because they are added *after* the accumulator has been multiplied
by albedo — a shaft seen against a dark wall must not be darkened by the wall.

## `phase_wallmarks`

**Contract** — binds the albedo target alone with the scene depth, admits only pixels the
G-buffer covered, culls back faces and enables colour writes on the three colour channels
only. Sets no geometry and issues no draw — the caller draws.

**Notes** — alpha is masked because albedo's fourth channel carries gloss, and a decal must
change a surface's colour without claiming to change how shiny it is. The two extra colour
slots are explicitly unbound first: the pass that ran before this one had the full G-buffer
bound, and leaving position and normal attached would have decals rewriting the geometry
buffer.

## Notes

**The off-screen final target is forced on unconditionally.** The code computes whether the
film post-process has anything to do, and then overrides the answer to "yes" — every frame
renders its final image into a target and copies it to the back buffer rather than drawing
to the back buffer directly. The reason recorded is that a screenshot must be able to read
the finished image back even in a window, and the swap chain's buffer is not reliably
readable. A rebuild whose seam filling can read back its own swap chain should restore the
branch; it costs one full-screen copy per frame.

**The downsampled-occlusion branch runs the depth downsample and then does not run
occlusion.** The occlusion call on that path is struck out in the source, so selecting that
quality setting produces a downsampled depth image nobody reads and no ambient occlusion at
all. This looks like a bug rather than a decision, and a rebuild should either call the
occlusion pass or drop the branch.

**Unrecovered**: a helper that maps a pixel coordinate to clip space is defined at the top
of the file and never called — a leftover from when the full-screen quads were built in
pixel coordinates rather than in clip space. Several quads in this file still compute
half-texel offsets that the current clip-space quads do not use.
