# src/xrGame/ai/monsters/states/monster_state_hitted_hide_inline.h

> Run flat out away from where the shot came from, until fifteen units of ground are between you
> and it.

**Needs** — [`monster_state_hitted_hide.h`](monster_state_hitted_hide.h.md)
**Used by** — [`monster_state_hitted_hide.h`](monster_state_hitted_hide.h.md)
**Tier floor** — T3: a retreat directive and a distance test

## Purpose

The outward half of the shot-from-nowhere oscillation. It is a pure retreat: no destination is
chosen, no cover is sought, no route is planned toward anything. The path builder is simply told
"away from this point" and the creature runs until it is far enough.

## State

`Stateless.` Completion is measured from the base's entry timestamp and from the hit memory.

## `check_start_conditions`

**Contract** — startable when the hit memory reports a hit and the creature has no enemy.

**Notes** — the "no enemy" clause is what confines this whole behaviour to unidentified attackers.
Once the creature knows who shot it, combat outranks this and the retreat is never offered.

## `execute`

**Contract** — request the running action and the panic voice, apply the aggressive acceleration
profile with braking disabled, and direct the path builder to retreat from the last recorded hit
position using the generic path parameters.

**Notes** — *retreat-from* is a different path-builder mode from *move-to*: it asks for a route
whose direction is away from a point, rather than a route to a destination, and it is re-evaluated
continuously. That is why this leaf needs no state — the anchor is read from the hit memory on
every update, so a second hit from a new direction bends the retreat immediately.

The panic voice, not the aggressive one, is the correct read: an animal shot by something it cannot
see is frightened, and the sound it makes is what tells a player at range that they connected.

## `check_completion`

**Contract** — finished once the creature is more than fifteen units from the last hit position
*and* a minimum duration has elapsed since entry.

```text
FUNCTION check_completion() -> bool
  IF distance(self.position, hit_memory.last_hit_position) < 15  RETURN false
  IF entry_time + MIN_HIDE_TIME > now()                          RETURN false
  RETURN true
```

**Notes** — fifteen units is hard-coded and is the leaf's real exit condition.

**The minimum-duration guard does not work.** It is declared as a real number with the value 3,
and the source comment beside it reads "hide more than 3 sec" — but it is added to a timestamp
measured in **milliseconds**. Three milliseconds elapse before the first update completes, so the
guard has never held back a single frame in shipped play: the leaf ends purely on distance. A
rebuild has to choose. Reproducing the original's *behaviour* means dropping the guard; reproducing
its *intent* means holding the creature in the retreat for three seconds even after it is far
enough, which measurably lengthens every oscillation cycle and changes how long a creature takes to
reach a shooter. This recipe records the behaviour as shipped and flags the intent.
