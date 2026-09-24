# src/Layers/xrRenderDX11/dx11StateUtils.cpp

> The rules that make state interning work: what every state field defaults to, which fields are irrelevant given the others, and how two state descriptions are compared and hashed.

**Needs** — [`dx11StateUtils.h`](dx11StateUtils.h.md) · [`xrRender/Utils/dxHashHelper.h`](../xrRender/Utils/dxHashHelper.h.md) · [`StateManager/dx11StateCacheImpl.h`](StateManager/dx11StateCacheImpl.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx11StateUtils.h`](dx11StateUtils.h.md)
**Tier floor** — T1: it hashes and compares driver description structures field by field precisely because their padding bytes are not defined.

## Purpose

Three jobs, all of them in service of the state caches in [`StateManager/`](StateManager/dx11StateCache.h.md):

1. **Translate** the engine's state vocabulary — which is the previous graphics generation's enumeration set, because that is what the shipped material files speak — into the device's.
2. **Default** each description, which is where "what does a material get if it says nothing" is answered.
3. **Normalize, compare and hash** descriptions, so that two materials expressing the same behaviour resolve to one device object.

## State

`Stateless.` Every function is a pure transformation of a description, except that the defaults consult one global: whether multisampling is enabled.

## the conversions

**Contract** — map a fill mode, cull mode, comparison function, stencil operation, blend factor, blend equation or texture addressing mode from the engine's vocabulary to the device's. Each rejects an unknown value loudly in a debug build and falls back to the most permissive value otherwise.

**Invariants** — Clockwise winding means *front-facing* and counter-clockwise means *back-facing*. That is the handedness the game's level and model data was authored with, and getting it backwards inverts culling everywhere.

**Notes** — The fill-mode conversion in the source maps wireframe to solid and solid to wireframe — the two cases are crossed. Nothing in the shipped content sets a fill mode, so the inversion is invisible in normal play and shows up only when a debug wireframe mode is requested. Treat it as a defect to fix in a rebuild, not as a convention to reproduce.

Two blend factors from the older generation (the "both source alpha" pair) have no equivalent and are left unconvertible; no shipped material uses them.

## the defaults

**Contract** — reset a description to the state a material inherits when it specifies nothing. These values *are* the engine's render-state defaults and a rebuild must match them, because the shipped material files were authored against them.

```text
rasterizer     : solid fill, cull back faces, clockwise = front, no depth bias,
                 depth clipping on, scissor off, antialiased lines off,
                 multisample = whatever the global multisampling option says
depth-stencil  : depth test on, depth write on, pass when nearer,
                 stencil test ON with keep/keep/keep and always-pass,
                 read and write masks = all bits, or the low 7 bits when
                 multisampling is on (see below)
blend          : blending off on every target, independent per-target blending off,
                 source one / destination zero / add, write all channels
sampler        : trilinear filter, clamp on all three axes, no mip bias,
                 anisotropy 1, no comparison, opaque white border,
                 unrestricted mip range
```

**Invariants** — Stencil testing defaults to *enabled with a no-op configuration* rather than disabled. That is not an accident: the deferred path uses the stencil buffer as a per-pixel classification mask throughout the frame, so a material that does not mention stencil must still not disturb it.

**The masks lose their top bit when multisampling is on.** With multisampling the renderer needs one stencil bit of its own to mark edge pixels for the per-sample resolve, so it reserves the highest bit and narrows every material's usable mask to the remaining seven. A rebuild that adds a multisampled path must reserve the same bit or the edge pass will read someone else's mark.

## `ValidateState` — normalization

**Contract** — force every field that cannot affect behaviour, given the other fields, to a fixed value. Called before hashing and comparing. This is the function that turns an unbounded space of descriptions into a handful of objects.

```text
FUNCTION normalize(depth_stencil)
  IF depth test off    THEN depth write := on, depth function := nearer
  IF stencil test off  THEN masks := all, both faces := keep/keep/keep, always

FUNCTION normalize(blend)
  IF no target has blending on THEN every target := one / zero / add
  ELSE  rewrite any *colour* factor appearing in an *alpha* slot as its alpha
        counterpart, because the device treats them as equivalent there and
        would otherwise hand back a description that no longer matches

FUNCTION normalize(sampler)
  IF no axis addresses by border THEN border colour := transparent black
  IF the filter is not anisotropic THEN anisotropy := 1

FUNCTION normalize(rasterizer)
  nothing: every rasterizer field is always meaningful
```

**Invariants** — The alpha-factor rewrite is the subtle one and it is required, not optional: the cache confirms a hash hit by reading the description *back out of the created object*, and the device normalizes colour-valued alpha factors on its way in. Without the same rewrite here, every such description would miss its own cached object forever and the cache would grow without bound.

## the comparisons

**Contract** — field-by-field equality. Fields made irrelevant by a disabled feature are skipped entirely (a depth-stencil description with depth testing off compares equal regardless of its depth function), which is belt-and-braces alongside normalization.

**Notes** — The source states why a byte comparison is not used: these structures contain padding the compiler does not initialize, so two descriptions with identical meaning can differ in their bytes.

Two exclusions in the sampler comparison and hash are deliberate and load-bearing: **anisotropy and mip bias are not part of a sampler's identity**, because they are global settings the cache overwrites on every sampler it holds. Including them would fragment the cache across a value that is the same everywhere.

The blend comparison examines only the first four targets while the hash covers all eight; the source marks this as a port-time fix. The consequence is real though currently harmless: two blend descriptions differing only in targets four to seven hash differently, so they still get separate objects — the effect is a missed *reuse*, not a wrong state. Four is also the number of targets this renderer ever binds.

## the hashes

**Contract** — accumulate a 32-bit checksum over exactly the fields the corresponding comparison examines. Used only as a pre-filter for the cache lookup; a collision is resolved by the comparison.

**Invariants** — Hash and comparison must cover the same field set, or the cache either misses (hash differs, meaning equal) or thrashes (hash equal, comparison always false). The sampler pair keeps this agreement by excluding the same two fields from both.
