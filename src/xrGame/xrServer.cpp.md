# src/xrGame/xrServer.cpp

> The authoritative world's heartbeat: the per-frame update, the table of every entity, the table of every client, and the switch that decides what a received message is allowed to do.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrServer_Objects_ALife_All.h`](../xrServerEntities/xrServer_Objects_ALife_All.h.md) · [`Level.h`](Level.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_sv_mp.h`](game_sv_mp.h.md) · [`ai_space.h`](ai_space.h.md) · [`file_transfer.h`](file_transfer.h.md) · [`screenshot_server.h`](screenshot_server.h.md) · [`xrServer_info.h`](xrServer_info.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`xrGameSpyServer.cpp`](xrGameSpyServer.cpp.md) · [`xrServer.h`](xrServer.h.md)
**Tier floor** — T1: copies packet buffers by raw size, downcasts every client record, and packs a 16-bit identifier into the wire format

## Purpose

Everything in the world that is *true* — where an entity is, who owns it, whether it exists —
is true because this object says so. Clients propose; the server decides and broadcasts. In
single player the same object runs in the same process as the client and the loop still goes
through it, which is the single most confusing thing about the architecture and the reason
the vocabulary distinguishes **server object** from **client object** everywhere.

This file holds the parts that are not one message family: the frame update, the two tables,
the broadcast machinery, and the message switch itself.

## State

See [`xrServer.h`](xrServer.h.md). Six tuning globals live here:

```text
reconnect_window_minutes  : int = 3       # how long a dropped player's state is kept
max_client_ping_ms        : int = 2000    # the ping a client is warned about
ping_check_interval_ms    : int = 15000   # minimum gap between two warnings to one client
max_ping_warnings         : int = 5       # warnings before the client is disconnected
```

**Notes** — the warning interval is what makes the policy humane: five warnings at fifteen
seconds apart is a bit over a minute of sustained bad connection before a kick, and a single
spike cannot spend more than one warning.

## `Update`

**Contract** — one server frame. Called from the level's own update. Does nothing at all while
a demo is playing back, because a demo *is* a recorded packet stream and a live server would
fight it.

```text
FUNCTION update()
  IF a demo is playing THEN RETURN

  proceed_delayed_packets()             # messages deferred out of the receive thread
  game.process_delayed_events()         # rule-layer events deferred by the game mode
  game.update()                         # the rules: scoring, rounds, respawn timers

  # entities whose respawn time has come
  now := async_time()
  WHILE the respawn queue is non-empty AND its earliest entry is due
    entry := take the earliest entry
    entity := entities[entry.phantom]
    packet := entity.write_spawn()
    process_spawn(packet, from: the server itself)

  send_updates_to_all()

  IF the game mode has demanded a full resync THEN perform_game_export()

  check_clients_for_max_ping()
  flush_client_buffers()

  once every 100 frames: refresh the banned-address list
```

**Invariants** — the respawn queue is ordered by due time, so the loop can stop at the first
entry that is not due. Delayed packets are processed *before* the game rules update, so a
message received since the last frame is reflected in this frame's rules rather than the next.

**Notes** — **The respawn queue re-spawns an entity from its own serialized form**: the
entity writes its spawn record and the server feeds that record back through the ordinary
spawn path, as if a client had asked. That is why respawn needs no separate code path, and it
is the clearest statement in the chapter that *a spawn record is the canonical description of
an entity*. A rebuild gets a lot for free by preserving it.

The banned-list refresh at every hundredth frame is a rate limit expressed in frames rather
than in time, so it happens more often on a fast machine. Harmless, and worth replacing with a
time interval.

## `SendUpdatesToAll`

**Contract** — the outbound half of a frame. **Returns immediately in single player**, which
is the whole optimization that makes running a server in a single-player process cheap: the
client and the server share memory, so there is nothing to send. Otherwise kicks pending
cheaters, sends each client its own game-state update, and — at a configured rate rather than
every frame — builds and broadcasts the per-entity updates.

