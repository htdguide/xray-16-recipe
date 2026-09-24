# src/xrSound/SoundRender_EffectsA_EAX.h

> Declares the vendor-extension reverb backend, and compiles to nothing where that extension's
> headers are absent.

**Needs** — [`SoundRender_Effects.h`](SoundRender_Effects.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`SoundRender_CoreA.cpp`](SoundRender_CoreA.cpp.md) · [`SoundRender_EffectsA_EAX.cpp`](SoundRender_EffectsA_EAX.cpp.md) · [`SoundRender_Environment.cpp`](SoundRender_Environment.cpp.md)
**Tier floor** — T1: it names vendor function-pointer types and property identifiers.

## Purpose

Declares the reverb backend implemented in
[`SoundRender_EffectsA_EAX.cpp`](SoundRender_EffectsA_EAX.cpp.md).

The whole declaration — the class and the flag that says reverb is available at all — exists only
when the vendor headers are present. That is the availability model for this feature: a build
without the extension has no reverb type, no reverb code, and preset fields that fall back to
built-in defaults, and it works. A rebuild should make reverb an optional module with a null
filling rather than a compile-time presence test, and will then get the same behaviour without the
conditional compilation.

## Exported units

- **`EaxEffects`** — the backend; holds the two resolved entry points and the two probed capability
  flags (immediate and deferred).
- **`eax_set` / `eax_get`** — one property by identifier, with an explicit size.
- **`query_support` / `test_support`** — the round-trip probe described in the implementation twin.
- The four operations of [`SoundRender_Effects.h`](SoundRender_Effects.h.md).
