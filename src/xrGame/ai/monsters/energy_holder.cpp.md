# src/xrGame/ai/monsters/energy_holder.cpp

> A rechargeable budget for a creature's always-on ability: it drains while the ability is on,
> refills while it is off, and can switch itself on and off at two different thresholds.

**Needs** — [`energy_holder.h`](energy_holder.h.md) · [Seam: Configuration (ltx)](../../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`energy_holder.h`](energy_holder.h.md)
**Tier floor** — T3: a clamped accumulator with a two-threshold latch

## Purpose

Some creature abilities are fields rather than actions: the psychic aura a controller radiates,
the poltergeist's ability to stay materially present. They have no animation and no target — they
are simply on or off — and what governs them is stamina. This file is that stamina, factored out
so that both of those creatures inherit it rather than each growing its own timer.

It is a mixin, not a component: the creature *is* an energy holder, and overrides the two
notification hooks to make the ability actually happen. That keeps the budget and the ability
inseparable, which is right — an energy value nobody spends is meaningless.

## State

```text
RECORD EnergyHolder
  value              : real   # 0..1, the remaining budget; invariant: always clamped
  active             : bool   # is the ability currently on
  time_last_update   : int    # monotonic; the accumulator's anchor
  enabled            : bool   # is the budget being simulated at all
  aggressive         : bool   # selects the faster of the two refill rates

  # all five from the creature's configuration section:
  restore_rate            : real   # budget per second while off
  aggressive_restore_rate : real   # budget per second while off and aggressive
  decline_rate            : real   # budget per second while on
  critical_value          : real   # below this, the ability should switch off
  activate_value          : real   # above this, the ability may switch on
```

**The two thresholds are separate numbers and that is the point.** If they were one value the
ability would chatter on and off at the boundary; with `activate_value` above `critical_value`
the budget must genuinely recover before the ability returns. The configuration data is what
guarantees the gap — nothing in the code enforces `activate_value > critical_value`, and a
section that inverts them produces an ability that flickers every update.

A budget starts full and active: a creature spawns with its field already on.

## `reload`

**Contract** — read the five rates and thresholds from a configuration section, and clear the
aggressive flag. The five key names are built by concatenating a caller-supplied prefix, a fixed
middle, and a caller-supplied suffix, so one creature section can carry several independently
tuned budgets (an aura and a materialisation budget, say) without name collisions. Every key is
required; a section missing one fails to load.

**Notes** — the prefix/suffix scheme is the only reason this is a function rather than five reads
at the call site, and it is worth preserving: it is how the data author addresses "the second
energy budget of this creature" without the engine enumerating them.

## `schedule_update`

**Contract** — advance the budget by the real time since the last call and, if automatic
switching is armed, act on the thresholds. Does nothing at all while disabled. Called from the
creature's coarse update, not every frame.

**Invariants** — the budget is clamped to 0..1 after every step, so a long pause between updates
cannot overshoot; the update anchor is stamped after the step, so no interval is counted twice or
skipped.

```text
FUNCTION schedule_update()
  IF NOT enabled  RETURN

  dt = seconds_since(time_last_update)

  IF active
    value = value - decline_rate * dt
  ELSE
    value = value + (aggressive ? aggressive_restore_rate : restore_rate) * dt

  clamp(value, 0, 1)
  time_last_update = now()

  IF active     AND value < critical_value AND auto_deactivate  THEN deactivate()
  IF NOT active AND value > activate_value AND auto_activate    THEN activate()
```

**Notes** — the update is driven by *elapsed real time*, not by a fixed number of updates, which
is what makes the rates meaningful as "per second" and keeps them correct when the scheduler
degrades a distant creature's update rate. This is the load-bearing reason the anchor exists;
a rebuild that decrements per update makes every rate depend on how far away the player is
standing.

Automatic switching is off by default and each direction is armed separately, so a creature may
choose to switch its ability on by hand but have it drop out automatically when exhausted. The
poltergeist's materialisation uses exactly that asymmetry.

## `activate` / `deactivate`

**Contract** — set the flag, and call the corresponding notification hook *only on a genuine
transition*. Idempotent: activating an already-active budget does nothing but re-assert the flag.

**Invariants** — the hook fires exactly once per transition. That is what lets the overriding
creature allocate and release the ability's real resources (a sound, a particle effect, a touch
registration) in the hook without reference counting.

## `enable` / `disable`

**Contract** — `disable` suspends simulation entirely; the budget freezes at its current value and
the ability's current on/off state is untouched. `enable` resumes it **and re-anchors the clock**,
so that the suspended interval is not charged against the budget when it resumes.

**Notes** — re-anchoring on resume is the whole reason `enable` is a function rather than a flag
assignment. Forgetting it means a creature that was offline for ten minutes has its field
instantly drained or instantly full on its first update back. This is the same hazard as any
accumulator that survives a pause, and it is the one line of this file most likely to be lost in
a rebuild.
