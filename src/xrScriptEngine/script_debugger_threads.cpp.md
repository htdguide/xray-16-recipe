# src/xrScriptEngine/script_debugger_threads.cpp

> Collects the live script coroutines of both processes into the list the editor shows.

**Needs** — [`script_debugger_threads.hpp`](script_debugger_threads.hpp.md) · [`script_debugger.hpp`](script_debugger.hpp.md) · [`script_process.hpp`](script_process.hpp.md) · [`script_thread.hpp`](script_thread.hpp.md) · [`script_engine.hpp`](script_engine.hpp.md) · [`Include/xrAPI/xrAPI.h`](../Include/xrAPI/xrAPI.h.md)

**Used by** — [`script_debugger_threads.hpp`](script_debugger_threads.hpp.md)

**Tier floor** — T3: it copies fields out of two lists into a third.

## Purpose

When the engine stops at a breakpoint the modder needs to know which coroutine they are in and
what else is running. This snapshots that, at the moment of the stop.

## State

```text
RECORD DebuggerThreads
  threads : list<ScriptThread>     # the snapshot the editor is looking at
```

Invariant: the snapshot is only valid while the engine is stopped. Nothing refreshes it while
scripts run, and nothing needs to, because scripts do not run while the engine is stopped.

## Contract

**`Fill`** — Asks the game process and then the level process for their coroutines and reports
the total. Tolerates either process being absent, which is the normal case: in single-player
there is no game process, and outside a loaded level there is no level process.

**`FillFrom`** — Snapshots one process's coroutines into records carrying the coroutine handle,
its registry reference, whether it is still alive, its script name and its process's name.

**`FindScript`** — Maps a reference identifier back to a coroutine handle, which is how the
editor's "show me this thread" selection is resolved.

**`DrawThreads`** — Clears the editor's thread pane and sends one record per coroutine.

**Notes** — `FillFrom` **clears the list before filling**, so calling `Fill` with both processes
present leaves only the second process's coroutines. The reported total still counts both. This
is a defect, not a design: the thread pane silently omits the game process's coroutines whenever
a level is loaded. A rebuild appends.
