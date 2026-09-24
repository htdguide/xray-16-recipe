# src/Layers/xrRenderPC_GL/gl_rendertarget_build_textures.cpp

> Builds the two procedural lookup tables the deferred shader set cannot do without: the lighting-model table and the sampling-jitter set.

**Needs** — [`gl_rendertarget.h`](gl_rendertarget.h.md) · [`../xrRenderGL/glSH_Texture.cpp`](../xrRenderGL/glSH_Texture.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`gl_rendertarget.h`](gl_rendertarget.h.md)
**Tier floor** — T1: it writes exact byte layouts into device storage, addressing them by computed row and slice pitch.

## Purpose

Two tables are computed on the host at startup and uploaded once, and both encode decisions that would otherwise be scattered through shaders.

The **lighting-model table** is the more interesting. Rather than branch between four shading models in the pixel program, the renderer pre-tabulates all four into one three-dimensional texture: the two axes are the two dot products a lighting model needs, the third axis selects the model, and the two channels hold the diffuse and specular responses. A surface's material index — which travels through the geometry buffer packed alongside its position — becomes the slice coordinate. **Shading model selection therefore costs one texture fetch and no branching**, which is exactly the trade the hardware of the era rewarded, and which still holds.

The **jitter set** supplies the pseudo-random offsets the soft-shadow and ambient-occlusion shaders sample. Generating them on the host rather than in the shader means the pattern is identical across frames and across pixels of the same screen position, which is what makes the filtering stable rather than boiling.

## `build_textures`

**Contract** — creates and fills both tables and registers them under the well-known names the shaders sample. Called once, after the device exists. Allocates device storage; blocks on the uploads.

### The lighting-model table

```text
FUNCTION build_material_table() -> ()
  allocate a 3D texture: two 8-bit channels,
      width  = the diffuse-dot resolution
      height = the specular-dot resolution
      depth  = the model count
  register it under the material-table name

  FOR EACH model slice, row y, column x
    d := x / (width - 1)                    # the diffuse dot product, 0..1
    s := y / (height - 1) + a small epsilon # the specular dot product
    # The specular axis is biased by the diffuse term so that a surface facing
    # away from the light cannot show a specular highlight.
    s := s × d^(1/32)

    SELECT slice
      0 -> diffuse := d^0.75 ; specular := s^16 × 0.5    # a rough, wide response
      1 -> diffuse := d^0.90 ; specular := s^24          # a middling response
      2 -> diffuse := d      ; specular := (s × 1.01)^128 # a tight response
      3 -> # the metallic slice: three interfering ridges, so the highlight
           # breaks up rather than forming one smooth lobe
           a := |1 - |0.05 × sin(33 d) + d - s||
           b := |1 - |0.05 × cos(33 d s) + d - s||
           c := |1 - |d - s||
           diffuse := d ; specular := max(a, b, c)^24 × d^(1/7)
      otherwise -> diffuse := 0 ; specular := 0

    store (specular, diffuse) as two 8-bit values, rounded

  # The very last texel of every slice is forced to full white in both
  # channels, so that a lookup clamped to the corner returns "fully lit"
  # rather than whatever the formula happened to produce there.
  FOR EACH slice: set the (width-1, height-1) texel to (255, 255)

  upload the whole volume
```

**Invariants** — the corner override is load-bearing: the shaders clamp their lookup coordinates, so the corner texel is what a surface pointing straight at the light receives. Leaving it to the formula makes the brightest possible surface slightly dim, which is visible as a ceiling on highlight intensity.

**Notes** — the four slices are named in the original after the classical shading models they resemble, and the resemblance is approximate: they are *fitted curves*, chosen by eye to look like those models, not evaluations of them. The exponents (0.75, 0.90, 1.0 on the diffuse side; 16, 24, 128, 24 on the specular side) and the metallic slice's frequency of thirty-three and final power of one-seventh are **tuning values with no recoverable derivation**. A rebuild must reproduce them numerically to match the shipped art's appearance; deriving them is not possible and not necessary.

The specular bias exponent of one thirty-second is the one constant with an evident purpose: it is close enough to zero that it barely attenuates a lit surface and still drives the specular term to zero as the diffuse term does.

### The jitter set

```text
FUNCTION build_jitter_textures() -> ()
  # All but the last are 8-bit four-channel; each texel packs TWO 2D offsets.
  FOR EACH texture index i IN 0 .. count - 2
      allocate an 8-bit four-channel 2D texture of the jitter extent
      register it under the jitter name plus i

  FOR EACH texel position (x, y)
      offsets := generate_poisson_set(count - 1)
      FOR EACH texture index i: write offsets[i] into that texture's texel

  upload each of them

  # The last texture is floating-point and holds a rotation plus a radius,
  # for the horizon-based occlusion filter rather than for shadow filtering.
  allocate a 32-bit four-channel 2D texture of the jitter extent
  register it under the jitter name plus the last index
  FOR EACH texel position
      directions := 4, 6 or 8 by the ambient-occlusion quality setting, else 1
      angle  := a uniform random turn divided by the direction count
      radius := a uniform random value in 0..1
      write (cos angle, sin angle, radius, 0)
  upload it

  # A mipped copy of the FIRST jitter texture, for shaders that sample the
  # pattern at varying scales.
  allocate an 8-bit four-channel 2D texture, upload the first pattern,
  generate its mip chain on the device
```

**Contract of `generate_poisson_set(n)`** — produces `n` pairs of two-dimensional offsets on a 256-by-256 grid such that no two are within a Manhattan distance of 32 of each other. Rejection sampling: draw a candidate, accept it only if it clears every accepted point, repeat until enough are accepted.

**Invariants** — the minimum-distance constraint is what makes this a *Poisson-disc-like* set rather than uniform noise, and it is the whole point: shadow filtering with clustered taps produces visible blotches, and the rejection rule guarantees spread. The threshold of 32 against a 256 range is the ratio a rebuild must preserve; the absolute numbers matter only because the offsets are stored as eight-bit channels.

**Notes** — the rejection loop has no iteration cap. With the shipped counts it terminates quickly; asking for many more offsets at the same minimum distance would make it spin forever. A rebuild should cap it and fall back to the best set found.

**Notes** — three defects are visible here and all three are the same kind: an index used after its loop has finished. The float jitter texture is uploaded against the *loop counter's* final value rather than its own index, and the mipped copy is generated from a texture whose storage was declared with a single level. Both happen to work — the counter's final value equals the intended index, and the mip generation on a single-level allocation is a no-op that the sampling shaders tolerate — which is why they have survived. A rebuilder copying this structure should index explicitly and allocate the full mip chain.

The last jitter texture's direction count is read from the occlusion quality setting *at startup*, so changing that setting requires a restart to regenerate the pattern. Nothing in the engine says so.
