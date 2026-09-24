# src/xrGame/ai/monsters/controlled_actor.cpp

> Taking the player's camera away: while a creature holds the actor, the camera is eased toward a point the creature chooses and nearly every input command is refused.

**Needs** — [`controlled_actor.h`](controlled_actor.h.md) · [`Actor.h`](../../Actor.h.md) · [`actor_input_handler.h`](../../actor_input_handler.h.md) · [`Inventory.h`](../../Inventory.h.md) · [`ai_monster_utils.h`](ai_monster_utils.h.md) · [Seam: Windowing and input](../../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — reached through its declarations in [`controlled_actor.h`](controlled_actor.h.md); callers name that, not this file.
**Tier floor** — T2: runs on the frame path and drives the player's camera

## Purpose

One creature family in the game takes the player over, and this is the mechanism. It is an
*input handler* installed on the actor: while it is installed, the actor's own input is
suppressed except for a whitelist, and each frame the actor's active camera is nudged toward
a point the creature names. The player retains the camera's motion — it is eased, not
snapped — which is what makes the effect read as being *dragged* rather than as a cutscene.

The class is inherited by the creature that does this, not held by it, so "is the actor
controlled" is answered by asking the creature.

## State

```text
RECORD ControlledActor
  target_point : vector      # where the camera is being turned toward
  turned_yaw   : bool        # the heading has arrived
  turned_pitch : bool        # the pitch has arrived
  need_turn    : bool        # keep turning; cleared to freeze the camera where it is
  lock_run     : bool        # declared, never used
  lock_run_started, lock_run_period : int (ms)   # likewise
```

**Invariants** — the two arrival flags are reset on install and on release, so a fresh hold
always reports "still turning" until it has actually arrived.

**Notes** — the run-lock trio is declared and never read or written anywhere. It was meant
to stop the player sprinting out of the hold and does not exist.

## Tuning

Four constants, in this file rather than in data:

| Constant | Value | Meaning |
|---|---|---|
| minimum turn speed | 0.5 | radians per second when the camera is nearly aligned or nearly opposite |
| maximum turn speed | 4.0 | radians per second at the midpoint of the turn |
| arrival tolerance | 1 degree | how close counts as arrived |
| speed normalizer | half a turn | the angular error at which the speed factor saturates |

## `update_turn`

**Contract** — ease the actor's camera one frame toward the target point, on both axes
independently, by issuing the same camera-move commands the player's own input would. Each
axis reports arrival separately.

```text
FUNCTION update_turn()
  get the camera's position and facing
  target_heading, target_pitch <- angles from the camera to the target point
  current_heading, current_pitch <- angles of the camera's facing

  FOR EACH axis IN { heading, pitch }
    factor <- angle_difference(current, target) / half a turn
    clamp factor INTO [0, 1]
    IF factor > 0.5 THEN factor <- 1 - factor      # see Notes
    IF current and target are within 1 degree THEN
      mark this axis arrived
    ELSE
      speed <- 0.5 + factor * (4.0 - 0.5)
      move the camera on this axis toward the target at speed * frame time
```

**Notes** — the folding of the speed factor at its midpoint is the decision that gives the
turn its feel. The factor rises with the remaining error up to a half-turn and is then
*mirrored*, so the camera moves slowest when it is nearly aligned **and** when it is nearly
opposite, and fastest through the middle of the swing. It is an ease-in-ease-out expressed
without a curve. Read literally it also means a camera pointing exactly backwards barely
moves, which is visible: the hold begins slowly.

Driving the camera through the same move commands the player's keys produce, rather than
setting the angles, means the hold composes with whatever camera effectors are running —
including the one the creature's psi attack installs.

## `authorized`

**Contract** — which input commands survive the hold. Exactly two: selecting the first
weapon slot, and firing *when the active slot is the first one*. Everything else is refused.

**Notes** — the whitelist is the whole design of the set piece. A held player can raise and
fire one weapon and nothing else — cannot run, cannot turn, cannot switch, cannot use an
item. Which weapon lives in slot one is the game's business, and in practice it is the
knife.

## `mouse_scale_factor`

**Contract** — an unbounded value, which in the actor's input handling means the mouse
contributes nothing. The player's own aim is dead for the duration.

## `install` / `release` / `reinit`

**Contract** — installing sets the handler on the actor and enables turning; releasing and
resetting clear the two arrival flags. The spelling without an actor argument re-installs on
the actor already known.

## `look_point`

**Contract** — set the point the camera is being turned toward. Called every tick by the
creature while the hold lasts, so the camera tracks a moving creature.

## `dont_need_turn`

**Contract** — stop turning and leave the camera where it is. The creature's psi attack uses
this: it installs the hold to suppress input, then immediately stops the turn because its
own camera effector is about to take over the view.

## `is_turning` / `is_installed` / `is_controlling` / `frame_update`

**Contract** — `is_turning` is true while *either* axis has not arrived. `is_installed` and
`is_controlling` both answer whether an actor is held. `frame_update` runs the turn each
frame while holding and turning is wanted.
