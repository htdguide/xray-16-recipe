# src/xrPhysics/ElevatorState.cpp

> Decides when a character is on a ladder, holds it there against gravity, and
> decides when it has let go.

**Needs** — [`ElevatorState.h`](ElevatorState.h.md) · [`IClimableObject.h`](IClimableObject.h.md) · [`PHCharacter.h`](PHCharacter.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`MathUtils.h`](MathUtils.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`ElevatorState.h`](ElevatorState.h.md)
**Tier floor** — T2: a state machine plus a handful of forces.

## Purpose

Climbing is the clearest case in the chapter of the **character controller's central
problem**: the character is a rigid body, but a climbing character is not obeying rigid-body
physics at all. This file resolves that by *switching gravity off* for the duration of a
climb and supplying its own holding force, while leaving the body otherwise in the solver so
that walls, other characters and shots still affect it.

Everything else here follows from that: the machine's states exist to bracket the interval
in which gravity is off, and its transitions exist so that the bracket opens and closes at
the right moment.

## State

```text
RECORD ElevatorState
  state          : Estate          # see IElevatorState.h
  ladder         : ClimableObject  # none when no ladder is in range
  character      : Character
  start_position : vector          # foot centre when the current state was entered
  start_time     : int             # wall-clock when the current state was entered
  # invariant: gravity is off on the character's body EXACTLY while state is one of
  #            the two climbing states.  Every transition in or out flips it.
```

## the tuning constants

```text
getting_on_dist     0.3    # how near an endpoint counts as "able to step on"
stop_climbing_dist  0.1    # how near an endpoint ends the climb
out_dist            1.5    # beyond this from the axis, the ladder is forgotten
look_angle_cosine   0.9238 # 22.5 degrees: how squarely you must face the ladder
lookup_angle_sine   0.342  # 20 degrees: the camera-pitch bias on climb direction
```

**Notes** — the two angles are the only ones that matter to feel. Twenty-two and a half
degrees of facing tolerance is deliberately tight: a character who steps onto a ladder while
looking sideways then has their movement input rewritten by the ladder, which reads as the
controls fighting the player. The twenty-degree pitch bias is subtler — see
`ClimbDirection`.

The endpoint distances are all measured against the character's *foot radius* added, so a
larger creature lets go further from the end. Distances that ignored body size would make
large creatures clip through the top of every ladder.

## `ClimbDirection`

**Contract** — signed intent: positive means the character is trying to climb up.

```text
FUNCTION climb_direction() -> real
  d = direction from the character to the ladder's plane
  dir = dot(character.control_acceleration, d)     # is the input pressing INTO the ladder?
  IF dir > small
    dir = dir * (camera_pitch_sine + lookup_angle_sine)
  RETURN dir
```

**Notes** — this is the file's most interesting decision and it is invisible in the
signature. Pressing "forward" at a ladder is ambiguous: it could mean climb up or it could
mean walk into it. The resolution is to fold the **camera pitch** in: the sine of the
camera's vertical angle, biased by twenty degrees, multiplies the forward intent. Looking
level or up gives a positive product — climb up. Looking more than twenty degrees down flips
the sign — climb down. So the player aims where they want to go and presses forward, and
there is no separate climb-down control. The twenty-degree bias is what makes "looking
roughly level" mean up rather than sitting on a knife edge.

## the state machine

**Contract** — every step, dispatch on the current state; each state evaluates its own exit
conditions and applies its own forces.

**`none`** — a ladder is known but unengaged. Exits to a climbing state when the character is
in touch, before the ladder, and facing it within the tolerance; the sign of
`ClimbDirection` picks up or down. Otherwise, if within stepping distance of either endpoint,
exits to the matching *near* state.

**`near_up` / `near_down`** — standing at an end. `near_down` exits to climbing-up when the
character is in touch, facing the lower endpoint, pressing towards it, and intending up.
`near_up` exits to climbing-down on a stricter set: in touch, camera pitched at least nine
degrees down, standing off the ladder's plane by at least a third of a foot radius, and
before the ladder with a loosened tolerance.

**Notes** — the asymmetry between the two is not an oversight. Getting *onto* a ladder from
below is a deliberate act and can afford to demand a clear forward press. Getting onto one
from above means backing off a ledge, where the character is not pressing towards anything
and the camera is the only reliable signal of intent — hence the pitch condition and the
loosened facing tolerance. The standoff distance stops a character standing directly on the
ladder's top rung from being grabbed by it.

Both *near* states also exit to `no_ladder` when the relevant endpoint recedes past the
forget distance.

**`climbing_up` / `climbing_down`** — the states in which gravity is off.

