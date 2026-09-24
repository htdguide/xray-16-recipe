# src/xrGame/wrapper_abstract_inline.h

> The two bind paths of the evaluator wrapper: from the concrete creature down to the facade, and from the facade up to the concrete creature.

**Needs** — [`wrapper_abstract.h`](wrapper_abstract.h.md) · [`script_game_object.h`](script_game_object.h.md)
**Used by** — [`wrapper_abstract.h`](wrapper_abstract.h.md)
**Tier floor** — T2: a checked downcast and two field assignments

## Purpose

Separated from the declaration for C++ compilation reasons; fold into the type. The
contracts are in [`wrapper_abstract.h`](wrapper_abstract.h.md); what is worth stating here
is the asymmetry between the two directions.

## State

`Stateless.` — see [`wrapper_abstract.h`](wrapper_abstract.h.md).

## The two directions

```text
# engine side: we have the creature, the base wants the facade
FUNCTION setup(concrete_owner, storage)
  REQUIRE concrete_owner EXISTS
  base.setup(concrete_owner.script_facade, storage)
  this.object := concrete_owner

# script side: we have the facade, we want the creature
FUNCTION setup(facade, storage)
  REQUIRE facade EXISTS
  base.setup(facade, storage)
  this.object := narrow(facade.underlying_object, to the concrete owner type)
  REQUIRE this.object EXISTS          # the narrowing must succeed
```

**Notes** — Every creature has a script facade, so the first direction cannot fail. Not
every facade wraps the creature this evaluator wants, so the second can — and it is checked
rather than tolerated. The consequence for a rebuild: the facade must expose the underlying
object in a form that can be tested against a concrete type, which constrains the facade to
be a wrapper over the object rather than a copy of its published fields.

The accessor requires the owner to be present, which is the same two-phase-construction
guard as everywhere else in the planner: build the table, then bind.
