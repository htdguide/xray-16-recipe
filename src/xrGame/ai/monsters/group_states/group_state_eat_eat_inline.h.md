# src/xrGame/ai/monsters/group_states/group_state_eat_eat_inline.h

> Take one bite per interval from the corpse, and stop when the meal is over, the body is out of
> reach, or the pack leader turns up.

**Needs** — [`group_state_eat_eat.h`](group_state_eat_eat.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Configuration (ltx)](../../../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`group_state_eat_eat.h`](group_state_eat_eat.h.md)
**Tier floor** — T3: a metered transfer on a clock, plus a distance predicate against a ragdoll

## Purpose

The rung of the pack feeding sequence where something actually changes in the world: the
creature's satiety rises and the corpse's remaining food falls, in discrete bites at a rate
authored per creature. Everything else about feeding is positioning; this is the transaction.

Its three tuning numbers all come from the creature's configuration section, which is the point
worth carrying into a rebuild: *how fast* a creature eats, *how much* a bite is worth to it, and
*how much* a bite costs the body are three independent authored numbers, not one.

## State

```text
RECORD GroupEatingState
  corpse        : object   # verified in the start check, re-verified every tick
  time_last_eat : int      # the bite clock; cleared on entry
```

`TIME_TO_EAT` is 20 seconds, authored in code: the hard cap on one uninterrupted meal.

## `check_start_conditions`

**Contract** — latch the creature's claimed corpse, find the nearest gripped bone of its ragdoll
(or the corpse's origin when it has no active physics shell), and accept only when that point is
closer than the section's *distance to corpse* **minus half a unit**.

**Notes** — measuring to the nearest *bone* rather than to the corpse's origin is the same decision
the approach rungs make, and for the same reason: a settled ragdoll's origin is not where its body
is. The half-unit margin below the nominal distance is deliberate hysteresis against the finish
test, which uses the same distance **plus** half a unit. The one-unit band between them is what
stops the creature flickering between approaching and eating as the ragdoll settles.

This predicate has a side effect — it latches the corpse — which is unusual for a start check and
is what the execute body's guard relies on.

## `execute`

**Contract** — refuse to do anything if the creature's claimed corpse is no longer the one latched.
Otherwise request the eating action and the eating sound, and transfer one bite each time the bite
interval has elapsed.

```text
FUNCTION execute()
  IF creature.claimed_corpse != corpse   RETURN     # the claim moved under us

  request_action(eat)
  set_state_sound(eating)

  IF time_last_eat + (1000 / section.eat_frequency) < now()
    creature.satiety     += section.eat_slice
    corpse.food_remaining -= section.eat_slice_weight
    time_last_eat = now()
```

**Notes** — the bite interval is expressed in the data as a *frequency* and inverted here into
milliseconds. A section that authors a frequency of zero divides by zero; nothing guards it, so
the data is required to be sane. A rebuild should either guard or state the requirement.

The two halves of a bite are **separately authored and need not match**: what the creature gains
(`eat_slice`) and what the corpse loses (`eat_slice_weight`) are different keys in the section.
That is not an oversight — it lets a small creature take many small bites from a body that depletes
slowly, and it means a rebuild cannot collapse them into one number.

The guard at the top is the only thing protecting the transfer from running against a corpse the
creature no longer owns. It compares against the *creature's* claim rather than re-checking the
lock, which is cheaper and, given that the claim is cleared on every exit from the feeding
composite, sufficient.

## `check_completion`

**Contract** — four independent ways to stop:

```text
FUNCTION is_finished() -> bool
  # 1. the pack leader arrived: yield the meal and growl about it
  IF in an active squad AND I am not the leader
     AND distance(self, leader) < 5
    request flavour clip 6 (growl while standing)
    RETURN true

  # 2. the meal has run its course
  IF time_state_started + TIME_TO_EAT < now()       RETURN true

  # 3. someone else's claim replaced ours
  IF creature.claimed_corpse != corpse              RETURN true

  # 4. we drifted out of reach of the nearest bone
  IF distance_to_nearest_bone > section.distance_to_corpse + 0.5  RETURN true

  RETURN false
```

**Notes** — the leader rule is the pack behaviour in this file and it is pure hierarchy: a
subordinate that is eating when the leader comes within 5 units gives up the body and growls. The
growl is requested through the numbered-animation machine, so the *completion test has a side
effect* — it writes an animation request into the creature. That is unusual and load-bearing: the
growl is what the player sees, and a rebuild that treats completion tests as pure will lose it.

The twenty-second cap is measured from when the state was entered, not from the last bite, so a
creature that keeps being interrupted never gets a full meal. Combined with the feeding
composite's twenty-second satiety clock, a creature can feed at most half the time.
