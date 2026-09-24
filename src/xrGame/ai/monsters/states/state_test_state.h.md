# src/xrGame/ai/monsters/states/state_test_state.h

> Declares two composite states written as harnesses; the cover one ships as the snork's enemy-search behaviour, the other is disabled everywhere.

**Needs** — [`state.h`](../state.h.md) · [`state_test_state_inline.h`](state_test_state_inline.h.md)
**Used by** — [`snork_state_manager.cpp`](../snork/snork_state_manager.cpp.md) · [`state_test_state_inline.h`](state_test_state_inline.h.md)
**Tier floor** — T3: two declarations

## Purpose

Declares the surface implemented in
[`state_test_state_inline.h`](state_test_state_inline.h.md). Both are *composite* states:
they own sub-states and choose between them, rather than driving the creature directly.

| State | Status |
|---|---|
| run to a random spot near the player | dead — the only line that added it is commented out |
| take the assigned cover and camp in it | **live** — the snork's behaviour tree selects it as its enemy-search state |

## State

```text
RECORD TestCoverState
  last_node : MeshVertexId   # the cover cell this state committed to on entry
```

## Exported units

- **the wander-near-player harness** — one sub-state, reselected unconditionally.
- **the take-cover-and-camp state** — two sub-states, a forced-reselection hook and a
  selection rule that keys entirely off whether the creature is standing on its assigned
  cover cell.
