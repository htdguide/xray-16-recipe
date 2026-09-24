# src/xrGame/hud_item_object.h

> Declares the base for every inventory item that also has a first-person view: it is simultaneously an inventory item and a held item.

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md) · [`HudItem.h`](HudItem.h.md)
**Used by** — [`Artefact.cpp`](Artefact.cpp.md) · [`Artefact.h`](Artefact.h.md) · [`Missile.cpp`](Missile.cpp.md) · [`Missile.h`](Missile.h.md) · [`Weapon.cpp`](Weapon.cpp.md) · [`Weapon.h`](Weapon.h.md) · [`WeaponHUD.h`](WeaponHUD.h.md) · [`flare.cpp`](flare.cpp.md) · [`flare.h`](flare.h.md) · [`hud_item_object.cpp`](hud_item_object.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`hud_item_object.cpp`](hud_item_object.cpp.md). Weapons,
detectors, the torch and every other item the player can raise into view derive from this: it
is the point where the "thing in a grid slot" role and the "thing animating in front of the
camera" role are joined into one object.

The class is abstract by construction — its constructor is not public — because joining the
two roles without being a specific item is meaningless.

Exported units: the merged lifecycle (construct, load, spawn, destroy), the merged inventory
transitions, activation and deactivation, the input action dispatch, the state machine
forwarding, the per-frame update, and the two rendering entry points. Every one of them exists
to sequence the two halves; the ordering is in the implementation and is load-bearing.

**Notes** — one override here is substantive rather than a forwarding: the item declines to
use its parent's navigation position on any frame in which its first-person view has been
rendered. A raised item is at the camera, not at the owner's feet, and letting its AI position
follow the owner while it is visibly in front of the camera makes it audible and detectable
from the wrong place.
