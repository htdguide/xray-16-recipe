# src/xrGame/ai/monsters/states/monster_state_home_point_danger_inline.h

> Frightened and away from home: claim a covered spot inside the territory, run there exactly, turn
> to face the most exposed direction, and hold — but if the spot was not covered, leave as soon as
> you arrive.

**Needs** — [`monster_state_home_point_danger.h`](monster_state_home_point_danger.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_look_point.h`](state_look_point.h.md) · [`state_custom_action.h`](state_custom_action.h.md) · [`state_data.h`](state_data.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../../../cover_point.h`](../../../cover_point.h.md)
**Used by** — [`monster_state_home_point_danger.h`](monster_state_home_point_danger.h.md)
**Tier floor** — T3: a claim, a three-step sequence and one direction query

## Purpose

The single object that gives both fear behaviours their territorial variant. It is registered as
the go-home leaf by the frightening-sound behaviour and by the shot-from-nowhere behaviour, and in
both it is tested *first* on every reselection — so for a creature with a lair, it is the reaction
to danger and the flee-and-cower or break-away-and-stalk sequences are what happens only when there
is no lair to run to.

Its counterpart in combat is
[`monster_state_home_point_attack_inline.h`](monster_state_home_point_attack_inline.h.md), and the
difference between them is instructive: the combat version hops from spot to spot continuously and
never settles; this one runs to exactly one spot and then stops moving.

## State

```text
RECORD DangerHomeState
  target_vertex : int      # claimed with the squad for the composite's whole lifetime
  skip_camp     : bool     # true when the destination is merely inside the territory, not covered
  danger_pos    : vector   # scratch; recomputed on every query
```

**Invariant** — exactly one vertex is claimed, from entry to exit, on both exit paths. Unlike the
combat version this destination is never re-chosen, so there is never a moment with two claims or
none.

## `get_most_danger_pos`

**Contract** — report the position of the thing currently most worth fleeing: the last hit position
if the creature has been hit, otherwise the last heard dangerous sound, otherwise the origin.

```text
FUNCTION most_danger_position() -> vector
  IF hit_memory.is_hit         RETURN hit_memory.last_hit_position
  IF senses.heard_danger_sound RETURN sound_memory.last_sound.position
  RETURN (0, 0, 0)
```

**Notes** — the ordering is the priority: a hit outranks a sound, because being shot is more
informative than hearing a shot. The origin fallback is not a sentinel — it is a real vector that
the start-condition test will then ask the home component about, and the origin is almost certainly
outside any territory, so a creature with neither a hit nor a sound *passes* the start test on the
strength of a meaningless position. In practice the composite is only ever offered by behaviours
that already established one or the other, so the case does not arise; a rebuild should use an
absent value and refuse rather than rely on that.

## `check_start_conditions`

**Contract** — startable when the creature is outside its home region **and** the danger is also
outside it.

**Notes** — both clauses matter and the second is the interesting one. A creature that is away from
home and whose danger is *inside* the home region does not retreat there — it would be running
toward the threat. It falls through to the ordinary flee-and-cower or break-away sequence instead.
That one test is what stops a pack from obligingly funnelling itself into a lair the player is
standing in.

## `initialize`

**Contract** — evaluate the danger position (for its side effect of filling the scratch vector),
ask the home component for a covered place inside the territory, and fall back to any place there
when it has none — recording which of the two happened. Claim the result with the squad.

```text
FUNCTION initialize()
  most_danger_position()                 # side effect only
  target_vertex = home.a_covered_place()
  skip_camp     = false
  IF target_vertex is absent
    target_vertex = home.any_place()
    skip_camp     = true
  squad.claim(target_vertex)
```

**Notes** — unlike the combat version there is no retry against drawing the creature's own cell and
no third fallback: if the home component yields nothing, the destination is absent and is claimed
and pathed to as if it were valid. A rebuild guards this; the original does not, and it does not
bite because the behaviour is only reachable for creatures whose home region has places in it.

`skip_camp` is the load-bearing output. It records *why* the destination was chosen, and the
completion test below turns that into a behavioural difference.

## `reselect_state`

**Contract** — run, then turn, then hold, with holding absorbing.

```text
FUNCTION next_leaf(previous) -> leaf
  IF previous is none   RETURN run_home
  IF previous == run_home RETURN face_open_ground
  RETURN hold
```

## `check_completion`

**Contract** — finished when a new hit has landed since entry, or when the destination was merely
inside the territory rather than covered and the run has ended.

```text
FUNCTION check_completion() -> bool
  IF hit_memory.last_hit_time > entry_time  RETURN true
  IF skip_camp AND previous_leaf EXISTS AND previous_leaf != run_home  RETURN true
  RETURN false
```

**Notes** — this is the payoff of `skip_camp`, and it is a genuinely nice piece of design.
Arriving at *covered* ground, the creature stays: it turns to face the open and holds there
indefinitely, which is the lair behaviour. Arriving at *uncovered* ground, the composite ends the
instant the run does — before the turn can even play — and the creature is handed back to the
behaviour above, which will pick the ordinary flee-and-cower sequence instead. So a territory
with no cover in it is not treated as a refuge, and a creature does not stand cowering in the open
just because the open happens to be home.

A fresh hit ends the composite in either case, which returns the creature to the oscillation in
[`monster_state_hitted_inline.h`](monster_state_hitted_inline.h.md) — being shot while retreating
means the retreat was not working.

## `setup_substates`

**Contract** — fill the parameters of whichever leaf was selected.

```text
run_home:
  vertex           = target_vertex
  point            = navigation.position_of(target_vertex)
  gait             = run, accelerating, no braking, aggressive profile
  action time_out  = none               # run however long it takes
  completion_dist  = 1                  # arrive essentially on the cell
  rebuild          = never              # keep the route planned on entry
  voice            = aggressive, delay = section key "attack_sound_delay"

face_open_ground:
  point            = self.position + least_covered_direction * 10
  action           = stand idle, time_out 2000 ms
  face_delay       = 0                  # begin turning immediately
  voice            = aggressive, delay = section key "idle_sound_delay"

hold:
  action           = look around
  time_out         = 7000 ms
  voice            = aggressive, delay = section key "idle_sound_delay"
```

**Notes** — the run's three settings are the same non-negotiable-destination set used by the
answer-a-call behaviour: no timeout, tight arrival tolerance, no re-planning. The creature commits
to the route it planned when it entered.

*The facing target is the least-covered direction*, ten units out — the same rule the non-
territorial fear behaviour uses. A creature that has just taken cover turns to watch the ground
something would have to cross to reach it. Only the direction matters; ten is an arbitrary lever
arm.

*The hold leaf has a seven-second timeout, and it still absorbs.* The reselection routes it back to
itself, so the timeout only causes the leaf to be re-entered — which restarts its animation and
re-rolls its voice throttle. That is how a cowering creature keeps twitching and calling instead of
freezing into one long pose. The source comment beside the number says "do not use time out",
copied from a leaf where the value was zero; here the value is not zero and the comment is wrong.

Two authored numbers appear: `attack_sound_delay` on the run and `idle_sound_delay` on the two
stationary leaves — a creature calls in its attack voice while fleeing and in its idle voice once
settled, even though the voice *type* is aggressive throughout. The mismatch between voice type and
repeat delay is deliberate in the original and easy to lose in a rebuild. Everything else — 1, 10,
2000, 7000 — is compiled in.
