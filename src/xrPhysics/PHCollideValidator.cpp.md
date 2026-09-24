# src/xrPhysics/PHCollideValidator.cpp

> The mutable half of collision filtering — the group counter, the class masks,
> and the named mutators that set an object's bits.

**Needs** — [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`PHObject.h`](PHObject.h.md) · [`ICollideValidator.h`](ICollideValidator.h.md)
**Used by** — [`PHCollideValidator.h`](PHCollideValidator.h.md)
**Tier floor** — T2: a counter and a set of bit assignments.

## Purpose

The decision rule lives in [`PHCollideValidator.h`](PHCollideValidator.h.md); this file
holds only what has to be mutable: the masks that say which bits are class bits and which
are refusal bits, the counter that mints group identifiers, and the named setters.

Read the header first — this page is not comprehensible on its own, and that is a property
of the split rather than of the writing.

## State

```text
MODULE STATE                       # one set, world-wide
  next_group   : int               # monotonically increasing; reset per world
  class_mask   : bit set           # every "is a" bit
  refusal_mask : bit set           # every "refuses" bit
  group_mask   : bit set           # just the in-group bit
```

**Invariants** — the three masks are derived at initialisation from the bit layout, not
written down twice. A rebuild that hard-codes them will silently drop a class when one is
added: the class will still be settable and will simply never filter anything.

## `Init`

**Contract** — resets the group counter to zero and fills the three masks. Called once per
world, at world creation.

**Invariants** — resetting the counter at world creation is what makes group identifiers
per-level rather than per-process. Objects surviving a level change must be re-registered;
one carrying a stale identifier from the previous level will be found to share a group with
an unrelated object, and the two will silently pass through each other. This is the strongest
argument for treating a group as a handle obtained from the world rather than a bare number.

## `RegisterGroup` / `LastGroupRegistred`

**Contract** — mint the next identifier and return it; or return the one most recently
minted. Identifiers are dense, increase by one, and are never released.

**Notes** — the "last registered" form exists so a composite object can mint one group and
then add its parts without carrying the identifier around. It is convenient and it is
fragile: anything that mints a group between the two calls silently reparents the parts. A
rebuild should pass the identifier explicitly and delete this call.

## `InitObject`

**Contract** — the default identity every physics object starts with: class bits cleared,
the *dynamic* class set, group identifier zero and not in a group.

**Invariants** — "dynamic" is the default class, so an object that nobody classifies collides
with everything that does not refuse dynamics. That default is why the mutators are all
phrased as restrictions: the model is permissive and each call takes something away.

## the mutators

**Contract** — one call each. `RegisterObjToGroup` and `RegisterObjToLastGroup` set the
group identifier and mark the object grouped; the group must already have been minted, which
is asserted. `SetStaticNotCollide` forbids the level. `SetNonDynamicObject` removes the
dynamic class. The remaining pairs set membership of, or refusal of, the character, ragdoll,
small and animated classes. `IsGroupObject` and `IsAnimatedObject` read two of the bits back.

**Notes** — `RestoreGroupObject` does nothing. It is the remaining half of a removed
mechanism for taking an object out of a group and putting it back, and its call sites still
exist. A rebuild should delete it and the calls; nothing depends on it having an effect.

The free function at the end of the file is the whole of
[`ICollideValidator.h`](ICollideValidator.h.md) — one forwarding call, so that the game layer
can mint a group without seeing the bit model.
