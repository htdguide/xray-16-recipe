# src/xrPhysics/PHUpdateObject.h

> Declares the lightweight participant in the physics step: something that wants the pre-solve and post-solve callbacks without being a collidable object.

**Needs** — [`PHItemList.h`](PHItemList.h.md) · [`PHObject.cpp`](PHObject.cpp.md) · [`xrPhysics.h`](xrPhysics.h.md)
**Used by** — [`AmebaZone.cpp`](../xrGame/AmebaZone.cpp.md) · [`AmebaZone.h`](../xrGame/AmebaZone.h.md) · [`Artefact.h`](../xrGame/Artefact.h.md) · [`Car.cpp`](../xrGame/Car.cpp.md) · [`Car.h`](../xrGame/Car.h.md) · [`CustomRocket.cpp`](../xrGame/CustomRocket.cpp.md) · [`CustomRocket.h`](../xrGame/CustomRocket.h.md) · [`telekinesis.h`](../xrGame/ai/monsters/telekinesis.h.md) · [`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md) · [`PHCapture.h`](PHCapture.h.md) · [`PHShellSplitter.h`](PHShellSplitter.h.md) · [`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md) · [`PHStaticGeomShell.h`](PHStaticGeomShell.h.md) · [`PHWorld.h`](PHWorld.h.md)
**Tier floor** — T2: two callbacks and a membership flag.

## Purpose

The world steps two collections each fixed step: physics objects (which collide) and *update
objects* (which do not). An update object exists when some bookkeeping must run in lockstep with
the solver but has no geometry of its own — the breakable-shell fracture accounting in
[`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) is the main case, and the static geometry proxy in
[`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md) is the other.

Implementation of activation lives in [`PHObject.cpp`](PHObject.cpp.md).

## Exported units

- `CPHUpdateObject` — the base. Demands two operations from an implementor:

```text
FUNCTION tune(step)          # called before the constraint solve, with every contact in place
FUNCTION data_update(step)   # called after the solve, with results available
```

  and offers one optional hook, `net_relcase(shell)`, which lets an implementor drop a reference
  to a shell that is about to be destroyed by a network correction.

- `PH_UPDATE_OBJECT_STORAGE` / `PH_UPDATE_OBJECT_I` — the intrusive-list alias pair.

**Notes** — deactivation is automatic on destruction, because the world holds an unowned link. That
ordering constraint survives into any rebuild: *the world's list must not outlive its members*.
