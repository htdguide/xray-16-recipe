# src/xrGame/xrServer_process_event.cpp

> The one gate through which every gameplay act reaches the authoritative world: a timestamped, addressed event, sorted into the handful of things the server is willing to do about it.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`game_sv_single.h`](game_sv_single.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [`game_base.h`](game_base.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: rewinds and rewrites a received packet in place before re-emitting it

## Purpose

Entities do not talk to each other across the network; they raise **events**. An event is a
timestamp, a type, a destination entity, and a type-specific payload. Everything a player does
that changes the world — picking something up, firing, dying, spending money — arrives here.

The value of this page is the **taxonomy**. Forty-odd event types are handled, and they fall into
six treatments. A rebuild that gets the six treatments right can add event types freely; one that
copies forty cases has learned nothing.

## State

`Stateless.`

## `Process_event`

**Contract** — handle one event. Reads the fixed header, delivers the event to the destination
entity if it exists, then dispatches on type. **An unhandled type is a hard failure**, so the
event vocabulary is closed and every type must be accounted for.

```text
FUNCTION process_event(packet, sender)
  timestamp   := packet.read_int(32-bit)
  type        := packet.read_int(16-bit)
  destination := packet.read_int(16-bit)      # an entity identifier

  receiver := entities[destination]
  IF receiver EXISTS
    REQUIRE receiver has an owning client
    receiver.on_event(packet, type, timestamp, sender)   # the entity's own reaction

  dispatch on type into one of the six treatments below
```

**Invariants** — **the entity's own reaction runs first, before the switch, and it reads from the
packet without consuming it** — the switch reads the payload again from the same position. So an
entity's event handler must leave the read cursor where it found it. That is an undocumented
contract on every server-object event handler and a rebuild should make it explicit by handing the
handler a view rather than the cursor.

A destination entity that does not exist is not an error: the event is still dispatched, and each
case decides whether it can proceed without one. This matters because an event about a dead entity
can legitimately arrive after its destruction — the source notes exactly this for the death and
killer-assignment cases.

### Treatment 1 — Relay to everyone

*Information transfer, weapon state change, zone state change, jumping, headshot particles,
attaching and detaching a vehicle, moving an item to a slot, a belt or the backpack, a grenade
exploding, attaching and detaching a weapon addon.*

**Contract** — rebroadcast the packet verbatim, reliably, to every client including the sender.
The server does not interpret it and keeps no record.

**Notes** — these are *presentation* events. They change what clients show, not what the server
believes. That is why the server can afford to be a dumb repeater: nothing about the authoritative
world depends on them, so a lost or forged one costs a wrong animation and nothing more.

### Treatment 2 — Relay to the host client only

*Inventory action, position change, sprint disable, weapon hide state, slot activation, eating an
item, using a booster.*

**Contract** — forward to the host's own client, which is the one running the full game logic.
The server is a conduit.

**Notes** — these exist because the *client* half of the game owns inventory and player-state
logic, and a remote client's request has to reach it. In single player the forward is an
in-process hand-off and costs nothing.

The booster case is the only one that forwards to the *receiver's* owner rather than the host, and
only when that is not the host — so a booster used on a remotely simulated entity reaches whoever
simulates it. It also composes a fresh event and then sends the original instead, which is a
straightforward bug: the composed packet is discarded.

### Treatment 3 — Defer to the rules layer's queue

*Generic game events, hits, hit statistics.*

**Contract** — push the event onto the game rules' own delayed queue, to be processed on the
simulation thread in the rules' own order.

**Notes** — the hit cases **rewind the read cursor by two bytes before queueing**, so that the
queued packet begins at the event type rather than after it and the rules layer can re-read the
header. The statistics variant additionally **truncates the packet by four bytes and appends the
sender's identifier** — rewriting a received message to carry server-known provenance, the same
trick as the ping stamp in [`xrServer.cpp`](xrServer.cpp.md). Both offsets are part of the frozen
format.

Deferring hits rather than applying them is what lets the rules decide about friendly fire,
difficulty scaling and scoring in one place.

### Treatment 4 — Delegate to a named rules operation

*Selling an item, teleporting, adding a restriction, removing one, removing all, requesting the
player list.*

**Contract** — call a specific operation on the game rules or the server, passing the packet and
the destination. The rules own the semantics.

**Notes** — this is the healthy pattern and the one a rebuild should prefer: the event switch
becomes a routing table, and each operation lives with the subsystem that owns it.

### Treatment 5 — Mutate the server's record directly

*Change visual, install an upgrade, inventory-box status, corpse-lootability status, money,
assign killer.*

**Contract** — read the payload, narrow the destination entity to the type that can hold the
field, and write the field. Silently does nothing when the destination is absent or is the wrong
kind of entity.

```text
# the shape all six share
value := packet.read(...)
target := narrow(receiver, to the type that owns this field)
IF target is none THEN RETURN
target.field := value
```

**Notes** — **these are unvalidated writes to authoritative state from a client message.** Money
is set from whatever number the packet carries. A remote client cannot reach them in the shipped
configuration because the transport is null, but a rebuild that enables multiplayer must validate
every one of them or hand a player an editor for their own wealth.

Two of them write a pair of booleans decoded from bytes compared against one — a container's
"may be looted" and "is closed", and the same pair for a corpse. The encoding is incidental; the
pair is not: *lootable* and *closed* are independent, so a container can be closed and lootable,
or open and not.

### Treatment 6 — The authoritative acts

These are the ones the server actually decides, and each has its own file or its own paragraph.

**Ownership taken, bought, forced** → [`xrServer_process_event_ownership.cpp`](xrServer_process_event_ownership.cpp.md)
**Ownership rejected, sold, a rocket launched** → [`xrServer_process_event_reject.cpp`](xrServer_process_event_reject.cpp.md)
**Destroy** → [`xrServer_process_event_destroy.cpp`](xrServer_process_event_destroy.cpp.md)
**Artefact activated** → [`xrServer_process_event_activate.cpp`](xrServer_process_event_activate.cpp.md)

**Respawn** — the destination must be a phantom; a respawn entry is queued at *the event's
timestamp plus the entity's configured respawn delay*, and the server frame drains the queue. The
delay is configured in seconds and converted here.

**Ammunition transfer** — the interesting one. An entity's ammunition is merged into another's,
and the donor is then destroyed outright rather than transferred.

```text
FUNCTION transfer_ammo(packet, sender, receiver)
  donor_id := packet.read_int(16-bit)
  donor := entities[donor_id]
  IF donor is absent THEN RETURN
  IF donor already has a parent THEN RETURN      # somebody else got there first
  REQUIRE the sender's client is the receiver's owner   # you may only take for yourself

  broadcast the event to everyone, including the sender
  destroy the donor
```

**Invariants** — the parent check is the race guard: two clients reaching for the same box of
ammunition, and the second finds it already owned. The ownership assertion is the authority check
— a client may not transfer ammunition into somebody else's gun.

The broadcast happens **before** the destruction, so clients learn what to merge before they learn
the donor is gone.

**Death** — the most elaborate case, and the only one that rewrites its own packet.

```text
FUNCTION on_death(packet, sender, timestamp, victim_id)
  killer_id := packet.read_int(16-bit)
  victim := entities[victim_id]
  IF victim is absent THEN RETURN              # a hit event can outrun a destroy event

  killer := entities[killer_id]
  IF killer is absent
    killer := the entity owned by the client whose identifier is killer_id
  IF killer is still absent THEN log and RETURN

  rules.on_death(victim, killer)

  IF the killer is its client's own main entity
    rewrite the packet: same header, plus killer id, plus the killer's CLIENT identifier
  broadcast to everyone

  IF the game is single-player
    send the killer's client a "you killed someone" event
```

**Invariants** — **the killer identifier is looked up twice, as an entity and then as a client.**
That is the file's one genuinely subtle line: a kill may be attributed to a client rather than to
an entity — an environmental death credited to a player, a kill by something already destroyed —
and the two identifier spaces are distinct but both 16 bits wide, so the fallback is a guess that
happens to work. A rebuild should tag the attribution with which space it is in.

The packet is **rewritten in place** to append the killer's client identifier, so that clients can
attribute the kill to a *player* for the scoreboard, not merely to an entity. It is appended only
when the killer is that client's main entity — a kill by somebody's deployed turret is not a kill
by them.

The extra single-player-only event is what drives the player-facing kill feedback; multiplayer
derives it from the broadcast instead.

**Freeze object** — accepted and ignored. The event type exists in the vocabulary and does
nothing; a rebuild may reject it.

## Notes

**The closed vocabulary is the design.** An unknown event type is a hard failure, which means the
sender and the receiver must have been built from the same event table — and since the type is a
16-bit value in the wire format, the table is frozen alongside it. The data-authenticity check at
connect time is what makes this safe to rely on.

**Every case that re-emits the packet re-emits the *received* bytes**, sometimes amended. Nothing
re-composes an event from parsed fields. That keeps the server cheap and means an event's payload
can carry fields the server does not understand, which is how the client halves of the game
exchange state through the server without the server knowing what it is.
