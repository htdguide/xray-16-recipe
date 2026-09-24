# src/xrGame/ExplosiveItem.cpp

> A canister or gas bottle: an inventory item that takes damage like an item, counts down on a fuse once damaged enough, and then explodes.

**Needs** — [`ExplosiveItem.h`](ExplosiveItem.h.md) · [`Explosive.h`](Explosive.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`DelayedActionFuse.h`](DelayedActionFuse.h.md) · [`ParticlesPlayer.h`](ParticlesPlayer.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a fuse over an item's condition, and three inheritance orderings

## Purpose

The environmental explosive: fuel canisters and gas bottles scattered through the levels for
the player to shoot. What makes it its own class rather than a configuration of the grenade
is the **fuse**: it does not explode when hit, it explodes a configured time *after* its
condition falls below a configured threshold, and it hisses visibly in between.

That delay is the entire gameplay value of these objects. A player who shoots a canister has
time to take cover, or to notice it and move away, and an enemy standing beside one does not.

## State

```text
RECORD ExplosiveItemTuning
  time_to_explode      : real   # seconds between the fuse lighting and the explosion
  condition_to_explode : real   # the condition below which the fuse lights
  set_timer_particles  : text   # invariant: this key must exist; asserted at load
  use_condition        : bool   # whether the item has a condition at all; default true
```

The fuse's own state — lit or not, time remaining — belongs to the fuse. Everything else
belongs to one of the three parents.

## `Load`

**Contract** — read the item's parameters, the explosion's parameters and the fuse's two
numbers from one section. Fails hard if the warning effect is not named.

**Notes** — the assertion on the warning effect is the only hard requirement, and it is
correct to make it hard: an explosive that lights its fuse invisibly is indistinguishable
from one that is about to kill you for no reason. The player must be shown the hiss.

The condition flag was added so that an item can opt out of having a condition at all, which
matters because the fuse's threshold is expressed in condition.

## `Hit`

**Contract** — take damage; light the fuse if this hit took the condition below the threshold;
remember who is to blame. **A hit that arrives while the fuse is already lit does nothing.**

```text
FUNCTION hit(descriptor)
  IF the fuse is already lit THEN descriptor.power = 0
  apply the hit as an ordinary inventory item
  REQUIRE the hit names an attacker
  IF the fuse is not lit AND the condition has now crossed the threshold THEN
    SetInitiator(the attacker's identifier)
```

**Invariants** — zeroing the power of subsequent hits is what stops a burning canister from
being destroyed by the next bullet before its fuse runs out. The countdown, once started, is
the only thing that can end the item.

The attacker recorded is **the one who lit the fuse**, not the last one to shoot it, and it is
recorded once. That is who the resulting kills are attributed to, which is the point:
shooting a canister next to an enemy is your kill.

**Notes** — the condition test is written so that the attacker's existence is folded into the
value being tested against the threshold rather than being a separate conjunct. It works
because an absent attacker makes the tested value zero, which is below any threshold — but it
means the assertion above it, not this expression, is what actually guarantees an attacker.
A rebuild should write the two conditions separately.

## `shedule_Update` and `shedule_Needed`

**Contract** — run the fuse. The item asks to be scheduled at all whenever its fuse is lit,
even if it would otherwise be idle.

```text
FUNCTION scheduled_update(elapsed)
  run the item's own scheduled work
  IF the fuse is lit AND advancing it against the current condition says it has expired THEN
    find the surface normal beneath us
    request an explosion at our position with that normal
    stop the warning effect

FUNCTION scheduled_needed() -> bool
  RETURN the item would be scheduled anyway, OR the fuse is lit
```

**Invariants** — the second function is what makes the first one run. A canister lying on the
floor far from the player is otherwise a candidate for the scheduler to skip almost entirely;
a lit fuse forces it back into the rotation, so the explosion happens on time whether or not
the player is nearby.

The surface normal is found by a downward ray (see [`Explosive.cpp`](Explosive.cpp.md)) so the
explosion's effect is oriented against the ground rather than arbitrarily.

## `StartTimerEffects`

**Contract** — play the configured warning effect on the item, oriented upward. Called by the
fuse when it lights.

**Notes** — the effect is tagged with the item's own entity identifier, which is how
`shedule_Update` finds it again to stop it. An explosion that is *interrupted* — which cannot
currently happen — would leave it playing.

## `GetRayExplosionSourcePos`, `ActivateExplosionBox`

**Contract** — the two explosion hooks this class fills. The blast's sampling rays start from
a uniformly random point inside the item's own bounding box. The expanding-shape step is
**overridden to do nothing**.

**Notes** — suppressing the expanding shape is a real decision, not an omission: the base
class's version pushes nearby bodies apart from the explosion's centre, and a canister
standing on the floor with an expanding shape at its centre launches itself and everything
resting on it. These objects rely on the blast wave's impulses alone.

## The dispatch

**Contract** — the remaining functions pick a parent. `net_Spawn`, `net_Export` and
`net_Import` are the item's alone; `net_Destroy`, `net_Relcase` and `OnEvent` run both;
`UpdateCL` runs the explosion's advance **first** so that a completed explosion tears the
object down before the item's update touches it; `ChangeCondition` is forced to the inventory
item's version, because the base explosive's would treat the condition as health.