```text
FUNCTION send_updates_to_all()
  IF the game type is single-player THEN RETURN
  kick_cheaters()

  FOR EACH client: send_game_update_to(client)      # per-client; scores, rules, its own view

  IF now - last_update >= 1000 / configured_updates_per_second
    make_update_packets()
    send_update_packets_to_all()
    IF the game mode demands a resync THEN perform_game_export()
    last_update := now

  IF file transfers are active
    pump them, and stop any receiver that has gone quiet
```

**Invariants** — the per-client game update goes out every frame; the per-entity world update
goes out at the configured rate. The two are separate because the first is small and
client-specific and the second is large and shared.

**Notes** — the update rate is the server's headline tuning number, and expressing it as
*milliseconds between updates* derived from a rate means the actual rate is quantized to the
frame. A rebuild should accumulate the remainder rather than resetting the timestamp to now,
which is what the original does and why the effective rate is always slightly below the
configured one.

## `SendGameUpdateTo`

**Contract** — one client's own view of the game state. Skipped for a client that has not yet
reported in, and **skipped when the transport says there is no bandwidth for it** — this is
the one place the server voluntarily drops a message rather than queueing it, because a stale
game-state update is worthless and a queue of them is worse than none.

## `MakeUpdatePackets`

**Contract** — serialize every entity that is worth sending and hand the results to the
compressor. Does not send. The filter is four conditions, and each one is a decision:

```text
FUNCTION make_update_packets()
  compressor.begin()
  FOR EACH entity IN entities
    IF entity has no owning client        THEN CONTINUE   # nobody is simulating it
    IF entity is not network-ready        THEN CONTINUE   # not finished spawning
    IF entity is flagged a phantom        THEN CONTINUE   # a server-side placeholder
    IF NOT entity.network_relevant()      THEN CONTINUE   # the entity's own say

    buffer := (entity id, then a length-prefixed block written by the entity)
    IF the entity wrote nothing THEN CONTINUE             # nothing changed
    compressor.write_update_for(entity id, buffer)
  compressor.end(yielding the packets to send)
```

**Invariants** — each entity's payload is written into a **one-byte length-prefixed chunk**,
which caps a single entity's update at 255 bytes. That is a hard limit on how much state one
entity may change per update, and an entity that needs more must split it across updates or
use an event. A rebuild must either keep the cap or widen the prefix in the frozen format.

**Notes** — **An entity that writes nothing is dropped entirely** rather than sent as an empty
update. That is the quiet-entity optimization the whole broadcast rests on: a level full of
motionless furniture costs nothing per frame. It requires every entity's serializer to be
honest about having nothing to say, which is a contract on the entity, not on the server.

*Network relevance* is the entity's own veto — an entity nobody can currently perceive can opt
out. That is where per-client interest management would go in a rebuild; the original's version
is global rather than per-client, so an entity is relevant to everyone or to no one.

## `SendUpdatePacketsToAll`

**Contract** — broadcast the compressor's packets, excluding the server's own client. **Skips a
packet carrying two bytes or fewer**, which is a packet with a header and no entities. Records
the total bytes sent for the statistics readout. Also writes each packet into the demo stream
when one is recording — which is what makes a demo replayable: a demo is exactly the broadcast
stream.

## `OnMessage`

**Contract** — the message switch. Receives every message from every client, dispatches on its
type, and falls through to the transport's own handling. Returns a non-zero value to ask the
caller to re-broadcast, which nothing in this switch does. **Runs on the receive path**, not
on the simulation thread, which is why several cases defer rather than act.

The cases divide into five kinds, and the kind is what matters:

**Acted on immediately** — state reports (`update`), gameplay events, save-packets, chat,
level-data queries, the client's digest, admin authentication, secure-message traffic and key
synchronization. These are cheap, or must be ordered against the receive stream.

**Deferred to the simulation thread** — the connection-data request, remote-control commands
and file transfers. These do work that is not safe off the simulation thread: the first
creates entities, the second runs console commands, the third touches the filesystem.

**Deferred to the game rules' own event queue** — player authentication and player-state
creation. The rules layer has its own ordering requirements and its own queue.

**Relayed, not interpreted** — a client's input and its client-side update are forwarded
verbatim to the *host* client; broadcast messages are re-broadcast. The server does not read
them.

