# src/xrGame/PHDestroyableNotificate.h

> Declares the debris-piece side of the destruction handshake, implemented in [`PHDestroyableNotificate.cpp`](PHDestroyableNotificate.cpp.md).

**Needs** — [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md)
**Used by** — [`Mincer.cpp`](Mincer.cpp.md) · [`PHDestroyable.cpp`](PHDestroyable.cpp.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`PHDestroyableNotificate.cpp`](PHDestroyableNotificate.cpp.md) · [`PHSkeleton.cpp`](PHSkeleton.cpp.md) · [`PHSkeleton.h`](PHSkeleton.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CPHDestroyableNotificate`, the interface a piece of debris presents. Substance is
in [`PHDestroyableNotificate.cpp`](PHDestroyableNotificate.cpp.md).

It is a separate, tiny file for a reason worth keeping: the breakable object and its pieces
refer to each other, and this is the half that breaks the cycle. Everything a breaking object
needs to know about a piece is here — it can be recognized, it has a body, and it can be told
to initialize.

Exported units:

- `CPHDestroyableNotificate` — the interface.
- `cast_phdestroyable_notificate` — the capability query, by which an arbitrary object is
  recognized as a piece of debris.
- `PPhysicsShellHolder` — demanded of every implementor: the way to reach this piece's body,
  visual and placement. The interface can then move the piece without knowing its type.
- `spawn_init` — the hook an implementor overrides to finish its own arrival; empty by
  default.
- `spawn_notificate` — report to the object this piece broke off from. Not overridable: the
  handshake is the same for every kind of debris.
