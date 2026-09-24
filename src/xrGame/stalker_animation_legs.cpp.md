# src/xrGame/stalker_animation_legs.cpp

> Locomotion: which direction the legs are carrying the body relative to where it is looking, with hysteresis so a stalker does not flicker between strafes, and which standing-still or turning-in-place motion to hold when it stops.

**Needs** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_animation_manager_impl.h`](stalker_animation_manager_impl.h.md) · [`stalker_animation_data.h`](stalker_animation_data.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: angle arithmetic against a hysteresis timer, per stalker per frame

## Purpose

A stalker walks in one direction while looking in another, and the leg animation has to
express the difference: forward, backward, or a strafe to one side. Deciding that from the
angle alone produces a stalker that snaps between strafe animations whenever it turns past a
boundary, so the real content of this file is the *hysteresis* that stops it — and, when the
stalker halts, a second decision about which of several standing poses to hold.

This is also where the speed the whole body will travel at is chosen, because the speed is a
property of the motion, not of the request. See the feedback loop in
[`stalker_animation_manager_update.cpp`](stalker_animation_manager_update.cpp.md).

## Constants

```text
forward_cone_half_angle   = a quarter turn / 2 on each side   # the forward sector is ±45°
direction_switch_interval = 500 ms      # how long a new direction must persist to take effect
look_back_delay           = 0 ms        # delay before a backward-moving stalker glances behind

direction_angles = [ 0, half turn, quarter turn, minus a quarter turn ]
                   #  forward,  backward,  left,          right
```

**Invariants** — the forty-five-degree half-angle makes the four sectors equal, so forward,
backward, left and right each claim a quadrant. It is not a tuning value: any other split
would make one strafe animation cover more of the circle than another, and the animations are
authored symmetric.

The direction table converts a chosen direction into the body yaw offset the movement system
should hold. It is indexed by the same enumeration the animation table is, which is what lets
one index serve both.

## `legs_process_direction` — choosing a direction with hysteresis

**Contract** — given the direction the stalker wants to travel, classify it against where the
head is looking, commit to it if it has persisted long enough, and set the body's target yaw
accordingly. Mutates the direction state and the movement system's body target.

```text
FUNCTION legs_process_direction(travel_yaw)
  factor = legs_switch_factor()
  head   = the head's current yaw
  # The forward and backward cones are measured from the side the target lies on,
  # so the classification is symmetric for a left and a right turn.
  toward_left = travel_yaw is counter-clockwise of head
  forward_limit  = forward_cone_half_angle
  backward_limit = a half turn - forward_cone_half_angle

  difference = unsigned angle between travel_yaw and head

  IF difference <= forward_limit        direction = forward
  ELSE IF difference > backward_limit   direction = backward
  ELSE IF toward_left                   direction = left
  ELSE                                  direction = right

  legs_assign_direction(factor, direction)
  the movement system's body target yaw = travel_yaw + direction_angles[current_direction]
```

**Invariants** — the body's target yaw is derived from the **committed** direction, not from
the direction just classified. That is what the hysteresis is protecting: the body turns when
the animation does, so the pose and the motion never disagree.

## `legs_assign_direction` — the commit rule

**Contract** — advance the direction state machine by one frame. Three outcomes: already
there, a new target, or a commit.

```text
FUNCTION legs_assign_direction(switch_factor, direction)
  IF current_direction IS direction
    direction_start = now                 # keep the dwell timer fresh
    RETURN

  IF target_direction IS NOT direction
    direction_start  = now                # a different candidate: restart the timer
    target_direction = direction
    RETURN

  # Same candidate as last frame: commit once it has persisted long enough.
  IF now - direction_start <= switch_factor * direction_switch_interval
    RETURN
  direction_start   = now
  current_direction = direction
```

**Invariants** — the dwell timer is reset in all three branches, which is what makes the rule
"has persisted continuously" rather than "was first seen half a second ago".

## `legs_switch_factor` — when hysteresis is skipped

**Contract** — the multiplier applied to the dwell interval: zero for a direct reversal,
one otherwise.

```text
FUNCTION legs_switch_factor() -> real
  IF the target and current directions are opposites   # forward<->backward, left<->right
    RETURN 0                                           # commit immediately
  RETURN 1
```

**Invariants** — a **reversal commits instantly**, and this is the most important decision in
the file. Hysteresis exists to damp flicker between *adjacent* sectors, which is what a
turning stalker produces. A stalker that reverses has genuinely reversed, and holding the old
direction for half a second would have it moonwalk. The same zero factor also resets the
speed state, so a reversal starts the new motion from a standstill rather than blending a
forward speed into a backward one.

## `legs_move_animation` — the moving case

**Contract** — select the locomotion motion, and set the target speed the body will travel at.
Two quite different paths depending on whether the stalker is being careful.

```text
FUNCTION legs_move_animation() -> motion
  no_move_actual = false                  # the standing-still bout is over

  IF mental state IS NOT danger
    # Relaxed: always face where you are going. One direction, one motion.
    target_speed = the movement system's forward speed
    RETURN the movement table entry for (body state, movement type, forward, family 1)

  # Careful: the body may travel in one direction while the head watches another.
  (yaw, pitch) = the direction the sight system is aiming
  yaw = normalized signed angle of -yaw
  legs_process_direction(yaw)             # commits a direction and turns the body

  # Classify the TRAVEL direction against the BODY, not the head: the head chose the
  # body's facing above, and the speed must match the legs, not the gaze.
  speed_direction = the quadrant of yaw relative to the body's current yaw,
                    by the same four-sector rule
  IF speed_direction changed since last frame
    refresh the change timestamp
    IF this was a reversal, zero both the previous and the target speed
    remember it

  target_speed = the movement system's speed for that direction
  RETURN the movement table entry for (body state, movement type, speed_direction, family 0)
