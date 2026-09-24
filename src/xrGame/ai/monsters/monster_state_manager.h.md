# src/xrGame/ai/monsters/monster_state_manager.h

> The root of every creature's state tree: a state that is also the manager the engine drives, generic over the creature type.

**Needs** — [`state_manager.h`](state_manager.h.md) · [`state.h`](state.h.md) · [`monster_state_manager_inline.h`](monster_state_manager_inline.h.md)
**Used by** — [`bloodsucker_state_manager.h`](bloodsucker/bloodsucker_state_manager.h.md) · [`boar_state_manager.cpp`](boar/boar_state_manager.cpp.md) · [`boar_state_manager.h`](boar/boar_state_manager.h.md) · [`burer_state_manager.cpp`](burer/burer_state_manager.cpp.md) · [`burer_state_manager.h`](burer/burer_state_manager.h.md) · [`cat_state_manager.cpp`](cat/cat_state_manager.cpp.md) · [`cat_state_manager.h`](cat/cat_state_manager.h.md) · [`chimera_state_manager.cpp`](chimera/chimera_state_manager.cpp.md) · [`chimera_state_manager.h`](chimera/chimera_state_manager.h.md) · [`controller_state_manager.cpp`](controller/controller_state_manager.cpp.md) · [`controller_state_manager.h`](controller/controller_state_manager.h.md) · [`dog_state_manager.cpp`](dog/dog_state_manager.cpp.md) · [`dog_state_manager.h`](dog/dog_state_manager.h.md) · [`flesh_state_manager.cpp`](flesh/flesh_state_manager.cpp.md) · _and 10 more_
**Tier floor** — T3: a type parameterised over the creature, forwarding to a state tree

## Purpose

Every creature in chapter 24 has a brain shaped the same way: a tree of states, each with an
enter / run / completion contract, with a selector at each level re-evaluated every tick.
This declares the **root** of that tree, and it is the one node that is two things at once — a
state like every other node, and the manager object the engine's scheduler calls into.

That double identity is the file's only real decision and it is what makes the tree uniform:
because the root is itself a state, a concrete creature's manager can `add_state` children and
`select_state` among them with exactly the same vocabulary its children use on *their*
children, all the way down. There is no separate "top level" shape to learn.

A concrete creature supplies two things and inherits the rest: a constructor that registers its
children, and a selector — `execute` — that decides which child is current this tick. Everything
else on this page is the shared machinery.

## What the engine demands of a manager

The manager side of the identity is an interface the engine drives, and every method of it is
answered here by forwarding into the state side:

| Called by | Answered as |
|---|---|
| `reinit` | reset the whole subtree |
| `update` | the per-tick entry point — see [`monster_state_manager_inline.h`](monster_state_manager_inline.h.md) |
| `force_script_state(state)` | select that state directly, bypassing the selector |
| `execute_script_state` | run whatever is currently selected, without re-selecting |
| `critical_finalize` | tear down the current state immediately, out of band |
| `current_state_type` | which state is selected, as a value the script layer can read |
| `forget_entity` | clear destroyed-entity references throughout the subtree — **each creature must supply this**; it is the one method with no default |
| `may_start_control(type)` | whether a given movement/animation controller may take over now |

**Notes** — `force_script_state` and `execute_script_state` are deliberately separate: script
sets the state on one call and the engine runs it on the next, so a scripted state persists
across ticks without the selector overriding it.

`forget_entity` is left abstract even though the obvious implementation — forward to the base —
is what every concrete creature writes. The intent is to force each creature's author to
consider whether its manager holds entity references of its own beyond the tree's.

## Shared selector helpers

Two predicates every concrete selector uses, provided here so the selectors stay short.

### `can_eat`

**Contract** — whether the creature should go and eat: it has a corpse selected, and the eating
state's own start-or-continue test passes.

### `should_run(state_id)` — the start-or-continue rule

**Contract** — the single most repeated decision in chapter 24, and the reason it is worth a
name. A state is run this tick if **either** it was running last tick and has not declared
itself finished, **or** it was not running and its start conditions are met.

```text
FUNCTION should_run(state_id) -> bool
  IF previous_substate = state_id
    RETURN NOT current_state.finished()      # continue until it says it is done
  ELSE
    RETURN state(state_id).may_start()       # otherwise, ask whether it may begin
```

**Invariants** — a running state is never re-asked whether it *may start*, and a state that is
not running is never asked whether it is *finished*. That asymmetry is what gives states
hysteresis: a state may be hard to enter and easy to stay in, or the reverse, and the two are
tuned independently. Every selector in chapter 24 is a chain of these tests, so getting this
rule right is most of getting the creature brains right.

**Notes** — the "continue" branch calls `finished` on the *currently selected* state rather
than on the state being asked about. Those are the same thing only because the branch is
guarded by the equality above; a rebuild should still ask the named state, which is clearer and
identical in effect.
