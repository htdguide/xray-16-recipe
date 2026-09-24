# src/xrSound/SoundRender_CoreA.cpp

> The device backend: open the mixer, discover what it can do, build the voice pool, and keep the
> listener transform in the mixer's coordinate system.

**Needs** — [`SoundRender_CoreA.h`](SoundRender_CoreA.h.md) · [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_TargetA.h`](SoundRender_TargetA.h.md) · [`OpenALDeviceList.h`](OpenALDeviceList.h.md) · [`SoundRender_EffectsA_EAX.h`](SoundRender_EffectsA_EAX.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it owns device and context handles and pushes raw float triples at the mixer.

## Purpose

The concrete filling of the audio device seam. Four things happen only here: choosing and opening a
device, discovering whether it offers float PCM and reverb, creating the voice pool, and converting
the listener's transform from the engine's handedness to the mixer's.

Everything else in the chapter is target-agnostic, which is the point of the three-way split — see
[`SoundRender_Core.cpp`](SoundRender_Core.cpp.md).

## `initialize_devices_list`

**Contract** — Enumerates the output devices without opening one. Marks the core absent if the
enumeration finds nothing. Runs before the settings screen is drawn, because the device list is a
setting.

## `initialize`

**Contract** — Opens the selected device, creates a context and makes it current, seeds the
listener at the origin facing forward, discovers the two optional capabilities, and then creates
voices until the device refuses. Marks the core absent and returns on any failure, leaving the game
running silently rather than failing to start.

```text
FUNCTION initialize()
  IF the device list was never built THEN mark absent; RETURN
  select_best_device()
  device ← open(chosen device name)          IF failed THEN mark absent; RETURN
  context ← create_context(device)           IF failed THEN close device; mark absent; RETURN
  make context current
  seed the listener: origin, forward, up, unit gain

  # Capability 1: float PCM. Two spellings, because one platform's mixer
  # uses a different case for the same extension name.
  supports_float_pcm ← device offers float-PCM buffers AND the player allows it

  # Capability 2: reverb. Construct a backend, ask it whether it really works,
  # and discard it if not.
  IF the device advertises reverb AND no backend exists THEN
    effects ← new reverb backend
    IF NOT effects.initialized() THEN discard effects

  start the clocks; mark present and ready

  # The voice pool: ask for the configured count, accept what we get.
  FOR i IN 0 .. targets_max - 1
    voice ← new device voice
    IF voice.initialize() THEN append it to targets
    ELSE log the real count and BREAK
```

**Invariants** — The voice pool is complete before any emitter can be updated, and never changes
size afterwards. Its *actual* size may be below the configured thirty-two; every ranking decision
reads the list rather than the constant, so a device that gives sixteen voices simply culls harder.

**Notes** — Asking until refusal, rather than querying a maximum, is the honest way to size the pool
on a seam that does not promise to report its limit. It costs one failed allocation at startup.

Float PCM changes the decode path and every byte count in the chapter (see
[`SoundRender_Source.cpp`](SoundRender_Source.cpp.md)) and is therefore decided here, once, before
any asset loads. It is both a capability and a player setting: the flag is the conjunction.

## `update_listener`

**Contract** — Extends the core's listener update with the device push. Converts to the mixer's
handedness first.

```text
FUNCTION update_listener(position, forward, up, right, dt)
  core.update_listener(...)                  # records the transform, blends the reverb
  m ← listener mirrored in Z                 # position and all three axes
  push m.position, zero velocity, and (m.forward, m.up) as the orientation
```

**Notes** — Velocity is always zero, so **there is no Doppler in this engine** even though the seam
offers it. That is a decision, not an omission: the player moves fast enough that a real Doppler
term on every source would pitch-shift the whole world audibly as they run. A rebuild that enables
it will not match the original.

The mirror is in Z, applied to the position and to every orientation axis, matching the per-source
mirror in [`SoundRender_TargetA.cpp`](SoundRender_TargetA.cpp.md). Mirroring both the listener and
the sources is self-consistent; mirroring one of them puts every sound on the wrong side.

## `set_master_volume`

**Contract** — The one global gain, applied at the listener. No-op when no device is present.

## `clear`

**Contract** — Tears down in strict reverse order: cached sources, the reverb backend, every voice,
then the context and the device. The order is the invariant — a voice released after its context is
gone has nowhere to release to, and a source freed while a voice still streams from it is a decoder
reading freed memory.

## `restart`

**Contract** — Re-applies the environment. A full device rebuild is not implemented: the seam's
device-lost path is unexercised here because the mixer implementations in use do not lose devices
the way graphics devices do. A rebuild on a mixer that does must rebuild the voice pool and
re-establish every streaming voice from its emitter's cursor.
