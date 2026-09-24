# src/xrPhysics/DamageSource.h

> Lets an object carry the identity of whoever set it in motion, so a thrown crate
> kills for the thrower.

**Needs** — _(none beyond the core types)_
**Used by** — [`Bolt.cpp`](../xrGame/Bolt.cpp.md) · [`Bolt.h`](../xrGame/Bolt.h.md) · [`Explosive.h`](../xrGame/Explosive.h.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md)
**Tier floor** — T3: two accessors behind an interface.

## Purpose

Damage attribution has to survive a chain of physical causes: the player throws a barrel,
the barrel knocks a crate, the crate crushes a creature. Physics cannot resolve intent, so
it propagates a single piece of data — the *initiator id* — along the chain, and this
interface is where an object exposes it.

## `IDamageSource`

**Contract** — set and read the initiator's object id, plus a self-cast that lets a holder
discover whether the object it is talking to participates in attribution at all (objects
that do not, such as world geometry, answer with nothing). The id is the no-object value
until something sets it.

**Notes** — the self-cast exists because the physics side holds objects only as shell
holders and must ask, at run time, whether this particular one also carries an initiator.
In a rebuild this is an optional capability lookup, not a cast.

Resolution walks exactly one link, and the fallback matters. When A strikes B, B asks A for
*A's* initiator — not for A itself — so the original thrower stays credited however long the
chain of intermediaries. If A carries no initiator, or A has already been destroyed by the
time B's impact is read, the credit falls back to **B itself**: an object hurt by an
unattributed physical cause counts as having hurt itself, which is what makes falling damage
and being crushed by scenery attribute correctly without a special case.
