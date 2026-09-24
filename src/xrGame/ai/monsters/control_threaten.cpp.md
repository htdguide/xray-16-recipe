# src/xrGame/ai/monsters/control_threaten.cpp

> The threat display: the creature stops, faces its enemy, plays a warning clip, and fires a single authored callback partway through it.

**Needs** — [`control_threaten.h`](control_threaten.h.md) · [`control_animation_base.h`](control_animation_base.h.md) · [`control_animation.h`](control_animation.h.md) · [`control_direction_base.h`](control_direction_base.h.md) · [`control_movement_base.h`](control_movement_base.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: seizes the body for the duration of a clip and tracks the enemy

## Purpose

A creature's warning before it attacks — the growl, the rear-up, the display. Mechanically
it is the simplest ability that still *does* something mid-clip: it registers a marker at
an authored fraction through the clip and, when that marker fires, calls back into the
creature. What the creature does then — a sound, a psi effect, a morale hit on the player —
is the creature's business.

It is also the only ability in this slice that keeps tracking its target after it starts:
it runs on the scheduled tick to re-aim at the enemy, so a creature threatening a
circling player turns to follow.

## State

```text
RECORD ThreatenPayload
  animation : text     # a clip name, resolved on the creature's model
  time      : real     # fraction through the clip at which the callback fires
```

The ability itself holds nothing across uses.

## `activate`

**Contract** — seize the body, subscribe to animation end and to the animation-signal event,
stop the path and the movement, set a turn rate of one radian per second aimed at the enemy,
start the named clip, and register a custom marker at the authored fraction through it.

**Notes** — the marker is registered on the *motion*, not on the playing blend, so it
persists for the creature's lifetime and will fire on every subsequent play of that clip
too. Registering it on every activation therefore accumulates duplicates. No shipped
creature threatens often enough for that to be visible, but a rebuild should register it
once at load.

The turn rate of one radian per second is a constant here — deliberately slow, so the
display reads as a menacing turn rather than a snap.

## `update_schedule`

**Contract** — re-aim the heading at the enemy each scheduled tick, if an enemy still
exists. The turn rate set at activation stands.

## `on_event`

**Contract** — animation end raises the threaten-end event; an animation signal carrying the
custom marker identifier calls the creature's threat-execute hook.

**Notes** — the signal handler compares the marker identifier and ignores anything else, so
the creature's other authored markers — footsteps, hit windows — pass through untouched.

## `check_start_conditions`

**Contract** — the ability is not running, no other ability holds the body, an enemy exists,
and the creature is facing it within a twelfth of a turn.

**Notes** — no cooldown and no distance test. The rate at which a creature threatens is
governed entirely by its state layer, which is the right place for it; the distance is
governed by the state that chooses to threaten.

## `on_release`

**Contract** — release the body and unsubscribe from both events.
