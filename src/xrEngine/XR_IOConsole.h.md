# src/xrEngine/XR_IOConsole.h

> Declares the console: the typed variable registry, the command line, and the tip list.

**Needs** — [`XR_IOConsole.cpp`](XR_IOConsole.cpp.md) · [`xr_ioc_cmd.h`](xr_ioc_cmd.h.md) · [`pure.h`](pure.h.md) · [`IInputReceiver.h`](IInputReceiver.h.md) · [`EventAPI.h`](EventAPI.h.md)
**Used by** — [`xrRender_console.cpp`](../Layers/xrRender/xrRender_console.cpp.md) · [`engine_impl.cpp`](../editors/xrWeatherEngine/engine_impl.cpp.md) · [`Engine.cpp`](Engine.cpp.md) · [`EngineAPI.cpp`](EngineAPI.cpp.md) · [`EventAPI.cpp`](EventAPI.cpp.md) · [`FDemoPlay.cpp`](FDemoPlay.cpp.md) · [`FDemoRecord.cpp`](FDemoRecord.cpp.md) · [`IGame_Persistent.cpp`](IGame_Persistent.cpp.md) · [`Text_Console.cpp`](Text_Console.cpp.md) · [`Text_Console.h`](Text_Console.h.md) · [`XR_IOConsole.cpp`](XR_IOConsole.cpp.md) · [`XR_IOConsole_callback.cpp`](XR_IOConsole_callback.cpp.md) · [`XR_IOConsole_control.cpp`](XR_IOConsole_control.cpp.md) · [`XR_IOConsole_get.cpp`](XR_IOConsole_get.cpp.md) · _and 35 more_
**Tier floor** — T2: a name-keyed command map and an edit buffer; substance is in the implementation files

## Purpose

Declares the surface implemented across
[`XR_IOConsole.cpp`](XR_IOConsole.cpp.md) (registry, execution, tip generation and drawing),
[`XR_IOConsole_control.cpp`](XR_IOConsole_control.cpp.md) (history and tip cursors),
[`XR_IOConsole_callback.cpp`](XR_IOConsole_callback.cpp.md) (key handling inside the edit
field), [`XR_IOConsole_get.cpp`](XR_IOConsole_get.cpp.md) (typed reads of a variable) and
[`XR_IOConsole_script.cpp`](XR_IOConsole_script.cpp.md) (the script surface).

The console wears four roles at once, which is why it is declared here rather than derived
from one of them: it takes part in the frame loop, it receives input, it receives deferred
events, and it is the handler that writes the user's configuration file back out — a crash
handler asks it for that filename so a report can name the settings in force.

Exported units:

- `CConsole` — the console. One process-wide instance.
- `Commands` — the registry itself: every console variable and command, keyed by name, in
  name order.
- `AddCommand` / `RemoveCommand` — registration.
- `Show` / `Hide` / `bVisible` — visibility, which also controls input capture.
- `Execute` / `ExecuteCommand` / `ExecuteScript` / `SelectCommand` — running a line.
- `GetBool` / `GetFloat` / `GetInteger` / `GetString` / `GetToken` / `GetXRToken` /
  `GetFVector` / `GetFVectorPtr` / `GetCommand` — typed reads.
- `TipString` — one completion suggestion plus the range within it that matched.
- `Console_mark` — the severity marks a log line may begin with.
- `ConfigFile` — the settings file this console reads and writes.

Sizes that matter: the edit buffer is 1024 bytes, at most 220 suggestions are collected,
and 14 of them are visible at once — the page size the page-up and page-down keys move by.
Command history holds 64 entries.
