# src/xrGame/stalker_animation_manager.cpp

> Brings a stalker's animation system to a known state on spawn and on every reinitialization, binds it to the shared animation table for its model, and fires one-shot reaction motions.

**Needs** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_animation_manager_inline.h`](stalker_animation_manager_inline.h.md) · [`stalker_animation_data_storage.h`](stalker_animation_data_storage.h.md) · [`stalker_animation_data.h`](stalker_animation_data.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md)
**Used by** — reached through its declarations in [`stalker_animation_manager.h`](stalker_animation_manager.h.md); callers name that, not this file.
**Tier floor** — T2: state reset and one table lookup per model

## Purpose

The two halves of bringing the animation system up, kept apart because they run at different
times and depend on different things.

- **Reinitialization** resets the *simulation* state — which direction the legs think they
  are facing, which channel is playing, what speed the last animation asked for. It runs on
  every transition back to a live simulated state and must not touch the model.
- **Reload** binds to the *model* — the skeleton, the shared animation table, the aiming
  bones, the crouch style. It runs when the visual changes.

Confusing the two is a real failure mode: resetting the skeleton binding on every alife
transition would re-resolve the animation table needlessly, and resetting the direction
state on a visual change would snap a walking stalker's legs.

## State

Declared in [`stalker_animation_manager.h`](stalker_animation_manager.h.md). The fields this
file sets, grouped by what they are for:

```text
# leg direction tracking
direction_start          : int    # world clock when the current target direction was chosen
current_direction        : enum   # the direction the legs are actually playing
target_direction         : enum   # the direction they are turning toward
previous_speed_direction : enum   # last frame's movement direction, for the look-back rule
change_direction_time    : int    # world clock of the last direction change
looking_back             : int    # non-zero while the stalker is glancing behind itself

# speed feedback to the movement system
previous_speed           : real
target_speed             : real
last_non_zero_speed      : real

# crouch style
crouch_state_config      : int    # -1 random per stop, or a fixed 0 or 1
crouch_state             : int    # the style currently in use

# channel bookkeeping
no_move_actual           : bool   # has the current standing-still bout chosen its style yet
call_script_callback     : bool   # an animation ended; run the script callback next frame
call_global_callback     : bool   # likewise for the global channel's callback
special_danger_move      : bool
```

## `reinit`

**Contract** — reset every piece of per-life animation state and empty the script queue.
Does not touch the skeleton, the animation table or the bone callbacks. Runs on spawn and on
every transition back online.

```text
FUNCTION reinit()
  current_direction = target_direction = forward
  direction_start = change_direction_time = 0
  looking_back = 0
  no_move_actual = false

  empty the script animation queue
  reset all five channels

  legs, global and script channels: step-dependent      # their timing drives footstep events
  global and script channels: whole-body                # they own every bone group

  clear both deferred callback flags
  previous_speed = target_speed = last_non_zero_speed = 0
  special_danger_move = false
```

**Invariants** — the channel flags set here are properties of the *channel*, not of any
particular motion, so they are set once at reset and never again. Marking a channel
step-dependent means the footstep events authored into its motions are honoured; marking one
whole-body means it claims every bone group when it plays, which is what makes the priority
ladder in the update actually exclusive.

Starting the direction state at *forward* rather than at whatever the body is doing is
deliberate: reinitialization happens when there is no continuity to preserve, and a defined
starting direction beats one read from a stale pose.

## `reload`

**Contract** — bind to the current model: take the skeleton, fetch the shared animation
table for its motion banks, read the crouch style from the stalker's character profile, and
install the aiming bone callbacks. Returns early for a dead stalker, before the bone
callbacks. Hard-fails if the visual is not an animated skeleton.

```text
FUNCTION reload()
  visual   = the stalker's current visual
  skeleton = visual as an animated skeleton          # must be one

  crouch_state_config = the character profile's crouch style   # -1, 0 or 1
  crouch_state = crouch_state_config

  animation_table = shared storage.object(skeleton)  # loaded once per motion-bank list

  IF NOT alive  RETURN                               # a corpse gets no aiming callbacks
  install the aiming bone callbacks
```

**Invariants** — the crouch style is one of exactly three values and is checked. Zero and one
select a fixed standing-still crouch pose; minus one means "choose randomly each time the
stalker stops", which is what makes a group of crouching stalkers not look cloned. The
random choice itself happens in [`stalker_animation_legs.cpp`](stalker_animation_legs.cpp.md),
at the moment movement stops.

The dead-stalker early return is before the bone callbacks and after the table binding,
which is the right split: a corpse still needs its animation table (death and ragdoll
motions come from it) but must not have its head and spine rotated toward a sight target it
no longer has.

**Notes** — when the head-as-a-separate-bone-group path is compiled in, this is also where
the bone-group mask for script animations is derived: it is every bone group *except* the
one the head's own motions are authored for, taken from the first head motion's definition.
That derivation from the data rather than from a constant is what lets a model author decide
which group the head belongs to.

## `play_fx`

**Contract** — play a one-shot additive reaction motion — a flinch, a hit reaction — at a
given strength, chosen by index from the whole-body table for the stalker's current body
state. Hard-fails on an index outside the table.

**Notes** — these are played directly on the skeleton rather than through a channel, because
they are additive: they layer over whatever is playing instead of replacing it, so they have
no channel to occupy and no end callback to wait for. That is why a hit reaction can happen
during a scripted animation without disturbing it.

The strength argument scales the motion's amplitude, so one authored flinch covers a graze
and a solid hit.
