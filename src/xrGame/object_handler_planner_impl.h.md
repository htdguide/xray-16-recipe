# src/xrGame/object_handler_planner_impl.h

> How one 32-bit word names both an item and a property: the packing, and the taking-apart.

**Needs** — [`object_handler_planner.h`](object_handler_planner.h.md) · [`object_handler_space.h`](object_handler_space.h.md)
**Used by** — [`object_handler_planner.cpp`](object_handler_planner.cpp.md) · [`object_handler_planner_missile.cpp`](object_handler_planner_missile.cpp.md) · [`object_handler_planner_weapon.cpp`](object_handler_planner_weapon.cpp.md)
**Tier floor** — T3: bit packing over an entity identifier

## Purpose

The object-handling planner plans over *every item the creature holds at once*, so its world
state needs a distinct property per item — "this rifle is loaded" and "that pistol is loaded"
must be different facts. This file is the trick that makes that cheap: the property
identifier is one word holding the entity identifier in its high half and the property name in
its low half.

It is a separate header from the other inline file only because it must be included after the
world-property enumeration, which the planner's own header cannot do. The split is incidental.

## State

`Stateless.`

## `uid` — the packing

**Contract** — combines an entity identifier and a property or operator name into one
identifier. Asserts the two do not overlap before combining.

```text
FUNCTION uid(entity, name) -> int (32-bit)
  REQUIRE ((entity shifted left 16) AND name) == 0   # the halves must not collide
  RETURN (entity shifted left 16) OR name
```

**Invariants** — the entity identifier's 16-bit width is exactly the high half, and the
property enumeration must therefore stay under 65,536 values — it uses about forty. A rebuild
widening the entity identifier must widen this word, and one representing properties as
strings loses the constant-time packing this whole planner is built on.

The all-ones entity identifier is reserved: it is the "no item" pseudo-entity that the
sentinels in [`object_handler_space.h`](object_handler_space.h.md) are built from, so a real
entity may never have it. That is already true of the engine's identifier allocation.

**Notes** — the assertion is written against the *arguments as named in the header*, which are
declared in the opposite order from the definition's parameter names. The check is therefore
performed on the shifted first argument against the second, which is the intended test; the
name mismatch between declaration and definition is a readability defect with no behavioural
consequence.

## The unpacking

**Contract** — `action_object_id` takes the high sixteen bits as an entity identifier;
`action_state_id` takes the low sixteen as a property or operator name. `current_action_object_id`
and `current_action_state_id` apply those to the planner's currently running operator.
`object_action(identifier, object)` answers whether an identifier belongs to a given object,
by comparing the high half to its identifier.

**Notes** — `current_action_state_id` is the query the object handler's sling-state logic
leans on: "which of the four sling operators is running right now" is answered by unpacking
the running operator's identifier and comparing the low half against four names.

## `add_condition` / `add_effect`

**Contract** — attach a precondition, or an effect, to an operator, naming a property of one
particular item. Both are thin wrappers that pack the identifier and hand the resulting
(property, value) pair to the generic planner.

**Invariants** — every precondition and effect in the whole object-handling operator set goes
through these two, which is what guarantees no operator can accidentally reference another
item's state. A rebuild should keep the packing behind a pair of functions for the same
reason.
