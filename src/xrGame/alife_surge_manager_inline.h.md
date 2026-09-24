# src/xrGame/alife_surge_manager_inline.h

> Construction of the repopulation layer.

**Needs** — [`alife_surge_manager.h`](alife_surge_manager.h.md)
**Used by** — [`alife_surge_manager.h`](alife_surge_manager.h.md)
**Tier floor** — T3: a forwarding constructor

## Purpose

One body: the constructor, which forwards the server and the configuration section to the
base and does nothing else. The layer holds no configuration of its own — the spawn
records carry their own gates and the clock comes from the time manager.

Substance is in [`alife_surge_manager.cpp`](alife_surge_manager.cpp.md).
