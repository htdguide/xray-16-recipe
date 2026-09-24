# src/editors/xrWeatherEngine/pch.hpp

> The module's compile-time prelude: it names the engine subsystems every file in the weather-editor engine layer reaches for.

**Needs** — [`xrEngine/device.h`](../../xrEngine/device.h.md) · [`xrCore/xrCore.h`](../../xrCore/xrCore.h.md) · [`xrSound/Sound.h`](../../xrSound/Sound.h.md)
**Used by** — [`pch.cpp`](pch.cpp.md)
**Tier floor** — T4: it is a build-time convenience with no runtime content.

## Purpose

Incidental. It exists because C++ recompiles headers per translation unit and a shared
prelude makes that cheap. What survives is the *dependency statement* it makes: every
file in this module assumes the core services (memory, strings, the virtual filesystem),
the running device, and the sound system are already available — this module is a plug-in
inside a live engine process, not a standalone library.

A rebuild deletes this file and lets its module system express the same thing.

## State

`Stateless.`
