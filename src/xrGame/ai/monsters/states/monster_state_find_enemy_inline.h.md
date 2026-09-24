# src/xrGame/ai/monsters/states/monster_state_find_enemy_inline.h

> A four-step search after losing sight of an enemy — charge past the last known spot, cast about,
> snarl, then mill around indefinitely.

**Needs** — [`monster_state_find_enemy.h`](monster_state_find_enemy.h.md) · [`monster_state_find_enemy_run.h`](monster_state_find_enemy_run.h.md) · [`monster_state_find_enemy_angry.h`](monster_state_find_enemy_angry.h.md) · [`monster_state_find_enemy_walk.h`](monster_state_find_enemy_walk.h.md) · [`monster_state_find_enemy_look.h`](monster_state_find_enemy_look.h.md)
**Used by** — [`monster_state_find_enemy.h`](monster_state_find_enemy.h.md)
**Tier floor** — T3: a transition table over four sub-behaviours

## Purpose

This is the creature-side answer to "the player broke line of sight". It is entered from the attack
behaviour when the enemy has not been seen for long enough, and it is what gives the player the
experience of being hunted rather than simply dropped. The composite itself is four lines of
transition; every interesting decision is in its leaves.

## State

`Stateless.` The composite carries only the base's current-leaf and previous-leaf record.

## `reselect_state`

**Contract** — advance one step along a fixed chain, with the last step absorbing.

```text
FUNCTION next_leaf(previous) -> leaf
  IF previous is none   RETURN charge          # run past the last known position
  IF previous == charge      RETURN cast_about
  IF previous == cast_about  RETURN threaten
  IF previous == threaten    RETURN mill_around
  RETURN mill_around                           # absorbing
```

**Notes** — the shape of this chain is the behaviour's whole design, and it has a deliberate
rhythm: **commit, search, display, give up without leaving.**

*Commit* is a charge to a point ten units *past* where the enemy was last seen, not to the spot
itself — a creature that stopped where the player was standing would always arrive behind them.

*Search* is the elaborate leaf: a randomized alternation of looking around and short dashes to
either side, described in
[`monster_state_find_enemy_look_inline.h`](monster_state_find_enemy_look_inline.h.md).

*Display* is four seconds of threatening posture and aggressive voice. It accomplishes nothing
mechanically; its entire purpose is to be seen and heard by a player who is hiding nearby, which is
the moment the encounter is decided.

*Give up without leaving* is the absorbing final leaf. It never completes on its own: the creature
stands in place making aggressive noise until something above — the attack behaviour's own
completion test, a new sighting, a hit — pulls it out. There is deliberately no path back to the
charge, so a creature searches once per loss of contact and does not re-charge the same stale
position forever.

The chain is entered fresh each time the attack behaviour selects it, so re-acquiring and re-losing
an enemy restarts at the charge.
