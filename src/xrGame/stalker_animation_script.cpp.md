# src/xrGame/stalker_animation_script.cpp

> The script-animation queue: how a script's request to play an animation is validated, queued, started, and advanced when one finishes.

**Needs** — [`stalker_animation_script.h`](stalker_animation_script.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_animation_pair.h`](stalker_animation_pair.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`game_object_space.h`](game_object_space.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`stalker_animation_script.h`](stalker_animation_script.h.md); callers name that, not this file.
**Tier floor** — T2: a callback arriving from the renderer's animation notification, carrying an opaque owner reference.

## Purpose

This is the part of the animation manager that belongs to the script layer. A script asks a
creature to play a named animation; the request joins a queue; the queue's head is what the
script animation channel plays; and when a motion ends the queue advances and the script is
told. Those four steps live here rather than with the rest of the manager because they are
the only part of animation driven from outside the engine, and because they are the part a
modder can break.

## State

The queue itself — a list of [`ScriptAnimation`](stalker_animation_script.h.md) records —
lives on the animation manager. Two flags on the manager are set from here:

```text
call_script_callback       : bool   # a motion just ended; tell the script layer next update
start_new_script_animation : bool   # the channel is free; the next queued entry may start
```

**Invariants** — both flags are *raised* here and lowered by the manager's update. They
exist because the notification arrives from the renderer at an arbitrary point in the
frame, and neither starting an animation nor calling into the script engine is safe at that
point.

## `script_play_callback`

**Contract** — invoked by the renderer when a motion on the script channel finishes. It
receives only the motion's blend record plus the opaque owner reference that was registered
with it, from which the creature is recovered. Pops the queue if the finished motion is
still the queue head, raises both deferred flags, and then notifies the channel's own
subscribers. Does not allocate, does not call into the script engine, and does not start
anything.

```text
FUNCTION on_script_motion_finished(blend)
  creature := blend.owner
  channel  := creature.animation.script_channel
  queue    := creature.animation.script_animations

  # Pop only if what finished is still what the head asks for. The head may already have
  # been replaced — by a plan switch clearing the queue, or by a script pushing a new
  # sequence — and popping then would discard an animation that never played.
  IF channel.wanted exists AND queue is not empty AND channel.wanted == queue.front.animation THEN
    pop queue front

  call_script_callback       := true
  start_new_script_animation := true
  channel.on_animation_end()
```

**Invariants** — the guarded pop is the load-bearing line. Everything else in the function
is bookkeeping; the guard is what makes the queue survive being cleared underneath a
running animation, which happens on every plan switch (see
[`stalker_base_action.cpp`](stalker_base_action.cpp.md)).

**Notes** — the two flags rather than direct action: the script callback can run arbitrary
Lua, including code that despawns the creature whose animation just ended. Deferring to the
manager's update means that happens at a point where the creature is known to be alive and
the animation channel is not mid-notification.

## `add_script_animation` — with a destination

**Contract** — queue an animation named by string, with a position and a rotation saying
where it should end. Looks the name up in the creature's motion bank; if there is no such
motion the request is **dropped with a logged script error** and nothing is queued. Always
requests movement-controller playback, because supplying a destination only makes sense for
an animation that carries the creature to it.

```text
FUNCTION add_script_animation(name, hand_usage, position, rotation_degrees, local_animation)
  motion := motion_bank.lookup(name)
  IF motion is none THEN
    script_error("there is no animation <name> on <creature>")
    RETURN                                    # a bad name is a mod bug, not a crash
  destination := transform_from(rotation_degrees converted to radians, position)
  queue.push(ScriptAnimation(motion, hand_usage, movement_controller = true,
                             destination, local_animation))
```

**Notes** — rotation arrives in degrees because it is written by hand in Lua, and is
converted at the boundary so that nothing downstream has to know. That conversion point is
the whole reason this overload exists separately from the one below.

A missing motion logs and returns rather than failing. The script surface is a modding
contract (conformance criterion 10): a typo in a mod's animation name must degrade to a
creature that does not play the animation, not to a stopped game.

## `add_script_animation` — without a destination

**Contract** — the same, for an animation that should play where the creature stands. Takes
the movement-controller flag explicitly, because without a destination it is a real choice:
the animation may still carry the creature (a stumble), or the movement manager may stay in
charge (a gesture). Same missing-motion handling.

## `assign_script_animation`

**Contract** — hand the queue head to the channel that is about to play it, and set up root
motion if this animation needs it. Returns the head record; asserts rather than tolerating
an empty queue, because the caller has already checked.

```text
FUNCTION assign_script_animation() -> ScriptAnimation
  entry := queue.front
  IF entry.use_movement_controller
     OR the motion itself is authored as root-moving THEN
    channel.target_matrix := creature.world_transform
  RETURN entry
```

**Invariants** — the destination handed to the channel here is the *creature's current
transform*, not the record's own destination. That is deliberate: this call sets the frame
the root motion is measured from, and the record's destination is applied later by the
channel relative to it. Conflating the two lands the creature at the destination instantly
instead of animating toward it.

**Notes** — root motion is enabled either because the caller asked for it or because the
motion is authored as root-moving. The second half of that test is what lets an animator
mark a clip as self-moving and have every script that plays it do the right thing without
the script knowing.
