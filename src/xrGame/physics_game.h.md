# src/xrGame/physics_game.h

> Declares the two contact callbacks the physics world calls back into the game with, implemented in [`physics_game.cpp`](physics_game.cpp.md).

**Needs** — [`physics_game.cpp`](physics_game.cpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`Level_network_start_client.cpp`](Level_network_start_client.cpp.md) · [`physics_game.cpp`](physics_game.cpp.md)
**Tier floor** — T3: two function references

## Purpose

Two names, no types. The physics layer knows nothing about wallmarks, particles or sounds; it
knows only that for each contact it may call a function the game installed. These are the two
functions the game installs — one for ordinary bodies, one for character bodies — and this
header is how the installation site names them.

Exported units:

- `ContactShotMark` — the contact effect callback for ordinary rigid bodies.
- `CharacterContactShotMark` — the same for character bodies, which differ only in their
  thresholds.

## State

`Stateless.`

**Notes** — both are function *references* rather than functions, so the installed behaviour
can be swapped at run time. Nothing in the shipped code swaps them; the indirection exists so
the physics layer can hold the reference without a link-time dependency on the game module.
The substance, including why there are two, is in
[`physics_game.cpp`](physics_game.cpp.md).
