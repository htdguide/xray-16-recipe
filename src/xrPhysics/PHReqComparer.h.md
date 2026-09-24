# src/xrPhysics/PHReqComparer.h

> How one script-installed physics callback is recognized as "the same request" as another, so that installing it twice does not stack two of them.

**Needs** — [`PHScriptCall.h`](PHScriptCall.h.md)
**Used by** — [`PHCommander.cpp`](PHCommander.cpp.md) · [`PHCommander.h`](PHCommander.h.md) · [`PHScriptCall.h`](PHScriptCall.h.md) · [`PHSimpleCalls.h`](PHSimpleCalls.h.md)
**Tier floor** — T3: it is an identity test over a closed set of request kinds.

## Purpose

The physics world holds a list of standing script requests — "when this object sinks in liquid, play
these particles", "while this condition holds, apply this force". Scripts install them repeatedly,
often every time a state is re-entered, and the world must be able to answer *does an equivalent
request already exist* so the list does not grow without bound and the same particles are not
started twice.

Equivalence is not identity: two requests are the same request when they name the same object with
the same parameters, even though they are different records. Only the kind of request knows what
"same parameters" means, so the test is dispatched on the kind.

## the comparison protocol

**Contract** — a *comparer* is a question, carried to every candidate in the list, that answers
"are you the request I am looking for?" The candidate hands itself to the comparer, and the comparer
answers based on what kind of thing it received. A comparer that does not recognize a kind answers
no.

```text
INTERFACE RequestComparer
  # one entry per kind of standing request; every one defaults to NO
  FUNCTION compare(candidate : ScriptCondition)         -> bool
  FUNCTION compare(candidate : ScriptAction)            -> bool
  FUNCTION compare(candidate : ObjectCondition)         -> bool
  FUNCTION compare(candidate : ObjectAction)            -> bool
  FUNCTION compare(candidate : ObjectConditionN)        -> bool
  FUNCTION compare(candidate : ObjectActionN)           -> bool
  FUNCTION compare(candidate : GameObjectCondition)     -> bool
  FUNCTION compare(candidate : GameObjectAction)        -> bool
  FUNCTION compare(candidate : ConstForceAction)        -> bool
  FUNCTION compare(candidate : LiquidParticlesPlayCall) -> bool
  FUNCTION compare(candidate : LiquidParticlesCondition)-> bool
  FUNCTION compare(candidate : FindLiquidParticles)     -> bool
```

**Invariants** — **default no.** A comparer looking for liquid particles must not accidentally match
a constant-force action, so every question it did not come to ask answers no without being written.
This matters: the list is heterogeneous and a false match silently removes or reuses the wrong
request.

**Notes** — the set of kinds is closed and enumerated here in one place. That is the decision a
rebuild inherits: the request kinds are a fixed vocabulary owned by this module, not an open
extension point, because the comparison is the *cross product* of kinds and cannot be written by
either side alone. A rebuild with an open set would instead give each request a comparable
identity — a kind tag plus its parameters — and compare those, which is simpler and loses nothing:
no comparer in the shipped code compares across kinds.

The definitions of the kinds are in [`PHScriptCall.h`](PHScriptCall.h.md) and
[`PHSimpleCalls.h`](PHSimpleCalls.h.md); the list they live in is the physics world's, see
[`PHWorld.cpp`](PHWorld.cpp.md).
