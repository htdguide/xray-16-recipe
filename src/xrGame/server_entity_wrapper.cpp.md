# src/xrGame/server_entity_wrapper.cpp

> Writes and reads one server object as a self-contained two-chunk record, by replaying the network spawn and update messages into a file.

**Needs** — [`server_entity_wrapper.h`](server_entity_wrapper.h.md) · [`xrServerEntities/xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: byte-exact message framing written into a chunked container

## Purpose

A server object already knows how to describe itself twice over — once as a *spawn*
message (identity, class, everything needed to create it) and once as an *update* message
(the mutable state). Those two descriptions are the wire format. This file's whole idea is
that a file is just a wire that is slow: to persist one entity outside a save game, write
its spawn message into one chunk and its update message into another, and to restore it,
replay both.

The payoff is that no separate on-disk schema exists for server objects. There is one
serialization contract, used by the network, by saves and by this; a change to an entity's
fields updates all three at once. The cost is that the file format inherits the network
format's frozen shape, including its 16-bit length prefix.

## State

```text
RECORD ServerEntityWrapper
  object : optional<ServerObject>    # owned: destroyed with the wrapper
```

**Invariants** — after a successful load the object is present and has had both its spawn
and its update applied, in that order. The wrapper owns it outright; there is no detach.

## `save`

**Contract** — writes two chunks into the given stream. Does not block beyond the stream,
allocates a message buffer on the stack.

```text
FUNCTION save(stream)
  chunk 0:
    packet = object.write_spawn(as_for_a_client = true)
    stream.write_int_16(packet.length)      # length prefix: 16-bit, the message cap
    stream.write_bytes(packet.bytes)

  chunk 1:
    packet = new message with header "update"
    object.write_update(packet)
    stream.write_int_16(packet.length)
    stream.write_bytes(packet.bytes)
```

**Invariants** — the spawn packet already carries its own message header (the spawn writer
puts it there); the update packet does not, so this code opens one explicitly. That
asymmetry is real and the load side depends on it: both chunks are read back expecting a
header, so the update chunk would be unreadable without the explicit one.

The length prefix is 16 bits, which caps a single entity's spawn or update record at
65535 bytes. That cap is inherited from the network message size and is not checked here —
an entity whose state exceeded it would silently truncate.

## `load`

**Contract** — reads the two chunks back and rebuilds the object. Fails loudly at each
step rather than producing a partly-built entity.

```text
FUNCTION load(stream) -> ServerObject
  chunk = stream.open_chunk(0)
  packet.length = chunk.read_int_16()
  packet.bytes  = chunk.read_bytes(packet.length)
  chunk.close()

  header = packet.read_header()
  REQUIRE header IS "spawn"                  # FAIL WITH "invalid packet id"

  class_name = packet.read_text()            # the first field of every spawn message
  object = entity_factory.create(class_name) # FAIL WITH "can't create entity"
  object.read_spawn(packet)                  # continues from where the name left off

  chunk = stream.open_chunk(1)
  packet.length = chunk.read_int_16()
  packet.bytes  = chunk.read_bytes(packet.length)
  chunk.close()

  header = packet.read_header()
  REQUIRE header IS "update"
  object.read_update(packet)
```

**Invariants** — the order is load-bearing and is the same order the network uses: the
class name is read *before* the object exists, because it is what selects the constructor;
the rest of the spawn record is read *into* the constructed object, continuing from the
same read cursor; and the update is applied only after the spawn is complete, because
update fields may be interpreted in terms of state the spawn established.

**Notes** — the object created here is destroyed by the wrapper's destructor through the
same factory that made it, so creation and destruction stay symmetric even though the
concrete class is known only as a string.

## `save_update` and `load_update`

**Contract** — declared, and do nothing. An abandoned third chunk that would have carried
a lighter-weight periodic update separate from the full state. Nothing calls them and
nothing writes the chunk they would have read. A rebuild should omit them.
