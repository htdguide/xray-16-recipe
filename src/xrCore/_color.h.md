# src/xrCore/_color.h

> Colour in two forms — four floats for arithmetic, one packed word for the device — and the exact conversion between them.

**Needs** — [`xr_types.h`](xr_types.h.md) · [`_bitwise.h`](_bitwise.h.md) · [`math_constants.h`](math_constants.h.md)
**Used by** — [`FS.h`](FS.h.md) · [`PPInfo.hpp`](PostProcess/PPInfo.hpp.md) · [`_stl_extensions.h`](_stl_extensions.h.md) · [`vector.h`](vector.h.md) · [`xr_ini.h`](xr_ini.h.md)
**Tier floor** — T1: the packed form's byte order is what a graphics device and several on-disk structures expect, so the shift positions are frozen.

## Purpose

Two representations and the bridge between them. The packed form is what vertex buffers, lightmaps and material descriptions store; the float form is what anything that blends, scales or interpolates uses. Getting the packing wrong swaps red and blue on every surface in the game, which is why the shift positions below are stated explicitly.

## The packed form

One 32-bit word, with the channels at fixed shifts:

```text
bits 31..24 : alpha
bits 23..16 : red
bits 15..8  : green
bits  7..0  : blue
```

**This ordering is frozen.** It is the ordering the graphics device of the era consumed and the ordering the shipped level and model data store. Two constructors exist — one taking the channels in alpha-red-green-blue order, one in red-green-blue-alpha order — and they produce the same word; the second exists only so call sites can name the channels in the order that reads naturally there.

A variant fixes alpha to fully opaque, and there are extractors for each channel, a substitution that replaces only the alpha, and a red/blue swap used where data arrives in the opposite convention. The swap is its own inverse and is exposed under both names for readability.

**Invariant** — the eight-bit clamp used by every float-to-packed conversion clamps to `[0, 255]` *after* flooring, not before. The order matters: flooring a value slightly above 255 and then clamping gives 255; clamping to 255.0 and then flooring gives 255 as well, but clamping a negative to 0.0 and flooring gives 0 where flooring first gives a large negative that the clamp then catches. Both orders happen to agree here; the code takes the first and a rebuild should too, since it is the one that tolerates a not-a-number by clamping it rather than by producing an arbitrary byte.

```text
FUNCTION pack(r, g, b, a: real) -> int (32-bit)
  RETURN (clamp8(floor(a * 255)) << 24)
      OR (clamp8(floor(r * 255)) << 16)
      OR (clamp8(floor(g * 255)) <<  8)
      OR  clamp8(floor(b * 255))
```

**Notes** — the scale is 255 and the rounding is *floor*, not round-to-nearest. A value of exactly 1.0 maps to 255 and a value of 0.999 maps to 254; a rebuild that rounds instead shifts every converted colour by up to one step, which is invisible on a single surface and visible as banding on a gradient. The quantizations in [`NET_utils.cpp`](NET_utils.cpp.md) round to nearest by adding a half first — these deliberately do not, and the difference is not explained anywhere.

## `Fcolor` — the float form

```text
RECORD Fcolor
  r, g, b, a : real
# no invariant: channels are NOT clamped to [0,1]. Values outside the range
# are routine — a light's colour is scaled past 1 before being tone-mapped,
# and a subtractive effect carries negatives.
```

**Contract** — constructible from four components, from a packed word, or uninitialized. Convertible back to a packed word. Every operation is total, allocates nothing, mutates in place and returns the record so calls chain.

### Operations

- **Component arithmetic** — add, subtract, multiply by a scalar, and modulate (per-channel multiply), each in a three-channel form that leaves alpha alone and a four-channel form that does not. The three/four split is the interesting part: alpha is *usually* not a colour and must not be scaled with the others, so the default is to leave it.
- **`lerp`** — linear interpolation between two colours by a factor, all four channels. A three-colour form interpolates across two segments, switching at the halfway point: below a half it blends the first pair at twice the factor, above it blends the second pair at twice the factor minus one. That is a piecewise-linear ramp through a mid colour, used by the weather system for dawn/day/dusk.
- **`adjust_contrast`** — scales each of the three colour channels away from **0.5**, the fixed pivot. A factor above one increases contrast.
- **`adjust_saturation`** — computes a grey by the weights `(0.2125, 0.7154, 0.0721)` and blends each channel toward or away from it. Those are the standard luminance weights for the colour primaries the engine assumes, and they are *different* from the flat thirds used as the neutral value in [`PostProcess/PPInfo.hpp`](PostProcess/PPInfo.hpp.md) — the two systems disagree about what grey means, deliberately or not.
- **`intensity`** — a flat average of the three channels, **not** the weighted luminance above. The source marks this as a known inconsistency. A rebuild should decide which it wants and use it in both places, accepting that the shipped content was tuned against these two different answers.
- **`negative`** — one minus each channel, alpha included.
- **`magnitude_rgb`**, **`normalize_rgb`** — treat the three channels as a vector; normalizing asserts a non-zero magnitude.
- **`similar_rgb`, `similar_rgba`** — per-channel comparison within a tolerance.
- **`get_windows`, `set_windows`** — the *other* packing, with red and blue exchanged relative to the one above. Used only where a colour crosses into a host interface that uses that convention. Note that this pair, unlike the main conversion, does **not** clamp — it truncates each scaled channel to a byte, so a channel above one wraps. Callers are expected to pass in-range values.

**Notes** — the arithmetic operations are declared as compile-time-evaluable where they can be, which lets a palette be built at build time. That is incidental; the values are what matter.

`_valid` reports whether all four channels are finite. It is used by the checked-build assertions throughout the rendering and effect code, where a single non-finite colour propagates into a constant buffer and blanks the screen with no other symptom.