```

**Invariants**

- The two classifications are against **different references** and both are needed. The first
  is head-relative and decides which way the body should face; the second is body-relative
  and decides which strafe motion and which speed. Collapsing them gives a stalker whose legs
  play a strafe while its body has already turned to face the travel direction.
- The relaxed path deliberately has no hysteresis and no direction tracking, because a
  relaxed stalker always walks forward — the movement system turns the whole body to face
  the path. Only a stalker in danger keeps its weapon pointed one way while moving another.
- Each direction has its **own** speed. A stalker strafes slower than it walks forward, and
  the movement system holds the per-direction values; the animation asks for the one matching
  the motion it picked, which is what keeps the feet from sliding.
- The two paths index different families of the movement table — the relaxed path takes
  family 1, the careful path family 0 — which is how one table holds both the plain walk
  cycles and the weapon-ready ones.

**Notes** — the yaw is negated and re-normalized before use because the sight system reports
an aiming angle in the opposite sense from the movement system's yaw. That is a convention
mismatch between two subsystems, and a rebuild should pick one sense.

## `legs_no_move_animation` — the standing case

**Contract** — select the standing-still or turning-in-place motion. Zeroes both speeds.
Picks a crouch style on entry to a standing-still bout. May retarget the body toward the
head.

```text
FUNCTION legs_no_move_animation() -> motion
  previous_speed = target_speed = 0

  IF this is the first frame of standing still
    no_move_actual = true
    crouch_state = crouch_config IF it is fixed ELSE a random choice of two
  change_direction_time = now             # standing still is not a direction change

  animations = the in-place table for the current body state
  current = the body's current yaw ; target = the body's target yaw

  IF the body has essentially reached its target yaw
    IF relaxed, or the sight system is not asking for a turn
      IF relaxed                          RETURN animations[1]   # relaxed idle
      IF crouching                        RETURN animations[crouch_state]
      RETURN animations[0]                                       # alert idle, standing
    # The sight system wants to look somewhere the body cannot: turn the body to follow.
    the body's target yaw = the head's target yaw
    target = that

  IF the turn is counter-clockwise
    RETURN animations[4] IF relaxed ELSE animations[2]           # turn left
  RETURN animations[5] IF relaxed ELSE animations[3]             # turn right
```

**Invariants** — the crouch style is chosen **once per bout** of standing still, not per
frame, which is what the first-frame flag exists for. Re-rolling it every frame would make a
crouching stalker twitch between two poses. The configured value of "random" is per
character, so a level's stalkers crouch in a mix of styles while any one of them is
consistent for as long as it stays down.

Retargeting the body to the head's target — rather than to the head's *current* yaw — is what
makes a turn-in-place complete instead of chasing a moving reference. The head leads, the
body follows to where the head is going.

**Notes** — the six in-place motions are two idles (alert, relaxed), two alert turns and two
relaxed turns, and the crouch style indexes into the first two. A rebuild should name them;
as written the literals are positions in an authored name list.

## `assign_legs_animation`

**Contract** — dispatch to the moving or standing selection, and on an invalid result call the
same selection once more before returning it.

**Notes** — the retry is a bug rather than a design: the second call's result is discarded and
the invalid one is returned anyway. It reads as an attempt to recover from a missing
animation that never worked, and its only real effect is to run the selection's side effects
— the direction commit, the speed assignment — twice. A rebuild should drop it. The genuine
failure it was reaching for is a model with an unresolved motion, which the containment in
[`stalker_animation_manager_update.cpp`](stalker_animation_manager_update.cpp.md) catches
instead.

## `need_look_back`

**Contract** — should the stalker glance over its shoulder? True while a glance is in
progress; otherwise true only if the stalker has been moving backward for longer than the
configured delay, in which case it starts a glance of randomized length.

**Invariants** — the randomized length is one of two values and is chosen when the glance
starts, so that a group of retreating stalkers do not all look back for the same duration.

**Notes** — the configured delay is zero, so a stalker begins glancing behind itself on the
first frame it moves backward. Whether that is the intended value or a left-over from tuning
is not recoverable; the constant exists solely so it could be non-zero.

## `legs_play_callback`

**Contract** — the renderer's notification that the leg motion ended. Tells the channel and
nothing more.
