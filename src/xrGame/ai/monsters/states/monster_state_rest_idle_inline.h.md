# src/xrGame/ai/monsters/states/monster_state_rest_idle_inline.h

> Standing around: take a nearby covered spot nobody else in the squad has claimed, walk to it
> exactly, turn to face the most exposed direction, and rest there.

**Needs** — [`monster_state_rest_idle.h`](monster_state_rest_idle.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_look_point.h`](state_look_point.h.md) · [`state_custom_action.h`](state_custom_action.h.md) · [`state_data.h`](state_data.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../../../cover_point.h`](../../../cover_point.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`monster_state_rest_idle.h`](monster_state_rest_idle.h.md)
**Tier floor** — T3: a two-tier cover search, a claim, and a three-step sequence

## Purpose

Where a creature spends most of its life. It is worth reading closely for exactly that reason: the
composition of the world's ambient wildlife — animals dotted about in cover rather than standing in
open fields — is produced by these thirty lines, and a rebuild that simplifies them to "stand
still" changes how every level looks from a distance.

## State

```text
RECORD RestIdleState
  target_vertex : optional<int>   # claimed with the squad while held
```

**Invariant** — claimed on entry if found, released on both exits.

## `initialize`

**Contract** — look for cover close by, between five and ten units away; failing that, look again
in the wider ten-to-thirty band; failing that, give up and leave the destination absent. Claim
whatever was found.

```text
FUNCTION initialize()
  target_vertex = absent
  point = cover_system.find_cover(from = self.position, min = 5,  max = 10)
  IF point is absent
    point = cover_system.find_cover(from = self.position, min = 10, max = 30)
    IF point is absent  RETURN
  target_vertex = point.vertex
  squad.claim(target_vertex)
```

**Notes** — the two tiers are the behaviour's one real decision. A resting creature prefers cover
it can reach in a couple of seconds; only if there is none does it accept a walk of up to thirty
units. That keeps idle animals dispersed across the concealment near where they already are,
instead of all of them converging on the best cover in the region.

Claiming the spot with the squad is what keeps two packmates from settling on the same rock. The
claim table is shared with every other behaviour in this directory that reserves ground — combat
fall-backs, danger retreats — so a resting creature's spot is also not taken by a fighting one.

Both bands are hard-coded, and the second reuses the generic 10-to-30 "near cover" band that
appears throughout this directory.

## `finalize` / `critical_finalize`

**Contract** — both release the claim.

**Notes** — the release is unconditional, so when no cover was found the composite releases an
absent vertex. The claim table's release is a remove-one on a list, so that is a no-op; a rebuild
guards it and nothing changes.

## `reselect_state`

**Contract** — walk to the cover if there is any, then face the open ground, then rest, with
resting absorbing.

```text
FUNCTION next_leaf(previous) -> leaf
  IF previous is none AND target_vertex EXISTS  RETURN walk_to_cover
  IF previous IN {none, walk_to_cover}          RETURN face_open_ground
  RETURN settle
```

**Notes** — the second clause catching "none" again is what handles the no-cover case: a creature
that found nothing skips the walk and goes straight to facing and settling, where it stands. So the
behaviour degrades to "stand still and look around" rather than failing, which is the correct
fallback for an idle animal in an open field.

The settle leaf absorbs. The composite has no completion test of its own; it is ended by the
peacetime cascade above it re-selecting something else, which it does every update.

## `setup_substates`

**Contract** — fill the parameters of whichever leaf was selected.

```text
walk_to_cover:
  vertex           = target_vertex
  point            = navigation.position_of(target_vertex)
  gait             = walk forward, accelerating, braking on arrival, calm profile
  action time_out  = none
  completion_dist  = 0                  # arrive exactly on the cell
  rebuild          = never              # keep the route planned on entry
  voice            = idle, delay = section key "idle_sound_delay"

face_open_ground:
  point            = self.position + least_covered_direction * 10
  action           = stand idle, time_out 2000 ms
  face_delay       = 0                  # begin turning immediately
  voice            = idle, delay = section key "idle_sound_delay"

settle:
  action           = rest
  time_out         = none               # absorbing
  voice            = idle, delay = section key "idle_sound_delay"
```

**Notes** — the walk uses the same commit-to-one-route settings as the danger retreats — no
timeout, exact arrival, no re-planning — but at a calm walk with braking. Nothing is urgent, and
re-planning would only make the creature wander off its chosen spot.

*Facing the least-covered direction is the same rule used by the fear behaviours and the danger
retreat*, and it means the same thing: an animal that has settled into cover watches the open
ground. It is why the wildlife in this game reliably has its back to the rock and its face to the
clearing, which is a large part of why the world looks alive. Only the direction matters; ten is an
arbitrary lever arm for the turn target.

*The rest action is distinct from standing idle* — it is the lying-down, grooming, low-alert
animation family. The two-second pause facing the open ground before settling is what makes the
transition read as an animal checking its surroundings and then relaxing.

`idle_sound_delay` is the only authored number. The three leaf identifiers are drawn from the
shared state vocabulary, and one of them — the settle leaf — is registered here under the same
identifier the peacetime cascade uses for this whole composite. The two live in different leaf
tables so they do not collide, but a rebuild should not assume identifiers are globally unique.
