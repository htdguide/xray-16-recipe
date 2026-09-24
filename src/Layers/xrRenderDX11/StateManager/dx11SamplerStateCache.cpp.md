# src/Layers/xrRenderDX11/StateManager/dx11SamplerStateCache.cpp

> Interns sampler settings behind stable handles, binds a whole stage's sampler set in one call, and rebuilds every sampler in place when the player changes texture filtering.

**Needs** — [`dx11SamplerStateCache.h`](dx11SamplerStateCache.h.md) · [`../dx11StateUtils.h`](../dx11StateUtils.h.md) · [`../dx11HW.h`](../dx11HW.h.md)
**Used by** — [`dx11SamplerStateCache.h`](dx11SamplerStateCache.h.md)
**Tier floor** — T1: it owns device objects and rebinds a fixed-size slot array per stage per draw.

## Purpose

Samplers are interned exactly like the pipeline state objects, with one difference that drives the whole design: **two sampler fields are global settings the player controls**, not properties of a material. Maximum anisotropy and the mip-level bias come from the graphics options, apply to every sampler in the game, and can change at run time.

So the cache hands out **handles — indices — rather than pointers**. When the player changes the anisotropy setting, every cached sampler is rebuilt with the new value and the array slot is overwritten. Every material's stored handle stays valid and now names the new object. A rebuild that hands out object references instead must find and patch every reference; a rebuild that hands out indices does not.

## State

```text
RECORD SamplerCache
  entries        : list<{key : int (32-bit), object : handle}>   # index == handle
  max_anisotropy : int    # global, clamped to 1..16
  mip_bias       : real   # global
```

Invariant: a handle is an index into `entries` and remains meaningful across a settings change. The sentinel handle (all bits set) means "no sampler in this slot".

## `GetState`

**Contract** — interns a sampler description and returns its handle. **Overwrites** the description's anisotropy and mip bias with the global values before interning, so a material cannot specify them; normalizes, hashes, confirms by read-back comparison, creates on miss.

**Notes** — Overwriting before normalizing is deliberate: normalization is what forces anisotropy back to a fixed value when the filter is not anisotropic, so the global setting cannot fragment the cache across filter modes that ignore it.

## `VSApplySamplers` and its five siblings

**Contract** — bind one stage's sampler set. Expands the compact handle list into a full slot-count array — slots beyond the list, and gaps inside it, become nothing — and binds *all* slots of that stage in a single call.

```text
FUNCTION apply(stage, context, handles)
  slots = array of stage's full sampler-slot count, all none
  FOR i IN 0 .. handles.count - 1
    IF handles[i] is not the sentinel THEN slots[i] = entries[handles[i]].object
  device.bind_samplers(stage, context, slot 0, all slots)
```

**Notes** — There is no redundancy filtering here: the whole stage's array is rebound on every pass-state application. That is the opposite choice from the shader-resource cache next door, which tracks a dirty range. The justification is that a sampler binding is cheap and the array is small, whereas texture bindings change far more often and are wider. A rebuild may unify the two; it should then measure, because the asymmetry is deliberate.

## `SetMaxAnisotropy` / `SetMipLODBias`

**Contract** — change a global filtering parameter. Returns immediately if unchanged. Otherwise walks every cached entry, reads its description back from the object, applies the new value, re-normalizes, releases the old object and creates a replacement **in the same slot**. Anisotropy is clamped to 1..16, the range the hardware defines.

**Notes** — The source warns that doing this often fragments device memory. It happens on a settings change, so never in a frame loop — but a rebuild that exposes these as live sliders needs to know.

This function is also the reason normalization is centralized: the new anisotropy is written unconditionally and normalization then decides whether the filter mode actually uses it. Putting that decision here as well would let the two copies drift.
