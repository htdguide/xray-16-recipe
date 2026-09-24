# src/xrServerEntities/xrServer_Objects_ALife_Monsters_script4.cpp

> Exports the monster level — the smart-terrain assignment, the brain, the travel speeds and the relation override — and the human level.

**Needs** — [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`alife_human_brain.h`](alife_human_brain.h.md) · [`alife_monster_brain.h`](alife_monster_brain.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

The fourth and last part of the creature export, and the one a mod's alife script spends most
of its time in. The monster level is where the **smart-terrain state machine** is reachable
from script, and every job-assignment mod in existence drives it through this surface.

## `cse_alife_monster_abstract`

**Contract** — the scheduled creature, at the monster level, plus:

**The smart-terrain assignment** — the state machine described in
[`xrServer_Objects_ALife_Monsters.cpp`](xrServer_Objects_ALife_Monsters.cpp.md), exported as
five entry points:

- `smart_terrain_id` — read the assignment.
- `m_smart_terrain_id` — the same field, **also exposed directly and writably** under its
  internal name.
- `clear_smart_terrain` — set it to unassigned.
- `smart_terrain_task_activate` / `smart_terrain_task_deactivate` — set and clear "has
  arrived".

**Invariants** — **the assignment is reachable through three names with different
capabilities**: a read-only accessor, a read/write field, and a clear operation. All three
address the same storage. That is redundancy frozen by conformance criterion 10 — shipped
scripts use all three — and a rebuild must provide all three.

**Nothing enforces the state machine.** A script can set "arrived" on a creature with no
assignment. The simulation then treats it as idle-but-arrived, which is a state the design
does not have a meaning for.

**The brain** — `brain` answers the record's offline decision-making, which is where a script
reads and steers an offline creature. It **fails hard on a null record** rather than
answering nothing, which is the right choice: a script calling a method on nothing has a
bug, and a silent nothing makes it a corruption instead.

**The rest** — `group_id` (read-only: which squad), `rank`, and in the game build only:

- `travel_speed` and `current_level_travel_speed`, each in **read and write forms under one
  name** distinguished by arity. These set how fast an *offline* creature crosses the game
  graph, which is the single most useful knob a pacing mod has.
- `kill` — the operation that unregisters from the squad and then zeroes health.
- `has_detector` — does this creature carry an artefact detector, answered by scanning its
  children.
- `force_set_goodwill` — override this creature's disposition toward another by identity,
  writing into the relation registry.

**Notes** — the relation override is the only method in the whole chapter that writes into a
subsystem outside this directory. It is here rather than in the relation system's own export
because a script holding a record wants to change *that record's* disposition, and reaching
the registry directly would mean the script needs both identities.

## `cse_alife_human_abstract`

**Contract** — the human, at the monster level with the identity mixin and the monster level
as bases. Adds `brain`, answering the **human** brain rather than the monster one, and
re-exports rank and its setter from the identity mixin.

**Invariants** — **rank is exported twice on this type**: once here from the identity mixin
and once inherited from the monster level, where it means the creature's combat rank. The two
are different numbers under one script name, and which one a call reaches depends on the
binding layer's resolution order. This is a genuine ambiguity in a frozen surface; a rebuild
should reproduce whichever the original resolves to and document it, because shipped scripts
read it.

The human brain accessor likewise fails hard on nothing.

## `cse_alife_psydog_phantom`

**Contract** — at the monster level, no added surface.
