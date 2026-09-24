# src/xrGame/ai/monsters/states/monster_state_help_sound_inline.h

> A packmate called for help: run to the exact navigation cell the call came from, look around for
> three seconds, and stop.

**Needs** — [`monster_state_help_sound.h`](monster_state_help_sound.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_custom_action_look.h`](state_custom_action_look.h.md) · [`state_data.h`](state_data.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`monster_state_help_sound.h`](monster_state_help_sound.h.md)
**Tier floor** — T3: a two-step sequence over a remembered navigation vertex

## Purpose

The mechanism by which one animal in trouble pulls the rest of its kind onto the player. It is
distinct from the ordinary interesting-sound reaction in three ways that all matter: the
destination is a **navigation vertex**, not a position; the creature **runs** rather than walks;
and the behaviour **terminates** instead of absorbing.

Those three together give the reinforcement its character — a fast, committed, finite convergence
rather than open-ended milling. Whether a creature's own distress call is loud enough to be heard,
and by whom, is decided by the senses and the sound's authored AI-perception attributes, not here.

## State

`Stateless.` The call's vertex is re-read from the sound memory each time the running leaf is set
up.

## `check_start_conditions`

**Contract** — begin only when the sound memory reports an unanswered distress call, and — for a
creature with an authored home region — only when that call's position lies inside the region.

```text
FUNCTION check_start_conditions() -> bool
  IF NOT sound_memory.heard_help_call  RETURN false
  IF home.exists
    RETURN home.contains(navigation.position_of(sound_memory.help_call_vertex))
  RETURN true
```

**Notes** — the territorial clamp here is a *refusal*, not a redirection. The curiosity behaviour
redirects an out-of-territory sound to the home point; this one simply declines to respond. A
packmate calling from outside the lair gets no help, which is what keeps a territory's defenders
in the territory.

## `reselect_state`

**Contract** — first the run, then the look-around, then nothing.

```text
FUNCTION next_leaf(previous) -> leaf
  IF previous is none                 RETURN run_to_call
  IF previous == run_to_call          RETURN look_around
  RETURN none                          # deliberately selects nothing
```

## `check_completion`

**Contract** — finished when no leaf is current.

**Notes** — the pairing of these two is the termination mechanism and it is worth stating plainly,
because it is the only behaviour in this directory that ends by itself. The reselection after the
look-around chooses nothing; the base machinery therefore leaves the current leaf unset; and the
completion test reads exactly that condition. A rebuild that makes "choose a leaf" total — say, by
looping the last leaf — turns this into an absorbing behaviour like all the others and the
reinforcement never disengages.

## `setup_substates`

**Contract** — fill the parameter record of whichever leaf was selected.

```text
run_to_call:
  vertex           = sound_memory.help_call_vertex
  point            = navigation.position_of(that vertex)
  gait             = run, accelerating, braking on arrival, aggressive profile
  action time_out  = none              # run until arrival, however long it takes
  completion_dist  = 0                 # arrive exactly on the vertex
  rebuild          = never             # keep the route planned on entry
  voice            = idle, delay = section key "idle_sound_delay"

look_around:
  action    = look around
  time_out  = 3000 ms
  voice     = idle, delay = section key "idle_sound_delay"
```

**Notes** — the three unusual settings on the run are a set, and they say "this destination is
exact and non-negotiable": no timeout means the creature keeps running however far it is; zero
arrival tolerance means it must reach the cell itself, not its neighbourhood; and never rebuilding
means it follows the route it planned on entry rather than continuously re-planning toward a moving
target. Nothing about the call moves, so re-planning would only cost time.

*The destination is a vertex because the call is remembered as a vertex.* The sound memory stores
the navigation cell a distress call came from, not its world position, which means the
reinforcement converges on ground that is known walkable. Running to a raw position would put
arriving creatures on the far side of the geometry the caller is behind.

**Dead field.** The look-around leaf here is the *facing* variant of the generic action leaf, whose
parameter record is the plain action record plus a point to face. This behaviour fills it with a
plain action record only — the facing point is never written. The leaf consequently turns the
creature toward whatever point that leaf object happens to hold, which is the origin the first time
it runs. The curiosity behaviour, which shares this leaf type, does write the point. A rebuild
either writes a facing point here or uses the non-facing leaf; reproducing the original exactly
means reproducing a turn toward the world origin, which is not a behaviour anyone designed.

`idle_sound_delay` is the only authored number; the three-second scan is compiled in.
