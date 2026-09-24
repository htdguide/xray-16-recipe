# src/xrGame/WeaponBinoculars.cpp

> Binoculars: a weapon that cannot fire, whose trigger zooms, and which draws a bracket around every living thing the actor can currently see.

**Needs** — [`WeaponBinoculars.h`](WeaponBinoculars.h.md) · [`WeaponBinocularsVision.h`](WeaponBinocularsVision.h.md) · [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md) · [`Level.h`](Level.h.md) · [`Inventory.h`](Inventory.h.md)
**Used by** — reached through its declarations in [`WeaponBinoculars.h`](WeaponBinoculars.h.md); callers name that, not this file.
**Tier floor** — T2: a per-frame overlay driven by the actor's vision memory.

## Purpose

Binoculars are modelled as a weapon because everything the player holds is: they occupy a
slot, they draw and holster, they have a first-person model and an aim pose. What this
file does is remove the weapon parts (it cannot kill, it has no crosshair, its trigger is
its zoom) and add the target-bracket overlay.

The stepped zoom helper defined here is used by every dynamically zoomable scope in the
game, not just by binoculars — an accident of where it was written.

## State

```text
RECORD Binoculars EXTENDS CustomPistol
  vision_enabled : bool                    # authored
  vision         : optional<TargetBrackets>  # alive only while zoomed
```

**Invariant** — the bracket overlay exists exactly while the binoculars are fully zoomed.
It is created on zoom-in and destroyed on zoom-out, so it never accumulates stale target
references across a holster.

## `Load`

**Contract** — adds zoom-in and zoom-out sounds, then reads whether the target brackets
are present. The flag is forced off when the bracket animation speed is absent or zero —
the speed is what drives the bracket's convergence, and a zero would make it never lock.
Data validation, expressed as a feature switch.

## `Action` — the trigger is the zoom

**Contract** — the fire binding is rewritten to the zoom binding before anything else
sees it; every other binding passes through. That single substitution is what makes
binoculars behave correctly under both zoom conventions (hold and toggle) without
reimplementing either.

## `OnZoomIn` / `OnZoomOut`

**Contract** — plays the matching sound, stopping the opposite one, routed as a
first-person sound when the carrier is the view entity. On the way in, the bracket
overlay is created if the weapon is authored for it *and* the player's display settings
enable it. On the way out it is destroyed.

**Invariants** — zoom-out only acts when the weapon is fully zoomed and not still
swinging, so a zoom cancelled mid-swing neither plays the sound nor tears down an overlay
that was never built.

## `UpdateCL` / `render_item_ui`

**Contract** — the overlay updates only while the carrier exists, the weapon is fully
zoomed, the overlay exists and the display setting is on; it draws under the same
conditions plus being the active item. The four-way guard is repeated in three places
because each has a slightly different set — a rebuild should compute "brackets are live"
once.

## `ZoomInc` / `ZoomDec` and the stepped-zoom rule

**Contract** — the shared rule for a dynamically zoomable optic. Given the optic's
maximum magnification (expressed as its narrowest field of view), it produces the step
size and the widest field of view the player may dial back to.

```text
FUNCTION zoom_data(narrowest_fov) -> (step, widest_fov)
  default_fov = the player's normal field of view
  total       = default_fov - narrowest_fov          # must be positive
  # the optic may be dialled back to 30% of its total magnification
  widest_fov  = default_fov - total * 0.3
  # the remaining 70% is divided into three steps
  step        = total * 0.7 / 3
```

**Invariants** — three steps, and a floor at thirty per cent of full magnification.
Those two numbers define how every dynamic scope in the game feels, and they are
hard-coded here rather than authored. The optic's *narrowest* field of view is the
authored end; the widest is derived, so an optic cannot be dialled back to naked-eye view.

**Notes** — the binoculars' own increment and decrement are the **opposite sense** to the
weapon base's: here "increase" widens the field of view and "decrease" narrows it, while
in [`Weapon.cpp`](Weapon.cpp.md) it is the reverse. One of the two is wrong relative to
its name; which is not recoverable, and both ship.

Note also that these overrides do not check the dynamic-zoom flag or a scope's presence,
unlike the weapon base's, because binoculars are always both.

## `save` / `load`

**Contract** — after the base weapon's payload, the remembered zoom factor. Binoculars
are the one item whose dialled-in magnification must survive a save, because the player
sets it deliberately and rarely.

## `can_kill` / `use_crosshair` / `GetBriefInfo`

**Contract** — `can_kill` is always false, which removes binoculars from every AI weapon
choice. The crosshair is suppressed. The inventory readout carries only the item's short
name and icon: no ammunition, no fire mode.

## `net_Relcase`

**Contract** — when any object is destroyed, the overlay drops its reference to it. This
is the engine-wide mechanism for severing links to a dying object, and the brackets hold
raw references to every visible creature, so they must participate.
