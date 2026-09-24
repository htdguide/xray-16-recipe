# src/xrGame/Wound.h

> Declares the per-bone injury record implemented in [`Wound.cpp`](Wound.cpp.md).

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`hit_immunity.h`](hit_immunity.h.md)
**Used by** — [`ActorCondition.cpp`](ActorCondition.cpp.md) · [`ActorCondition_script.cpp`](ActorCondition_script.cpp.md) · [`EntityCondition.cpp`](EntityCondition.cpp.md) · [`Wound.cpp`](Wound.cpp.md) · [`entity_alive.cpp`](entity_alive.cpp.md) · [`entity_alive.h`](entity_alive.h.md)
**Tier floor** — T3: a declaration plus trivial field access

## Purpose

Declares `CWound` — one accumulated injury on one bone, carrying a magnitude per damage
type — and the accessors that owners use to attach a particle to it, to mark it for
removal and to read its bone. Substance is in [`Wound.cpp`](Wound.cpp.md).

Exported units:

- `CWound` — the record; constructed against a bone index.
- `save` / `load` — the persistent form (bone plus quantized magnitudes only).
- `TotalSize`, `TypeSize`, `BloodSize` — magnitude queries; the last is the bleeding
  subset.
- `AddHit` — fold a hit into the wound.
- `Incarnation` — heal the wound by an absolute amount per damage type.
- Bone index, particle bone index and particle name — read and write; presentation
  bindings the owner sets after construction.
- Destruction mark — the owner's flag that this wound should be swept away.
- Drop timer — a public field the owner advances between blood drops; deliberately not
  encapsulated because the owner, not the wound, runs the clock.
