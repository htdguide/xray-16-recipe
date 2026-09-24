# src/xrGame/xrServerMapSync.cpp

> Answers a joining client's claim about which level it has, and whether its copy is the same one.

**Needs** — [`xrServerMapSync.h`](xrServerMapSync.h.md) · [`xrServer.h`](xrServer.h.md) · [`Level.h`](Level.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`xrServerMapSync.h`](xrServerMapSync.h.md)
**Tier floor** — T1: reads a fixed wire record and replies with a one-byte tag

## Purpose

Before a client may play, the server must know it has *the same level*, not merely a level with
the same name. Name and version establish identity; a checksum over the geometry establishes
that the bytes match. This is that exchange, and it exists as its own file because it is the
one handshake step with three distinguishable outcomes.

## State

`Stateless.`

## `OnProcessClientMapData`

**Contract** — reads the client's claimed level name, version and geometry checksum; compares
each against the server's own; replies with one of the three outcome tags, sent reliably.
**Never disconnects the client** — the reply is informational and the client decides what to do.

```text
FUNCTION on_client_map_data(packet, client)
  claimed_name     := packet.read_string()
  claimed_version  := packet.read_string()
  claimed_checksum := packet.read_int(32-bit)

  IF claimed_name != level.name OR claimed_version != level.version
    reply(other_map)
  ELSE IF NOT level.checksum_matches(claimed_checksum)
    reply(invalid_checksum)
  ELSE
    reply(success)
```

**Invariants** — the order of the two tests is load-bearing: a checksum is only meaningful
against the same level, so identity is established first.

**Notes** — the strings are read into fixed buffers with a bounded read, which is what stops a
hostile client's overlong name from overwriting the stack. That bound is the security-relevant
part of the function, and a rebuild in a language with growable strings has it for free.

**Nothing enforces the answer.** The source flags the missing disconnect in a comment: a client
told it has the wrong level may simply carry on. So this exchange is advisory as shipped, and a
rebuild that wants it to mean anything must refuse the connection on a non-success outcome. The
anti-cheat value of the checksum is entirely lost otherwise.

The download location for a mismatched level is not sent here — it was already sent in the
connection description, before the client had a chance to say what it has. That ordering is
slightly odd and entirely harmless.
