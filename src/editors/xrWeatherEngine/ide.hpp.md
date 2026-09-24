# src/editors/xrWeatherEngine/ide.hpp

> One accessor: reach the editor application from anywhere inside the engine-side weather model.

**Needs** — [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md) · [`xrEngine/device.h`](../../xrEngine/device.h.md)
**Used by** — [`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md) · [`editor_environment_ambients_effect_id.cpp`](editor_environment_ambients_effect_id.cpp.md) · [`editor_environment_ambients_manager.cpp`](editor_environment_ambients_manager.cpp.md) · [`editor_environment_ambients_sound_id.cpp`](editor_environment_ambients_sound_id.cpp.md) · [`editor_environment_effects_effect.cpp`](editor_environment_effects_effect.cpp.md) · [`editor_environment_levels_manager.cpp`](editor_environment_levels_manager.cpp.md) · [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) · [`editor_environment_sound_channels_channel.cpp`](editor_environment_sound_channels_channel.cpp.md) · [`editor_environment_sound_channels_source.cpp`](editor_environment_sound_channels_source.cpp.md) · [`editor_environment_suns_flare.cpp`](editor_environment_suns_flare.cpp.md) · [`editor_environment_suns_sun.cpp`](editor_environment_suns_sun.cpp.md) · [`editor_environment_thunderbolts_collection.cpp`](editor_environment_thunderbolts_collection.cpp.md) · [`editor_environment_thunderbolts_gradient.cpp`](editor_environment_thunderbolts_gradient.cpp.md) · [`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md) · _and 5 more_
**Tier floor** — T2: a global lookup with a precondition.

## Purpose

Every editable object in this module needs the editor application for exactly one reason:
to ask it for a property holder when it is created, and to hand that holder back when it
is destroyed. Threading an editor reference through every constructor would double the
parameter list of thirty classes for no gain, so the reference is fetched from the running
device instead.

## State

`Stateless.` The editor reference lives on the device.

## `ide`

**Contract** — returns the editor application currently driving this process. Fails a
precondition if no editor is attached.

**Invariants** — callers must already know an editor is attached. Every destructor in this
module tests that separately before calling, because a module object may outlive the
editor during shutdown — see the note.

```text
FUNCTION editor() -> Ide
  REQUIRE device.editor IS NOT none
  RETURN device.editor
```

**Notes** — The precondition and the destructor-side test are the same problem seen from
two ends: **the editor outlives every editable object except during teardown, where the
order reverses.** Every destructor in this module opens with "if no editor is attached,
skip releasing the property holder" for that reason, and a rebuild that gives the editor
ownership of its holders (so releasing is the editor's job, not the object's) deletes the
test everywhere it appears.
