# src/xrGame/ai/monsters/states/state_data.h

> The parameter records a composite creature behaviour fills in before handing control to one of the generic leaf states.

**Needs** — _(none: a pure record vocabulary)_
**Used by** — [`bloodsucker_attack_state.h`](../bloodsucker/bloodsucker_attack_state.h.md) · [`group_state_eat_inline.h`](../group_states/group_state_eat_inline.h.md) · [`group_state_panic_inline.h`](../group_states/group_state_panic_inline.h.md) · [`group_state_squad_move_to_radius.h`](../group_states/group_state_squad_move_to_radius.h.md) · [`monster_state_eat_inline.h`](monster_state_eat_inline.h.md) · [`monster_state_find_enemy_look_inline.h`](monster_state_find_enemy_look_inline.h.md) · [`monster_state_hear_danger_sound_inline.h`](monster_state_hear_danger_sound_inline.h.md) · [`monster_state_help_sound_inline.h`](monster_state_help_sound_inline.h.md) · [`monster_state_home_point_danger_inline.h`](monster_state_home_point_danger_inline.h.md) · [`monster_state_panic_inline.h`](monster_state_panic_inline.h.md) · [`monster_state_rest_idle_inline.h`](monster_state_rest_idle_inline.h.md) · [`monster_state_smart_terrain_task_inline.h`](monster_state_smart_terrain_task_inline.h.md) · [`monster_state_squad_rest_follow_inline.h`](monster_state_squad_rest_follow_inline.h.md) · [`monster_state_squad_rest_inline.h`](monster_state_squad_rest_inline.h.md) · _and 13 more_
**Tier floor** — T2: these records are copied by raw byte length into a leaf state's storage, so their size and layout are load-bearing in the original; a rebuild that passes them by value or by reference lifts this to T3

## Purpose

A creature's behaviour tree is built from a small set of *generic leaf states* — go to a
point, retreat from a point, turn to face a point, stand and do a thing. A composite state
selects one of those and must then tell it *which* point, *how fast*, *what animation* and
*what sound*. This header is the vocabulary for that hand-off: one record per leaf state,
plus a shared record for the animation-and-sound half that every leaf state has.

It is a separate file because the leaf states and their many callers both need these
records, and the leaf states are templates that cannot be included from each other without
a cycle. A rebuild that makes the leaf states generic over their own parameter type can
delete this file and put each record beside its state.

## State

Stateless — this file declares records only.

```text
RECORD StateAction                 # the animation-and-sound half of every leaf state
  action      : ActionId           # default: stand idle
  spec_params : int (bit field)    # extra animation modifiers, meaning is per-creature
  time_out    : int (milliseconds) # 0 means "no timeout"; the state ends on its own terms
  sound_type  : optional<SoundId>  # absent means "play nothing"
  sound_delay : optional<int>      # absent means "use the state's own default delay"
```

```text
RECORD StateMoveToPoint
  point           : vector3
  vertex          : optional<MeshVertexId>   # absent: let the path builder resolve the point
  target_direction: vector3                  # zero magnitude means "no required facing on arrival"
  accelerated     : bool                     # engage the walk-to-run acceleration chain
  braking         : bool                     # decelerate into the destination rather than stopping dead
  accel_type      : AccelerationProfile      # calm / aggressive; selects the acceleration curve
  completion_dist : real (metres)            # 0 means "arrive within one mesh cell"
  action          : StateAction
```

```text
RECORD StateMoveToPointEx EXTENDS StateMoveToPoint
  time_to_rebuild : optional<int (milliseconds)>
    # absent  : never re-path, follow the route computed at entry
    # 0       : re-path at the path builder's own default cadence
    # positive: re-path no more often than this
```

```text
RECORD StateHideFromPoint
  point              : vector3      # the thing to get away from
  accelerated        : bool
  braking            : bool
  accel_type         : AccelerationProfile
  distance           : real         # UNREAD by the state that owns this record (see Notes)
  cover_min_dist     : real         # UNREAD
  cover_max_dist     : real         # UNREAD
  cover_search_radius: real         # UNREAD
  action             : StateAction
```

```text
RECORD StateLookToPoint
  point      : vector3
  face_delay : int (milliseconds)   # hold the turn request this long before it may be revised
  action     : StateAction
```

```text
RECORD StateMoveAroundPoint          # never reaches a running state — see Notes
  point   : vector3
  vertex  : optional<MeshVertexId>
  radius  : real (default 10)
  accelerated, braking, accel_type
  action  : StateAction
```

```text
RECORD StateActionLook EXTENDS StateAction
  point : vector3                   # what to face while performing the action
```

**Invariants**

- `sound_type` and `sound_delay` are both encoded as "all bits set means absent". Two
  distinct absences are meaningful: no sound at all, and a sound with the creature's own
  configured repeat delay rather than a caller-chosen one. A rebuild using a real optional
  must keep both.
- `time_out` of zero is *not* "expire immediately"; it is "this state decides its own
  completion". Every leaf state checks the timeout first and falls through to its own test
  when the timeout is zero, so a caller that wants a state to run to its natural end leaves
  the field at zero.
- `StateMoveToPointEx.time_to_rebuild` has *three* meanings packed into one integer, and
  the "never re-path" case is the all-bits-set value rather than a fourth flag. A rebuilder
  wanting a plain integer must pick a different sentinel and audit every caller.

## Notes

**The four unread cover fields.** `distance`, `cover_min_dist`, `cover_max_dist` and
`cover_search_radius` on the retreat record are authored by several callers — the eating
behaviour sets 20 / 30 / 25 — but the state that owns the record never reads them. It asks
the path builder for a retreat route and lets the builder apply its own generic cover
parameters. So tuning those three numbers in a caller changes nothing. A rebuild should
either wire them through to the retreat search or delete them; the shipped behaviour is the
one where they are ignored.

**`StateMoveAroundPoint` is dead.** Nothing instantiates the state that consumes it, and
that state's implementation is commented out besides. The record is retained here only so
the twin mirrors the source.

**Defaults are code, not data.** Every default in these records — 10 metre orbit radius,
1 metre retreat distance, 10/30/20 cover distances — is a compile-time initializer, not a
configuration value. The numbers a designer actually tunes arrive through the creature's
configuration section and are applied by the composite states that fill these records.
