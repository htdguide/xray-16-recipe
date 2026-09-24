# src/xrGame/ElectricBall.cpp

> An artefact that, while carried, keeps its own transform pinned to its carrier's instead of following the usual attachment rules.

**Needs** — [`ElectricBall.h`](ElectricBall.h.md) · [`Artefact.h`](Artefact.h.md)
**Used by** — reached through its declarations in [`ElectricBall.h`](ElectricBall.h.md); callers name that, not this file.
**Tier floor** — T3: a transform copy once per frame

## Purpose

An artefact class whose only departure from the base is where it sits when someone is
holding it. The base artefact, when attached to a parent, is placed by the parent's
attachment logic — a bone, a slot, an offset. This one simply adopts the parent's own
transform wholesale each frame, so the effect it renders is centred on the carrier rather
than on a hand or a belt slot. That is the entire decision in the file, and it exists
because the artefact's visual is a glow the player should be standing inside.

## State

`Stateless.` The transform it writes is the object's own, owned by the base.

## `CElectricBall`

**Contract** — construction, destruction and configuration load delegate to the base
artefact.

## `UpdateCLChild`

**Contract** — the per-frame update step the base artefact reserves for its subclasses.
After the base's own work, if the artefact currently has a parent, its transform is
overwritten with the parent's. Unparented — lying on the ground or in flight — it does
nothing and the base's physics keeps the transform.

**Notes** — copying the whole transform, orientation included, rather than only the
position means the effect also inherits the carrier's facing. Nothing visibly depends on
that, but a rebuild that copies only position will differ if the visual is ever made
directional.
