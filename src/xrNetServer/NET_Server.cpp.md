# src/xrNetServer/NET_Server.cpp

> The server end of a session: hosting, admitting or refusing a peer, keeping the client
> roster, broadcasting, pacing per-client traffic, and maintaining the ban list.

**Needs** — [`NET_Server.h`](NET_Server.h.md) · [`NET_Common.h`](NET_Common.h.md) · [`NET_Messages.h`](NET_Messages.h.md) · [`NET_Log.h`](NET_Log.h.md) · [`NET_PlayersMonitor.h`](NET_PlayersMonitor.h.md) · [`ip_filter.h`](ip_filter.h.md) · [`xrCore/net_utils.h`](../xrCore/net_utils.h.md) · [`xrCore/xr_ini.h`](../xrCore/xr_ini.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Threads](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — reached through its declarations in [`NET_Server.h`](NET_Server.h.md); callers name that, not this file.
**Tier floor** — T1: it reinterprets received datagrams as system-packet memory images and
mutates one in place before bouncing it.

## Purpose

The mirror of [`NET_Client.cpp`](NET_Client.cpp.md), with one structural difference that
changes everything: the server has *many* peers, so every decision the client makes once the
server makes per client, and the roster that holds them is shared between the transport's
delivery thread and the simulation thread.

The file also carries three things that have no better home — the address record, the ban-list
entry, and the implementation of the per-link statistics declared in
[`NET_Shared.h`](NET_Shared.h.md). All three are here for historical reasons; a rebuild should
place the statistics with its transport adapter and the address record wherever addresses are
defined.

## State

```text
RECORD ServerSession
  clients            : ClientRoster          # see NET_PlayersMonitor.h
  local_client       : optional<ClientRecord> # the listen server's own client, if any
  port               : int
  connect_options    : text                  # the option string, kept verbatim
  bans               : list<BanEntry>
  subnet_filter      : SubnetFilter
  statistics         : ServerStatistics
  dedicated          : bool                  # rendering disabled, no local client
  message_lock       : Mutex                 # serializes inbound handling against broadcast

RECORD ClientRecord
  id              : int (32-bit)      # from the transport
  name            : text
  password        : text
  guid            : text              # filled by the game layer's validation, if any
  address         : Address
  port            : int
  process_id      : int               # the peer's process identifier, from its identity blob
  last_update_time: int (32-bit, wraps)   # governor bookkeeping
  statistics      : LinkStatistics
  local           : bool              # this is the server's own client
  connected       : bool
  reconnect       : bool
  verified        : bool              # cleared while content authentication is outstanding

RECORD Address                        # four octets, addressable as one 32-bit value
RECORD BanEntry
  address : Address
  expires : timestamp                 # wall clock, not the monotonic clock

RECORD ServerStatistics
  bytes_in, bytes_out : int
  bytes_in_real, bytes_out_real : int     # declared; never updated
  bytes_sent, send_time, bytes_per_second : int
```

**Invariants** — the first client to connect to a listen server is that server's own client,
and it is exempt from the subnet check. A dedicated server has no local client and its player
limit is raised by one extra slot to compensate for the reserved one.

A record marked not-`connected` is skipped by broadcasts but still occupies the roster until
the game layer destroys it.

`verified` starts *true* and is cleared by the game layer when it begins content
authentication — the opposite of the obvious polarity, and the reason a server with
authentication disabled needs no special case.

## `Connect` — hosting

**Contract** — parses the option string, creates the transport endpoint, advertises the
session and begins accepting. Returns success or a single undifferentiated failure. Blocking.
On success, admission events begin arriving on the transport's thread immediately, so
everything the admission path reads must already be initialized.

```text
FUNCTION host(options : text, description : GameDescription) -> result
  keep options verbatim
  direct_connect = options contains "/single"

  session_name = options up to the first '/'
  password     = value of "psw="
  max_players  = value of "maxplayers=", clamped to 1..32, default 32
  port         = value of "portsv=", default the LAN base + 1, remembered as explicit

  IF NOT direct_connect THEN
    create transport endpoint
    advertise: client-server topology, the application identifier, the session name,
               max_players + (2 if dedicated else 1), the description blob, the password
    disable the transport's own NAT traversal
    FOR port FROM the chosen port WHILE hosting fails
      IF the port was given explicitly THEN FAIL WITH port_busy
      IF port passes the end of the LAN window THEN FAIL WITH no_free_port

    load the ban list
    load the subnet filter
  RETURN ok
```

**Notes** — the player limit is clamped to at most 32, and a request outside 1..32 becomes 32
rather than an error. Thirty-two is the engine's ceiling everywhere, not just here.

The extra reserved slot exists because the transport counts the host itself as a participant.
A dedicated server reserves two — one for the host, one for the client slot it will never use
— which is a compensation for a vendor accounting quirk, not a decision a rebuild inherits.

NAT traversal is explicitly turned off. The engine's story for reaching a server behind a NAT
was the matchmaking service, which no longer exists; a rebuild designing a fresh one owns this
decision again.

In direct-connect mode nothing is created and nothing is advertised. The ban list and subnet
filter are not even loaded, because there is nobody to filter.

## `net_Handler` — the admission and delivery path

**Contract** — the transport's event callback, running on the transport's thread. Four event
kinds matter; everything else is ignored.

```text
ON discovery_query(probe)
  IF the probe is the client's marker THEN answer
  IF the probe's leading identifier is not this application's THEN stay silent
  IF the game layer declines to advertise THEN stay silent
  otherwise answer

ON admit_request(peer_address)
  IF the address is banned THEN refuse with the ban message
  IF there is already a local client AND the address is outside the subnet filter
     THEN refuse with the subnet message
  # note: the first connection is exempt - it is the server's own client
  otherwise admit

ON client_created(handle, identity_blob)
  IF the handle is the server's own participant THEN it is not a client; record the
     server type string once and stop
  read name, password and process identifier out of the identity blob
  ask the game layer to build a client record for it

ON client_destroyed(handle)
  find the record; mark it not connected and not reconnecting
  tell the game layer, then let the game layer destroy the record

ON datagram(from, bytes)
  IF it is a probe packet (magic prefix, probe length) THEN
     stamp it with the server's clock and bounce it back
     on an unreliable, unordered, high-priority, immediate channel
  ELSE split the envelope and deliver each message
```

**Invariants** — the probe bounce writes the server's clock **into the received buffer** and
returns that same buffer. This is what makes the round trip measure one network traversal each
way plus the server's handling, with no server-side queueing in between, and it is why the
bounce goes out immediately rather than being batched.

**Notes** — the refusal messages are human-readable text handed back through the transport's
reply channel. They are the only bytes an un-admitted peer ever receives, and the client turns
them into the message the player sees.

The subnet check is skipped entirely while there is no local client. The comment in the source
says why: on a listen server, the host's own client connects first, from an address that is
usually not in the allow-list. A dedicated server never has a local client, so this exemption
applies to its *first* connecting player instead — an inherited quirk that a rebuild should
close by keying the exemption on locality rather than on ordering.

The bounce is sent through the buffered path, not the raw one; the original tried both and the
raw version is commented out beside it.

## `_Recieve` — inbound message dispatch

**Contract** — one message, already split out of its envelope, tagged with the sender's
identifier. Handed to the game layer under the message lock. If the game layer returns a
non-zero channel descriptor, the message is re-broadcast to every other client on that
channel — a relay path that lets the game layer forward a client's message without
recomposing it.

```text
FUNCTION on_message(data, size, sender)
  IF size >= datagram_limit THEN
    report a suspiciously large packet and DROP IT      # the only hostile-input check here
  wrap the bytes as a message
  LOCK message_lock DURING
    channel = game_layer.on_message(message, sender)
  IF channel is non-zero THEN broadcast the message to everyone except the sender
```

**Notes** — the oversize check is the module's entire input validation. Everything past it
trusts the bytes, which is defensible only because the format has no self-describing
structure to be confused by: a malformed message causes the game layer to read nonsense
values, not to jump anywhere.

The broadcast happens *outside* the message lock while the roster iteration takes its own
lock, so a message can be relayed while the next one is being handled. That is deliberate;
the message lock exists to serialize the game layer's own state, not the wire.

## `SendTo` / `SendTo_Buf` / `SendBroadcast` / `Flush_Clients_Buffers`

**Contract** — three send paths with different batching, and they are not interchangeable:

- **`SendTo`** bypasses the envelope entirely and hands one message straight to the transport.
  Used where a message must not wait behind others.
- **`SendTo_Buf`** queues into that client's envelope accumulator. This is the normal path.
- **`SendBroadcast`** sends one message to every connected client except one, each through
  that client's *unbuffered* path, under the roster lock.
- **`Flush_Clients_Buffers`** closes every client's accumulator, once per simulation tick.

Each client owns its own accumulator — the envelope is per link, so it has to. The server's
job is only to remember to flush them all.

**Notes** — that broadcast uses the unbuffered path means a broadcast message never shares a
datagram with that client's pending updates. Given that broadcasts are usually reliable state
changes and updates are usually not, the flags-change rule would have forced a flush anyway;
the unbuffered path just makes it explicit.

The server-wide byte counters are only updated in debug builds. A rebuild that wants
server-level throughput numbers has to add them.

## `HasBandwidth` — the server governor

**Contract** — asked per client before composing that client's update. Same two brakes as the
client's, with the server's own limits; a `true` answer reserves the slot.

```text
FUNCTION has_bandwidth(client) -> bool
  IF direct_connect THEN                     # in-process: always allowed
    refresh the client's statistics
    client.last_update_time = now
    RETURN true

  interval = 1000 / server_update_rate       # default rate 30 per second
  IF minimize_updates THEN interval = 1000
  IF server_update_rate = 0 THEN RETURN false
  IF now - client.last_update_time <= interval THEN RETURN false

  pending = transport's outbound queue depth for this client
  IF pending > server_pending_limit THEN     # default 3
    client.statistics.times_blocked = client.statistics.times_blocked + 1
    RETURN false
  refresh the client's statistics
  client.last_update_time = now
  RETURN true
```

**Notes** — the rate is *per client*, so a server with thirty clients at thirty updates per
second is composing nine hundred updates per second. The depth brake is what keeps a single
slow client from consuming the server's send capacity: once that client's queue is three
datagrams deep, it stops receiving updates until it catches up, and the others are unaffected.

## `IClientStatistic` — the per-link statistics

**Contract** — implemented here, declared in [`NET_Shared.h`](NET_Shared.h.md), used by both
ends. Folds a transport link report into the record and rolls the four per-second rates over
once a second. The message-rate counters are derived by differencing the transport's
cumulative counts, so they are only meaningful if the report is refreshed regularly — which
the governor guarantees, since it refreshes on every allowed send.

The transport reports *three* transmitted-message counters, one per priority class, and the
engine sums them into one. A substitute transport that reports a single count satisfies this
directly.

## `ip_address`

**Contract** — four octets with a whole-value view. Parses from dotted-decimal text and
formats back to it. Malformed text yields the zero address and a complaint rather than an
error.

**Invariants** — equality is *not* plain equality. Two addresses are equal if all four octets
match, **or** if the first three match and the right-hand side's fourth octet is zero — so a
stored address ending in zero acts as a whole-class-C wildcard. That is how a ban can cover a
subnet without a mask. It is asymmetric (the wildcard must be on the right) and therefore not
a proper equivalence, which a rebuild should fix by making the wildcard explicit.

## `IBannedClient` — ban entries

**Contract** — an address and an expiry, persisted as one section of a configuration file in
the writable data root, the section name being the address and the single key being the expiry
as `dd.mm.yyyy_hh:mm:ss` local time. Loaded whole at host time, rewritten whole on every
change.

**Notes** — the expiry is *local wall-clock* time, formatted and parsed with the local time
zone. A server that moves time zone, or a machine whose clock is corrected, changes the
meaning of every stored ban. Storing an absolute instant would cost nothing and a rebuild
should.

Expiry sweeping sorts the list by expiry descending and tests only the last entry, retiring at
most one ban per sweep. With a sweep per tick this is not observable, but it is an accident of
implementation rather than a rate limit anyone chose.

Banning by client resolves that client's address and bans the address; the client is not
disconnected as a side effect. Disconnecting an address is a separate operation that walks the
roster collecting every client at that address and ejects each, because several clients may
share one address behind a NAT.

## `DisconnectClient` / `DisconnectAddress` / `BanAddress` / `UnBanAddress`

**Contract** — eviction, carrying a reason string that reaches the evicted client's
session-terminated hook. Banning and eviction are deliberately separate operations: a ban
without an eviction leaves an existing player alone and refuses their next connection, which
is what an administrator usually wants.

## Notes

Two protected operations are declared for aborting a half-formed client link and are defined
nowhere. Their absence is invisible because nothing calls them.

The subnet filter's query takes an address in host order while the filter stores its entries
in network order and swaps on the way in. Nothing else in the module cares about byte order,
so this is a local inconsistency rather than a convention; a rebuild should pick one order and
hold it everywhere.
