# src/xrEngine/EventAPI.h

> Declares the named-event queue and what a receiver must implement.

**Needs** — [`xrCore/Threading/Lock.hpp`](../xrCore/Threading/Lock.hpp.md)
**Used by** — [`CustomHUD.h`](CustomHUD.h.md) · [`Engine.cpp`](Engine.cpp.md) · [`Engine.h`](Engine.h.md) · [`EventAPI.cpp`](EventAPI.cpp.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_Persistent.cpp`](IGame_Persistent.cpp.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`XR_IOConsole.cpp`](XR_IOConsole.cpp.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`XR_IOConsole_script.cpp`](XR_IOConsole_script.cpp.md) · [`xr_input.cpp`](xr_input.cpp.md) · [`xr_ioc_cmd.cpp`](xr_ioc_cmd.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`EventAPI.cpp`](EventAPI.cpp.md).

## Exported units

- **`EventReceiver`** — one entry point: handle an event, given its identity and two opaque payload words. The identity is passed so a receiver subscribed to several events can discriminate by comparing against the identities it holds — comparison is by identity, never by name, which is why the identity is returned from subscription.
- **`EventQueue`** — create and destroy an identity; attach and detach a handler; signal immediately or defer, by identity or by name; drain; peek; dump; shut down. See the implementation twin.

**Notes** — Every operation exists in both an identity-taking and a name-taking form. The name-taking form creates and destroys an identity around the call, so it costs a table scan and possibly an allocation per use. It exists for call sites that fire an event rarely and do not want to hold state; a rebuild may keep both, but should make the cost difference visible at the call site as the original does not.
