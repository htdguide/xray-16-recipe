# src/xrGame/ai/monsters/states/monster_state_hear_int_sound_inline.h

> Heard something worth a look: walk toward it — or toward home, if it came from outside the
> territory — then stand and scan the least-covered direction.

**Needs** — [`monster_state_hear_int_sound.h`](monster_state_hear_int_sound.h.md) · [`state_custom_action_look.h`](state_custom_action_look.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`monster_state_hear_int_sound.h`](monster_state_hear_int_sound.h.md)
**Tier floor** — T3: a two-step sequence and one clamping rule

## Purpose

The curiosity behaviour, and the counterpart to the fear behaviour in
[`monster_state_hear_danger_sound_inline.h`](monster_state_hear_danger_sound_inline.h.md). Which of
the two runs is not decided here — it is decided by the senses, which classify each heard sound by
the emitter's AI-perception attributes. That classification is the whole difference between an
animal that investigates and one that bolts, and it lives in the data attached to each sound, not
in this file.

## State

`Stateless.` The sound position is re-read from the sound memory each time a leaf is set up.

## `reselect_state`

**Contract** — on the first selection, walk to the source if the walk leaf's own start condition
allows, otherwise skip straight to looking around. Every subsequent selection looks around.

```text
FUNCTION next_leaf(previous) -> leaf
  IF previous is none
    IF walk_to_source.can_start()  RETURN walk_to_source
    RETURN look_around
  RETURN look_around                 # absorbing
```

**Notes** — the look-around leaf is absorbing and has no timeout configured, so the creature stands
scanning until the brain selects a different top-level behaviour. Curiosity, like fear and like the
lost-contact search, ends by being outranked rather than by concluding.

## `get_target_position` — the territorial clamp

**Contract** — the walk destination is the sound's position, unless the creature has an authored
home region and the sound came from outside it, in which case the destination is the home region's
own reference point.

```text
FUNCTION target_position() -> vector
  sound_position = sound_memory.last_sound.position
  IF NOT home.exists           RETURN sound_position
  IF home.contains(sound_position) RETURN sound_position
  RETURN navigation.position_of(home.reference_vertex)
```

**Notes** — this is the behaviour's one real decision and it is easy to miss, because it changes
what the creature is doing rather than how far it goes. A territorial animal that hears something
outside its territory does not investigate it and does not ignore it: it walks *back to the middle
of its own ground* and looks around there. So a player skirting a lair draws the pack inward, not
outward — which is what stops one noisy player from pulling an entire region's wildlife across the
map.

A creature with no home region investigates the sound directly, which is the behaviour of the
wandering, non-territorial species.

## `setup_substates`

**Contract** — fill the parameter record of whichever leaf was selected.

```text
walk_to_source:
  point            = target_position()
  vertex           = unknown, let the path builder resolve it
  gait             = walk forward, accelerating, no braking, calm profile
  completion_dist  = 2                       # hard-coded
  voice            = idle, delay = section key "idle_sound_delay"

look_around:
  action           = look around
  time_out         = none
  face             = self.position + least_covered_direction * 10
  voice            = idle, delay = section key "idle_sound_delay"
```

**Notes** — the look-around leaf is the *facing* variant of the generic action leaf (see
[`state_custom_action_look.h`](state_custom_action_look.h.md)): it turns the creature toward a point
while playing the action. The point is ten units away along the direction the cover component
reports as least covered.

That is the same rule the fear behaviour uses and it means the same thing here: after investigating,
the animal turns to watch the open ground, because that is where something would come from. The ten
units only set the turn target's distance; only the direction matters, and the distance is
hard-coded.

`idle_sound_delay` is the only per-creature number in the behaviour. Two units of arrival tolerance
and the ten-unit facing distance are compiled in.
