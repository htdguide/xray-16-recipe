# src/Layers/xrRender/tss_def.cpp

> The recorded state list: a material pass's state assignments accumulated as a flat set, then translated into the device's state objects.

**Needs** — [`tss_def.h`](tss_def.h.md) · [`tss.h`](tss.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`tss.h`](tss.h.md) · [`tss_def.h`](tss_def.h.md)
**Tier floor** — T1: the translation is a switch over a frozen numeric state vocabulary into a device's own description structures, including a bit-level construction of a sampler filter value.

## Purpose

A material pass in the shipped data is described in a state vocabulary from the graphics API of 2003: set this render state to that value, set this sampler's filter to that. Modern devices do not have individually settable states — they have immutable state *objects* built from a description and bound as a unit.

This file bridges the two. A pass records its assignments into a flat list; when the pass is finalised, the list is translated into a rasterizer description, a depth-stencil description, a blend description and a set of sampler descriptions. That translation is the file's whole content, and every line of it is a mapping a rebuild must reproduce, because the numeric values come from the shipped material files.

**The old vocabulary is frozen.** It is not a legacy convenience: the material files that ship with all three games name these states and values, so a rebuild must understand them whatever it targets.

## `set_render_state`, `set_texture_stage_state`, `set_sampler_state`

**Contract** — Record one assignment, replacing any earlier assignment to the same target. The "same target" key differs per category: a render state is keyed by which state; a texture-stage or sampler state by the stage or slot *and* which state.

```text
FUNCTION set(category, target..., value)
  remove the existing assignment with this category and target, if any
  append a new assignment
```

**Invariants** — Removal is a linear scan and appends go to the end, so the list ends up in *last-assignment order*, not first. This matters because `equal` compares the list literally: two passes that set the same states in different orders compare unequal, and the renderer will change state between them needlessly. That is a known inefficiency, not a correctness problem. A rebuild should sort the list by target before comparing.

## `equal`

**Contract** — Whether two lists hold the same assignments in the same order. Implemented as a raw comparison of the whole array, which is why the record has no padding-sensitive fields and no pointers.

## `record`

**Contract** — Turn the list into whatever the backend binds. The three backends differ fundamentally:

- On the oldest, the device itself records a state block: the assignments are replayed against the device between a begin and an end, and the device hands back an opaque block. One substitution happens during the replay — an anisotropic magnification filter is downgraded to linear, because magnification has no anisotropy to exploit and some drivers reject it.
- On the modern Direct3D backend, a state object is constructed from the list by the four description-filling functions below.
- On the OpenGL backend, a state object is constructed by replaying render-state and sampler-state assignments into it. Texture-stage assignments are skipped entirely: the fixed-function texture environment they describe does not exist, and every material that relies on it has a programmable equivalent. The maximum-anisotropy value is **overwritten from the current console setting** during the replay, so that changing the anisotropy setting takes effect on the next pass record without reloading materials.

## `extract_reference_values`

**Contract** — Pull out the two values that are not part of any state object and must be set separately at bind time: the stencil reference value and the alpha-test reference value. Both are per-draw parameters on a modern device rather than pieces of immutable state, which is why they cannot be folded into a description.

## `fill_rasterizer_description`

**Contract** — Translate the fill mode, cull mode and scissor-enable assignments into a rasterizer description. Everything else a rasterizer description holds — depth bias, clipping, multisample and antialiased-line flags — is left at the caller's defaults.

**Notes** — Depth bias and slope-scaled depth bias are explicitly *rejected*: a material that sets either trips an assertion. The old and new devices scale depth bias differently — the old one in device depth units, the new one in units of the depth buffer's least significant bit — and no conversion was ever worked out. No shipped material uses them, so the assertion has never fired. A rebuild that wants depth-biased materials (for decals, say) must solve this; the decal path instead offsets geometry in world space, which is why [`dxWallMarkArray.cpp`](dxWallMarkArray.cpp.md)'s material does not need bias.

## `fill_depth_stencil_description`

**Contract** — Translate depth enable, depth write, depth comparison, stencil enable, the stencil read and write masks, and the four stencil operations and comparison for each face. Front and back faces are addressed by separate state names in the old vocabulary and map onto the two halves of the modern description.

## `fill_blend_description`

**Contract** — Translate alpha-to-coverage, the source and destination blend factors and blend operation for colour and for alpha separately, blend enable, and the four per-target colour write masks.

**Invariants** — Every state except the colour write masks is written to **all eight** render targets. The old vocabulary has no per-target blend state, so a material's blend settings must apply uniformly; only the write mask has per-target names and only the first four targets have them. A rebuild with independent per-target blending gains nothing, because no shipped material can express it.

## `fill_sampler_descriptions`

**Contract** — Translate every sampler assignment into the matching sampler description, then validate. Takes a base slot index so a pass's samplers can be offset into a stage's slot range; assignments falling outside the range are ignored, which is how one recorded list serves several stages.

The filter value is the interesting part. The old vocabulary sets magnification, minification and mip filters independently, plus separate anisotropic and comparison flags; the modern description packs all five into a single enumerated value. The translation builds that value **bitwise**, because the enumeration is constructed rather than arbitrary:

```text
bit 0  mip filter is linear
bit 2  magnification filter is linear
bit 4  minification filter is linear
bit 6  anisotropic
bit 7  comparison
```

Each assignment sets or clears its own bit and leaves the rest. This is a real dependency on the numeric layout of the target API's filter enumeration; a rebuild against a different API must write an explicit mapping instead.

The remaining mappings are direct: the three address modes, the mip level-of-detail bias (which arrives as a real number smuggled through an integer field, and must be reinterpreted rather than converted), the maximum anisotropy, the comparison function, the border colour (unpacked from a packed thirty-two-bit colour, with the alpha byte in the high position), and the minimum and maximum level of detail.

Validation, after every assignment is applied:

- A sampler marked anisotropic has all three linear-filter bits forced on. Anisotropic filtering with a point filter is not a thing the device accepts, and some materials set only the anisotropic flag.
- The minimum level of detail must not exceed the maximum; when it does, the maximum is raised to match. This protects against a material that sets a minimum without a maximum.

**Notes** — Two states in the old vocabulary have no modern equivalent and are silently dropped: the alpha-test comparison function (modern devices have no alpha test; the shipped shaders perform it themselves) and the whole texture-stage-state category (fixed-function texture combining). Every shipped material that used them has a programmable path; a rebuild that does not provide one will render those materials wrongly rather than fail.
