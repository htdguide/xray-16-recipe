# src/xrGame/actor_mp_server_import.cpp

> Accepts a player's own report of their state as authoritative, unless they are dead.

**Needs** — [`actor_mp_server.h`](actor_mp_server.h.md) · [`actor_mp_state.h`](actor_mp_state.h.md) · [`xrPhysics/phvalide.h`](../xrPhysics/phvalide.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: consumes the protocol's byte image

## Purpose

The server's receive path, and the place where this engine's multiplayer trust model is
visible in one function: **the client is authoritative over its own player's position,
pose, health and radiation.** The server copies what it is told into the authoritative
record with no validation beyond "is this a finite coordinate". That is why the codebase
has a separate anti-cheat comparison pass at all — there is no server-side simulation to
disagree with the client, so cheating is detected by comparing declared tuning values
rather than by rejecting impossible movement. A rebuild designing fresh multiplayer should
not copy this; it should copy the *format* and put a simulation behind it.

## State

None owned; writes into the authoritative record's own fields and into the wire-state
holder declared in [`actor_mp_server.h`](actor_mp_server.h.md).

## `UPDATE_Read`

**Contract** — resets three fields that are not carried by the compact form, then either
consumes and discards the update (if the player is already dead) or reads it into the
authoritative record and raises the freshness flag.

```text
FUNCTION update_read(packet)
  record.flags = 0
  record.item_count = 1            # a player record always carries exactly one item block
  record.velocity = zero

  IF record.health <= 0 THEN
    read a wire record into a throwaway holder and discard it
    RETURN                          # the packet must still be consumed: the stream is positional

  state = read_wire_record(packet)
  REQUIRE state.position is a valid coordinate
  copy every field of state into the authoritative record
  ready_to_update = true
```

**Invariants**

- The dead-player branch **still reads the packet**. The update stream is a concatenation
  of per-entity records with no per-entity length, so skipping the read would desynchronize
  every entity after this one in the same packet. Reading into a throwaway holder is how
  the original expresses "skip"; a rebuild with length-prefixed records can genuinely skip.
- The three reset fields are the ones the compact form dropped in the size-reduction pass.
  Resetting them rather than leaving them stale is what keeps the full-state form
  consistent with the compact one, since the full form still carries them.
- The freshness flag is raised so that the send path relays these exact bytes rather than
  re-deriving them.
