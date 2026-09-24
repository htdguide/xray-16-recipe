# src/xrGame/item_manager.cpp

> Decides which of the items a creature can currently see is worth walking over to pick up.

**Needs** — [`item_manager.h`](item_manager.h.md) · [`object_manager.h`](object_manager.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`restricted_object.h`](restricted_object.h.md) · [`movement_manager.h`](movement_manager.h.md) · [`ai_space.h`](ai_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — reached through its declarations in [`item_manager.h`](item_manager.h.md); callers name that, not this file.
**Tier floor** — T3: a filter and a ranking over a small remembered set

## Purpose

A stalker that walks past a better rifle than the one in his hands looks stupid; a stalker
that walks across an anomaly field to reach a bandage looks worse. This file is the whole
of that judgement. It plugs a filter and a ranking function into the generic remembered-set
machinery, and the machinery does the rest: keep the set current, score every member, hand
the best one to the planner as "the item I want".

It is one file per *kind* of selection — items here, enemies and dangers elsewhere —
because the filter and the score are the only things that differ, and they are exactly what
a designer tunes.

## State

```text
RECORD ItemManager EXTENDS ObjectManager<GameObject>
  owner   : Creature        # the creature whose judgement this is
  stalker : optional<Stalker>   # the same object, when it is a stalker; none otherwise
```

**Invariant** — the stalker reference is resolved once, at construction, from the owner.
The two fields always denote the same object; the second exists only so the per-frame
filter can ask stalker-specific questions without a type test on every item.

**Invariant** — every object in the remembered set is reachable under the owner's current
restrictors. This is not merely established by the filter on entry: it must *remain* true,
which is why a change of restrictors re-checks the selection, and why the set is re-filtered
every update. The invariant is asserted on the whole set and on the selection each frame in
a development build, because a violation here surfaces much later as a creature walking into
a place it was forbidden.

## `useful`

**Contract** — the admission filter, run against every candidate item every update. Returns
whether the item may be in the remembered set at all. Pure; no side effects. An item that
fails is not remembered rather than remembered-and-ignored, so the set stays small.

The tests, in order — and the order is a cost ordering, cheapest and most-rejecting first:

```text
FUNCTION useful(item) -> bool
  IF the base machinery rejects it              THEN RETURN false
  IF the owner is being destroyed               THEN RETURN false
  IF the OWNER is attached to a parent          THEN RETURN false  # see note
  IF the item does not participate in navigation THEN RETURN false
  IF the item's POSITION is outside the owner's restrictors      THEN RETURN false
  IF the item's NAVIGATION VERTEX is outside them                THEN RETURN false
  IF the item is an inventory item the creature has no use for   THEN RETURN false
  IF the owner is a stalker AND (he cannot take this item
                                 OR its position is outside his restrictors)
                                                THEN RETURN false
  IF there is no level graph                    THEN RETURN false
  IF the item's navigation vertex is invalid    THEN RETURN false
  IF the item's position is not INSIDE that vertex               THEN RETURN false
  RETURN true
```

**Invariants** — reachability is checked twice over, once against the item's free position
and once against the navigation vertex it claims, because the two can disagree: an item
resting on the boundary of a forbidden volume has a vertex on one side and a centre on the
other. Both must be admissible, because the creature will path to the vertex and then reach
for the position.

The last test — position inside the claimed vertex — catches an item whose recorded
navigation vertex has gone stale, typically because it was dropped or thrown and has not
re-registered. Such an item is not merely mis-ranked; pathing to its vertex would put the
creature somewhere else entirely.

**Notes**

- The parent test rejects on the **owner** having a parent, not the item. An attached
  creature — riding, carried, being a turret's gunner — does not shop for items. The
  original's comment describes the opposite intent; the code is what ships and what the
  gameplay reflects.
- The "no use for this item" test is the designer's hook: it is a property of the item
  section, which is how a bandage is takeable and a quest document is not.
- The final three tests dereference the item as an inventory item without having confirmed
  it is one — a non-inventory object that reaches them is undefined. In practice nothing
  else survives the earlier tests, but a rebuild should hoist the type test.

## `evaluate`

**Contract** — the ranking. Returns a score for one item, **lower is better** — the
machinery selects the minimum. The score is a large constant minus the item's monetary
cost, so that the most valuable item wins.

```text
FUNCTION evaluate(item) -> real
  RETURN 1_000_000 - item.cost
```

**Notes** — the constant exists only to keep the result positive for every cost the shipped
data contains, because the machinery's sentinel for "nothing selected" is a comparison
against a large value rather than an absent marker. It is not a cap and no item approaches
it. A rebuild with a proper optional selection should rank by negated cost and drop the
constant entirely.

Cost is the *only* term. Distance, the creature's current needs, whether he already carries
one — none of it enters. A stalker will cross a room for a marginally pricier rifle. This
is the shipped behaviour and changing it changes how the game reads.

## `is_useful` · `do_evaluate`

**Contract** — the two override seams. Both forward the question to the owning creature,
passing this manager along so the creature can tell which selection is asking. That is how a
subclass or a script-driven creature substitutes its own judgement without replacing the
manager. `do_evaluate` additionally asserts, before delegating, that the item is still
reachable — the last point at which a stale set member can be caught before its score
influences the choice.

## `update`

**Contract** — runs one selection cycle: re-filter, re-rank, re-select. Delegates to the
base machinery; everything this file adds is assertion. Called on the creature's schedule,
not every frame.

## `remove_links`

**Contract** — forgets an object that is being destroyed. Removes it from the set by
identity, and clears the selection if it *was* the selection. Must be called before the
object's memory is released, from the global object-teardown notification, or the set holds
a dangling member.

**Invariant** — the selection is compared by entity identifier rather than by reference,
while the set is searched by reference. Both work here; the identifier comparison is the one
that would still be correct if the object had already been replaced.

## `on_restrictions_change`

**Contract** — called when the owner's restrictors change. Drops the current selection if it
is no longer reachable, by navigation vertex or by position; leaves the rest of the set
alone, since the next update re-filters it anyway. The selection is special-cased because it
is the one thing the planner may already have acted on.
