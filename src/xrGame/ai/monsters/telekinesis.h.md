# src/xrGame/ai/monsters/telekinesis.h

> Declares the telekinesis controller: the thing that owns a set of levitated objects on behalf of a creature or an anomaly.

**Needs** — [`telekinetic_object.h`](telekinetic_object.h.md) · [`telekinesis.cpp`](telekinesis.cpp.md) · [`xrPhysics/PHUpdateObject.h`](../../../xrPhysics/PHUpdateObject.h.md) · [Seam: Rigid-body dynamics](../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`GraviZone.cpp`](../../GraviZone.cpp.md) · [`GraviZone.h`](../../GraviZone.h.md) · [`TeleWhirlwind.cpp`](../../TeleWhirlwind.cpp.md) · [`TeleWhirlwind.h`](../../TeleWhirlwind.h.md) · [`burer.cpp`](burer/burer.cpp.md) · [`burer.h`](burer/burer.h.md) · [`burer_state_attack_tele.h`](burer/burer_state_attack_tele.h.md) · [`burer_state_attack_tele_inline.h`](burer/burer_state_attack_tele_inline.h.md) · [`poltergeist.cpp`](poltergeist/poltergeist.cpp.md) · [`poltergeist.h`](poltergeist/poltergeist.h.md) · [`poltergeist_telekinesis.cpp`](poltergeist/poltergeist_telekinesis.cpp.md) · [`telekinesis.cpp`](telekinesis.cpp.md) · [`telekinetic_object.cpp`](telekinetic_object.cpp.md)
**Tier floor** — T2: a list of owned objects and a physics-step subscription

## Purpose

Declares the surface implemented in [`telekinesis.cpp`](telekinesis.cpp.md).

Four very different things in the game lift objects into the air and throw them — the
poltergeist, the burer, the gravity anomaly and the whirlwind anomaly — and all four use
this one controller. It is a *component*, not a base class: an owner holds one and drives
it. Two extension points exist: the owner may subclass it to supply a richer per-object
type, and it inherits a physics-step subscription so it can act between collision detection
and the constraint solve.

## Exported units

- **activate an object** — takes a physics-bearing object plus strength, target height,
  hold duration and a rotate flag; allocates a per-object record, refuses and reports
  failure if the object has no physics body.
- **deactivate everything / deactivate one object** — release and destroy.
- **clear, clear-and-deactivate, clear-not-relevant** — three different degrees of
  forgetting; see the implementation twin, they are not interchangeable.
- **fire all at a target / fire one with a power factor / fire one with a flight time** —
  the three throw forms.
- **queries** — is the controller active, is this object held, how many objects are held in
  the lifting or holding phases, how many are held at all.
- **the scheduled update** — advances each held object's phase machine.
- **link removal** — drops a destroyed object from the set.
- **the per-object allocator hook** — the single virtual an owner overrides to get its own
  per-object type.

**Notes** — one accessor returns a held object's record *by value*. Since owners subclass
that record, this copies the base part only and discards the rest. Callers that use it get
a snapshot of the base fields and nothing else; a rebuild should return a borrowed
reference.
