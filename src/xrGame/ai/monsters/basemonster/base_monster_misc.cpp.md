# src/xrGame/ai/monsters/basemonster/base_monster_misc.cpp

> The perception refresh: advance every memory, let the managers choose what matters, and derive the two summary flags the whole behaviour layer branches on.

**Needs** — [`base_monster.h`](base_monster.h.md) · [`EntityCondition.h`](../../../EntityCondition.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: five sub-updates and two derived predicates

## Purpose

One routine, and it is the first thing a creature's think does. It is small because all the
work is in the memories and managers it drives; what it *decides* is the last four lines,
where a creature's whole situation is collapsed into two flags.

## `update_memory`

**Contract** — advances the four memories and the two managers over them, then derives the
creature's heard-sound state, its damaged flag and its aggression flag. Called once per
think, and again from the scripted-action path.

```text
FUNCTION update_memory()
  enemy_memory.update()          # decay and revalidate remembered enemies
  sound_memory.update()          # decay heard sounds
  corpse_memory.update()
  hit_memory.update()

  enemy_manager.update()         # choose the current enemy and grade its danger
  corpse_manager.update()        # choose the corpse worth going to

  heard_dangerous = heard_interesting = false
  IF a sound is remembered THEN
    take the strongest remembered sound; it is one or the other, never both

  damaged   = health < settings.damaged_threshold
  aggressive = heard_dangerous OR enemy_count > 0 OR recently hit
```

**Invariants** — the memories are advanced before the managers, because a manager's choice
must be made over current memory. The two sound flags are mutually exclusive by
construction: a remembered sound is classified as dangerous or not, and the interesting flag
is the negation.

**Notes** — the **damaged** flag is the trigger for the wholesale movement-animation
substitution described in [`ai_monster_defs.h`](../ai_monster_defs.h.md). It is recomputed
every think from a threshold in the settings block, so a creature healed above the threshold
walks normally again without anything resetting it.

The **aggressive** flag is the single predicate that separates a creature going about its
business from one that is roused, and it is deliberately generous: hearing one dangerous
sound is enough, and it stays true as long as a hit is within the hit memory's fifty-second
window. It gates the transition-table shortcuts (an aggressive creature skips polite
stand-ups), the sound selection, and several states' entry conditions.

Only *one* remembered sound is consulted, not all of them. A creature that hears a shot and
then a footstep reports only the louder, and the rest of the sound memory is read directly
by whichever state cares.
