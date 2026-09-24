# src/xrGame/ai/monsters/states/monster_state_find_enemy_angry_inline.h

> Four seconds of standing still, posturing and snarling, in the middle of a failed search.

**Needs** — [`monster_state_find_enemy_angry.h`](monster_state_find_enemy_angry.h.md)
**Used by** — [`monster_state_find_enemy_angry.h`](monster_state_find_enemy_angry.h.md)
**Tier floor** — T3: an animation flag and a timer

## Purpose

A pure display leaf. Nothing about the world changes while it runs: no movement, no path, no memory
update, no effect on the enemy. It exists so that a player hiding a few metres away hears and sees
an animal that knows they are there, which is the moment a lost-contact search pays off.

That makes it the clearest example in this directory of a state whose entire contract is
*presentation*, and a rebuild that optimizes it away because it "does nothing" removes a designed
beat.

## State

`Stateless.` Completion is measured against the base's entry timestamp.

## `execute`

**Contract** — request the standing-idle action with the threatening special-parameter flag, and the
aggressive ambient voice. Runs every update; sets no path and touches no memory.

**Notes** — the threat posture is delivered as a *special parameter* on the animation request
rather than as a distinct action, because the animation layer picks a variant of the idle motion
from that flag. Which variant exists is per-creature model data, so a creature with no threatening
idle simply stands there — the behaviour degrades to a pause rather than failing.

## `check_completion`

**Contract** — finished four seconds after entry.

**Notes** — four seconds is hard-coded and identical for every creature. It is long enough to be
unmistakable and short enough that a player who has decided to run gets a usable head start; nothing
in the source derives it.
