# src/xrGame/ai/monsters/controller/controller_animation.cpp

> Two-part animation for the controller: a torso clip chosen from what it is doing and a legs clip chosen from the angle between where it looks and where it walks.

**Needs** — [`controller_animation.h`](controller_animation.h.md) · [`controller.h`](controller.h.md) · [`controller_direction.h`](controller_direction.h.md) · [`../control_animation_base.h`](../control_animation_base.h.md) · [`../control_animation.h`](../control_animation.h.md) · [`../control_direction_base.h`](../control_direction_base.h.md) · [`../control_path_builder_base.h`](../control_path_builder_base.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md) · [`../../../detail_path_manager.h`](../../../detail_path_manager.h.md)
**Used by** — reached through its declarations in [`controller_animation.h`](controller_animation.h.md); callers name that, not this file.
**Tier floor** — T2: per-frame clip selection over two body partitions

## Purpose

The controller's substitute for the standard animation driver. Where the base drives one clip
over the whole body, this drives two: a **torso** clip saying what the creature is doing, and
a **legs** clip saying how it is moving — and the legs clip is chosen from the angle between
the creature's gaze and its travel direction, so a controller walking backwards while staring
at the player plays a backwards-walk clip rather than turning around.

That is the intent of the file. In the shipped build **most of it is disabled**, and the twin
must say so plainly: the per-frame entry point returns immediately after delegating to the
base, and the body-state setter ignores its arguments and pins the creature to one pose. What
survives and runs is the psi-attack torso clip, the event routing and the path parameter
selection. The rest is a complete, readable design that the game does not use.

## State

```text
RECORD ControllerAnimation                  # beyond the animation base
  current_legs_action  : LegsAction         # a tagged bit set, see below
  current_torso_action : one of { idle, steal, psy_attack, run }
  legs   : map<LegsAction, Motion>
  torso  : map<TorsoAction, Motion>
  path_rotations : map<LegsActionFamily, list<(angle, LegsAction)>>
  wait_torso_anim_end : bool                # a psi-attack clip is playing; do not interrupt
```

The legs actions are a **tagged bit set**, and the encoding is the file's one real data
structure. A high bit marks the family — standing, sneaking, sneak-moving, walking, running —
and the low bits index within it. So a single mask test asks "is this creature running,
whichever run clip it is playing", which is what the movement tests and the path-rotation
lookup both need.

```text
ENUM LegsActionFamily (one bit each, from bit 16 up)
  stand, steal, steal_motion, walk, run

ENUM LegsAction = family | index
  stand          = stand | 1        stand_damaged      = stand | 2
  steal          = steal | 1
  run            = run | 1          back_run           = run | 2
  run_fwd_left   = run | 3          run_fwd_right      = run | 4
  run_bkwd_left  = run | 5          run_bkwd_right     = run | 6
  run_damaged    = run | 7          back_run_damaged   = run | 8
  run_strafe_left_damaged = run | 9 run_strafe_right_damaged = run | 10
  walk           = walk | 1         walk_damaged       = walk | 2
  steal_fwd      = steal_motion | 1 steal_bkwd         = steal_motion | 2
  steal_fwd_left = steal_motion | 3 steal_fwd_right    = steal_motion | 4
  steal_bkwd_left= steal_motion | 5 steal_bkwd_right   = steal_motion | 6
```

**Invariants** — the family bits begin at bit sixteen so that the index bits never collide
with them. A rebuild may use a record of (family, index) instead; what must survive is that
"same family" is a cheap test.

## `load`

**Contract** — resolve every legs and torso clip on the model by literal name, and register
the path-rotation tables for the running and sneak-moving families.

The path rotations are the interesting half: each family declares six entries mapping an
angle to a clip — straight ahead, straight behind, and the four diagonals at a quarter turn
and three quarters of a turn either side.

**Notes** — six of the damaged legs clips resolve to the *same* clip as the ordinary run.
This creature has no damaged animation set; the entries exist so the lookup never finds a
hole. The forward run, the walk and both damaged gaits are likewise the same clip in the
base driver's own table (see [`controller.cpp`](controller.cpp.md)), so the creature's whole
locomotion is two clips plus their directional variants.

All clip names are literals here, not authored, so this creature's model is frozen by name.

## `select_legs_animation`

**Contract** — choose the legs clip. When the creature is moving, take the gaze-versus-travel
angle and pick the nearest path-rotation entry of the current family. When it is not, pick
the first clip in the table belonging to the current family — a stand or sneak pose.

```text
FUNCTION select_legs_animation()
  IF is_moving() THEN
    action <- nearest path rotation to my current heading
  ELSE
    action <- the first legs entry whose family matches current_legs_action
    REQUIRE one was found

  IF the legs partition is not already playing legs[action] THEN mark it stale
  set the legs partition's motion to legs[action]
```

## `get_path_rotation`

**Contract** — given a heading, find the path-rotation entry of the current family whose
angle best matches the signed difference between that heading and the path's travel
direction.

