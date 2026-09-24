# src/xrGame/ai/monsters/ai_monster_defs.h

> The shared vocabulary of every non-human creature: the animation alphabet, the abstract action set, the transition and attack-timing records, and the small value types their managers exchange.

**Needs** — [`Include/xrRender/KinematicsAnimated.h`](../../../Include/xrRender/KinematicsAnimated.h.md) · [`xrEngine/CameraManager.h`](../../../xrEngine/CameraManager.h.md)
**Used by** — [`ai_monster_motion_stats.h`](ai_monster_motion_stats.h.md) · [`ai_monster_shared_data.h`](ai_monster_shared_data.h.md) · [`anti_aim_ability.h`](anti_aim_ability.h.md) · [`base_monster.h`](basemonster/base_monster.h.md) · [`boar.cpp`](boar/boar.cpp.md) · [`control_animation_base.cpp`](control_animation_base.cpp.md) · [`control_animation_base.h`](control_animation_base.h.md) · [`control_animation_base_load.cpp`](control_animation_base_load.cpp.md) · [`control_movement_base.h`](control_movement_base.h.md) · [`control_path_builder_base.h`](control_path_builder_base.h.md) · [`controller_animation.h`](controller/controller_animation.h.md) · [`monster_corpse_manager.h`](monster_corpse_manager.h.md) · [`monster_corpse_memory.cpp`](monster_corpse_memory.cpp.md) · [`monster_corpse_memory.h`](monster_corpse_memory.h.md) · _and 6 more_
**Tier floor** — T2: an enumeration used as a dense array index, and records read from configuration lines

## Purpose

Every creature in chapter 24 speaks this vocabulary. It is a pure definitions file with no
implementation, and it is the one page a reader must have in hand before any monster twin
makes sense: the animation identifiers here are what the state machines request, the action
identifiers are what the animation layer resolves them against, and the records here are
what the configuration parser fills.

The two enumerations at its centre — the **motion alphabet** and the **action set** — are
the chapter's central design decision, so they get most of this page.

## The motion alphabet

A single flat enumeration names roughly seventy *abstract* animations — stand idle, walk
forward, run, turn left while running, attack, attack from behind, drag corpse, sleep,
howl, and so on — plus a handful that only one creature uses (psi attack, telekinesis,
gravity fire, shield start). Its values are used as **dense array indices** into per-creature
animation tables, so the order is load-bearing within a build but not across the shipped
data: the names in the data are strings and the enumeration is the code's own indexing.

Two conventions inside it are worth naming:

- **Upper stance.** A creature that can rear onto two legs has a parallel family
  (`upper stand idle`, `upper walk forward`, `upper attack`) plus the two transition
  animations between stances. The stance is not a separate state machine; it is a different
  region of the same alphabet.
- **A reserved "undefined" value** distinct from index zero, so "no animation selected" and
  "stand idle" are different answers. Stand idle is the default an unset creature falls back
  to.

## The action set

Sixteen *generic actions* — stand idle, sit idle, lie idle, walk forward, walk backward,
run, eat, sleep, rest, drag, attack, steal, look around, prepare capture, and two home-area
walks. This is deliberately a much smaller alphabet than the motion one.

**The relationship between the two is the chapter's core idea.** A creature's behaviour
asks for an *action*; the animation layer turns that into a *motion*, choosing among the
motions that serve that action according to the creature's current posture, its damage
state, its aggression, and whether it must turn. A state machine therefore never names an
animation, and adding a new animation variant to a creature is a data change.

## Posture and hit geometry

```text
ENUM Posture      = { standing, sitting, lying, standing_upright }
ENUM HitSide      = { front, back, left, right }
ENUM DangerLevel  = { weak, normal, strong, very_strong, none }
```

Posture is the state the transition table routes between: a creature asked to walk while
lying down must play "stand up from lying" first. Hit side is resolved from the geometry of
an incoming hit and selects which damage reaction plays. Danger level grades a perceived
threat and feeds the state selectors.

