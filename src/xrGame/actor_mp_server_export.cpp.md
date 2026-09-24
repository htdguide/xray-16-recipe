# src/xrGame/actor_mp_server_export.cpp

> Fills the server's wire record from the authoritative player record, and relays it.

**Needs** — [`actor_mp_server.h`](actor_mp_server.h.md) · [`actor_mp_state.h`](actor_mp_state.h.md) · [`xrPhysics/phvalide.h`](../xrPhysics/phvalide.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: assembles the protocol's byte image

## Purpose

The server's send path. It is deliberately the mirror image of the client's — the same
fields, in the same order, into the same wire form — with one structural difference: the
server does not usually *compute* the state, it already holds it, received from the
player's own client. Gathering is therefore the fallback, not the normal path.

## State

None owned; fills the wire-state holder and the freshness flag declared in
[`actor_mp_server.h`](actor_mp_server.h.md).

## `fill_state`

**Contract** — copies the player's state out of the authoritative record's own fields:
the rigid-body half from the record's alive-state block, the logical half from the
record's position, acceleration, body facing, torso aim, timestamp, active weapon slot,
movement flags, health and radiation. Raises the freshness flag.

**Invariants** — every angle is normalized to a single turn before storing, for the same
aliasing reason as on the client.

**Notes** — the radiation value is stored **unscaled** here, while the client's gather
divides it by one hundred and the client's apply multiplies it back. A state that goes out
along this path therefore carries radiation on a different scale from one that came from a
client. Since the wire form clamps its normalized scalars into the unit interval, a server-
gathered radiation above one saturates. The path is only taken when no client update has
arrived — on the death capture, and on the first update of a player the server owns — so
the discrepancy is rarely visible, but it is a real inconsistency and a rebuild should pick
one scale.

## `UPDATE_Write`

**Contract** — if the held state is not already fresh, gathers it from the record; asserts
the position is a valid coordinate; writes the wire form.

**Invariants** — the freshness flag is what makes the relay a relay. When a client's
update has been received this update, the flag is up and the received state is forwarded
verbatim, bit for bit. Only when nothing was received does the server synthesize a state
from its own record. A rebuild that always synthesizes turns a lossless relay into a
re-quantization at every hop.
