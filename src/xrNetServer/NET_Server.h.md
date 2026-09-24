# src/xrNetServer/NET_Server.h

> Declares the server endpoint and, more importantly, the operations a game layer must supply
> before a server can exist at all.

**Needs** — [`NET_Shared.h`](NET_Shared.h.md) · [`NET_Common.h`](NET_Common.h.md) · [`NET_PlayersMonitor.h`](NET_PlayersMonitor.h.md) · [`ip_filter.h`](ip_filter.h.md)
**Used by** — [`game_sv_base.h`](../xrGame/game_sv_base.h.md) · [`xrServer.h`](../xrGame/xrServer.h.md) · [`NET_Client.cpp`](NET_Client.cpp.md) · [`NET_Client.h`](NET_Client.h.md) · [`NET_Server.cpp`](NET_Server.cpp.md) · [`NET_Client.cpp`](empty/NET_Client.cpp.md)
**Tier floor** — T2: it is an interface plus a few value records. The implementation in
[`NET_Server.cpp`](NET_Server.cpp.md) is what pins the module to T1.

## Purpose

Declares the surface implemented in [`NET_Server.cpp`](NET_Server.cpp.md). What belongs here
rather than there is the **extension contract**: this is an abstract endpoint, and the list of
operations it leaves unimplemented is the boundary between chapter 9 and chapter 23. A
rebuilder holding only this page should be able to say what a game layer owes the transport.

## State

Declarations only; the records themselves are described in
[`NET_Server.cpp`](NET_Server.cpp.md). One invariant belongs here rather than there, because
it is a property of the *interface*: an endpoint is abstract and cannot exist without a game
layer. There is no such thing as a bare transport server in this design — the eleven
operations below are not optional extensions, they are the constructor's preconditions.

## Exported units

- **`IPureServer`** — the server endpoint. Abstract: it cannot be instantiated.
- **`IClient`** — one connected peer's record. Also an envelope sender, because every link
  batches independently.
- **`IServerStatistic`** — aggregate byte counters, only maintained in debug builds.
- **`IBannedClient`**, **`ip_address`**, **`SClientConnectData`** — contracts in
  [`NET_Server.cpp`](NET_Server.cpp.md).

## What a game layer must supply

Eleven operations, in three groups. Every one of them is a place where the transport needs
something only the game knows.

```text
# Group 1 - client lifecycle. The transport decides WHEN; the game decides WHAT.
  new_client(identity)   -> ClientRecord   # a peer was admitted; produce its record
  client_create()        -> ClientRecord   # allocate an empty record
  client_replicate()                       # push current world state to a new client
  client_destroy(record)                   # release it

# Group 2 - entity identity. The server mints and recycles the identifiers that
# appear in every entity-bearing message, and only the game knows the rules.
  perform_id_generation(preferred) -> id   # allocate an entity identifier
  free_id(id, at_time)                     # return one, not before the given time
  game_state()           -> GameState      # the authoritative world

# Group 3 - entity records. The transport receives a spawn message and must turn
# it into something; only the game knows the class registry.
  entity_create(class_name) -> EntityRecord
  entity_destroy(record)
  perform_destroy(record, mode)
  process_spawn(message, sender, ...) -> EntityRecord
```

**Notes** — group 3 is the seam's leak. A transport module has no business knowing about
entities, and it does not: it never calls these on its own behalf, only as a way of letting
one of *its* callers reach the game layer through the object it already has. The honest
reading is that `IPureServer` is two interfaces wearing one name — a transport endpoint and a
server-side entity factory — and a rebuild should separate them. Nothing in this module would
change if group 3 moved out.

`free_id` taking a time is the one genuinely load-bearing oddity in group 2: an entity
identifier may not be reused immediately, because messages referring to the old occupant may
still be in flight. The delay is the game layer's to choose; the transport only carries it.

## What a game layer may override

```text
  on_message(message, sender) -> ChannelFlags
      # Zero means handled. Non-zero means "relay this to everyone else on this channel",
      # which is how a server forwards a client's message without rebuilding it.
  on_client_connected(record) / on_client_disconnected(record)
  on_query_host() -> bool            # answer a discovery probe at all?
  check_server_access(record) -> (bool, reason)
  assign_server_type(text) / get_server_info(sink)     # for the browser listing
  disconnect_client / disconnect_address / ban / unban  # policy overrides
```

## Roster access

The roster is never exposed directly. Callers get four shapes, all of which take the roster
lock for their duration:

```text
  for_each_client(action)
  for_each_client_as_sender(action)   # also takes the message lock, see below
  find_client(predicate) -> optional<ClientRecord>
  client_by_id(id)       -> optional<ClientRecord>
  client_count()         -> int
```

**Invariants** — `for_each_client_as_sender` additionally holds the inbound-message lock
across the iteration, so composing an outbound broadcast cannot interleave with handling an
inbound message. That is the difference between the two iteration forms and the only reason
both exist; a rebuild that names them `iterate` and `iterate_while_sending` will have
documented it better than the original does.

## Notes

The only way to reach a client by position is commented out, with a note calling it a very
bad method. The decision behind the comment is sound and should survive: a roster index is not
stable across a disconnect, so nothing may hold one across a frame.

A parallel roster of *disconnected* clients is declared, together with four operations over
it, and all of it is commented out. Reconnection-preserves-state was designed and abandoned;
the `reconnect` flag on the client record is what is left of it.