**Unpacked and re-entered** — an event *pack*, which is several messages concatenated with
one-byte lengths, is split and each part fed back through the same switch.

```text
FUNCTION on_message(packet, sender) -> broadcast_flags
  type := packet.read_type()
  client := client_for(sender)
  SELECT type
    spawn request      : accept ONLY from a local client          # see Notes
    state report       : process_update(packet, sender)
    event              : process_event(packet, sender)
    event pack         : WHILE NOT packet.at_end
                           len := packet.read_u8()
                           on_message(packet.read_bytes(len), sender)
    client update      : mark client ready; stamp the measured ping into
                         the packet in place; relay to the host client
    client input       : mark client ready; relay to the host client
    ... (the remaining cases as classified above)
  RETURN transport.on_message(packet, sender)
```

**Invariants** — **a spawn request is honoured only from a client flagged local.** That is the
single most important line in the switch: a remote client may not create entities. Everything
a remote client wants to create goes through a gameplay event the server validates instead.

The ping is written *into the received packet in place*, at a fixed offset past the type, and
the packet is then relayed. The server is amending a client's message with a value only the
server knows, which means the offset is part of the frozen wire format.

**Notes** — the switch dispatches before checking that the sender resolves to a known client,
and several cases then dereference the result. A message arriving from a client that has just
been dropped is a real window. Some cases check, some do not; a rebuild should resolve the
client once, up front, and drop the message when it fails.

**Marking a client ready on its first update or input** is how the handshake completes: there
is no explicit "I am ready" from the client for this purpose, only the first message that could
only come from a running client.

The whole switch is wrapped by a sibling that takes a lock, so message handling is serialized
against itself but not against the simulation. That is the reason the deferral list exists.

## `OnDelayedMessage`

**Contract** — the three cases too heavy for the receive path, run at the top of the server
frame: the connection-data request (which admits a client), the remote-control command, and
file-transfer traffic.

**Notes** — the remote-control path is worth stating in full because it is a **remote code
execution surface by design**: an authenticated administrator's string is concatenated with
their client identifier and executed as a console command, with the console's log output
captured and sent back line by line as the reply. The identifier is appended so that a command
addressed at a player can be attributed. A client without administrator rights gets a refusal
string.

The log capture works by *globally redirecting* the log sink for the duration of the command,
which means any other thread logging concurrently has its output captured into this
administrator's reply. A rebuild should capture per-invocation.

## `ProceedDelayedPackets` / `AddDelayedPacket`

**Contract** — the deferral queue. Adding takes a lock and **copies the packet by its raw
buffer size**, not by its used length; draining takes the same lock and runs every queued
message through the delayed handler, in order.

**Invariants** — the queue is drained completely each frame, so a message is delayed by at most
one frame.

**Notes** — copying the whole fixed-size packet buffer rather than the used bytes is a real cost
per deferral and exists because the packet is a plain value with no ownership story. A rebuild
with a length-carrying buffer copies what was used.

The queue is also swept when a client disconnects, so a departed client's pending messages are
not run against a freed record — see `client_Destroy`.

## `client_Create`, `client_Find_Get`, `client_Replicate`

**Contract** — three points in the client record's life. Creation produces a bare record and is
*virtual*, so an authentication layer can extend it. `client_Find_Get` builds the record for a
newly connected identifier, resolves its network address from the transport — or substitutes the
loopback address when the build is in direct-connect mode — and registers it in the client table.
Replication is empty; the server does not push a client table to clients.

## `client_Destroy`

**Contract** — remove a client from the table and park its record for possible reconnect. Four
things happen and the order matters.

```text
FUNCTION client_destroy(client)
  record := client_table.find_and_erase(client)
  IF record is absent THEN RETURN

  owned := record.owner
  IF owned is a spectator
    broadcast a destroy event for it, excluding this client   # spectators are not persistent

  remove every queued delayed packet whose sender is this client
  IF owned EXISTS THEN game.clear_delayed_events_for(owned)

  disconnected_pool.add(record)          # ownership transfers; see xrClientsPool
```

**Invariants** — the record is removed from the client table *before* anything else touches it,
so no concurrent lookup can find a record that is being torn down.

