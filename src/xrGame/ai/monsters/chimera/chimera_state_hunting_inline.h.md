# src/xrGame/ai/monsters/chimera/chimera_state_hunting_inline.h

> An abandoned design for ambush hunting, preserved as a shell: alternate between hiding and emerging, forever.

**Needs** — [`chimera_state_hunting.h`](chimera_state_hunting.h.md) · [`chimera_state_hunting_move_to_cover.h`](chimera_state_hunting_move_to_cover.h.md) · [`chimera_state_hunting_come_out.h`](chimera_state_hunting_come_out.h.md)
**Used by** — [`chimera_state_hunting.h`](chimera_state_hunting.h.md)
**Tier floor** — T3: selection only

## Purpose

The intended behaviour is legible from the shape even though none of it is implemented: a chimera that stalks by moving from cover to cover, emerging between moves, rather than by circling in the open as the shipped attack does. What survives is the alternation and nothing else.

**This subtree is not built.** Its two substate implementations collide — [`chimera_state_hunting_come_out_inline.h`](chimera_state_hunting_come_out_inline.h.md) defines the *move to cover* state a second time rather than the *come out* state — so a translation unit that pulled this tree in would not compile. No translation unit does. A rebuild should treat the whole hunting subtree as a design note, not as code to port.

## State

Stateless.

```text
SUBSTATES
  move_to_cover
  come_out
```

## `reselect_state`

**Contract** — Alternates. The first selection is *move to cover*; thereafter, whatever ran last, the other one runs next — except that anything other than *move to cover* leads back to *move to cover*, so the cycle is cover, out, cover, out.

```text
FUNCTION reselect_state()
  IF previous == none            select(move_to_cover)
  ELSE IF previous == move_to_cover  select(come_out)
  ELSE                           select(move_to_cover)
```

## `check_start_conditions` / `check_completion`

**Contract** — Always startable, never finished. Entering this state would trap the creature in it until the state manager selected something else from above.
