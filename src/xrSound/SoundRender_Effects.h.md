# src/xrSound/SoundRender_Effects.h

> The reverb seam: four operations that stand between a blended preset and whatever the device's
> reverb actually is.

**Needs** — [`SoundRender_Environment.h`](SoundRender_Environment.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_EffectsA_EAX.cpp`](SoundRender_EffectsA_EAX.cpp.md) · [`SoundRender_EffectsA_EAX.h`](SoundRender_EffectsA_EAX.h.md)
**Tier floor** — T2: a four-operation interface.

## Purpose

An interface with no implementation file — it is the contract a reverb backend satisfies. The
engine ships one filling ([`SoundRender_EffectsA_EAX.cpp`](SoundRender_EffectsA_EAX.cpp.md)) and
runs perfectly well with none, in which case the listener's blend is computed and discarded.

Everything about the engine's reverb model is visible in how narrow this is: reverb is **global to
the listener**, not per-source. One preset describes the room the player is in; every sound is
heard through it. There is no per-source wet/dry send, no per-source reverb zone. A rebuild on a
mixer that offers per-source reverb may do better, but need not, and the shipped content is tuned
for the global model.

## Exported units

- **`initialized`** — did the backend actually find a working reverb? Checked right after
  construction; a backend that reports false is discarded and the engine runs dry.
- **`set_listener(preset)`** — apply a preset. Called every update with the freshly blended value,
  so it must be cheap and must tolerate being handed a value that barely changed.
- **`get_listener(preset)`** — read back what the device actually has. The device may clamp or
  quantize; reading back is how the editor shows the true values.
- **`commit`** — publish a batch of parameter changes as one atomic step, where the device supports
  deferred setting. Called immediately after `set_listener` every update.

## Notes

The `set_listener` / `commit` split exists because a reverb applied one parameter at a time passes
through combinations that are not any real room, and on some devices that is audible as a click.
Where a device cannot defer, `commit` is a no-op and the parameters are applied as they arrive.