```text
FUNCTION climbing_step(going_up)
  IF climb_direction() has flipped sign AND still before the ladder
    SWITCH to the other climbing state
  (to_axis_distance, to_axis_dir) = ladder.distance_and_dir_to_axis(character)
  (control_dir, control_magnitude) = split(character.control_acceleration)
  # Pressing across the ladder rather than along it means letting go.
  IF to_axis_distance is non-zero AND control_magnitude is non-zero
     AND |dot(control_dir, ladder.normal)| < cos(45 degrees)
    SWITCH to depart
  IF the axial distance to this climb's far endpoint is within stop_climbing_dist
    SWITCH to the matching near state
  climbing_common(to_axis_dir, to_axis_distance, control_dir, control_magnitude)
  IF going_down AND the body's upward velocity is positive
    apply a downward force of mass * gravity     # cancel the bounce; see Notes
  IF the axial distance past the other endpoint has gone negative
    SWITCH to no_ladder

FUNCTION climbing_common(to_axis_dir, to_axis_distance, control_dir, control_magnitude)
  IF to_axis_distance - foot_radius > out_dist
    SWITCH to no_ladder
  IF control_magnitude is zero AND the character is on the ladder's front side
    apply force mass * gravity along to_axis_dir   # HOLD ON
```

**Invariants** — with gravity off, the *only* thing keeping a released climber in place is
the hold-on force, and it is applied only when the player is giving no movement input. That
is deliberate: a climber pressing a direction is moving under their own control and must not
also be pulled towards the axis.

**Notes** — the hold-on force is exactly one gravity, which is what makes an idle climber
neither fall nor be sucked into the ladder — it cancels the force the solver would have
applied had gravity been on. It is applied *towards the ladder's axis*, not upwards, so it
doubles as the thing that keeps a climber hugging the ladder.

The extra downward force while descending exists because a climber going down accumulates
upward velocity from contacts with the rungs, and without cancelling it the character
visibly bounces on the way down.

The 45-degree let-go test reads "if you are pressing more across the ladder than along it,
you meant to step off" — the one place where lateral input has a meaning distinct from
climbing.

**`depart`** — a one-step state. It checks whether an endpoint is within stepping distance
and if so goes to the matching *near* state, then unconditionally falls through to
`no_ladder`.

**Notes** — the distance-and-time gate that was meant to make departure take a while is
present in the transition table (departure is the only transition with a non-zero entry:
two metres or three seconds) but the code path that would enforce it is commented out, so
departure is immediate. A rebuilder should treat the gate as *intended and not shipped*.

**`no_ladder`** — clears the ladder reference and does nothing further.

## `SwitchState`

**Contract** — the only way the state changes. Consults the transition-inertia table, and if
the transition is not yet permitted, does nothing. On a permitted transition it flips gravity
if the transition crosses the climbing boundary in either direction, restamps the start
position and time, and assigns the new state.

**Invariants** — gravity is flipped **only** on transitions that cross the climbing/
not-climbing boundary, never on climbing-up to climbing-down, so switching direction
mid-climb does not momentarily restore gravity.

**Notes** — the transition table is a square matrix over all states holding a minimum
distance and a minimum time. Every entry is zero except departure-to-no-ladder, and since
the gate passes when *either* the distance or the time threshold is exceeded, a zero entry
always passes. The table is therefore inert in the shipped build. It is worth keeping in a
rebuild as the place where "don't let this transition happen twice in a frame" would live —
which is a real problem the shipped game solves by luck.

## `SetElevator`

**Contract** — a proposal, not an assignment. The proposed ladder is rejected if it is
beyond the forget distance, if it is already the current one, or if the current one is
nearer. Accepting resets the state to `none`.

**Notes** — this is what makes overlapping ladders behave: the character is never told which
ladder to use, it is offered each one that it touches, and nearest wins.

## `GetControlDir`

**Contract** — rewrite the character's desired direction into one the ladder allows, and
report whether motion is permitted at all.

- In a *near* state: the desired direction becomes the direction to the endpoint, but only
  if the character is facing it and pressing towards it. Otherwise the input passes through
  unchanged.
- **Climbing up**: the direction is the ladder's axis plus the direction to the axis,
  normalized — climb and hug at once.
- **Climbing down**: the same with the axis inverted, but *only* if the character is before
  the ladder or already moving towards the axis. If neither holds, the function reports
  **false** and the character may not move at all.

**Notes** — that false return is the one place in the chapter where the controller flatly
refuses a movement request rather than transforming it, and it stops a descending character
from walking off the back of the ladder into space.

## `GetJumpDir`

**Contract** — jumping off a ladder goes along the ladder's outward normal, unless the
player is pressing sideways by more than 45 degrees, in which case the side direction is
added and the result renormalized.

**Notes** — never along the axis: you cannot jump *up* a ladder, only off it.

## `UpdateMaterial`

**Contract** — while climbing, reports the ladder's own material index so footstep sounds
and surface effects come from the ladder. Reports nothing in every other state.

## `NetRelcase`

**Contract** — when the ladder object is destroyed, drop the reference and fall to
`no_ladder` immediately, bypassing the transition table. Physics must never hold a dangling
reference across an object's destruction — the
[destroyed-entity invariant](../../SYSTEM-REQUIREMENTS.md#6-conformance).

## Notes

`PhDataUpdate`, `InitContact` and `EvaluateState` are present and empty. The state machine
is driven entirely from the tune phase of the step, before the solver runs, and needs no
post-step hook and no contact hook. Keeping the empty overrides is a C++ habit; a rebuild
does not need them.
