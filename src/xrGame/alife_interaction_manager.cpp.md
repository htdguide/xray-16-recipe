# src/xrGame/alife_interaction_manager.cpp

> Where two off-screen entities meeting on the world graph would have been made to fight, trade or touch a smart terrain. Disabled; only the constructor remains.

**Needs** — [`alife_interaction_manager.h`](alife_interaction_manager.h.md) · [`alife_combat_manager.h`](alife_combat_manager.h.md) · [`alife_communication_manager.h`](alife_communication_manager.h.md)
**Used by** — [`alife_interaction_manager.h`](alife_interaction_manager.h.md)
**Tier floor** — T3: an index walk and a dispatch.

## Purpose

The off-screen simulation's *meeting* pass: for each scheduled entity, look at everything on
its graph vertex and on every adjacent vertex, decide whether an interaction happens, and
run it. This is the layer that joined the combat model
([`alife_combat_manager.cpp`](alife_combat_manager.cpp.md)) to the trading model
([`alife_communication_manager.cpp`](alife_communication_manager.cpp.md)); with both of
those disabled, it is disabled too.

## `CALifeInteractionManager` construction

**Contract** — Joins the two disabled layers over one shared simulation base, constructing
the base once and each layer against it. That single construction of the shared base is the
only live decision in the file: both layers inherit it virtually so there is exactly one
simulation state, and this constructor is the one place that says so.

**Notes** — The disabled setup read the player inventory's slot count from configuration and
sized two scratch buffers from it, one of them large enough to mark every possible entity
identifier. A rebuild sizing a mark array by the identifier space is buying 64 kilobytes to
avoid a set; at 16 bits that was a reasonable trade in 2004 and is a poor one now.

## The disabled interaction pass

```text
FUNCTION check_for_interaction(entity)
  IF the entity is not active THEN RETURN
  check_for_interaction(entity, its own graph vertex)
  FOR EACH neighbouring graph vertex
    check_for_interaction(entity, that vertex)

FUNCTION check_for_interaction(entity, vertex)
  FOR EACH object indexed at that vertex          # mutation-safe iteration
    skip it if it was already visited this pass   # see the visit counter below
    skip the entity itself
    skip anything that is not schedulable
    IF no interaction is detected between them THEN CONTINUE
    SWITCH what the detecting side decides to do
      CASE attack        : run the combat loop, then settle the aftermath
      CASE interact      : both must be human; run the trade
      CASE ignore        : nothing
      CASE smart terrain : tell the smart terrain the creature touched it
```

**Invariants** — Interactions are checked against the entity's own vertex **and every
adjacent one**. Two entities on neighbouring vertices are close enough to meet, because a
graph vertex is a place-sized region, not a point. That doubles the cost and is the reason
the pass needs a visit counter at all.

Each object carries a *switch counter* stamped with the current pass's identifier; an object
already stamped is skipped. That is what stops the same pair from interacting twice when
both are reachable from the other's neighbourhood. A rebuild needs the same idea — a
per-pass visited mark carried on the object, not a set, because the iteration is
mutation-safe and a set would go stale.

### The combat loop

```text
result = "both retreated"
FOR at most twice the configured maximum iteration count
  IF the current side chooses to attack
    perform one round of attacks
    clear the "other side also declined" flag
  ELSE
    IF the other side already declined THEN BREAK        # neither will fight
    IF the retreat roll succeeds
      result = "this side retreated"; BREAK
    set the "declined but could not retreat" flag
  swap sides
  IF the now-current side has no members left
    result = "the other side killed this one"; BREAK
settle the aftermath with the result
```

**Invariants** — The iteration bound is twice the configured count because the loop swaps
sides every pass, so each side gets the configured number of turns. Both sides declining in
succession ends the fight without a winner — that is what the initial *both retreated*
result means, and why the decline flag is cleared on every attack rather than only on entry:
one side attacking makes the other's earlier refusal stale.

A side that chooses to retreat but fails its retreat roll does not get a free turn; it is
marked and play passes to the other side, so the failed retreat costs it a turn. That is the
whole cost of trying to run away.

### Smart terrain

**Contract** — The one branch that is not symmetric: a creature meeting a smart terrain is
simply announced to the terrain, which decides what job to hand out. No combat, no trade.
This is how the shipped games actually populate the world with purposeful behaviour, and it
is the only branch whose *live* equivalent exists elsewhere in the engine.
