# src/xrGame/ai/monsters/states/state_test_state_inline.h

> Implements the two harness composites; the live one is a two-state loop — walk to your assigned cover cell, then stand on it — and it is the snork's search behaviour.

**Needs** — [`state_test_state.h`](state_test_state.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_custom_action.h`](state_custom_action.h.md) · [`state_data.h`](state_data.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`state_test_state.h`](state_test_state.h.md)
**Tier floor** — T3: a two-way selection and two parameter fills

## Purpose

Shows the composite-state pattern at its smallest: a composite owns named sub-states,
answers three questions per tick — must I abandon my current sub-state, which sub-state do
I want, and what parameters does it get — and delegates everything else.

One of the two is shipped behaviour. That is the finding: the snork's "I have lost my enemy
and I am searching" state is this test composite, so the snork does not search. It walks to
a cover cell somebody else assigned it and stands there.

## `CStateMonsterTestCover` — take cover and camp

**Contract** — owns two sub-states: an extended move-to-point under the *hide in cover*
identifier and a stand-and-act under the *camp in cover* identifier. Entry latches the
creature's currently assigned target cell. Each tick:

- **forced reselection** — if the assignment changed since the last tick, re-latch it and
  drop the current sub-state so selection runs fresh. Separately, if the creature is
  camping but is no longer standing on the latched cell — it was pushed, or the cell moved
  underneath it — drop the sub-state too.
- **selection** — standing on the latched cell means camp; anything else means move to it.
- **parameter fill** — the move gets an aggressive run with exact arrival, continuous
  re-pathing and a 200-second timeout; the camp gets the idle stance. Both get the
  creature's configured idle vocalisation with the creature's own configured repeat delay.

```text
FUNCTION initialize()
  last_node = object.assigned_target_node

FUNCTION check_force_state()
  IF last_node != object.assigned_target_node
    last_node = object.assigned_target_node
    current_substate = none                 # force a fresh selection
    RETURN
  IF current_substate == camp_in_cover AND object.location.level_vertex_id != last_node
    current_substate = none

FUNCTION reselect_state()
  IF object.location.level_vertex_id != last_node
    select(hide_in_cover)
  ELSE
    select(camp_in_cover)

FUNCTION setup_substates()
  IF current_substate == hide_in_cover
    fill move-to-point-extended with
      vertex          = last_node
      point           = level_graph.vertex_position(last_node)
      action          = run, aggressive acceleration, no braking
      completion_dist = 0                   # stand exactly on the cell
      time_to_rebuild = 0                   # re-path at the builder's own cadence
      time_out        = 200 seconds
      sound           = idle, with the creature's configured idle delay
  ELSE IF current_substate == camp_in_cover
    fill stand-and-act with
      action = stand idle
      sound  = idle, with the creature's configured idle delay
```

**Invariants** — the latched cell is the *only* memory this composite has, and it is
compared against the creature's live assignment every tick. That makes the composite
correct under an assignment that changes mid-approach, and it is why the camping branch
rechecks the creature's actual cell: camping is defined as *being on* the latched cell, not
as *having selected* camping.

**Notes** — the 200-second timeout is effectively "never", present so the move state has a
non-zero timeout at all. Exact arrival with continuous re-pathing is expensive; it is
affordable here because the destination rarely changes.

## `CStateMonsterTestState` — wander near the player

**Contract** — one sub-state, selected unconditionally: run to a random position within 20
metres of the player, clamped into the creature's permitted space by the nearest-accessible
query when the random point falls outside it, with a calm acceleration profile, a
3-metre arrival tolerance and a 20-second timeout.

**Dead.** The single line that would add it to a creature is commented out in the chimera's
behaviour tree. It is documented because it is the clearest example in the creature layer
of the "pick a random reachable point near the player" idiom, which a rebuild will want
somewhere.
