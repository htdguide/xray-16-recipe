# src/xrGame/autosave_manager.h

> Declares the autosave timer and the readiness counter implemented in [`autosave_manager.cpp`](autosave_manager.cpp.md).

**Needs** — [`autosave_manager_inline.h`](autosave_manager_inline.h.md) · [`xrEngine/ISheduled.h`](../xrEngine/ISheduled.h.md)
**Used by** — [`ActorCondition.cpp`](ActorCondition.cpp.md) · [`Level.cpp`](Level.cpp.md) · [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md) · [`autosave_manager.cpp`](autosave_manager.cpp.md) · [`autosave_manager_inline.h`](autosave_manager_inline.h.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`enemy_manager.cpp`](enemy_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`autosave_manager.cpp`](autosave_manager.cpp.md), with the counter and timestamp
operations split into [`autosave_manager_inline.h`](autosave_manager_inline.h.md). The class
is a scheduler participant, which is how it gets a heartbeat without owning a thread.

Exported units:

- **`shedule_Update`** — the timer and the refusal rules.
- **`shedule_Scale`, `shedule_Needed`, `shedule_Name`** — the scheduler's view of it: cheap,
  always wanting an update, named for the profiler.
- **`on_game_loaded`** — reset the timestamp after a load.
- **`inc_not_ready`, `dec_not_ready`, `ready_for_autosave`, `not_ready_count`** — the
  readiness protocol every subsystem that must not be saved mid-transaction uses.
- **`autosave_interval`, `last_autosave_time`, `update_autosave_time`, `delay_autosave`** —
  the timestamp.
