# src/xrGame/rat_states.h

> Declares the twelve behaviour states a rat can be in.

**Needs** — [`rat_state_base.h`](rat_state_base.h.md) · [`rat_state_manager.h`](rat_state_manager.h.md)
**Used by** — [`ai_rat.cpp`](ai/monsters/rats/ai_rat.cpp.md) · [`rat_states.cpp`](rat_states.cpp.md)
**Tier floor** — T3: twelve declarations against one interface

## Purpose

Names the rat's behaviour repertoire. Each entry implements the enter/run/leave contract of
[`rat_state_base.h`](rat_state_base.h.md) and adds no state of its own — every state is a
pure decision procedure over the rat it is handed. The substance is in
[`rat_states.cpp`](rat_states.cpp.md).

The list is closed and its order matches the identifier enumeration the rat registers them
under, so a rebuild that reorders it must reorder both together.

Exported units, in identifier order:

- `death` — the rat is dead; the only state that can end the entity's existence.
- `free active` — the default: roam, watch for enemies, food and noise.
- `free passive` — roam without reacting; the offline-ish idler.
- `attack range` — hold a firing position against the selected enemy.
- `attack melee` — close on the enemy; also the squad-leader election site.
- `under fire` — morale has broken; back away from the last noise.
- `retreat` — withdraw while the enemy is still live.
- `pursuit` — chase an enemy that is out of perception but not yet given up on.
- `free recoil` — flinch away from a noise that is not an enemy.
- `return home` — walk back inside the home radius.
- `eat corpse` — feed on a remembered item.
- `no way` — the path is blocked or a patrol path is in force; walk the way points.
