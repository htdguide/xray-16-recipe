# src/xrGame/xrServer_svclient_validation.cpp

> Answers whether an entity the server knows about is actually a live, usable object on the client that simulates it.

**Needs** — [`xrServer_svclient_validation.h`](xrServer_svclient_validation.h.md) · [`GameObject.h`](GameObject.h.md) · [`Level.h`](Level.h.md)
**Used by** — [`xrServer_svclient_validation.h`](xrServer_svclient_validation.h.md)
**Tier floor** — T2: a registry lookup and three state tests

## Purpose

A **server object** and a **client object** are two different things with the same identifier, and
their lifetimes do not coincide. The server's record exists from the spawn message; the client's
live instance exists from when the client got around to constructing it, and persists — marked
dead — for a while after it is logically gone.

The ownership path needs to know about the *client* half, because attaching an item to a parent
whose client object is dead or gone produces an item that exists in the ownership tree and nowhere
the player can see. This is that question, asked from the server side about the host's client
registry, which single player makes cheap because both halves are in one process.

## State

`Stateless.`

## `is_object_valid_on_svclient`

**Contract** — given an entity identifier, answer whether the host client has a live game object
for it. Four ways to answer no:

```text
FUNCTION valid_on_simulating_client(entity_id) -> bool
  object := client_object_registry.find_by_network_id(entity_id)
  IF object is absent                THEN RETURN false   # never constructed, or already gone
  IF object is not a game object     THEN RETURN false   # a bare engine object, no game state
  IF object is marked for destruction THEN RETURN false  # dying this frame
  IF object has been removed          THEN RETURN false  # logically gone, record lingering
  RETURN true
```

**Invariants** — the last two are distinct states and both must be tested. *Marked for destruction*
is the engine's own deferred-deletion flag, set this frame and acted on at the frame's end;
*removed* is the game layer's flag, set when an object leaves play but its record is still needed
— the window in which an entity is being taken out of the world. An object in either state must not
become anybody's parent or child.

**Notes** — **the existence of this function is an admission that the server's entity table and the
client's object registry can disagree.** They can, and that is not a bug: they are populated by
different code at different times, and the gap is exactly the spawn-to-construction latency. What
would be a bug is acting as if they agreed, and every caller of this predicate is a place where the
original found that out.

It is only meaningful for the *host's* client, because that is the only registry this process has.
For a remote client the server has no way to ask, which is a real gap in the model — a remote
client's parent could be equally invalid and nothing would catch it. That gap is invisible in the
shipped configuration, where the host simulates everything.

The narrowing from a bare engine object to a game object is not a type-safety formality: engine
objects exist that have no game-layer state at all, and an ownership relation between them is
meaningless.
