# src/xrSound/SoundRender_EffectsA_EAX.cpp

> The shipped reverb backend: detect a vendor listener-reverb extension, probe every parameter it
> claims, and push a preset through it.

**Needs** — [`SoundRender_EffectsA_EAX.h`](SoundRender_EffectsA_EAX.h.md) · [`SoundRender_Effects.h`](SoundRender_Effects.h.md) · [`SoundRender_Environment.h`](SoundRender_Environment.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it addresses device properties by identifier with explicit sizes and passes
raw field pointers.

## Purpose

Fills the reverb interface against a vendor extension that exposes a *listener-global* reverb as a
set of named properties, each set and got by identifier with an explicit size. The whole file is
capability probing plus a twelve-property push.

It is compiled only where the extension's headers exist, and is discarded at runtime when the
device does not answer. Reverb is optional throughout the engine.

## `construct` — capability probing

**Contract** — Resolves the extension's two entry points; if either is missing the backend is not
initialized and the caller discards it. Then probes the property set twice: once for deferred
setting, once for immediate.

```text
FUNCTION construct()
  set_property ← device.resolve("set")
  get_property ← device.resolve("get")
  IF either is missing THEN RETURN      # initialized() will report false

  supports_deferred  ← probe(deferred = true)
  supports_immediate ← probe(deferred = false)

FUNCTION probe(deferred) -> bool
  # For each of the twelve properties: read the device's current value,
  # then write that same value straight back. Both must succeed.
  FOR EACH property IN the twelve listener properties
    IF get(property) failed THEN RETURN false
    IF set(property, the value just read, deferred) failed THEN RETURN false
  RETURN true
```

**Notes** — Probing by *round-tripping the device's own value* is the interesting decision. Asking
"do you support this extension" is unreliable — drivers of the era claimed extensions they only
partly implemented — so the backend instead writes back exactly what it read, which is
observationally a no-op and therefore safe to do at startup, and checks that every one of the
twelve properties it will ever touch actually answers. A partial implementation is rejected as a
whole rather than failing later on the one property it lacks.

Deferred support is probed separately because it is an independent capability: a device may accept
every property immediately and refuse to batch them.

## `initialized`

**Contract** — True only when both entry points resolved and the immediate probe passed. The
manager constructs this backend, asks this question, and deletes it on a false answer.

## `set_listener`

**Contract** — Pushes a preset at the device: twelve properties, each by identifier with its own
size, deferred where the device supports it. Called every update with a freshly blended preset.
Does not clamp — the preset arrives clamped.

**Notes** — Three of the twelve are integers at the device and reals in the engine, and are
floored, not rounded, on the way out. They are decibel-like levels where the quantization step is
0.1 dB, so the bias is inaudible; a rebuild may round instead.

Two fields the engine carries are *not* pushed: the room kind and the environment size. The kind
is replaced by the extension's own default because setting it would reset every other property —
the vendor model treats the room kind as a macro that overwrites the parameter set, which is
exactly what the engine does not want after blending. Environment size is skipped for the same
reason: on this extension it rescales the other parameters.

## `get_listener`

**Contract** — Reads all twelve back in one call and fills a preset. Used by the editor to show
what the device actually holds after its own clamping.

## `commit`

**Contract** — Publishes the deferred batch, or does nothing when the device does not defer. Called
immediately after every `set_listener`, so a preset change reaches the mixer as one step rather
than twelve.
