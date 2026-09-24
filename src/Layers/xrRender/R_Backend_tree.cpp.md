# src/Layers/xrRender/R_Backend_tree.cpp

> The wind channel: the eight named constants through which the engine tells a vegetation shader how to bend a tree and how to unpack its quantized vertices.

**Needs** — [`R_Backend_tree.h`](R_Backend_tree.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`FTreeVisual.h`](FTreeVisual.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`R_Backend_tree.h`](R_Backend_tree.h.md)
**Tier floor** — T3: eight bindings and a guarded write each.

## Purpose

Wind-animated vegetation is the one visual class whose *geometry* is computed in the shader from constants rather than supplied by the engine. The engine never moves a leaf; it publishes a wind direction, a wave description and a time value, and the vertex program displaces each vertex by an amount that depends on the vertex's own baked sway weight. This module is the set of named slots that conversation happens through, and the names are part of the shipped material contract.

The same eight slots also carry the *dequantization* constants, because a tree's vertices are stored as fixed-point integers and only the shader knows how to turn them back into positions. That pairing — animation constants and unpacking constants in one binding group — is why this is one module and not two.

## State

```text
RECORD TreeCache
  loc_m_xform_v : optional<ConstantLocation>   # shader name "m_xform_v"
  loc_m_xform   : optional<ConstantLocation>   # shader name "m_xform"
  loc_consts    : optional<ConstantLocation>   # shader name "consts"
  loc_wave      : optional<ConstantLocation>   # shader name "wave"
  loc_wind      : optional<ConstantLocation>   # shader name "wind"
  loc_c_scale   : optional<ConstantLocation>   # shader name "c_scale"
  loc_c_bias    : optional<ConstantLocation>   # shader name "c_bias"
  loc_c_sun     : optional<ConstantLocation>   # shader name "c_sun"
```

**Invariants**

- All eight shader-side names are **frozen**. They are short and unprefixed — `wave`, `wind`, `consts` — which is a hazard worth naming: they are in the same flat name space as every other constant in a pass, so a rebuild may not reuse those words for anything else.
- Locations belong to the currently installed pass and are cleared by `unmap` on every pass change.
- This record publishes only; every value it sends is computed by the vegetation visual (per instance) or by a once-per-frame setup derived from the weather state.

## What travels through each slot

The names carry no self-description, so the contract is only recoverable from the caller. It is:

```text
m_xform     : matrix   # the tree's object-to-world transform
m_xform_v   : matrix   # the same composed with the view matrix, i.e. object-to-camera.
                       # Published only on the deferred-shading renderers, which need
                       # the sway computed in camera space; the oldest renderer omits it.
consts      : (s, s, 0, 0)          # s = the dequantization scale, see below
wave        : (wx, wy, wz, phase)   # the weather's three wave frequencies and a phase
                                    # that advances with global time; all four divided
                                    # by a full turn so the shader can use them directly
wind        : (dx, 0, dz, 0)        # unit wind direction in the horizontal plane,
                                    # rotating once per the weather's rotation period,
                                    # scaled by the weather's amplitude
c_scale     : (r, g, b, hemi)       # per-tree colour and ambient scale, times a global
c_bias      : (r, g, b, hemi)       # per-tree colour and ambient offset, times a global
c_sun       : (sun_scale, sun_bias, 0, 0)
```

**Invariants**

- **The dequantization scale is the reciprocal of the vertex quantizer**, and the quantizer is fixed by the vegetation vertex format: a tree's positions are stored as signed 16-bit integers spanning a tile of 16 world units, so the quantizer is `32768 / 16 = 2048` steps per unit and the scale is `1/2048`. Both numbers are frozen by the shipped tree meshes. The scale is published twice in the same constant (first and second components) because the shader reads it through two different swizzles; the third and fourth components are unused.
- The **colour scale and bias are pre-halved when the visual is loaded** and then multiplied by a global correction factor at draw time. The halving is baked into the loader, not into this module, and the two together are what maps the authored range onto the shader's expected range. A rebuild that applies neither, or both in the wrong place, gets vegetation that is uniformly too bright or too dark — the most common symptom of getting this pair wrong.
- On the oldest renderer the weather's ambient colour is **added into the bias** at draw time, because that renderer has no separate ambient term to add it to later. The deferred renderers do not, and instead scale the correction factor up by one third. Both facts are decisions of the vegetation visual, recorded here because they are invisible from the constant names.
- The wave vector's fourth component is a phase, not a frequency: it advances with wall-clock time multiplied by the weather's tree speed. Dividing all four components by a full turn is what lets the shader take a fractional part instead of a trigonometric reduction.

## `set_m_xform_v` / `set_m_xform` / `set_consts` / `set_wave` / `set_wind` / `set_c_scale` / `set_c_bias` / `set_c_sun`

**Contract** — Each publishes its value into the corresponding binding, or does nothing when the current pass did not bind that name. No validation, no caching, no ordering requirement between them. Cheap enough to be called unconditionally for every vegetation instance.

**Notes** — There is deliberately no redundancy filter here even though most of these values are identical across every tree in a frame. The values are recomputed once per frame and then re-published per instance; comparing eight four-component values costs about what writing them costs, and the constant cache one level down already filters identical writes.

## `unmap`

**Contract** — Clears all eight bindings. Called on every pass change and on command-list invalidation.
