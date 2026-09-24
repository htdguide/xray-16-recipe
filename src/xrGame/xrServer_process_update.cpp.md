# src/xrGame/xrServer_process_update.cpp

> Applies a batch of per-entity state to the server's records, and detects the moment a sender and a receiver disagree about a record's shape.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: length-prefixed records parsed from a byte stream with exact-consumption checking

## Purpose

Two nearly identical readers, for the two directions state travels *into* the server: the live
update stream and the save stream. Both are a sequence of (identifier, length, payload) records,
and both rest on the same rule — **the length prefix is the contract, and the payload's own
reader must consume exactly it.**

That rule is what makes the format extensible without versioning: an unknown entity is skipped by
its length, and a reader that consumes the wrong amount is caught immediately rather than
corrupting everything after it.

## State

`Stateless.`

## `Process_update`

**Contract** — apply a batch of entity updates. **Accepted only from a client flagged local**,
which in practice means the host or the server itself: a remote client cannot rewrite entity
state directly. Marks each touched entity network-ready. Skips an unknown identifier by its
length. **Fails fatally if an entity's reader consumes a number of bytes other than the declared
length.**

```text
FUNCTION process_update(packet, sender)
  client := client_for(sender)          # must exist
  REQUIRE client is local

  WHILE NOT packet.at_end
    id   := packet.read_int(16-bit)
    size := packet.read_int(8-bit)
    start := packet.position
    entity := entities[id]

    IF entity EXISTS
      entity.net_ready := true
      entity.read_update(packet)
      IF packet.position - start != size
        FAIL WITH a fatal error naming the entity's class, the sender and the offsets
    ELSE
      packet.skip(size)
```

**Invariants** — the payload length is **one byte**, so an entity's update is capped at 255 bytes.
That is the same cap the writer in [`xrServer.cpp`](xrServer.cpp.md) imposes, and it is frozen.

The consumption check is exact, not a bound. Reading too little leaves the stream misaligned for
every following record; reading too much has already consumed somebody else's. Either is
unrecoverable, which is why it is a fatal error rather than a skip.

**Notes** — the fatal message names the entity's class identifier and the sending client, because
that is exactly the pair needed to find the bug: one class's writer and reader disagree, and the
sender tells you which build produced the bytes. The original's phrasing of this message is a joke
at the offending class author's expense; the *content* is what a rebuild should keep.

Requiring the sender to be local is the second half of the authority rule stated in
[`xrServer.cpp`](xrServer.cpp.md)'s message switch, where a spawn from a remote client is ignored.
Here the check is an assertion rather than a silent ignore, because a remote client sending updates
is a protocol violation rather than a plausible mistake.

Marking an entity network-ready on its first update is how an entity graduates from "spawned" to
"being simulated", and it is what the update writer's own readiness filter tests.

## `Process_save`

**Contract** — apply a batch of saved entity state. The same shape as the update reader with three
differences: the length prefix is **16 bits** rather than 8; the payload is read by the entity's
*save* reader rather than its update reader; and a consumption mismatch is **recovered from** by
seeking to where the record should have ended, with a warning.

```text
FUNCTION process_save(packet, sender)
  client := client_for(sender)          # must exist
  client.net_ready := true
  REQUIRE client is local

  WHILE NOT packet.at_end
    id   := packet.read_int(16-bit)
    size := packet.read_int(16-bit)
    start := packet.position
    entity := entities[id]

    IF entity EXISTS
      entity.net_ready := true
      entity.read_saved_state(packet)
    ELSE
      packet.skip(size)

    IF packet.position - start != size
      warn, naming the entity
      packet.seek(start + size)         # resynchronize and carry on
```

**Invariants** — the wider length prefix is the difference that matters: saved state per entity may
exceed 255 bytes where live update state may not. Live updates are bounded because they are sent
many times a second; saved state is written once.

**Notes** — **the two readers treat the same class of error oppositely, and both are right.** A
live update mismatch means the two sides' code disagrees and nothing downstream can be trusted; a
save mismatch means one entity's saved state is unreadable, and abandoning the whole save over one
entity is worse than losing that entity's state. So the update path dies and the save path
resynchronizes. A rebuild should reproduce the asymmetry, not unify it.

The save path's recovery also fires on the *unknown entity* branch, where the skip has already
advanced exactly the right amount and the seek is a no-op. Harmless.
