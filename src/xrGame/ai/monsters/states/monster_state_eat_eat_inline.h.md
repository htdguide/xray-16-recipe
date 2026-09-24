# src/xrGame/ai/monsters/states/monster_state_eat_eat_inline.h

> Stand at a corpse and convert its food reserve into satiety, one authored slice at a time, for at
> most twenty seconds.

**Needs** — [`monster_state_eat_eat.h`](monster_state_eat_eat.h.md) · [`../monster_corpse_manager.h`](../monster_corpse_manager.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`monster_state_eat_eat.h`](monster_state_eat_eat.h.md)
**Tier floor** — T2: queries the rigid-body layer for the position of the ragdoll part nearest the
creature

## Purpose

The leaf where feeding actually changes world state. Everything else in the feeding sequence is
positioning; this is the only place a corpse's food reserve goes down and a creature's satiety goes
up. It is also the sequence's distance gate: its start-condition test is what the surrounding
sequence uses to decide whether to eat or to walk closer, so a rebuild must keep the test on this
object and not fold it into the caller.

## State

```text
RECORD EatingState
  corpse         : optional<entity>   # captured during the start-condition test, not on entry
  time_last_eat  : int (milliseconds)
```

**Invariant** — `corpse` is written by the start-condition test and read by entry, execute and the
completion test. The order of those calls is therefore load-bearing: the sequence above always asks
"can you start" before selecting this leaf, and that question is what binds the target.

**Invariant** — `corpse` is cleared when that entity is destroyed, and `execute` refuses to run
when the memory component's current corpse is not the bound one, so a destroyed body cannot be
eaten.

## `check_start_conditions`

**Contract** — bind the corpse from the memory component, then answer whether the creature is close
enough to bite: the distance from the creature to the nearest part of the body must be at least half
a unit *below* the creature's authored corpse distance.

```text
FUNCTION check_start_conditions() -> bool
  corpse = memory.current_corpse
  target = nearest_reachable_part_of(corpse)     # entity origin if no live physics body
  RETURN distance(target, self.position) + 0.5 < section["distance_to_corpse"]
```

**Notes** — the half-unit is the inner edge of a hysteresis band, and the matching outer edge is in
the completion test below. Without it the creature would start eating and immediately stop on the
frame the body twitched. The authored key `distance_to_corpse` is the only number here that varies
by creature; the half-unit margin is compiled in.

*Nearest part of the body, not the body's origin.* A corpse with an active rigid body has settled
away from the position its entity record reports. The state asks the physics layer for the element
nearest the creature; a corpse with no active body, or one whose simulation is asleep, falls back
to the entity position.

## `execute`

**Contract** — do nothing when the memory component's corpse is no longer the bound one. Otherwise
play the eating action and the eating voice, and — no more often than the authored eating
frequency allows — move one authored slice of satiety into the creature and subtract one authored
slice-weight of food from the corpse.

```text
FUNCTION execute()
  IF memory.current_corpse != corpse  RETURN
  action = eat
  voice  = eating

  interval = 1000 / section["eat_freq"]        # eat_freq is bites per second
  IF time_last_eat + interval < now()
    self.satiety  = self.satiety + section["eat_slice"]
    corpse.food   = corpse.food  - section["eat_slice_weight"]
    time_last_eat = now()
```

**Notes** — three authored keys shape this and nothing else does: `eat_freq` (how often a bite
lands), `eat_slice` (how much satiety a bite is worth to the eater) and `eat_slice_weight` (how
much reserve a bite costs the corpse). They are deliberately independent — a big animal can take
large bites out of a body that only has a few of them in it — and a rebuild must not collapse the
two slice values into one.

The corpse's food reserve is decremented here but **nothing in this leaf reads it**. Whether the
body still has food in it is the corpse-memory component's question, asked when it decides what to
offer as a target; this leaf keeps eating regardless and relies on the twenty-second cap and on the
memory component re-targeting to stop it. A body can therefore go negative for the remainder of one
sitting.

The interval is computed from a rate every frame rather than stored. If the authored frequency is
zero the interval is undefined; no configuration in the shipped data sets it that way, and nothing
guards it.

## `check_completion`

**Contract** — finished when twenty seconds have elapsed since entry, when the memory component now
names a different corpse, or when the creature has drifted more than half a unit *beyond* its
authored corpse distance from the nearest part of the body.

**Notes** — the two half-unit margins make the band: the creature must get to
`distance_to_corpse - 0.5` to begin and may drift to `distance_to_corpse + 0.5` before it stops.
The one-unit-wide band is what stops the eat/walk pair in the surrounding sequence from oscillating
every frame.

Twenty seconds is hard-coded and is a *sitting* limit, distinct from — and equal in value to — the
twenty-second hunger timer in the surrounding sequence. The coincidence means a fed creature is
almost exactly ready to be hungry again when it finishes a full sitting; nothing in the source says
whether that was intended.