**Notes** — **a spectator's entity is destroyed on disconnect and a player's is not.** A player
who drops may come back and reclaim their body; a spectator has no body worth keeping. This is
the only entity-lifetime decision made in the disconnect path, and everything else about a
dropped player's entity is decided by the game mode.

Purging the delayed queue and the rules layer's event queue of the departing client's work is
the dangling-reference cleanup: both queues hold a client identifier or an entity identifier and
would otherwise run against a record the pool now owns.

## `GetPooledState`

**Contract** — restore a reconnecting player's state. Asks the pool for a record matching this
new client; if one is found, **serializes the parked player state and deserializes it into the
new client's**, marks the new client as a reconnect, and frees the parked record.

**Notes** — round-tripping the state through a packet rather than copying it is the load-bearing
choice: the player state's serializer is the only complete, maintained description of what the
state *is*, and a field-by-field copy would drift from it. A rebuild should do the same wherever
a structure already has a wire format — the serializer is the deep copy.

## `SendTo_LL`

**Contract** — the single exit for every outbound message. **Short-circuits to a direct in-process
delivery** when the destination is the host's own client or the build is in direct-connect mode;
otherwise hands the bytes to the transport. Refuses to send to a disconnected client.

**Notes** — this is where single player pays nothing for the client/server split: the message is
handed straight to the level's receive path with no serialization round trip. A rebuild must
keep the short-circuit or single player becomes a loopback network game.

## `SendBroadcast`

**Contract** — send to every client except one. The exclusion predicate is three conditions: not
the excluded identifier, still connected, and **accepted**. A connected-but-not-yet-accepted
client receives no broadcasts, which is what keeps a half-joined client from seeing world traffic
it cannot interpret.

## `entity_Create` / `entity_Destroy`

**Contract** — construct a server object by class name through the factory, and tear one down.
Destruction removes the entity from the table, **returns its identifier to the allocator stamped
with the current time**, and breaks the mutual reference with its owning client.

**Invariants** — the identifier is freed with a timestamp so it cannot be reused while messages
naming it may still be in flight. That is the reason the allocator takes a time at all.

**The entity is not always actually destroyed.** When the alife simulation is running and the
entity is under its control, the object is removed from the server's table but *not* freed —
because the alife simulation owns it and will go on advancing it offline. This is the
online-to-offline transition, and getting it wrong is either a leak or a crash. A rebuild must
make the ownership explicit rather than inferring it from two flags.

## `ID_to_entity`

**Contract** — look up an entity by identifier. The all-ones identifier is the reserved *no
entity* value and answers nothing. An unknown identifier answers nothing rather than failing.

**Notes** — the source marks this as redundant with the game rules layer's own lookup, which
calls through to it. A rebuild should have one table and one lookup.

## `Server_Client_Check`

**Contract** — decide whether a connecting client is the *host's own*, by comparing its reported
process identifier with this process's. Marks it local and records it as the server client if so.
Also clears the server-client reference when that client disconnects.

**Notes** — **identifying the host client by process identifier is the mechanism behind the
"local client may spawn" rule in the message switch.** It is only as trustworthy as the client's
self-report, which for a same-machine client is fine and for a remote one is a value it chose.
Since a remote client's process identifier matching the server's is a coincidence rather than a
proof, a rebuild should compare the transport's own notion of a loopback connection instead.

## `PerformCheckClientsForMaxPing`

**Contract** — warn or disconnect clients whose ping exceeds the maximum. Skips the server's own
client and any client without a player state. A client is warned at most once per interval; on
the fifth warning it is disconnected with a localized reason.

The warning message carries the measured ping, the warning number and the maximum, so the client
can display "warning 3 of 5". Sending the counts rather than a bare warning is what lets the
player know how much rope is left.

## `CheckAdminRights`

**Contract** — validate a remote administrator's name and password against a file in the
application data directory. Answers false with a distinct reason for each failure: no file, no
such user, wrong password.

**Notes** — **the password is stored and compared in plain text.** That is the original's
scheme and a rebuild should not reproduce it; store a salted hash and compare against that. The
protocol above it does not care — it sends a name and a password over a channel that may or may
not be encrypted, which is the deeper problem.

