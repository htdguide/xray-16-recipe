# src/xrGame/xrServer_balance.cpp

> Chooses which client should take over simulating an entity — and, as shipped, always answers "the host".

**Needs** — [`xrServer.h`](xrServer.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: returns one reference

## Purpose

In the architecture this engine was designed for, simulating an entity is *work*, and that work
is distributed: a client near an entity simulates it and tells the server what happened. Moving
an entity's simulation from one client to another is *migration*, and this file is where the
destination is chosen.

As shipped the answer is always the host's own client, so **migration is effectively off** and
the server simulates everything. The file is worth a page because the decision it was meant to
make is a real one a rebuild must face, and the abandoned alternatives are in the source.

## State

`Stateless.`

## `SelectBestClientToMigrateTo`

**Contract** — given an entity and a flag demanding a destination other than its current owner,
return the client that should simulate it. **Always returns the host's own client.**

The two policies the source abandoned:

```text
# what was intended, as recorded in the source
FUNCTION select_best(entity, force_another) -> client
  IF force_another
    RETURN the first connected client that is not the entity's current owner, or none
  ELSE IF entity has an owner
    RETURN that owner                     # no reason to move it
  ELSE
    RETURN a uniformly random connected client
```

**Notes** — both abandoned policies are labelled *dumb* in the source, and they are: neither
consults distance to the entity, the candidate's measured latency, or how much that candidate is
already simulating. Those three are what a real policy would weigh, and the fact that they were
never written is why the whole scheme collapsed to "the host does everything".

The force-another flag exists for one case: the current owner is leaving, so the entity must move
regardless of whether moving is otherwise a good idea. A rebuild implementing migration needs
that distinction — evacuate versus rebalance — even if it weighs candidates differently.

**A rebuild should think hard before restoring distributed simulation at all.** It buys server
CPU and costs authority: a client that simulates an entity can lie about it, and every
anti-cheat measure elsewhere in the chapter exists because of that. The shipped behaviour —
server simulates, clients report input — is the modern answer, and the original arrived at it by
abandoning this file rather than by deciding.
