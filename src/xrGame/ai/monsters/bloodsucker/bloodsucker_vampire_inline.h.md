# src/xrGame/ai/monsters/bloodsucker/bloodsucker_vampire_inline.h

> The whole vampire behaviour as one tree: close on the player half-cloaked, feed if the chance comes, then run and vanish — and a global cooldown so no two bloodsuckers in the world can chain feeds.

**Needs** — [`bloodsucker_vampire.h`](bloodsucker_vampire.h.md) · [`bloodsucker_vampire_approach.h`](bloodsucker_vampire_approach.h.md) · [`bloodsucker_vampire_execute.h`](bloodsucker_vampire_execute.h.md) · [`bloodsucker_vampire_hide.h`](bloodsucker_vampire_hide.h.md) · [`state_hide_from_point.h`](../states/state_hide_from_point.h.md) · [`bloodsucker.h`](bloodsucker.h.md) · [`state.h`](../state.h.md)
**Used by** — [`bloodsucker_vampire.h`](bloodsucker_vampire.h.md)
**Tier floor** — T3: selection logic and authored parameters; nothing here touches a device or a byte layout

## Purpose

The bloodsucker's signature ability, expressed as a four-node tree under the shared state contract. The creature's ordinary attack is elsewhere; this tree replaces it whenever the creature "wants" a feed and the player is a legal victim.

Two decisions here are load-bearing beyond this file. First, the creature drops to *partial* visibility for the whole tree and back to full on every exit — so the cloak is a property of being in this behaviour, not of any one step. Second, the cooldown that gates the next feed is held on the creature's *class*, shared by every bloodsucker in the world, not per creature: two bloodsuckers cannot take turns draining the player.

## State

```text
RECORD VampireState
  enemy : reference to the victim captured at entry   # see check_completion
```

```text
SUBSTATES
  approach_enemy : close on the player while cloaked
  execute        : the feed itself
  run_away       : generic "flee from a point" state, parameterised below
  hide           : the composite retreat (flee, then stalk)
```

```text
SHARED ACROSS ALL BLOODSUCKERS
  time_last_vampire : int   # stamped on every exit from this tree, successful or not
```

Authored numbers:

```text
run_away_distance = 50 world units     # literal here
vampire_min_delay = from the creature's configuration section
```

## `VampireState` construction

**Contract** — Registers the four substates, each owned for the creature's lifetime.

## `initialize`

**Contract** — Entering the tree drops the creature to partial visibility, remembers who the victim is, and plays the hunt-start cue. Side effects on the creature's render state and the sound layer.

```text
FUNCTION initialize()
  set_visibility(partial)
  enemy = current_enemy
  play_sound(vampire_start_hunt)
```

## `reselect_state`

**Contract** — Picks the next substate from what ran last. Pure over the substates' start conditions.

```text
FUNCTION reselect_state()
  next = none

  IF previous == approach_enemy AND execute.check_start_conditions()
    next = execute
  IF previous == execute
    next = hide
  IF previous == approach_enemy
    next = hide          # approach ended without a chance: give up and retreat
  IF previous == hide
    next = hide          # keep retreating; `hide` decides when it is done
  IF next == none
    next = approach_enemy

  select(next)
```

**Notes** — The tests are written as a straight run of overwrites rather than a chain of alternatives, so the *last* matching line wins. That matters for the approach case: the feed test is written first and the unconditional "approach ended, go hide" second, so reaching this function straight from a finished approach always goes to `hide` — the only way into `execute` is the forced re-selection below, mid-approach. A rebuild that turns this run of assignments into an if/else chain silently changes the behaviour.

## `check_force_state`

**Contract** — Runs every tick before the substate does, and is what actually lets a feed start. If the approach is running and the feed's preconditions have just become true, it invalidates the current substate, which makes the tree re-select on the same tick.

```text
FUNCTION check_force_state()
  IF current == approach_enemy AND execute.check_start_conditions()
    invalidate_current_substate()
```

## `check_start_conditions`

**Contract** — Whether the whole vampire behaviour may be entered at all, asked by the creature's state manager. Reads only.

```text
FUNCTION check_start_conditions() -> bool
  IF NOT wants_to_feed()                              RETURN false
  IF creature is configured always-berserk            RETURN false
  IF enemy is not the player                          RETURN false
  IF NOT can_see_enemy_right_now()                    RETURN false
  IF player is already controlled by a creature       RETURN false
  IF player's input is held by something else         RETURN false
  IF time_last_vampire + vampire_min_delay > now()    RETURN false   # world-wide cooldown
  RETURN true
```

**Notes** — "Always-berserk" creatures are excluded because the feed needs a victim it can hold still; a berserk creature attacks anything including other creatures, and the feed is only defined against the player.

## `check_completion`

**Contract** — Ends the tree. Three ways out, and the second is why the victim is captured at entry.

```text
FUNCTION check_completion() -> bool
  IF current == hide AND hide.check_completion()      RETURN true
  IF enemy != current_enemy                           RETURN true   # target changed under us
  IF current != execute AND player already controlled RETURN true   # another one got there first
  RETURN false
```

**Notes** — The third test excludes the feeding state on purpose: during `execute` *this* creature is the one controlling the player, so the test would fire immediately and abort its own feed.

## `finalize` / `critical_finalize`

**Contract** — Identical on both paths: restore full visibility and stamp the shared cooldown with the current time. Stamping on the aborted path too is deliberate — an attempt that failed still buys the player the same respite as one that succeeded.

## `setup_substates`

**Contract** — Fills the flee state's parameter record when it is selected: flee from the last known player position, out to fifty units, running, aggressive acceleration, no braking, aggressive vocalisations spaced by the creature's configured attack-sound delay, giving up after fifteen seconds. The same record is authored in [`bloodsucker_vampire_hide_inline.h`](bloodsucker_vampire_hide_inline.h.md) for its own flee substate.

## `remove_links`

**Contract** — Clears the captured victim reference when that entity is destroyed, so `check_completion` compares against nothing rather than a dead handle. This is the creature-side half of the engine's object-teardown notification; a rebuild whose references can outlive their target safely still needs the *semantic* effect, namely that a destroyed victim reads as "no victim".
