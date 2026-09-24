# src/xrGame/PHDestroyableNotificate.cpp

> A freshly spawned piece of debris reporting back to the object it broke off from.

**Needs** — [`PHDestroyableNotificate.h`](PHDestroyableNotificate.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`Level.h`](Level.h.md) · [`xrServerEntities/xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one registry lookup and a callback

## Purpose

The other half of the destruction handshake described in
[`PHDestroyable.cpp`](PHDestroyable.cpp.md). A piece of debris is spawned as an independent
entity carrying, in its server record, the identity of the object it came from. On arriving
in the world it must find that object and announce itself, so the original can count down its
outstanding pieces and eventually vanish.

## State

`Stateless.` The back-reference lives in the piece's server record, not here.

## `spawn_notificate`

**Contract** — called once as the piece comes online, with its own server record. Resolves
the recorded source identity to a live object, tells it a piece has arrived, and then
**clears the recorded source** so the announcement can never happen twice.

```text
FUNCTION spawn_notificate(own_server_record)
  source_id = own_server_record.source_id, if the record carries one
  IF source_id names something
    source = level.objects.find(source_id)
    IF source can receive destruction notifications
      source.on_piece_arrived(this)
  own_server_record.source_id = none     # consumed: a piece announces itself once
```

**Invariants** — clearing the source is what makes the announcement one-shot, and it matters
because the same record is written back into a save. A piece reloaded from a save must not
announce itself to an object that has long since been removed.

An unresolvable source is not an error. The original may already be gone — destroyed
through another path, or on a different level — and a piece with no parent to report to is
simply debris.

**Notes** — the record is cleared unconditionally at the end, including on the path where it
was not a skeleton record at all. Reaching that path would fail; nothing in the tree calls
this with any other kind of record, so the hole is unreachable, but a rebuild should guard
it.
