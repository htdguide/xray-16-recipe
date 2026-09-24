# src/xrGame/ai/monsters/states/monster_state_panic_run_inline.h

> Run flat out away from the enemy until there is both distance and time between you.

**Needs** — [`monster_state_panic_run.h`](monster_state_panic_run.h.md)
**Used by** — [`monster_state_panic_run.h`](monster_state_panic_run.h.md)
**Tier floor** — T3: a retreat directive and a two-clause exit test

## Purpose

The running half of the panic alternation. Structurally identical to the break-away leaf of the
shot-from-nowhere behaviour — a pure retreat with no destination — but with a different anchor and
a much harder exit condition.

## State

`Stateless.` Both exit clauses are read from the enemy memory on each update.

## `execute`

**Contract** — request the running action and the panic voice, apply the aggressive acceleration
profile with braking disabled, and direct the path builder to retreat from the enemy's last known
position using the generic path parameters.

**Notes** — *retreat-from* is a path-builder mode, not a destination: it asks for a route leading
away from a point and re-evaluates it continuously. Because the anchor is the enemy's remembered
position and that position keeps updating while the enemy is visible, the retreat bends away from a
pursuer in real time — the creature does not run into a corner the enemy has cut it off in, it runs
along whatever direction currently increases separation.

Braking is disabled, so the creature does not decelerate into the pause; it goes from a full run to
a standstill, which reads as an animal stopping short to listen.

## `check_completion`

**Contract** — finished only when **both** hold: the creature is at least fifteen units from the
enemy's remembered position, and the enemy has not been seen for fifteen seconds.

```text
FUNCTION check_completion() -> bool
  IF distance(self, enemy_memory.position) < 15                RETURN false
  IF now() - enemy_memory.time_last_seen < 15000               RETURN false
  RETURN true
```

**Notes** — the conjunction is what makes the panic behaviour work, and either clause alone would
break it.

Distance alone would let a creature stop the moment it crossed fifteen units with the player in
plain sight behind it. Time alone would let a creature that lost sight of the player two paces away
stop right next to them.

Together they mean: the pause happens only when the creature has both broken contact *and* opened
ground. Since the pause itself then re-establishes contact — the creature is standing still, facing
open ground, making noise — a player who keeps following simply resets the cycle, and a player who
stops following lets the creature settle. The fifteen-second clause is why a panicking animal in
this game is genuinely hard to catch and yet never simply disappears.

Both numbers are hard-coded and shared by every species. The fifteen-unit distance coincides with
the break-away distance in the shot-from-nowhere behaviour, which suggests one notion of "out of
immediate danger" reused rather than two independent calibrations.
