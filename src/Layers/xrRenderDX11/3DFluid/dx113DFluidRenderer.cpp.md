# src/Layers/xrRenderDX11/3DFluid/dx113DFluidRenderer.cpp

> Making a density field visible: find where each view ray enters and leaves the volume, march it at reduced resolution, and composite the result against the scene, re-marching only at the silhouettes where the reduction shows.

**Needs** — [`dx113DFluidRenderer.h`](dx113DFluidRenderer.h.md) · [`dx113DFluidBlenders.h`](dx113DFluidBlenders.h.md) · [`dx113DFluidData.h`](dx113DFluidData.h.md) · [`xrRender/BufferUtils.h`](../../xrRender/BufferUtils.h.md) · [`../dx11R_Backend_Runtime.h`](../dx11R_Backend_Runtime.h.md) · [`../dx11SH_RT.cpp`](../dx11SH_RT.cpp.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx113DFluidRenderer.h`](dx113DFluidRenderer.h.md)
**Tier floor** — T1: it creates render targets with explicit formats, generates texture contents byte by byte, and owns geometry buffers.

## Purpose

The solver produces a three-dimensional field of numbers. This file turns one into pixels. The technique is ray marching: for every pixel the volume covers, walk a ray from the eye through the field, accumulating — absorption for fog, emission for fire — and composite the accumulated colour over the frame.

The whole design is governed by one cost: **a march is many samples per pixel, and the number of pixels a fog volume covers is not small.** Three decisions follow, and they are what a rebuild must reproduce:

1. The entry point and the ray length are computed **once into a texture**, not per march step, by rasterizing the volume's box twice and letting the blend hardware subtract.
2. The march runs at **reduced resolution**, capped by the grid's own resolution — marching at more pixels than the field has voxels resolves nothing.
3. The reduction is visible only at **silhouettes**, where the volume meets scene geometry or its own edge, so those pixels are detected and re-marched at full resolution while everything else is upsampled.

## State

```text
RECORD FluidVolumeRenderer
  grid_dimensions   : (real, real, real)
  max_dimension     : real
  render_width      : int      # the reduced march resolution
  render_height     : int
  grid_matrix       : matrix   # centres the unit cube and restores its aspect

  targets:
    ray_data        : 4 channels, full precision, at SCREEN resolution
    ray_data_small  : 4 channels, full precision, at march resolution
    ray_cast        : 4 channels, full precision, at march resolution
    edges           : 1 channel, full precision, at march resolution

  geometry:
    box             : 8 vertices, 12 triangles, the unit cube 0..1
    screen_quad     : 4 vertices

  generated textures:
    jitter          : 256 x 256, one byte per texel, uniform random
    filter_weights  : 16 texels, 4 half-precision channels
```

**Invariant** — the ray-data target is at *screen* resolution while the other three are at march resolution. The first must be sharp because it is what gets downsampled and edge-detected; if it were already reduced there would be nothing to detect.

**Invariant** — the ray-data targets are full 32-bit precision per channel, unlike every field in the solver, which is half. They carry a position in grid space and a distance in view space, and a half-precision position quantizes the march's start point to a visible stair-step across the volume's face. This is the one place in the subsystem where half precision is not enough.

## `Initialize`

**Contract** — builds the grid matrix, compiles the eight passes, creates the box and quad geometry, and generates the two lookup textures. The render targets are *not* created here; they depend on the window size and are created by `SetScreenSize`.

```text
grid_matrix := translate by (-0.5, -0.5, -0.5)
             · scale by (dimensions / max dimension)
```

**Invariants** — Two jobs in one matrix. The translation moves the box's own `0..1` texture-coordinate space so the volume is centred on the placement transform's origin — which is what makes the placement transform mean "where the centre of the fog is". The scale preserves the grid's aspect: a grid that is twice as wide as it is tall must render as a box twice as wide as it is tall, or the fog is stretched. Dividing by the largest dimension rather than by a fixed value keeps the longest side at unit length, so the placement transform's scale means "the size of the longest side".

## `CalculateRenderTextureSize` — the resolution cap

**Contract** — chooses the march resolution from the window size and the grid size.

```text
FUNCTION march_resolution(screen_width, screen_height)
  # The most screen space the volume can possibly need is its own
  # diagonal: a cube of side N seen corner-on spans sqrt(3)*N voxels.
  # Three samples per voxel is the marching rate.
  max_useful := 3 * sqrt(3) * max_grid_dimension
  IF max(screen_width, screen_height) <= max_useful
    use the screen resolution unchanged
  ELSE
    set the longer axis to max_useful and the shorter one to match
    the screen's aspect ratio
```

**Invariants** — The cap is on the **longer** axis and the other follows from the aspect ratio, so the march always renders a correctly-proportioned image. Below the cap the march runs at native resolution and the edge-detection path costs a little for nothing — accepted, because at that point the volume is small on screen anyway.

**Notes** — The factor of three is the oversampling rate: three marched samples per voxel along the ray. Nothing in the source derives it. Lowering it banded the fog; that is as much as is recoverable.

## `Draw` — the whole sequence

**Contract** — renders one fluid volume into the frame. Releases the depth target first, because nothing in the sequence depth-tests — the march does its own occlusion from the scene's depth image. Restoring the frame's state afterwards is the solver's job, not this file's.

```text
FUNCTION draw(volume)
  lighting := gather_lighting(volume)
  is_fire  := volume.settings.type is fire

  compute_ray_data(volume)          # entry point and ray length, full res
  compute_edge_texture(volume)      # downsample, then detect silhouettes

  # march, at reduced resolution, into a scratch target
  clear the raycast target to black
  target := ray_cast
  pass   := march_fire or march_fog
  bind the per-draw constants for the march resolution
  draw the screen quad

  # composite to the frame, re-marching where an edge was found
  target := the frame's colour target, with the scene's depth bound
  pass   := composite_fire or composite_fog
  bind the per-draw constants for the SCREEN resolution
  bind the accumulated light intensity
  draw the screen quad
```

**Invariants** — The per-draw constants are bound **twice with different resolutions**, once for the march and once for the composite. They encode, among other things, the reciprocal of the target size, which the composite needs at screen resolution to sample the march result correctly. Binding them once would misplace every sample by half a pixel of the wrong size.

Fog and fire are separate passes end to end rather than one pass with a mode flag. Fire marches a temperature field through an emissive transfer function and does not attenuate what is behind it the way absorption does; the two accumulation loops share almost nothing.

**Notes** — The first pass bound in `Draw` is bound solely so that the constant layout is established before anything is written into it, and the source says so and warns that the hack depends on every fluid pass sharing one constant layout. That dependency is invisible and would break the moment one pass declared its constants differently. A rebuild with an explicit constant-block description does not have this problem and should not reproduce the workaround.

## `ComputeRayData` — entry point and ray length in one target

**Contract** — rasterizes the volume's box twice into one four-channel target, arranging the blend so that the difference of the two passes is the answer.

```text
FUNCTION compute_ray_data(volume)
  clear ray_data to zero
  target := ray_data

  # pass 1: the faces pointing away from the eye — where the ray leaves
  pass := ray_data_back
  writes  (0, -1, 0) and  min(scene depth, box exit depth)
  draw the box

  # pass 2: the faces facing the eye — where the ray enters.
  #         blended reverse-subtractively, so the target becomes
  #         (exit - entry): a grid-space entry point and a ray length.
  pass := ray_data_front
  writes  the entry position in grid space, and the box entry depth,
          unless the scene occludes this pixel entirely, in which case
          it writes a marker the march recognizes as "skip"
  draw the box
```

**Invariants** — **Clamping the exit depth to the scene's depth is how the volume is occluded by geometry**, and it happens in the first pass. A wall standing inside the fog shortens every ray that hits it, so the march stops at the wall; there is no per-sample depth test in the march at all. A rebuild must do the occlusion here or pay for it once per sample.

The two passes write into the same target with opposite cull modes, and the second's blend is reverse-subtractive on both colour and alpha. This is the arithmetic that turns two rasterizations into one difference without a second target or a readback. See [`dx113DFluidBlenders.cpp`](dx113DFluidBlenders.cpp.md) for why the blend state has to be assembled in two steps.

A pixel the volume does not cover keeps the cleared value of zero, which is a zero ray length and so a march that terminates immediately.

**Notes** — The target is re-bound between the two passes although it does not change. Removing the redundant bind is safe.

## `ComputeEdgeTexture` — where the reduction will show

**Contract** — downsamples the ray data to march resolution by point sampling, then runs an edge detector over the reduced image and writes a one-channel mask.

**Invariants** — **Point sampling, not filtering.** The ray data is a position and a distance; averaging four of them across a silhouette produces a position that is inside neither surface and a distance that belongs to nothing. The detector is looking for exactly those discontinuities, and filtering them away is the one thing that would defeat it.

The detected edges are the places where the march's own inputs change abruptly between neighbouring pixels: the volume's silhouette against the background, and the line where scene geometry cuts into it. Those are the pixels where upsampling the marched result would produce a visible staircase, so the composite re-marches them at full resolution.

## `CreateJitterTexture`

**Contract** — generates a 256-by-256 single-byte texture of uniform random values at startup and registers it under an engine texture name so the passes can bind it by name.

**Invariants** — The march offsets each ray's first sample by this value, tiled across the screen. Without it every ray samples the field at the same set of planes and the result bands visibly — concentric shells of density where there should be a gradient. Jittering converts the banding into high-frequency noise, which the eye reads as fog. **This is not optional at low sample counts**, and a rebuild that finds its fog banded should look here first.

**Notes** — Generated with the global random source at startup, so it differs between runs. Nothing depends on its exact contents.

## `CreateHHGGTexture` — the filter-weight table

**Contract** — generates a sixteen-texel lookup table of cubic B-spline filter weights and offsets, in half precision, registered under an engine texture name.

```text
FOR i IN 0..15
  a := i / 15                     # fractional position within a texel
  texel[i] := ( -h0(a), h1(a), 1 - g0(a), g0(a) )
where g0, g1 are the summed B-spline weights of the two texel pairs
and h0, h1 are the offsets at which a linear sample returns the
correctly-weighted sum of the pair
```

**Invariants** — This is the standard trick for cubic filtering on hardware that offers linear filtering: **a weighted pair of adjacent texels is one linear sample taken at the right fractional offset**, so a cubic reconstruction over four texels costs two linear samples per axis instead of four point samples plus arithmetic. The table precomputes the offsets and weights for sixteen sample positions.

The values lie in `-1 .. +1`, which is why half precision suffices; the source notes the range for exactly this reason.

**Notes** — The table has sixteen entries, so the fractional position is quantized to sixteen steps and sampled with clamping. The quantization is invisible because the result feeds a smoothing filter. A rebuild that computes the weights in the shader can delete the table entirely; it exists because evaluating four cubic polynomials per sample was more expensive than a texture fetch on the hardware this was written for, and that tradeoff has since reversed.

## `CalculateLighting`

**Contract** — reduces all the light reaching a fog volume to a **single colour**, computed once per volume per frame on the host.

```text
FUNCTION gather_lighting(volume)
  intensity := environment.hemisphere_colour * volume.settings.hemisphere_scale
  intensity := intensity + environment.ambient
  FOR EACH light source overlapping the volume's bounding box
    skip it if it is a static light
    d := distance from the light to the volume's centre
    skip it if d >= light.range + the volume's largest half-extent
    falloff := clamp(1 - d / light.range, 0, 1) * 2
    intensity := intensity + light.colour * falloff
  RETURN intensity
```

**Invariants** — **One colour for the whole volume.** The fog is not lit per voxel and casts no shadow on itself; a torch at one end of a fog bank brightens the entire bank uniformly. This is the largest fidelity concession in the subsystem, and it is also why the fog costs what it does: per-voxel lighting would need a second march per light.

Static lights are skipped because the hemisphere and ambient terms already account for the level's baked lighting; counting a baked light again would double it. The distance test uses the light's range plus the volume's largest half-extent, which is a conservative sphere-versus-sphere test — it admits some lights that do not actually reach, which costs an addition.

**Notes** — The dynamic-light falloff is multiplied by two, via a conditional that can only ever take the dynamic branch because static lights were already skipped. The source shows the factor arriving as a "dynamic lights count double" rule from when static ones were also summed, and surviving the removal of that case. It now simply means dynamic lights brighten fog twice as much as their own intensity — a tuning value, and worth naming as one.

## `PrepareCBuffer` — the per-draw constants

**Contract** — computes and binds everything the marching passes need to relate the volume, the camera and the target. Called before each of the four screen-space draws, with the target's dimensions.

```text
near and far planes              # to unproject the scene's depth image
grid_scale_factor                # the length of one axis of the
                                 # world-view matrix: converts a ray length
                                 # from view space into grid space
world_view_projection            # box -> screen
inverse world_view_projection    # a point on the near plane -> grid space
eye_in_grid_space                # the ray origin, one value for the whole draw
target width and height
```

**Invariants** — The grid matrix is **prepended** to the world-view transform, not applied to the geometry, so the box geometry stays the literal unit cube `0..1` and the same eight vertices serve every volume regardless of its grid aspect ratio.

**The ray length must be converted from view space to grid space**, because the march steps through grid space while the entry/exit distances were computed in view space against the scene's depth. The conversion factor is the length of one axis of the world-view matrix — the world-space length of the box's longest side — which is correct precisely because the grid matrix already normalized the longest side to one. A rebuild that skips the normalization needs a per-axis conversion instead of a scalar.

The eye position is transformed into grid space on the host and bound as one constant, rather than being derived per pixel from the inverse transform. It is constant over the draw; deriving it per pixel is a matrix multiply per pixel for the same answer.

## `SetScreenSize` / `CreateRayDataResources`

**Contract** — recreate the four intermediate targets for a new window size. The full-resolution ray-data target takes the window's dimensions; the other three take the capped march resolution. Called on window resize and at startup.

**Notes** — Each target is released before being recreated, so a resize does not transiently hold two sets. The volume fields themselves are untouched by a resize — only the screen-space intermediates depend on it, which is why this is the renderer's job and not the solver's.
