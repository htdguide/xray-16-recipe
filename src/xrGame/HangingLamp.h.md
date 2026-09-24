# src/xrGame/HangingLamp.h

> Declares the destructible light fixture, implemented in [`HangingLamp.cpp`](HangingLamp.cpp.md).

**Needs** — [`GameObject.h`](GameObject.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md)
**Used by** — [`HangingLamp.cpp`](HangingLamp.cpp.md) · [`script_game_object_script3.cpp`](script_game_object_script3.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CHangingLamp` as a physics-shell holder that is also a jointed skeleton — the
combination that lets a fixture both swing on its mount and be damaged bone by bone.
Substance is in [`HangingLamp.cpp`](HangingLamp.cpp.md).

Exported units:

- `CHangingLamp` — the fixture.
- `TurnOn`, `TurnOff` — the whole script-visible surface, and also how the lamp reacts
  to being destroyed.
- `Load`, `net_Spawn`, `net_Destroy`, `SpawnInitPhysics`, `CopySpawnInit` — the
  lifecycle, including the two hooks that build the body and re-derive the on/off state
  from the restored bone visibility.
- `shedule_Update`, `UpdateCL` — the scheduled tick and the per-frame light placement.
- `net_Save`, `net_SaveRelevant`, `save`, `load` — persistence; the lamp's own
  contribution is one byte.
- `Hit` — damage, with the bulb bone fatal and every other bone merely damaging.
- `Center`, `Radius` — the bounding sphere, from the model.
- `UsedAI_Locations` — always false; a lamp holds no navigation position.
- `net_Export`, `net_Import` — empty; the lamp replicates only through events.
- `renderable_ShadowGenerate`, `renderable_ShadowReceive` — both true.
