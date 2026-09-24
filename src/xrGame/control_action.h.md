# src/xrGame/control_action.h

> The base an animated-behaviour step derives from: a bound subject and five lifecycle hooks that all default to doing nothing.

**Needs** — [`control_action_inline.h`](control_action_inline.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`control_action_inline.h`](control_action_inline.h.md) · [`sight_action.h`](sight_action.h.md)
**Tier floor** — T3: an interface with default behaviour; the whole point is that it is trivially cheap to derive from

## Purpose

A humanoid's behaviour is built from small steps: raise the weapon, turn toward a sound, take
a step to the left, play a reaction. Each is a short-lived object that is set up, ticked until
it says it is done, and torn down. This is the base those steps share.

Its defining property is that **every hook has a do-nothing default**. A step that only needs
to run one thing at setup overrides exactly one hook. That matters because there are many of
these and the ceremony of implementing five methods to use one would dominate them.

Its second property is that the subject is *not* a constructor argument. A step is default-
constructed — typically as a member of the behaviour that owns it, or in an array of steps —
and bound to its subject afterwards. That is what lets a behaviour own its steps by value
rather than allocating each one.

## State

```text
RECORD CControlAction
  object : optional<reference to the humanoid this step acts on>   # none until bound
```

**Invariants** — the subject must be bound before any hook is called, and binding it is a
one-way operation asserted on a non-null subject. Reading the subject before binding is a
programming error, caught by an assertion rather than handled: a step with no subject has
nothing to do and the caller has skipped a required step.

## the five hooks

**Contract** — the interface a step implements, in the order the owner calls them:

```text
INTERFACE ControlAction
  FUNCTION applicable() -> bool    # default true.  May this step run at all right now?
  FUNCTION initialize()            # default nothing. Called once, before the first execute.
  FUNCTION execute()               # default nothing. Called repeatedly while running.
  FUNCTION completed() -> bool     # default true.  Has this step finished?
  FUNCTION finalize()              # default nothing. Called once, after the last execute.
```

**Invariants**

- `applicable` is asked **before** `initialize`; a step that is not applicable is skipped
  entirely and its setup and teardown never run.
- `initialize` and `finalize` are exactly paired for any step that ran.
- The defaults are chosen so that an unoverridden step is *applicable and immediately
  complete* — it runs once and disappears rather than blocking the behaviour forever. A base
  defaulting `completed` to false would deadlock every step that forgot to override it, which
  is the wrong failure.

## `remove_links`

**Contract** — tells the step that a particular game object is being destroyed and any
reference it holds to that object must be dropped. Defaults to nothing, because most steps
hold no object references.

**Invariants** — this is the lifetime contract for the whole behaviour layer: nothing may hold
a reference to a destroyed entity, and the notification reaches every level of the behaviour
tree. A step that caches an enemy, a cover point's owner or a target item **must** override
this. A rebuild using weak handles rather than raw references removes the whole obligation,
and should.

## `set_object` and `object`

**Contract** — bind the subject, and read it. Binding requires a real subject; reading
requires one to have been bound.

**Notes** — the subject's type is the humanoid class specifically, not a general entity. The
step hierarchy is not reusable for other creature kinds, which is a real limitation of the
original: the monster behaviour layer has its own parallel hierarchy.
