# src/Layers/xrRender_R2/r3_rendertarget_draw_rain.cpp

> The wetness rewrite: three full-screen passes that read the rain shadow map and edit the
> G-buffer's normal, albedo and gloss in place, so sky-exposed surfaces shade wet.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`r2_types.h`](r2_types.h.md) ·
[`r3_R_rain.cpp`](r3_R_rain.cpp.md) ·
[`r2_rendertarget_phase_accumulator.cpp`](r2_rendertarget_phase_accumulator.cpp.md) ·
[`xrRender/light.h`](../xrRender/light.h.md) ·
[`xrRender/blenders/dx11RainBlender.h`](../xrRender/blenders/dx11RainBlender.h.md) ·
[`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) ·
[Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it edits bound render targets in place under a stencil and a
write-channel mask, and depends on the exact channel packing of the G-buffer.

## Purpose

This is the only pass in the frame that *modifies* the G-buffer after it is filled and
before it is lit. Everything else either adds to the accumulator or writes a new target;
this reaches back into position/normal/albedo and changes what the scene said about
itself, so that every light which runs afterwards — starting with the sun — sees wet
surfaces.

What "wet" means here is three edits:

- the **normal** is perturbed by a scrolling water-bump pattern, so flat ground ripples;
- the **albedo** is darkened, because a wet surface absorbs more;
- the **gloss** is raised, because a wet surface is smoother.

The interesting decision is that these cannot be done in one draw. Gloss lives in albedo's
fourth channel and the perturbed normal must be computed before either the normal or the
gloss can be written — and a pass cannot sample a target it is writing. So the work splits
into a *compute* pass into scratch and two *apply* passes that read the scratch.

## State

`Stateless.`

Everything it works on is passed in or owned elsewhere: the render-target set, and the
rain light record prepared by [`r3_R_rain.cpp`](r3_R_rain.cpp.md).

## `draw_rain(command_list, rain_light)`

**Contract** — rewrites the G-buffer for every covered pixel within the wet-surface
distance range. Reads the rain shadow map, the position, normal and albedo targets, the
material lookup, the cloud mask, a jitter texture and two water normal maps. Writes, in
order: the accumulator (as scratch), then the normal target (or the position target when
the G-buffer is packed), then the albedo target. Does not blend into the accumulator and
does not advance the light marker — the rain light is not a light.

**Invariants** — this runs *before* any lighting, which is what makes the accumulator
usable as scratch: the accumulator is cleared by whichever lighting step touches it first,
and that has not happened yet. Reordering rain after the sun destroys both passes at once —
the sun would shade dry surfaces, and the scratch write would erase the sun's contribution.

**Invariants** — the target set is restricted to covered geometry by the stencil (`equal
1`, the bit the G-buffer fill set) and to the wet-surface distance range by depth. Sky
pixels carry no stencil bit and are skipped; the depth restriction is how the range is
enforced and is described below.

```text
FUNCTION draw_rain(cmd, rain_light) -> ()
  density = the weather's current rain density

  # --- constants every sub-pass needs ------------------------------------------
  L_dir   = the rain light's direction in eye space, normalized   # straight down,
                                                                  # rotated by the view
  W_x, W_z = the world x and z axes in eye space, normalized
      # the water pattern is addressed in world space so puddles stay put on the
      # ground as the camera moves; the program reconstructs a world-horizontal
      # frame from these two and the eye-space position

  texel_adjust = the matrix that maps clip space onto the rain map's texture
      space: a half-scale-and-offset on each axis, scaled to the map rectangle
      (always the whole map), offset by half a texel, and biased in depth
  shadow_transform = texel_adjust * rain_light.transform * inverse_view
  cloud_transform  = the cloud mask's projection, built from the inverse view alone

  # --- the quad ------------------------------------------------------------------
  # ONE oversized triangle, not two, given directly in clip space. Its depth is
  # the projected depth of a point `far` metres ahead of the camera.
  clip_depth = project(camera position + camera forward * wet_surface_far).depth
  fill three vertices covering the screen at that depth, carrying a screen
      coordinate and a jitter coordinate scaled to the jitter texture's size

  # --- pass 1: compute the wet normal, into scratch --------------------------
  bind the accumulator with the multisample depth
  set the rain material's PATCH element; pass L_dir, W_x, W_z, the shadow and
      cloud transforms, the density, and the near/far fade distances
  draw under stencil `equal 1`                    # once per pixel

  # --- pass 2: write the normal ----------------------------------------------
  set the rain material's APPLY-NORMAL element; pass L_dir and the two transforms
  IF the G-buffer is packed
      bind the position target      # the normal lives in its spare channels
  ELSE
      bind the normal target
  draw under stencil `equal 1`

  # --- pass 3: write albedo and gloss -----------------------------------------
  set the rain material's APPLY-GLOSS element
  bind the albedo target
  draw under stencil `equal 1`
```

### The depth restriction

**Contract** — wetness fades out with distance and stops entirely at a configured far
distance. The far cut is not a test in the program; it is the geometry of the quad.

The screen-covering triangle is placed at the projected depth of a point exactly
`wet_surface_far` metres ahead of the camera, and the material declares an **inverted
depth test with depth writes off**. A pixel therefore survives only where the scene is
*nearer* than that plane — which is exactly the set of pixels that can be wet. The near
end and the fade are handed to the program as a pair of distances instead, because a fade
cannot be expressed as a clip.

**Notes** — this is the reason the pass draws one oversized triangle rather than two
triangles of a quad. A quad's diagonal would give the rasterizer two coplanar halves to
resolve at a constant depth, and the single triangle also avoids shading the diagonal
twice. The vertex positions are in clip space with no half-pixel offset, unlike most other
full-screen work in the chapter — see the backend chapters for why that inconsistency
exists.

### Why the accumulator is the scratch target

**Contract** — pass 1 writes the perturbed normal into the accumulator; passes 2 and 3
sample it from there.

**Invariants** — the accumulator is the only screen-sized four-channel float target that
is (a) already allocated, (b) not part of the G-buffer this pass is editing, and (c) not
yet holding anything this frame. Its contents at the end of pass 3 are garbage; the first
lighting step clears it. A rebuild may use any scratch surface of that shape, and probably
should — the reuse is a memory economy from an era with less of it, and it is the single
strongest ordering constraint in the frame graph.

### Which channels each apply pass may write

**Contract** — the apply passes do not write whole pixels. Pass 2 writes only the three
colour channels of the normal target (two, when the G-buffer is packed and it is really
editing the position target), leaving the fourth alone. Pass 3 writes all four channels of
albedo but under a *multiplicative* blend — destination times source — rather than a
replace.

**Invariants** — the fourth channel of the normal target carries the hemispheric-ambient
factor and the fourth channel of the position target carries the material id. Neither has
anything to do with rain, and writing them would silently reassign every wet pixel's
material. Under the packed G-buffer only two channels are writable because the normal is
packed into two of position's channels and the rest belong to somebody else.

**Invariants** — the albedo edit *must* be a multiply, not a replace, because it performs
two edits at once: darkening the colour and raising the gloss in the fourth channel. A
multiply lets one program output both factors without knowing what was there. A rebuild
that separates gloss into its own channel can replace instead.

### Multisampling

**Contract** — under multisampling each of the three passes runs twice: once over the
interior pixels (stencil bit 0 set, edge bit clear) with the ordinary program, and once
over the edge pixels (edge bit set) with a program variant that resolves per sample. Both
variants exist as separate compiled materials.

**Invariants** — the edge pass disables culling. The interior and edge passes together
must cover exactly the covered set: the interior draw's stencil mask includes the edge bit
so that an edge pixel fails it, and the edge draw tests for the edge bit being set.
Overlapping them would apply the darkening twice, and since the albedo edit is
multiplicative that is visible immediately as a black patch along every silhouette.

**Notes** — a second, unoptimized multisample path exists which loops once per sample with
a sample mask, for devices that cannot address individual samples in a program. The OpenGL
backend refuses it outright; a rebuild targeting a modern API needs only the optimized
form.

**Notes** — a large block of the original is commented-out state for facilities of an
earlier generation: a depth-bounds test that would have replaced the depth-clip trick
above, a four-tap depth fetch requested through an abused sampler setting, and a stencil
recompression call. None survives, and their absence is a hint that the depth-clip trick is
the portable way to express the range restriction.

**Notes** — the cloud-shadow transform is built from an identity matrix and the inverse
view, so it is presently just "the world, in eye space"; the drift that would make the
cloud mask crawl across the ground is commented out, along with the wind direction it
would follow. The mask is still sampled, so the effect is a static cloud pattern. Whether
the drift was removed for a reason or lost in a port is **not recoverable**.
