# src/Layers/xrRender/blenders/Blender_Particle.cpp

> The particle material: one texture, one of six named blend modes, and nothing else.

**Needs** — [`Blender_Particle.h`](Blender_Particle.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

Every particle system in the shipped data names a material using this template, and the only thing that distinguishes smoke from a muzzle flash from a blood spray, at the material level, is which of six blend modes it selected. The six are frozen: the selector is stored as an index, and the labels written beside it are for the authoring tools.

Forward renderer filling; the deferred renderers use [`Blender_Particle_deferred.cpp`](Blender_Particle_deferred.cpp.md).

## State

```text
RECORD Parameters
  blend_mode : enum, one of 6, default 0    # stored as an index; see below
  clamp      : bool, default true           # clamp the texture instead of wrapping
  alpha_ref  : int in [0,255], default 32   # loaded, and never used — see Notes
```

**The six modes, in their frozen order:**

```text
0  SET        replace the frame; an alpha test at 200 does the cutting
1  BLEND      source alpha over inverse source alpha — ordinary translucency
2  ADD        one plus one — fire, sparks, light
3  MUL        destination colour times zero-source — darkening smoke
4  MUL_2X     destination times source, doubled — contrast without brightening
5  ALPHA-ADD  source alpha over one — additive weighted by the particle's own alpha
```

**Invariants** — the index, not the label, is what the data stores. Reordering the six breaks every shipped particle effect.

## `Save` / `Load`

**Contract** — the blend selector with its six labels inlined, then the clamp flag, then the alpha reference. The selector's option count is **re-asserted after reading**, because the stored record carries a count that a file written by an older tool may understate.

## `Compile`

```text
FUNCTION compile(context)
  SELECT blend_mode
    SET       -> pass with fog, depth test on, depth WRITE ON, no blend,
                 alpha test at 200
    BLEND     -> depth test on, depth write off, blend src-alpha : inv-src-alpha,
                 alpha test at 0
    ADD       -> depth test on, depth write off, blend one : one, alpha test at 0
    MUL       -> depth test on, depth write off, blend dest-colour : zero, alpha test at 0
    MUL_2X    -> depth test on, depth write off, blend dest-colour : src-colour,
                 alpha test at 0
    ALPHA-ADD -> depth test on, depth write off, blend src-alpha : one, alpha test at 0

  bind s_base <- instance texture 0, addressed clamped or wrapped per the flag
```

**Invariants**

- **Only the replace mode writes depth.** Every other mode is a translucent particle that must not occlude the particles behind it. The replace mode is the exception because it is used for opaque, cut-out particles — debris chunks, shell casings — which do occlude, and its alpha test at 200 is what makes the cut-out sharp.
- Every mode enables the alpha test, at 200 for the replace mode and at 0 for the rest. A reference of 0 is not a no-op on this hardware: it discards fully transparent texels before they reach the blend unit, which is a measurable saving for a screen full of mostly-empty particle quads.
- The template declares no capabilities: particles take neither detail nor lightmap.

**Notes** — The alpha reference parameter is loaded from every shipped particle material and **never read**: the references used are the two literals above. The parameter predates the fixed pair and cannot be removed without changing the parameter block's length. A rebuild must still consume its bytes.

Clamping matters more than it looks: a particle quad's uv range is set by the effect's frame animation, and a wrapped sampler shows the opposite edge of the sprite sheet bleeding in at the seam. Clamping is the default for that reason.
