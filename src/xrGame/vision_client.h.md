# src/xrGame/vision_client.h

> Declares the standalone vision sensor — a scheduled eye that can be attached to an entity that is not a creature — implemented in [`vision_client.cpp`](vision_client.cpp.md).

**Needs** — [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`Entity.h`](Entity.h.md) · [`xrEngine/ISheduled.h`](../xrEngine/ISheduled.h.md) · [`xrEngine/Feel_Vision.h`](../xrEngine/Feel_Vision.h.md) · [`vision_client_inline.h`](vision_client_inline.h.md)
**Used by** — [`actor_memory.cpp`](actor_memory.cpp.md) · [`actor_memory.h`](actor_memory.h.md) · [`car_memory.cpp`](car_memory.cpp.md) · [`car_memory.h`](car_memory.h.md) · [`vision_client.cpp`](vision_client.cpp.md) · [`vision_client_inline.h`](vision_client_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `vision_client`, an abstract sensor that owns a visual memory manager and drives
it from the scheduler rather than from a creature's brain. It exists so that something
which is not a `CCustomMonster` — a camera, a turret, a mounted device — can still see.
Substance is in [`vision_client.cpp`](vision_client.cpp.md).

Exported units:

- `vision_client(object, update_interval)` — binds to an entity, creates the visual memory
  manager, and registers with the scheduler at a fixed interval (both bounds equal, so the
  rate does not degrade with load).
- `shedule_Scale` / `shedule_Update` / `shedule_Name` / `shedule_Needed` — the scheduler
  contract. The update alternates between two half-passes.
- `feel_vision_mtl_transp` — delegates the per-material transparency question to the
  visual memory manager.
- `feel_vision_isRelevant` and `camera` — **the two things an implementor must supply**:
  which objects this sensor is allowed to consider, and where the eye is and what frustum
  it has. Everything else is inherited.
- `reinit` / `reload(section)` / `remove_links(object)` — lifecycle, delegated straight
  through to the visual memory manager.
- `visual()` — the owned visual memory manager (see
  [`vision_client_inline.h`](vision_client_inline.h.md)).

## Notes

**This is an abstract base whose two pure entries are the whole point.** A rebuild should
read it as an interface with a default implementation: the scheduling, the two-phase
query, and the memory bookkeeping are provided; the eye's placement and the relevance
filter are demanded. Splitting it any other way loses what the type is for.
