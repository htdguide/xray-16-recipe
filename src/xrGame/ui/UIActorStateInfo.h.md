# src/xrGame/ui/UIActorStateInfo.h

> Declares the actor's condition panel — fourteen readouts of health, protection and
> restoration — and the one polymorphic readout they are all built from.

**Needs** — [`UIActorStateInfo.cpp`](UIActorStateInfo.cpp.md) · [`../../xrUICore/Hint/UIHint.h`](../../xrUICore/Hint/UIHint.h.md) · [`../../xrServerEntities/alife_space.h`](../../xrServerEntities/alife_space.h.md)
**Used by** — [`UIActorMenu.cpp`](UIActorMenu.cpp.md) · [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorStateInfo.cpp`](UIActorStateInfo.cpp.md)
**Tier floor** — T2: widget tree polled from simulation state

## Purpose

Declares the surface implemented in [`UIActorStateInfo.cpp`](UIActorStateInfo.cpp.md).

## `ui_actor_state_wnd`

The panel. Holds **fourteen** readouts, one per named state, in a fixed enumeration:
stamina, health, bleeding, radiation, armour, a main sensor, and then eight protection
readouts — fire, radiation, acid, psi, wound, fire wound, shock and power restoration.

- `init_from_xml(document)` — the *plain* form: three readouts only (health, psi, radiation),
  used by the older split layout dialect where the panel is three bars in a frame.
- `init_from_xml(document, path)` — the full form: all fourteen, plus a shared hint window.
- `UpdateActorInfo(owner)` — poll everything from the actor's condition and equipment.
- `UpdateHitZone()` — drive the main sensor from the overlay's zone sensing.

The two init forms are not variants of one another: the plain form leaves eleven readouts
constructed but unconfigured, and every set operation on an unconfigured readout is a no-op
that reports failure. That is how one panel serves two incompatible layouts.

## `ui_actor_state_item`

One readout. It is *whichever* of five presentations the layout document gave it, and it may
be several at once:

- a progress bar,
- a circular progress shape,
- a needle with an optional shadow needle,
- up to three icons, each independently shown or hidden,
- a numeric caption on the first icon.

Every setter returns whether the corresponding part exists, and **that return value is the
dispatch**: the panel sets a progress bar first and only falls back to a needle when there was
no bar. So what a readout *looks like* is entirely a layout decision, and the same state can
be a bar in one game's data and a dial in another's.

It derives from the toolkit's hint-bearing window, so every readout carries its own tooltip
text and dwell delay from the document.
