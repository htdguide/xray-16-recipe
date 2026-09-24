# src/xrGame/wrapper_abstract.h

> The adapter that lets a planner evaluator or operator written against a concrete creature be constructed from either the creature or its script facade.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`wrapper_abstract_inline.h`](wrapper_abstract_inline.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`smart_cover_evaluators.h`](smart_cover_evaluators.h.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`wrapper_abstract_inline.h`](wrapper_abstract_inline.h.md)
**Tier floor** — T2: a mixin that narrows a script-facing base to a concrete owner type

## Purpose

The planner's evaluators and operators come in two flavours: ones the engine instantiates
with a concrete creature, and ones a script instantiates with a *game object* — the
script-visible facade. Both must end up holding the same thing: the concrete creature, so
that the evaluator's condition can ask questions the facade does not expose.

This file is the one adapter that makes both entry points land in the same place. An
evaluator written on top of it declares the creature type it wants, inherits the
script-facing base, and gets a typed accessor to its owner regardless of which side
constructed it.

It is a header with no implementation file because it is a compile-time adapter; the
substance is the two setup paths, and they are here and in
[`wrapper_abstract_inline.h`](wrapper_abstract_inline.h.md).

## State

```text
RECORD AbstractWrapper                # parameterized by the concrete owner type
  object : optional<ref concrete owner>   # none until setup; every accessor requires it
  ...    : whatever the script-facing base carries
```

**Invariants** — the owner reference is absent between construction and setup, and every
accessor requires it to be present. A wrapper is therefore unusable until set up, and the
two-phase construction is deliberate: the planner builds its evaluator table before it knows
which creature the plan is for, then binds each entry once.

## `setup(concrete owner, property storage)`

**Contract** — bind the wrapper when the caller already holds the concrete creature. Passes
the creature's *script facade* down to the script-facing base, so that base sees what it
expects, and records the concrete creature here. Requires the creature to exist.

## `setup(script facade, property storage)`

**Contract** — bind the wrapper when the caller holds only the script facade, which is the
path a script-authored evaluator takes. Passes the facade straight down, then narrows it to
the concrete creature type and records that. Requires the facade to exist and the narrowing
to succeed — a script that attaches a stalker evaluator to a dog is a hard failure here, not
a silent no-op.

**Notes** — the narrowing is the whole point of the class. It is where "this evaluator only
makes sense for this kind of creature" is enforced, and it is enforced at bind time rather
than at evaluation time so the failure names the misconfiguration rather than the symptom.

## `object`

**Contract** — the concrete owner. Requires setup to have happened.

## Notes

A second, simpler form of the same adapter — one that takes no property storage — is present
in the source but commented out and unused. The property storage is the planner's world-state
blackboard; an evaluator without one would have nothing to read, so the simpler form was
likely a leftover from before the blackboard existed. A rebuild needs only the one form.

The adapter is expressed with C++ templates and a template-template parameter. What a rebuild
actually needs is: a generic mixin over (concrete owner type, script-facing base) that
supplies a typed, checked owner accessor and two constructors. Any mechanism that gives an
evaluator a statically typed owner while still being constructible from the dynamically typed
script side will do; the variadic constructor forwarding in the original is pure C++ noise.
