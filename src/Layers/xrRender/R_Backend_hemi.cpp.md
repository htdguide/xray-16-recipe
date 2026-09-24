# src/Layers/xrRender/R_Backend_hemi.cpp

> The per-object lighting channel: the six-face ambient cube, the surface material selector, and three free-form data slots that a model carries into its own shaders.

**Needs** — [`R_Backend_hemi.h`](R_Backend_hemi.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`RenderVisual.h`](../../Include/xrRender/RenderVisual.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`R_Backend_hemi.h`](R_Backend_hemi.h.md)
**Tier floor** — T3: six named constants and a guarded write each. It sits at T2 only because it is inlined into the per-object path of the draw loop.

## Purpose

Two things vary per *object* rather than per pass or per frame: how much of the sky reaches it (the ambient term), and what kind of surface it is (the material selector that picks a lighting response curve). Both are computed elsewhere — the ambient cube by the light-tracking pass, the material index by the texture description data — and both must arrive in the shader under names the shipped shader sources already use. This module is that delivery channel and nothing more.

It also carries three slots that are not lighting at all: a camouflage matrix, a free-form four-component vector, and a "living object" vector holding health, radiation, condition and infra-red visibility. They live here because here is where per-visual data is published, not because they are hemispherical.

## State

```text
RECORD HemiCache
  loc_pos_faces    : optional<ConstantLocation>   # shader name "hemi_cube_pos_faces"
  loc_neg_faces    : optional<ConstantLocation>   # shader name "hemi_cube_neg_faces"
  loc_material     : optional<ConstantLocation>   # shader name "L_material"
  loc_camo_data    : optional<ConstantLocation>   # shader name "m_obj_camo_data"
  loc_custom_data  : optional<ConstantLocation>   # shader name "m_obj_custom_data"
  loc_entity_data  : optional<ConstantLocation>   # shader name "m_obj_entity_data"
```

**Invariants**

- The six shader-side names are **frozen**: the shipped high-level shader sources declare them verbatim, and a rebuild that renames one silently loses that channel — the write goes through a `none` binding and does nothing, with no error.
- Every location is valid only for the currently installed pass and must be cleared when the pass changes (`unmap`). Same rule, same reason, as the transform cache.
- This record holds no *values*, only places. The values live in the command list (`ambient`, `ambient_cube`, `sun`) or on the visual being drawn; this module is a pure publisher.

## The ambient cube

**Contract** — Ambient light arriving at an object is stored as six scalars, one per axis direction, and published as two three-component constants: the positive faces (+X, +Y, +Z) in one and the negative faces (−X, −Y, −Z) in the other. The fourth component of each is written as zero.

**Invariants** — The face order inside each constant is **+X, +Y, +Z** and **−X, −Y, −Z**, matching the face enumeration the light-tracking pass fills. Shuffling the order lights objects from the wrong side and produces a plausible-looking image, which is why it is worth stating: this is not a bug that announces itself.

**Notes** — Six scalars rather than a set of spherical-harmonic coefficients is the whole design: it is the cheapest ambient representation that still has a direction, it is reconstructed in the shader with three lerps against the surface normal's sign, and it is what the shipped shaders expect. The unused fourth component is not padding to be reclaimed — the constant is a four-component register either way, and writing a defined zero is cheaper than leaving whatever the previous object left there.

## `set_material` — the material selector and the two lighting scalars

**Contract** — Publishes four values into one constant: the object's hemispherical ambient term, its sun visibility term, an unused zero, and the **material coordinate**. Does nothing if the pass did not bind `L_material`.

**Notes** — The material coordinate is a texture coordinate, not an index: the material index is turned into the centre of one of four rows in a lookup texture holding the shipped lighting response curves. The conversion and the divisor by four are described where it is computed, in [`R_Backend_Runtime.cpp`](R_Backend_Runtime.cpp.md#apply_lmaterial). The third component is a hole in the packing with no discoverable consumer.

## `c_update` — publish a visual's own shader data

**Contract** — Given the visual about to be drawn, publishes its three per-object data slots into whichever of `m_obj_camo_data`, `m_obj_custom_data`, `m_obj_entity_data` the current pass bound. Reads through the visual's render-data record; that record is always present on a visual that reaches the draw stream.

```text
RECORD VisualShaderData          # carried on every visual, per instance
  camo_data    : matrix          # camouflage transform, consumed by camo shaders
  custom_data  : (real, real, real, real)   # meaning is agreed per-shader, engine does
                                            # not interpret it; default all zero
  entity_data  : (real, real, real, real)   # health, radiation, condition, infra-red
                                            # visibility; default (-2, -2, -2, 0)
```

**Invariants** — The sentinel `-2` in the first three components of the entity data means *this object has no such property*, and the shaders test against it. Zero is a legitimate value (a corpse has zero health), so the absence sentinel has to sit outside the valid range; the valid range is `0..1`, so any negative would do and `-2` is the one the shipped shaders compare against.

**Notes** — These three slots are an extension point rather than an engine feature: the engine defines where the bytes go and what the entity-data components mean, and leaves the custom slot's meaning entirely to whichever shader reads it. A rebuild must keep all three, because shipped and modded shaders alike read them.

## `unmap`

**Contract** — Clears all six bindings. Called on every pass change and on command-list invalidation.