A creature also declares its **leg count**, four or two, which selects footstep-sound
grouping and a few animation conventions.

## Records

```text
RECORD VelocityParam              # one configuration line, five comma-separated values
  linear         : real           # world units per second
  angular_real   : real           # radians per second the body actually turns at
  angular_path   : real           # radians per second the path follower plans against
  min_factor     : real           # multiplier bounds applied by the speed manager
  max_factor     : real
```

The split between *real* and *path* angular velocity is load-bearing: the path follower
plans corners assuming one turn rate while the body turns at another, which is what lets a
creature visibly overshoot a corner and correct, rather than pivoting on the spot.

```text
RECORD AnimationItem              # one entry in a creature's animation table
  base_name    : text             # e.g. "stand_idle_" — variants append an index
  spec_id      : int              # -1 = serves any specialisation; else a tag the
                                  #     selector matches against
  count        : int              # how many numbered variants exist
  posture      : Posture          # the posture this animation leaves the creature in
  velocity     : VelocityParam
  damage_fx    : { front, back, left, right : text }   # hit-reaction effect motions
```

```text
RECORD Transition                 # one edge of the posture/animation transition table
  from    : { by_posture : bool, animation, posture }
  target  : { by_posture : bool, animation, posture }
  via     : animation             # the bridging motion to play
  chain   : bool                  # may this transition be chained with another
  skip_if_aggressive : bool       # an angry creature skips the polite version
```

**Invariants** — each end of a transition is matched *either* by posture or by a specific
animation, never both; the flag selects which. `skip_if_aggressive` is how a creature that
has just spotted an enemy gets out of a sleep or a sit without playing the leisurely
stand-up.

```text
RECORD MotionItem                 # maps an abstract action to a motion, with turn variants
  animation     : animation
  has_turn      : bool
  turn_left     : animation
  turn_right    : animation
  min_angle     : real            # below this heading error, do not play a turn at all
```

```text
RECORD ReplacedAnimation          # a conditional substitution
  when     : animation
  becomes  : animation
  condition: reference to bool    # re-read each time the substitution is considered
```

**Notes** — the substitution table is how a creature swaps in its damaged-movement
animations wholesale once a health threshold is crossed, without every state knowing about
damage. In the original the condition is a pointer to a flag the creature owns; a rebuild
wants a named predicate, which is what the pointer stands for.

```text
RECORD AttackAnimation            # when, within a motion, does a melee attack land
  animation   : animation
  variant     : int
  window      : (time_from, time_to)   # ms into the motion; the hit may land only here
  trace_from  : vec3                   # trace origin and endpoint, relative to the
  trace_to    : vec3                   #   creature's centre
  flags       : set of { attacks_rats, no_trace_required }
  damage      : real
  hit_dir     : vec3                   # direction the impulse is applied in
  cone        : (yaw_from, yaw_to, pitch_from, pitch_to)
  dist        : real
```

**Invariants** — the hit window is the whole contract of a melee attack: a creature may
damage its target only while the motion's clock is inside it, which is what makes a melee
attack dodgeable. The `no_trace_required` flag exists for attacks that must connect
regardless of intervening geometry — a rebuild that always traces makes those attacks miss.

```text
RECORD AbilityAttackParam         # the same idea for a scripted or special ability
  motion, time, damage, impulse, impulse_dir, cone, dist
```

```text
RECORD CurrentAnimationInfo       # what the animation layer is playing right now
  index           : int           # which numbered variant
  time_started    : int
  motion          : animation     # private; read and written through accessors
  name            : text
  blend           : handle to the active blend
  speed           : { current, target : real }   # written only through setters that
                                                 # assert |v| < 1000
  speed_change_vel: real          # how fast current chases target
```

**Invariants** — the speed guard is not decoration. A creature's movement speed is derived
from animation and terrain and can be computed as a division; a degenerate result sends the
creature across the level in one frame. The guard is the recorded symptom of that bug.

