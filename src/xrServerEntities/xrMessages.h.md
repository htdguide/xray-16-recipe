# src/xrServerEntities/xrMessages.h

> The frozen identifier space of everything that travels between a client and a server: transport messages, entity events, match events, and the spawn message's option bits.

**Needs** — [Data: network protocol](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`BottleItem.cpp`](../xrGame/BottleItem.cpp.md) · [`BreakableObject.cpp`](../xrGame/BreakableObject.cpp.md) · [`CustomRocket.cpp`](../xrGame/CustomRocket.cpp.md) · [`Explosive.cpp`](../xrGame/Explosive.cpp.md) · [`Grenade.cpp`](../xrGame/Grenade.cpp.md) · [`HelicopterWeapon.cpp`](../xrGame/HelicopterWeapon.cpp.md) · [`Hit.cpp`](../xrGame/Hit.cpp.md) · [`HudItem.cpp`](../xrGame/HudItem.cpp.md) · [`InfoDocument.cpp`](../xrGame/InfoDocument.cpp.md) · [`Level.h`](../xrGame/Level.h.md) · [`Level_GameSpy_Funcs.cpp`](../xrGame/Level_GameSpy_Funcs.cpp.md) · [`Level_network_messages.cpp`](../xrGame/Level_network_messages.cpp.md) · [`Message_Filter.cpp`](../xrGame/Message_Filter.cpp.md) · [`Mincer.cpp`](../xrGame/Mincer.cpp.md) · _and 49 more_
**Tier floor** — T1: these are wire values; their numbering is the protocol.

## Purpose

Four numbering spaces, each dense from zero, each written on the wire as its ordinal.
Renumbering any of them is a protocol change, and two of them reach the save file as well —
a save is a stream of spawn messages, so the spawn message's identifier and its option bits
are part of conformance criterion 7.

Nothing here is a decision a rebuild gets to make. It is an input.

## The transport messages

The outermost envelope: what kind of thing this packet is. The first two matter far more
than the rest.

**`M_SPAWN` (1) carries a full record** — the class tag, the identity, and the class's own
serialized state. It is the same encoding a save file uses and the same one a spawn file
uses, which is the single most important fact in this chapter: one record type, three
consumers, one format.

**`M_UPDATE` (0) carries a delta** — the per-tick state of already-spawned entities, packed
for size rather than completeness. What a given class puts in it is entirely different from
what it puts in a spawn, and the two are not interchangeable.

The rest, grouped by what they do:

- **session setup** — new-client configuration, game configuration, configuration finished,
  client-ready, connection-data request, connection result;
- **server migration** — deactivate on the old server, activate with full state on the new
  one; a feature of the original's dedicated-server architecture;
- **level and save control** — change level, change level with a game mode, load, reload,
  save, a save payload chunk, and the switch distance that sets how far online simulation
  reaches;
- **events** — a single game event, a pack of them, a game message, client input, a client
  update, an object-update batch and its compressed form;
- **chat** — two identifiers, an older and a newer;
- **authentication and integrity** — an authentication challenge and response, a key
  validation challenge and response, a ping challenge and response, a bullet-check response,
  an anti-cheat channel, a secure key exchange and a secure message, a map-name and digest
  exchange, and a client warning;
- **statistics and administration** — a statistics update and response, remote-control
  authentication and command, a rename, a file transfer, a screenshot request;
- **multiplayer mechanics** — player fire, move players, move artefacts, a move
  acknowledgement, create player state.

**Notes** — several of these reach the dead matchmaking service (see
[Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts))
and are unreachable in practice. **Their ordinals must still be reserved**, because removing
one renumbers everything after it. A rebuild that drops multiplayer keeps the gaps.

**Two identifiers in the middle are marked as having been added for a trade-show build** and
never cleaned up. They are live numbering all the same.

## The entity events

What *happened to* an entity, delivered inside an event message. This is the game's verb
list and it is worth reading as one: it is the complete set of things one entity can do to
another across the client/server boundary.

