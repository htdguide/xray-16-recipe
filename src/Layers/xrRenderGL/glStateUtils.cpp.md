# src/Layers/xrRenderGL/glStateUtils.cpp

> The state translation table, and the three places where the two APIs genuinely disagree rather than merely spell things differently.

**Needs** — [`glStateUtils.h`](glStateUtils.h.md) · [`CommonTypes.h`](CommonTypes.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`glStateUtils.h`](glStateUtils.h.md)
**Tier floor** — T1: it produces the exact enumerants a driver call takes.

## Purpose

Seven of the eight converters here are flat one-to-one maps and deserve no more than a table. The value of the page is the three disagreements, because those are the ones a rebuild will hit whatever API it targets, and two of them silently change what the player sees if you get them wrong.

## State

`Stateless.`

## `convert_cull_mode(mode)`

**Contract** — maps the engine's cull selector to the face this API discards. **The mapping is crossed**: the engine's "cull clockwise" becomes "discard back faces" and "cull counter-clockwise" becomes "discard front faces". Any other value is a fault.

**Invariants** — the crossing is not a bug and must be reproduced. The two APIs disagree on which winding is front-facing by default *and* the engine's enumerant names describe the winding to remove rather than the face to remove. The shipped level and model geometry has a fixed winding; get this backwards and every surface in the game shows its inside.

**Notes** — the "cull nothing" case never reaches this converter — the caller in [`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md) intercepts it and disables face culling altogether, because on this API "no culling" is a switch, not a mode.

## `convert_texture_filter(requested, current_combined, is_mip_setting)`

**Contract** — folds one filter request into the device's combined minification-and-mip enumerant. Takes the enumerant's current value and returns the new one. When `is_mip_setting` is false the request selects the *minification* quality; when true it selects the *mip* quality.

```text
FUNCTION convert_texture_filter(requested, current, is_mip) -> int
  # The device's combined enumerant is bit-structured: one bit selects linear
  # vs nearest within a level, one selects linear vs nearest between levels,
  # and one says whether mip levels are consulted at all.
  SELECT requested
    none ->
        IF is_mip THEN RETURN current WITH mip-linear and mip-enable cleared
        FAIL WITH "a filter of none is only meaningful for the mip setting"
    point ->
        IF is_mip THEN RETURN current WITH mip-linear cleared, mip-enable set
        RETURN current WITH within-level linear cleared
    linear, anisotropic ->
        IF is_mip THEN RETURN current WITH mip-linear and mip-enable set
        RETURN current WITH within-level linear set
    otherwise -> FAIL
```

**Invariants** — anisotropic and linear produce the same enumerant here. Anisotropy is *not* a filter mode on this API; it is a separate sampler parameter, set elsewhere ([`glState.cpp`](glState.cpp.md)) and only when the device offers it. A material asking for anisotropic filtering therefore gets linear filtering plus, if available, an anisotropy setting — which is the right behaviour, but a rebuild that treats anisotropic as a filter *mode* will drop it entirely on a device without the extension, where the original degrades to linear.

**Notes** — Exploiting the bit structure of a device enumerant is a liberty the specification happens to permit and a rebuild should not copy. The honest form is to keep the two filter choices as separate fields in the block and compose the device value once, at apply time.

## `convert_blend_factor(factor)`

**Contract** — maps a blend factor. All eleven factors the shipped materials use map one-to-one: zero, one, source colour and its complement, source alpha and its complement, destination alpha and its complement, destination colour and its complement, and saturated source alpha. Anything else faults.

**Notes** — Five Direct3D 9 factors are deliberately unmapped: the two "both source alpha" forms, the two constant-blend-factor forms, and the dual-source colour forms. None appears in the shipped material set. A rebuild can leave them unimplemented but should fault loudly rather than substitute, because a silently-wrong blend factor is very hard to see in a screenshot and very obvious in motion.

## `convert_stencil_op(op)`

**Contract** — maps a stencil operation. The subtlety is at the two increment/decrement pairs: the engine's *saturating* increment and decrement map to this API's plain increment and decrement (which clamp), and the engine's *plain* increment and decrement map to this API's wrapping forms. The names cross over; the semantics line up. Both forms are used — the deferred lighting path marks pixels with a saturating increment and the stencil-optimization path relies on wrapping — so neither can be collapsed into the other.

## `convert_comparison_func(func)` · `convert_blend_op(op)` · `convert_fill_mode(mode)` · `convert_address_mode(mode)`

**Contract** — flat maps with no surprises.

- comparison: never, less, equal, less-or-equal, greater, not-equal, greater-or-equal, always.
- blend operation: add, subtract, reverse subtract, minimum, maximum.
- fill mode: point, wireframe, solid.
- address mode: wrap, mirror, clamp-to-edge, clamp-to-border. The Direct3D "mirror once" mode has no equivalent and is unmapped; no shipped material uses it.

**Notes** — every converter faults on an unrecognized value in a checked build and returns a benign default in a shipping build (always-pass, add, solid, clamp, keep). That pairing — loud in development, survivable in the field — is the right policy for a translation layer over frozen data, because the data can contain a value nobody anticipated and the player should still get a frame.
