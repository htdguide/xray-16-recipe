# src/editors/xrWeatherEngine/engine_impl.hpp

> Declares the one object that answers everything the editor application asks of a running engine.

**Needs** — [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md) · [`engine_impl.cpp`](engine_impl.cpp.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md)
**Used by** — [`entry_point.cpp`](../xrWeatherEditor/entry_point.cpp.md) · [`engine_impl.cpp`](engine_impl.cpp.md)
**Tier floor** — T1: it is the symbol the editor application resolves across a module boundary, and it forwards raw platform window messages.

## Purpose

Declares the surface implemented in [`engine_impl.cpp`](engine_impl.cpp.md): the host side
of [`engine_base`](../../Include/editor/engine.hpp.md), plus the single process-wide
instance the editor application binds to.

The instance is a module-level object, not something the editor creates. That is the one
decision this file makes: **there is exactly one engine per process, it is constructed
before the editor exists, and the editor is handed a reference rather than a factory.**

## State

```text
RECORD EngineHost
  input_receiver  : InputReceiver   # the claim on keyboard and mouse
  input_captured  : bool            # whether that claim is currently held
```

Everything else the interface exposes is a view onto the running weather system, which
belongs to [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md).

## Exported units

- **the process-wide instance** — the object the editor application binds to by name.
- **frame and window** — `on_message`, `on_idle`, `on_resize`, `pause`, `capture_input`,
  `disconnect`, `quit_requested`.
- **string interning** — convert plain text to and from the engine's interned string type.
- **the weather model** — `environment`, the current cycle and keyframe, the two timeline
  positions, pause and time factor, the current time of day as text.
- **the three property views** — current keyframe, interpolated result, target keyframe.
- **persistence and clipboard** — save; the four reload granularities; copy, paste into
  current, paste into target, add as new keyframe.