```text
RECORD RememberedEnemy    { position : vec3, vertex : int, time : int, danger : real }
RECORD RememberedCorpse   { position : vec3, vertex : int, time : int }
RECORD RememberedHit      { source, position : vec3, time : int, side : HitSide }
```

**Invariants** — every remembered position is stored **as a navigation mesh vertex plus a
position**, never as a bare coordinate, because only a vertex is something the pathfinder
and the cover system can reason about. This is the chapter-wide convention stated in the
glossary, and it is enforced by these three records.

```text
RECORD StepSound        { volume, frequency : real }
RECORD AttackEffector   { post_process : PostProcessInfo,
                          time, attack_time, release_time : real,
                          camera : { time, amplitude, periods, power : real } }
RECORD Velocity         { current, target : real }
RECORD MotionVelocity   { linear, angular : real }
ENUM  AccelerationType  = { calm, aggressive }
ENUM  AccelerationPhase = { accelerating, braking }
```

## Animation specialisation flags

A bitset of about fifteen tags — moving backward, dragging a corpse, checking a corpse,
attacking a rat, standing scared, threatening, attacking from behind, rotation jump,
running attack, psi attack, upper stance, smelling while moving. These are matched against
an animation item's `spec_id` to choose between several motions that serve the same action.
The set is open-ended by nature and grew one tag at a time as creatures were added; a
rebuild may model it as a set of named tags rather than a fixed-width mask, since nothing
serializes it.

## Enemy observation flags

A bitset recording what a creature has noticed about a specific enemy: that it died, that
it went out of sight, that it is closing, that it is closing *fast*, that it is retreating,
that it is standing still, that it is hiding, that it is running away, that it does not
know about me, that it cannot see me, that it went offline, and that its statistics are not
yet available. These are the inputs the state selectors weigh; they are computed by the
perception side and consumed by the behaviour side, and naming them here is what keeps the
two halves decoupled.

**Notes** — "does not know about me" and "cannot see me" are separate flags and mean
different things: the first is about the enemy's memory, the second about its current line
of sight. A creature that sneaks (the bloodsucker) branches on the first; a creature that
takes cover branches on the second.

## Timing constants

```text
CRITICAL_STAND_TIME       = 1400 ms   # how long a creature may be stuck at zero speed
                                      #   while trying to move before it is treated as stuck
TIME_STAND_RECHECK        = 2000 ms   # how often that check re-runs
SOUND_ATTACK_HIT_MIN_DELAY= 1000 ms   # floor between two attack-hit sounds from one creature
MORALE_NORMAL             = 0.5       # the midpoint a creature's morale returns to
```

The default melee cone — ±30 degrees in yaw and pitch at 3.5 world units — is defined here
as the shared fallback for attacks that do not declare their own.

**Notes** — none of these four numbers is derived anywhere. The stuck-detection pair is the
most consequential: 1400 ms of zero progress is what it takes for a creature to conclude it
is jammed against geometry, and a rebuild that shortens it gets creatures that abandon
valid paths.

## Timing and once-only helpers

Three textual patterns used throughout the chapter, worth naming because their *intent* is
easy to lose:

- **do once** — a guarded block that runs on the first pass after a flag is cleared. The
  flag is owned by the state, and clearing it on state entry is what re-arms it.
- **timed out** — a comparison against the creature's own cached current time, not the
  device clock. Every monster caches the time once per think so that all its decisions in
  one tick agree on "now".
- **do at intervals** — the same, plus the re-arm.

A separate helper asks "does the path need rebuilding", which it answers as "am I within
two nodes and half a unit of the end of my current path". That threshold is what makes a
creature re-plan *before* it arrives rather than stalling at the end of a path.

## Aliases

Time is a 32-bit millisecond count throughout the chapter. Enemy and corpse memories are
maps keyed by the remembered entity. The animation-to-name map exists only for debug
display.
