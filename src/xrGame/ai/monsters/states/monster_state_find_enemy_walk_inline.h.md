# src/xrGame/ai/monsters/states/monster_state_find_enemy_walk_inline.h

> The search's terminal leaf: stand where you are, keep making aggressive noise, and wait for
> something else to decide.

**Needs** — [`monster_state_find_enemy_walk.h`](monster_state_find_enemy_walk.h.md)
**Used by** — [`monster_state_find_enemy_walk.h`](monster_state_find_enemy_walk.h.md)
**Tier floor** — T3: two calls

## Purpose

Named `WalkAround` in the original and registered under a "walk around" identifier, but it does not
walk: it requests the standing-idle action. A rebuild should keep the *behaviour* — an alert,
stationary, vocal animal — rather than the name, and should not add wandering because the name
suggests it.

It is where the search chain ends. The chain's own transition table routes this leaf back to
itself, and its completion test never fires, so the creature stays here until the attack behaviour
above it decides the enemy is gone, a new sighting re-enters combat, or a hit pre-empts everything.
That is the intended exit path: the search does not conclude, it is concluded.

## State

`Stateless.`

## `execute`

**Contract** — request the standing-idle action and the aggressive ambient voice. No path, no
memory access, no timer.

## `check_completion`

**Contract** — always false.

**Notes** — the constant answer is the design, not an oversight, and it is why the leaf needs no
state at all. A rebuild that gives it a timeout changes when creatures disengage across the whole
game.
