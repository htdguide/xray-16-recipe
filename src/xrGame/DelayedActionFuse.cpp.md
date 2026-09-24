# src/xrGame/DelayedActionFuse.cpp

> A fuse that arms when its host's condition falls to a threshold, then burns the remaining condition away over a fixed time and fires when either runs out.

**Needs** — [`DelayedActionFuse.h`](DelayedActionFuse.h.md)
**Used by** — [`DelayedActionFuse.h`](DelayedActionFuse.h.md)
**Tier floor** — T3: one scalar integrated against the frame clock

## Purpose

The mechanism behind an item that breaks and *then* explodes a moment later — a damaged
grenade, a leaking fuel tank. It couples two clocks that would otherwise be independent:
a countdown in seconds, and the host's own condition, which the fuse consumes at whatever
rate makes it reach zero exactly when the countdown does. The coupling is the whole point:
the player sees the item's condition draining and can read how long is left off the
condition bar, without the item needing a second visible timer.

The fuse does not know what it is attached to or what happens when it fires. It demands
two operations from its host — change my condition by this much, and start whatever
telegraphs an armed fuse — and answers one question per frame: have I fired yet.

## State

```text
RECORD Fuse
  active                : bool   # counting down
  initialized           : bool   # configured and not yet fired
  no_condition_change   : bool   # pure timer mode; the fuse does not drain condition
  time                  : real   # before arming: the configured duration.
                                 # after arming: the absolute clock time it fires.
  condition_drain_rate  : real   # before arming: the arming threshold.
                                 # after arming: condition consumed per second.
```

Invariants, and the reason this record is worth writing down: **two fields change meaning
when the fuse arms.** `time` is a duration until arming and an absolute deadline after;
`condition_drain_rate` is a threshold until arming and a rate after. Nothing in the type
marks the transition except the active flag. A rebuild should use four fields and stop
paying for this.

Further: a fuse is armed at most once — firing clears both the active and initialized
flags, so a fired fuse must be re-initialized before it can arm again. Arming with a zero
duration is forbidden unless the fuse is in pure-timer mode, because the drain rate is
computed by dividing by the duration.

## `Initialize`

**Contract** — configures the fuse with a duration and a condition threshold, and marks it
usable. Refuses silently if the fuse is already counting down, so a second hit on a
burning fuse cannot restart it. A zero duration zeroes both fields, and a zero threshold
selects pure-timer mode: the fuse will count down without touching the host's condition.

## `CheckCondition`

**Contract** — the per-hit test, and the only way a fuse arms. Answers true and arms the
fuse when the fuse is initialized, not already counting, and the host's condition has
fallen to or below the configured threshold. Otherwise answers false and does nothing.
Cheap enough to call on every condition change.

## `SetTimer`

**Contract** — arms the fuse. Consumes the difference between the threshold and the
current condition immediately, so that the fuse always starts from exactly the threshold
however far past it the hit carried the host; converts the remaining threshold into a
per-second drain rate; converts the configured duration into an absolute deadline against
the global clock; and tells the host to start its armed telegraph.

```text
FUNCTION arm(current_condition)
  active = true
  host.change_condition(threshold - current_condition)   # usually negative: snap down to the threshold
  IF NOT no_condition_change THEN
    drain_rate = threshold / duration                    # so condition reaches 0 as the timer does
  deadline = duration + now()
  host.start_timer_effects()
```

**Notes** — the immediate condition adjustment is what makes an overkill hit and a
just-sufficient hit produce identical fuse behaviour. Without it, a big hit would leave
the host with less condition than the drain schedule expects and the fuse would fire
early through the condition path rather than the timer path.

## `Update`

**Contract** — advances the fuse one frame and answers whether it has fired. In pure-timer
mode the answer is simply whether the deadline has passed. Otherwise the fuse computes
where the host's condition *should* be for the time remaining, drains the difference, and
fires when the condition reaches zero. Firing clears both flags, so the caller gets the
answer exactly once. Must only be called while the fuse is counting down.

```text
FUNCTION update(current_condition) -> bool
  remaining = deadline - now()

  IF no_condition_change THEN
    fired = remaining <= 0
  ELSE
    # where the schedule says the condition should be, minus where it is
    delta = drain_rate * remaining - current_condition
    IF delta > 0 THEN delta = 0      # never restore condition; see Notes
    host.change_condition(delta)
    fired = current_condition + delta <= 0

  IF fired THEN
    active = false
    initialized = false
  RETURN fired
```

**Notes**

- Clamping the delta at zero makes the fuse a floor, not a schedule: if something else
  damages the host faster than the fuse would, the fuse lets it, and the host dies early
  through the condition test. If something *restores* the host above the schedule the
  fuse does nothing that frame rather than undoing the repair. Either way the fuse can
  only ever bring the fire forward, never push it back.
- Deriving the drain from the remaining time every frame rather than integrating a fixed
  rate means the fuse is self-correcting: a dropped frame, a paused game or an external
  condition change never desynchronizes the two clocks.
- Driving the deadline off the global wall clock rather than accumulating frame deltas
  means a fuse survives a frame-rate change exactly, and means a saved game must store
  the *remaining* time, not the deadline — which is what `Time` exists for.

## `Time`

**Contract** — answers the remaining duration in both states: the configured duration when
the fuse is merely initialized, the time left until the deadline when it is counting.
This uniformity is what lets the host serialize and display the fuse without asking which
state it is in.