## `OnChatMessage`

**Contract** — relay a chat message to the clients entitled to see it. Three filters, in order:
the recipient must be ready and have a player state; a team-addressed message reaches only that
team; **and a message from a permanently dead player reaches only other permanently dead
players.**

**Notes** — the dead-to-dead rule is the interesting one. It keeps an eliminated player from
relaying what they can see to the living, which is the standard spectator-information problem.
The addressed team is read from the message as a signed value, with a negative meaning *all*.

## `KickCheaters`

**Contract** — disconnect every client queued as a cheater, with its recorded reason, and
broadcast the reason to everyone else. Runs once per frame, at the top of the outbound work, so
that detection anywhere in the frame can queue rather than disconnect mid-operation.

**Invariants** — the broadcast reason **skips the first two characters of the recorded reason**.
Those two characters are a marker prefix the detection sites add and the broadcast strips. A
rebuild should carry the marker as a separate field rather than as a string prefix.

## `AddCheater`

**Contract** — queue a client for disconnection with a reason. Deferred rather than immediate,
because it is called from deep inside validation code where disconnecting would invalidate the
caller's own state.

## `MakeScreenshot` / `MakeConfigDump`

**Contract** — ask a suspected client to upload a screenshot or a dump of its configuration, over
the file-transfer channel, into one of a fixed pool of proxies. Fails with a message when every
proxy is busy. **The screenshot path is disabled**: it logs that it is unsupported and returns
before doing anything.

**Notes** — these exist as anti-cheat instruments: an administrator can demand evidence from a
client. The proxy pool is sized at twice the maximum player count, so two transfers per player
can be in flight. A rebuild that keeps this should treat it as a privacy-sensitive operation,
which the original does not.

## `SendPlayersInfo`

**Contract** — send one client the address and copy-protection digest of every connected player.
Used by the administrative console.

**Notes** — it sends **every player's network address to whoever asks**, gated only by the
message reaching the switch at all. That is an information leak a rebuild should gate on
administrator rights.

## `GetServerInfo`

**Contract** — fill the console's server description with the base server's items: port, uptime,
game type with its mode-specific limit, and the in-game clock with the statistics period. Each
item is a label, a value and a colour.

**Notes** — the game-type line is assembled by string concatenation with the limit that matters
for that mode — frag limit for the deathmatch modes, artefact count for the artefact modes —
which is a small piece of per-mode knowledge living in the wrong file. A rebuild should ask the
game mode to describe itself.

## `verify_entities` / `verify_entity`

**Contract** — a debug-only sweep of the entity table, run before and after every message. Checks
the table's key against each entity's own identifier, that no entity is the reserved invalid
identifier, that every entity has a version, and that parent and child references are mutually
consistent in both directions. Can be disabled by a command-line switch, because it is
quadratic-ish and dominates a debug build's frame.

**Notes** — **this is the best available statement of the entity table's invariants**, which is
why it is worth a heading in a recipe that otherwise discards debug code. A rebuild should keep
these checks, and the command-line escape hatch says the original could not afford them
continuously.

## `create_direct_client`

**Contract** — manufacture the single-player client: identifier one, the name `single_player`,
and this process's own identifier so that the host-client check above recognizes it. This is how
single player gets a client without a network connection.

## `DumpStatistics`

**Contract** — write the server's per-frame timing — update time and compression time — into the
debug overlay. Resets both frame timers as a side effect, so it must be called exactly once per
frame.

## `level_name` / `level_version`

**Contract** — parse the level's name and version out of the session option string. Both delegate
to the game-rules layer's parser, which owns the option-string grammar.

## `OnCL_QueryHost`

**Contract** — whether this server should answer a host-discovery probe. Always no in single
player; otherwise yes only if at least one client is connected.

**Notes** — refusing to answer when empty is deliberate: an empty single-player-derived server
should not appear on a local network browse.

## `GetEntity`

**Contract** — the *n*-th entity in the table, by walking the table. Linear, and used only by
inspection code. A rebuild should not offer it — index-into-a-hash-table is not a meaningful
operation, and the order it walks is the same undefined order that the load-order note in
[`xrServer.h`](xrServer.h.md) warns about.
