# src/xrSound/Sound.cpp

> Module lifetime: build the device list, create and destroy the renderer, and own the reverb
> preset library.

**Needs** — [`SoundRender_CoreA.h`](SoundRender_CoreA.h.md) · [`Sound.h`](Sound.h.md) · [`SoundRender_Environment.h`](SoundRender_Environment.h.md) · [`Include/xrAPI/xrAPI.h`](../Include/xrAPI/xrAPI.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: lifetime wiring and one file load.

## Purpose

The engine's entry and exit points for the whole chapter. Small, but it fixes the startup order that
everything else assumes.

## Startup order

```text
FUNCTION create_devices_list()
  no_sound ← the command line asked for silence
  renderer ← new device-backed sound core
  IF NOT no_sound THEN renderer.initialize_devices_list()
  IF no device is present THEN append a terminator to the device list anyway
  publish renderer into the global environment

FUNCTION create()
  IF a device is present THEN
    load the reverb preset library        # before the renderer: level geometry needs it
    renderer.initialize()                 # opens the device, builds the voice pool

FUNCTION destroy()
  withdraw the renderer from the global environment   # first: nobody may reach it mid-teardown
  renderer.clear()
  delete renderer
  unload the preset library
  free the device-name list
```

**Invariants** — The two-phase startup is required, not stylistic: the device *list* must exist
before the settings screen draws, and the settings screen's choice must be known before a device is
opened. The preset library loads before the renderer because a level's environment geometry resolves
its preset names against the library at load time and a missing library silently disables
regions.

The command-line silence switch is read once and cached: sound cannot be turned on later in the
session. A no-sound run still constructs the renderer, so every call site keeps working — it simply
reports no device, and every play becomes a no-op. That is the engine's answer to "what if there is
no audio hardware", and it is cheaper than making every call site conditional.

## `is_sound_enabled`

**Contract** — True when a renderer exists and it found a device.

## `env_load` / `env_unload` / `refresh_env_library` / `get_env_library`

**Contract** — Load the reverb preset library from a single fixed-name file in the game data;
release it; reload it and force every listener blend to re-evaluate; read it. The reload path exists
for the editor and for modders tuning presets against a running game.

**Notes** — The library is optional: an installation without the preset file gets no reverb regions
and the identity preset everywhere, rather than a failure.

## Module globals

The default scene and the selected device index live here. The default scene is the one the handle's
`play` reaches for when no scene is named; the engine fills it when the level's sound world is
created. The device index starts at a sentinel meaning "not chosen", which is what makes the
backend pick the system default on first run.
