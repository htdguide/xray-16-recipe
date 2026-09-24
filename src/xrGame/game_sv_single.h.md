# src/xrGame/game_sv_single.h

> Declares the single-player session: the game mode that owns an alife simulator.

**Needs** — [`game_sv_base.h`](game_sv_base.h.md) · [`alife_simulator.h`](alife_simulator.h.md)
**Used by** — [`GamePersistent.cpp`](GamePersistent.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`game_sv_single.cpp`](game_sv_single.cpp.md) · [`xrServer_CL_disconnect.cpp`](xrServer_CL_disconnect.cpp.md) · [`xrServer_Connect.cpp`](xrServer_Connect.cpp.md) · [`xrServer_process_event.cpp`](xrServer_process_event.cpp.md) · [`xrServer_process_event_destroy.cpp`](xrServer_process_event_destroy.cpp.md) · [`xrServer_sls_clear.cpp`](xrServer_sls_clear.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`game_sv_single.cpp`](game_sv_single.cpp.md). The one
piece of substance here is the shape: the single-player session *is* the game mode whose
session state is an alife simulator, and every other override on the class either delegates
to that simulator or, when there is none, falls back to the base session's answer.

Exported units:

- `game_sv_Single` — the session; type name `"single"`.
- The optional alife simulator, created at session creation and destroyed with the session.
- Time questions — start time, current time, time factor, and a second *environment* clock.
- Attachment rules — create, touch, detach: maintaining the alife graph's ownership tree.
- Persistence — save, load, reload, change level, switch distance.
- Restriction commands — add, remove, remove-all, applied to an entity by identifier.
- `restart_simulator` — tears the whole simulation down and rebuilds it from a save.
- Accessors for the server and the simulator, both of which assert they exist.
