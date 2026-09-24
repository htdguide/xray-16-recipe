# src/xrGame/alife_creature_abstract.cpp

> What every server-side creature settles at spawn: its faction's team, its restrictor sets, and the death timestamp of a creature that was authored already dead.

**Needs** — [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`monster_community.h`](monster_community.h.md) · [`Level.h`](Level.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a configuration lookup at spawn.

## Purpose

Every creature in the off-screen simulation — human or not — passes through one spawn hook
that turns its authored **section** into a faction allegiance. This file is that hook.

## `on_spawn`

**Contract** — Runs after the base server object's own spawn handling. Clears the creature's
dynamic restrictor sets, resolves its faction from configuration, and normalizes the death
timestamp of a creature spawned dead. Group entities skip everything after the restrictor
clear.

```text
FUNCTION on_spawn()
  base spawn handling
  clear the dynamic out-restrictors      # volumes it may not enter
  clear the dynamic in-restrictors       # volumes it may not leave
  IF this is a group entity THEN RETURN

  community = the "species" key of this creature's configuration section
  IF the community declares a team THEN adopt it
  IF the creature is not alive THEN death time = 0
```

**Invariants** — Only *dynamic* restrictors are cleared. The static ones come from the spawn
record and are part of the authored level; the dynamic ones are added at run time by scripts
and by smart terrains, and must not survive a save-load or a re-spawn — clearing them here
is what guarantees a creature starts with exactly the restrictions the level author gave it.

A group entity — a squad record rather than an individual — has no species of its own; its
members do. Returning early is not an optimization, it is the statement that a group has no
faction until its members give it one.

**Notes** — The team is adopted only when the community declares one, signalled by a
reserved value meaning "no team". Factions in the shipped data map many-to-one onto teams,
and the team is what the combat rules actually consult; the community name is what
reputation and dialogue consult.

Setting a dead creature's death time to zero, rather than to the current game time, is a
deliberate change — the commented-out original stamped the clock. Zero means "died before
the game began", which is what an authored corpse is, and it keeps corpse decay and loot
rules from treating level dressing as a fresh kill.

## `add_online` / `add_offline` (actor)

**Contract** — The player's server record promotes and demotes through the *trader* path
rather than the creature path. The two overrides exist solely to select which inherited
implementation runs, because the player is simultaneously a creature and a trading party
and the two bases both define the transition.

**Notes** — This is the diamond the engine resolves by hand. A rebuild that composes rather
than inherits — a creature record that *has* a trading aspect — never faces the choice, and
the two functions disappear.
