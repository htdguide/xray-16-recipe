# src/xrGame/action_script_base_inline.h

> Resolves the script-visible game object handed to an engine action back into the concrete client object the action actually needs.

**Needs** — [`action_script_base.h`](action_script_base.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`xrServerEntities/smart_cast.h`](../xrServerEntities/smart_cast.h.md)
**Used by** — [`action_script_base.h`](action_script_base.h.md)
**Tier floor** — T2: a downcast per installation; the cast itself is the incidental part

## Purpose

One decision, made twice: **the facade and the concrete object are derived from each
other, not passed separately.** A caller constructing one of these actions supplies the
concrete creature; the base's facade binding is obtained from it. Conversely the planner
installs the action by handing it the facade; the concrete object is recovered from it.
Either direction, the two bindings can never disagree — which is the invariant this file
exists to hold.

## State

Adds one field to the base action: the concrete object, alongside the base's facade
binding for the same entity.

Invariant: both bindings name the same entity at all times, or both are absent.

## Construction

**Contract** — takes the concrete object, optionally with an initial precondition and
effect set, and a diagnostic name. Derives the base's facade binding from the object, or
leaves it absent when no object was given, and records the concrete object. Does not
allocate.

```text
FUNCTION construct(object, preconditions, effects, name)
  base.construct(facade_of(object) IF object present ELSE none, preconditions, effects, name)
  concrete_object = object
```

## `setup`

**Contract** — two entries into the same operation. The planner calls the facade-taking
one; it runs the base's setup, then recovers the concrete object from the facade and calls
the typed one, which subclasses override. The typed one alone rebinds the concrete object
and, notably, does *not* rebind the world-state storage — the base already did.

```text
FUNCTION setup(facade, storage)
  REQUIRE facade present
  base.setup(facade, storage)                    # binds the facade and the storage
  setup(facade.underlying_object AS concrete type, storage)

FUNCTION setup(object, storage)                  # subclasses override this one
  REQUIRE object present
  concrete_object = object
```

**Invariants** — the recovery is a *checked downcast that is not checked here*: the
facade's underlying object is narrowed to the action's expected type, and installing an
action on a creature of the wrong type yields an absent binding rather than a diagnosed
error. Every concrete action's own setup then asserts the binding is present, so the
failure surfaces one level down. A rebuild should reject the mismatch at installation.

**Notes** — the two overloads share a name and are separated only by parameter type, which
is why the facade-taking one can call the object-taking one without recursion. In a
rebuild they are two differently named operations and the subtlety disappears.
