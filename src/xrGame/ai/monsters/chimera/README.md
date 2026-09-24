# src/xrGame/ai/monsters/chimera — the chimera

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

The chimera is the only creature whose attack is built entirely out of leaps, and the only
one carrying two whole behaviour trees that the shipped build cannot reach. Both facts are
worth as much to a rebuilder as the working behaviour, so this page gives them equal space.

## What works

**An attack made only of pounces.** Circle behind the enemy, pounce, and after a run of
pounces spend a couple of them purely repositioning — with a crouched pause whenever the
creature gets behind the enemy unseen. There is no walk-up, no melee state, no ranged
option. Seven authored numbers tune the whole thing.

**A creature that never flinches.** Its selector has six states and no response to being
hit. A chimera under fire keeps doing what it was doing, which is what makes it frightening
and is a deliberate absence rather than an oversight — the state exists and is simply not
registered.

**An animation table built out of almost nothing.** Two clips carry most of the creature,
with two velocity profiles for spinning and launching.

## What does not work

**The hunting tree cannot compile.** It was to be an ambush behaviour: move to cover, wait,
come out, repeat. Three of its four files are stubs. The fourth —
[`chimera_state_hunting_come_out_inline.h`](chimera_state_hunting_come_out_inline.h.md) — is
a **verbatim copy of its sibling**, so it defines the same state's methods a second time in
the same translation unit. The only reason the build succeeds is that nothing instantiates
the tree, so none of it is ever generated. What the "come out" half was supposed to do is
not recoverable; the file that should say is a copy of a different file.

**The threaten tree is registered nowhere.** Four complete, working states — roar in place,
creep closer, walk closer, roar again — implementing an intimidation display for anything
that is not yet a sworn enemy. They are finished. No state manager registers them and the
global state identifier they were written for is never selected by any creature in the
chapter. A rebuild that wires them up gets a creature that warns before it attacks, which is
a materially different animal.

## What could not be recovered

- The intended body of the hunting tree's "come out" state. The file that should hold it is
  a copy of its sibling, so the design is simply gone.
- Why the threaten tree was never wired up. It is complete and self-consistent; nothing in
  the source suggests it was abandoned rather than merely not yet connected.
- The eight-unit and five-unit thresholds in the threaten approaches are bare constants.

## Twins

| Twin | Role |
|---|---|
| [`chimera.cpp`](chimera.cpp.md) | The chimera's definition: an animation table built almost entirely out of two clips, two velocity profiles for spinning and launching, and the pounce's clip triple. |
| [`chimera.h`](chimera.h.md) | Declares the chimera: a base creature whose whole attack is a pounce, tuned by seven authored numbers. |
| [`chimera_attack_state.h`](chimera_attack_state.h.md) | Declares the chimera's attack: the only creature attack in the chapter built entirely out of pounces. |
| [`chimera_attack_state_inline.h`](chimera_attack_state_inline.h.md) | The chimera fight: circle behind the enemy, pounce, and after a run of pounces spend a couple of them just repositioning — with a crouched pause whenever it gets behind the enemy unseen. |
| [`chimera_script.cpp`](chimera_script.cpp.md) | Exposes the chimera class to the script layer under its frozen name. |
| [`chimera_state_hunting.h`](chimera_state_hunting.h.md) | Declares an unfinished hunting behaviour: take cover, then come out. |
| [`chimera_state_hunting_come_out.h`](chimera_state_hunting_come_out.h.md) | Declares the "emerge from cover" half of the unbuilt hunting behaviour. |
| [`chimera_state_hunting_come_out_inline.h`](chimera_state_hunting_come_out_inline.h.md) | Nothing: this file defines the wrong state, and is the reason the hunting subtree cannot be built. |
| [`chimera_state_hunting_inline.h`](chimera_state_hunting_inline.h.md) | An abandoned design for ambush hunting, preserved as a shell: alternate between hiding and emerging, forever. |
| [`chimera_state_hunting_move_to_cover.h`](chimera_state_hunting_move_to_cover.h.md) | Declares the "get into cover" half of the unbuilt hunting behaviour. |
| [`chimera_state_hunting_move_to_cover_inline.h`](chimera_state_hunting_move_to_cover_inline.h.md) | The cover half of the unbuilt hunting behaviour: every contract point is a stub. |
| [`chimera_state_manager.cpp`](chimera_state_manager.cpp.md) | The chimera's mood chart: six states, and a creature that never flinches from being shot. |
| [`chimera_state_manager.h`](chimera_state_manager.h.md) | Declares the chimera's top-level state selector. |
| [`chimera_state_threaten.h`](chimera_state_threaten.h.md) | Declares the chimera's intimidation behaviour: roar, stalk closer, roar again — used on anything that is not yet a sworn enemy. |
| [`chimera_state_threaten_inline.h`](chimera_state_threaten_inline.h.md) | A display rather than an attack: the chimera roars, closes in a stalk or a walk, and roars again, until the target either becomes a real enemy or gets too close to bluff. |
| [`chimera_state_threaten_roar.h`](chimera_state_threaten_roar.h.md) | Declares the leaf that stands still and bellows at the target. |
| [`chimera_state_threaten_roar_inline.h`](chimera_state_threaten_roar_inline.h.md) | Four seconds of standing, facing and bellowing. |
| [`chimera_state_threaten_steal.h`](chimera_state_threaten_steal.h.md) | Declares the creeping approach used during an intimidation display. |
| [`chimera_state_threaten_steal_inline.h`](chimera_state_threaten_steal_inline.h.md) | Creep toward the target until within eight units, then stop and let the display continue. |
| [`chimera_state_threaten_walk.h`](chimera_state_threaten_walk.h.md) | Declares the walking approach used during an intimidation display. |
| [`chimera_state_threaten_walk_inline.h`](chimera_state_threaten_walk_inline.h.md) | Walk toward the target while it is nearer than eight units, and stop at five. |
