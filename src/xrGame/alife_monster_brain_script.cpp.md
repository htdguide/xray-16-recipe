# src/xrGame/alife_monster_brain_script.cpp

> Exports the offline creature brain to scripts: read its movement manager, force an update, and toggle whether it may accept simulation-assigned tasks.

**Needs** — [`alife_monster_brain.h`](../xrServerEntities/alife_monster_brain.h.md) · [`alife_monster_movement_manager.h`](alife_monster_movement_manager.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one registration.

## Purpose

Registers the offline creature brain as a script class with no constructor — scripts reach
one through a creature record, never build one — and three members.

## `script_register`

**Contract** — Exports the type under its own name with:

- **`movement`** — the brain's offline movement manager, read-only;
- **`update`** — force one brain step *now*, bypassing the simulation's own scheduling;
- **`can_choose_alife_tasks`** — set whether this brain may be handed tasks by the
  simulation. Turning it off is how a script takes exclusive control of a creature's
  off-screen behaviour: the simulation stops assigning it anywhere to go.

**Invariants** — The forced update is exported with its *forced* flag hard-set, so the script
surface has no way to request a non-forced one. That is deliberate: a script calling update
wants the brain re-evaluated immediately, which is the only reason to call it at all.

**Notes** — The exported names are frozen by conformance criterion 10.
