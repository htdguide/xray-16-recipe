# src/xrGame/ai/monsters/control_com_defs.h

> The vocabulary of the monster control bus: which control channels exist, which events travel on it, and the three resources an ability can seize.

**Needs** — _(none: this file is the root of the monster control vocabulary)_
**Used by** — [`control_combase.h`](control_combase.h.md) · [`control_manager.cpp`](control_manager.cpp.md) · [`control_manager.h`](control_manager.h.md) · [`state.h`](state.h.md) · [`state_manager.h`](state_manager.h.md)
**Tier floor** — T3: three enumerations and two empty marker types

## Purpose

Every creature in chapter 24 is driven by a small bus of cooperating *control elements*
("coms"). This file names the channels on that bus, the events that flow across it, and
the subset of channels an ability declares it will seize. It is a separate file for one
reason: every com header includes it, and nothing else, so it must have no dependencies of
its own.

The names are load-bearing beyond this chapter — script-facing capture and release take a
channel identifier by name, and a save-compatible rebuild must keep the channel set.

## State

Stateless.

## `EControlType` — the channels

The bus is layered, and the layering is the whole design. A channel of a lower layer is a
*resource*; a channel of a higher layer is a *client* that seizes resources.

```text
ENUM ControlChannel
  # layer 1 — the four resources a creature's body exposes
  movement          # linear velocity along the path
  path              # the path builder: where the body is heading
  direction         # the model's heading and pitch
  animation         # the animation mixer

  # layer 2 — animation clients that seize only the animation channel
  sequencer         # play a list of clips back to back
  triple_animation  # a prepare / execute / finish triad

  # layer 3 — abilities. each seizes several layer-1 resources at once
  jump              # seizes path, movement, animation, direction
  rotation_jump     # seizes path, movement, sequencer, direction
  run_attack        # seizes path, movement, sequencer
  threaten
  melee_jump        # seizes path, movement, sequencer, direction

  # the base controllers — the default occupant of each layer-1 channel
  animation_base
  movement_base
  path_base
  direction_base

  custom            # a creature-specific com
  custom_1          # a second creature-specific slot
  critical_wound
  anti_aim

  channel_count
  invalid           # sentinel, the all-ones value of the tag's width
```

**Notes** — the tag is one byte wide and `invalid` is its all-ones value, so the channel
count must stay under 255; nothing else depends on the width. The comments beside the
layer-3 entries are the *only* record of which resources each ability takes — the
abilities themselves seize whatever they like at run time and nothing checks the comment.
A rebuild would do well to make the declaration real.

There is a channel for sound in the original, commented out. Sound is driven from the
state layer instead.

## `EEventType` — the bus events

One creature's coms talk to each other only through these. A com subscribes to an event
and the manager delivers it to every subscriber in subscription order.

```text
ENUM ControlEvent
  animation_start          # the mixer has begun the clip a com asked for
  animation_end            # ... and finished it
  legs_animation_end       # same, for the lower-body partition
  torso_animation_end      # same, for the upper-body partition
  animation_signal         # an authored marker inside a clip fired
  sound_start / sound_end
  particles_start / particles_end
  step                     # a footfall marker
  triple_animation_change  # the prepare/execute/finish triad advanced
  velocity_bounce          # the body's speed changed abruptly (see below)
  sequence_end
  rotation_end             # heading or pitch reached its target
  travel_point_change      # the body passed a waypoint of the detailed path
  path_built
  path_updated
  path_selector_failed
  jump_end
  rotation_jump_end
  melee_jump_end
  run_attack_end
  threaten_end
  critical_wound_end
```

**Notes** — the `*_end` events are how an ability tells the creature's custom manager "I
am finished, release my channel". They are not an ability finishing cleanly versus
failing: an ability that cannot start raises its own end event immediately, and the
manager cannot tell the two apart. That is deliberate — the only thing the manager does
either way is release the channel.

`velocity_bounce` is the engine's stand-in for a landing detector: the physical body's
speed is sampled each frame and a ratio between consecutive samples beyond a threshold
raises the event, with the sign saying whether the body sped up or was stopped. A jump in
flight uses a sharp *deceleration* as its "I hit something" signal.

## `ECaptureType` — the seizure declaration

A bit set naming layer-1 resources, used only by the triple-animation com, which lets its
caller choose which of path, movement and direction it should freeze while it plays.

```text
ENUM CaptureFlags (bit set)
  capture_path
  capture_movement
  capture_direction
```

## `IComData` and `IEventData`

Two empty marker types. Every channel's payload record and every event's payload record
derives from one of them, and the bus passes them as the marker type and the receiver
casts back. In a rebuild these are a tagged union or a generic parameter; what must
survive is the fact that *each channel owns exactly one payload record* whose shape both
the resource and its current capturer agree on.
