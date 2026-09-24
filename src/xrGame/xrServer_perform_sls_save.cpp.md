# src/xrGame/xrServer_perform_sls_save.cpp

> Writes the level's entire server-side state to a save: for each entity, the record that creates it and the record that brings it up to date.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: writes the frozen chunked container format with explicit widths

## Purpose

A save is the world, and the world is the entity table. This writes it, and the format it writes
is one of the chapter's frozen contracts: a save written by the original must load here and vice
versa.

The decision worth naming is that **a save reuses the network format**. An entity is saved as the
same spawn message a joining client would receive, followed by the same update message. There is
no separate persistence format, which is why adding a field to an entity's wire format changes the
save format too.

## State

`Stateless.`

## `SLS_Save`

**Contract** — write every entity in the table into the writer, one container chunk per entity,
numbered by iteration position. Each chunk holds two length-prefixed records: the spawn and the
update.

```text
FUNCTION save_level_state(writer)
  n := 0
  FOR EACH entity IN entities
    writer.open_chunk(n)

    spawn := entity.write_spawn(with_client_data: true)
    writer.write_u16(spawn.length)
    writer.write(spawn)

    update := begin(UPDATE)
              + entity.id (16-bit)
              + a one-byte-length-prefixed block written by the entity
    writer.write_u16(update.length)
    writer.write(update)

    writer.close_chunk()
    n := n + 1
```

**Invariants** — each record is prefixed with a **16-bit** length, while the update's inner
payload block carries a **one-byte** length. The two widths are both frozen and they differ for a
reason: the outer length must accommodate a full spawn record, the inner is the same per-entity
payload cap that the live update path uses, so the same serializer serves both.

The chunk numbering is **the table's iteration position**, not the entity identifier. It is
therefore whatever order the table happens to yield, which is the undefined order that
[`xrServer.h`](xrServer.h.md) warns about. The loader does not care about the number, only about
the sequence — but see the load twin: it *does* care about the order.

**Notes** — the client-data blob *is* written, unlike in the join replay where it is written once
and cleared. A save must carry it because there may be no client to have taken it yet.

The chunk index variable is written in the source as a shadowed name — the loop's entity shadows
the table's end iterator — which is a C++ hazard with no effect here and no meaning for a
rebuild.
