# src/xrGame/ai/stalker/ai_stalker_impl.h

> Two inline helpers that need heavy includes: reaching the squad's shared brain, and the recoil-to-aim conversion that is switched off.

**Needs** — [`ai_stalker.h`](ai_stalker.h.md) · [`agent_manager.h`](../../agent_manager.h.md) · [`EffectorShot.h`](../../EffectorShot.h.md)
**Used by** — [`ai_stalker.cpp`](ai_stalker.cpp.md) · [`ai_stalker_fire.cpp`](ai_stalker_fire.cpp.md) · [`ai_stalker_misc.cpp`](ai_stalker_misc.cpp.md) · [`stalker_movement_restriction.h`](../../stalker_movement_restriction.h.md)
**Tier floor** — T3: a registry walk and a disabled angle composition

## Purpose

Exists only so that the two functions below can be inline without their includes leaking
into every file that sees the stalker's declaration. A rebuild with modules or with a
different compilation model deletes the file and puts both beside their callers.

## `agent_manager`

**Contract** — returns the squad's shared coordination object by walking the world's
hierarchy: seniority level, then team, then squad, then group. Every stalker in the same
(team, squad, group) triple reaches the same object, and that object is where shared
knowledge lives — who is in combat, which enemies the squad collectively knows about, which
covers are taken, where the danger locations are.

**Invariants** — squad membership is therefore *derived from three small integers on the
entity*, not from a membership list. Changing a stalker's team, squad or group silently
moves it to a different shared brain, which is why the class has explicit before- and
after-team-change hooks that carry combat registration across the move.

**Notes** — this walk happens on every access and there are many. It is one of the clearest
places where the recipe's global-environment note applies: a rebuild should hand the stalker
its agent manager rather than making it look one up.

## `weapon_shot_effector_direction`

**Contract** — would convert the recoil effector's current angular offset into a perturbed
aim direction. **It returns its input unchanged.** The composition that would apply the
recoil is compiled out.

The consequence is worth stating plainly, because half a dozen call sites in
[`ai_stalker_fire.cpp`](ai_stalker_fire.cpp.md) guard calls to this with "if the recoil
effector is active": **a stalker's aim is unaffected by its own recoil.** The effector still
runs, and it still shakes the camera when the player is possessing the stalker, but the
rounds go where the stalker aimed. All scatter comes from the dispersion model instead.

A rebuilder deciding whether to implement recoil-driven aim should know they would be adding
behaviour, not restoring it, and that stalker accuracy is tuned against its absence.