```text
FUNCTION get_path_rotation(heading) -> (angle, legs_action)
  travel <- -(heading of the detailed path's current direction)
  diff   <- angle_difference(heading, travel)
  IF travel is to the right of heading THEN diff <- -diff
  normalize diff
  RETURN the entry of path_rotations[current family] whose angle is nearest diff
```

**Notes** — this is the mechanism the whole class exists for. The creature's body heading is
aimed at what it is *looking at*; the path points where it is *going*; the difference selects
a strafing or backing clip. A controller can therefore retreat while staring, which is its
signature look.

The normalization of the difference before the nearest-angle search is suspect: a signed
difference normalized to the positive range makes a left turn of a quarter and a right turn
of three quarters compare equal in magnitude, so the diagonals may be chosen on the wrong
side. Nothing exercises it in the shipped build, where the selection is disabled.

## `set_path_direction`

**Contract** — set the body's heading to the travel direction *plus* the offset of the
chosen path rotation, at a fixed turn rate of half a turn per second. Disabled along with
the rest of the per-frame path.

**Notes** — the pairing with `get_path_rotation` is circular by design: the clip is chosen
from the angle and then the body is turned to the angle the clip assumes, so clip and body
converge rather than fighting.

## `select_torso_animation`

**Contract** — choose the torso clip. A pending psi attack wins: if the creature's psi-bolt
conditions hold, the attack clip is chosen and the partition is latched until it ends.
Otherwise the clip for the current torso action. Does nothing at all while latched.

**Notes** — this is the one caller of the creature's psi-bolt condition check, and that check
stamps its own cooldown, so asking here *is* firing. The latch is what stops the selector
re-entering the attack every frame.

## `select_velocity`

**Contract** — set the path builder's desirable speed from the legs family: four units when
running forward, two when running backward or diagonally backward, a little over one when
sneak-moving, zero otherwise.

**Notes** — four literal speeds, in this file rather than in the creature's configuration
section, and they bypass the authored gait table entirely. Disabled along with the rest of
the per-frame path. A rebuild reviving this should route them through the gait table.

## `set_path_params`

**Contract** — this *is* live, and it is what the creature's states call before moving.
Choose the gait mask and desirable gait from whether the creature is looking roughly toward
its destination or away from it, and enable the path; a non-moving legs family disables the
path instead.

```text
FUNCTION set_path_params()
  IF the current legs family is not a moving one THEN disable the path; RETURN

  direction <- the set target minus my position
  looking_forward <- true
  IF direction is non-degenerate THEN
    looking_forward <- angle_difference(heading of direction, my current heading) <= quarter turn

  IF looking_forward THEN use the creature's forward gait
  ELSE                     use the creature's backward gait
  enable the path
```

**Notes** — the two gaits are the creature's own authored extra pair, loaded in
[`controller.cpp`](controller.cpp.md), and this is the only place they are selected. So the
backward gait is chosen by *gaze*, not by any decision to retreat: a controller whose
destination is behind where it is looking walks there backwards, at its authored backwards
speed. That is the behaviour the disabled backwards clips were meant to dress.

## `is_moving`

**Contract** — the creature is moving on a path **and** its legs family is one of the three
moving ones. The second half is what keeps a standing pose from being replaced mid-path.

## `set_body_state`

**Contract** — as written, ignores both arguments and pins the creature to the sneak-moving
legs family and the sneak torso action.

**Notes** — this is the disabling. Every caller — the creature's states, the mental-state
switch, the reset — passes a meaningful pair and gets the sneak pose. The intended
assignment is present and commented out on the next two lines. The shipped controller
therefore always sneaks, which is its recognizable silhouette; whether that was a deliberate
art decision made in code or an unfinished feature is not recoverable.

## `on_event`

**Contract** — four events. A whole-body animation end re-selects from the base driver and
clears the attack latch on the base. A torso animation end clears the psi latch and
re-selects the torso. A legs animation end re-selects the legs. An animation signal carrying
the hit marker fires the creature's psi bolt when it came from the attack clip, and otherwise
falls through to the base driver's hit check.

**Notes** — the signal case falls through into the next case when the marker is not a hit
marker, because its branch does not break. Nothing follows it, so the effect is benign, but
it is a latent bug a rebuild must not copy.

## `update_frame`

**Contract** — delegate to the base driver and return. Everything after the return —
setting the path direction while moving, selecting both partitions and the velocity — is
unreachable and is the design this file describes.

## `on_switch_controller`

**Contract** — the mental-state hook. In the danger state, clear the psi latch, set the body
state and re-select both partitions; in the idle state, re-select through the base driver
instead.

**Notes** — since the body-state setter is pinned, the two branches differ only in which
selector runs. This is the one live entry point into the two-partition selection.

## `reinit`

**Contract** — find the creature at its derived type, load the clip tables, reset the base
driver, pin the body state, register the psi-attack hit marker at half the clip's duration,
and clear the latch.

**Notes** — the marker fraction of one half is a constant here. The load runs *before* the
base's own reset, because the base's reset reads the tables this load fills.