- **Life cycle** — respawn, hit, die, assign a killer, destroy, reject a destroy, freeze,
  teleport, change position, change visual.
- **Ownership** — take, a forced multiplayer take, reject, and the combined
  destroy-and-reject. Ownership is a *request*: a client asks, the server decides.
- **Inventory** — transfer ammunition between weapons, an inventory action, container
  status, owner status, buy, sell, trade in either direction, money, install an upgrade,
  attach/detach/change an attachment, add ammunition, change weapon state, explode a
  grenade, launch a rocket.
- **Restriction** — add, remove, remove all. See
  [`restriction_space.h`](restriction_space.h.md).
- **Knowledge** — transfer a newly learned information piece to a PDA. See
  [`InfoPortionDefs.h`](InfoPortionDefs.h.md).
- **Player-directed** (a distinct prefix) — activate a slot, move an item to a slot, to the
  belt or to the rucksack, eat it, sell it, activate an artefact, use a booster, hide or
  show the weapon, disable sprinting, attach or detach a vehicle, play a headshot effect.
- **Match bookkeeping** — a hit statistic, a kill credit, a request for player information,
  a zone state change, and the actor's instantaneous move, jump, maximum power and maximum
  health.

**Invariants** — the ownership request/reject pair is the protocol's only two-phase
operation and the reason `GE_DESTROY_REJECT` exists as its own identifier rather than as two
messages: destroying an item the client thought it owned must undo the ownership claim in
the same step, or the item leaks into two inventories.

## The match events

A third space, between the match rules on the two sides: readiness, a requested suicide, the
buy menu's open/close/finish, the game menu and its response, connect, disconnect, enter
game, killed, hit, join a team, round start and end, the five artefact events (spawned,
destroyed, taken, dropped, on base), base entry and exit, three menu-closed notifications,
create client, hit and touch, the six voting events, authentication, name, speech, money
changed, a server string or dialogue message, player started, make data, receive server
logo, create player state, and a player-information reply.

**Invariants** — the space ends with a **sentinel marking where script-defined events
begin**, and the file says in as many words that nothing may be added after it. A mod's
event numbers start there, so inserting an engine event at the end silently collides with
every mod's first event. This is the one rule in the file that is easy to break and hard to
diagnose.

## The spawn message's option bits

A bit field accompanying every spawn, saying what else the message contains and what the
receiver should do with the entity.

```text
ENUM SpawnOption (bit positions in a 32-bit field)
  local        = bit 0   # after spawning, this side is authoritative for the entity
  has_update   = bit 2   # an update payload follows inside this message
  as_player    = bit 3   # the receiver should view through this entity
  phantom      = bit 4   # visible but not simulated
  versioned    = bit 5   # a format version precedes the payload
  with_update  = bit 6   # an update packet follows the spawn
  with_time    = bit 7   # a spawn timestamp follows
  denied       = bit 8   # do not spawn this entity at all
```

**Invariants** — **bit 1 is unused.** No reason is recorded and no code reads it; it is most
likely a removed option whose neighbours were never renumbered, and it must stay unused.

**`versioned` is the one that matters for saves.** A record whose spawn carries it is
preceded by a format version, which is what makes the version-gated reads throughout this
chapter possible — see [`xrServer_Objects.h`](xrServer_Objects.h.md) for the changelog those
versions index.

**`has_update` and `with_update` are two different bits** with confusingly similar names:
one says an update is embedded in this message's body, the other that a separate update
packet accompanies it. Both ship.

**`denied` is a spawn that must not happen**, carried as a spawn anyway so that the sender's
and receiver's record streams stay aligned. A receiver that simply skipped it would
desynchronize the identity allocation.

## The connection results

Five reasons a connection was refused: data verification failed, key validation failed,
password wrong, banned, profile error. Three of the five involve the dead matchmaking
service and can no longer occur.
