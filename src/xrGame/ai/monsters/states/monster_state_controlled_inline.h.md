# src/xrGame/ai/monsters/states/monster_state_controlled_inline.h

> Being someone else's puppet: two behaviours, follow and attack, chosen by a task another creature writes onto this one — with a validity check that quietly converts an impossible order back into "stay with me".

**Needs** — [`monster_state_controlled.h`](monster_state_controlled.h.md) · [`monster_state_controlled_attack.h`](monster_state_controlled_attack.h.md) · [`monster_state_controlled_follow.h`](monster_state_controlled_follow.h.md)
**Used by** — [`monster_state_controlled.h`](monster_state_controlled.h.md)
**Tier floor** — T3: a two-way dispatch with one guard

## Purpose

The puppet mechanic's receiving half. What makes this interesting is *where the decision comes
from*: nothing in this creature chooses. A controller writes a task and a target onto the
creature's controlled-entity facet, and this state reads them. The creature's own perception,
morale and memory are bypassed entirely for as long as the state runs.

## `execute`

**Contract** — dispatches on the written task, runs the chosen child, records it as previous.

```text
FUNCTION execute()
  CASE creature.controlled_data.task OF
    follow:
      child = follow
    attack:
      target = creature.controlled_data.object
      IF target IS none OR target is being destroyed OR target is dead
        creature.controlled_data.object = my controller     # rewrite the order in place
        child = follow
      ELSE
        child = attack
    otherwise:
      unreachable

  select_state(child)
  current_child.execute()
  previous_child = current_child
```

**Invariants**

- **The order is validated on every tick, not on entry.** A puppet ordered to attack something
  that then dies does not sit idle or return to its own brain — it falls back to following, on
  the same tick.
- **The fallback rewrites the shared data rather than merely branching.** The target field is
  overwritten with the *controller itself*, which is what makes the follow child work: that
  child follows whatever the target field names. So "follow" and "attack" are the same
  mechanism pointed at different objects, and the fallback is a single assignment rather than a
  second code path. That is the load-bearing trick on this page.
- **There is no default case.** A task value outside the two is a programming error and fails
  hard. Since the only writer is the controller, the enumeration is closed in practice.
- **Unlike most containers in the chapter, this one selects unconditionally every tick** rather
  than deferring to `reselect_state`. Selection is idempotent, so re-asserting the same child
  costs nothing, and the effect is that the task is re-read every tick with no latency.

**Notes** — the creature under control keeps running its own brain's outer selector, which is
what re-selects this state each tick; see for instance
[`../pseudogigant/pseudogigant_state_manager.cpp`](../pseudogigant/pseudogigant_state_manager.cpp.md),
where being controlled is the first branch and shortcuts everything else.
