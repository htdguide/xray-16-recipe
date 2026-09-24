# src/xrGame/Level_network_messages.cpp

> The client's message dispatch: one pass over everything the transport delivered this frame, routing each message either into the deferred game-event queue or straight into the object it names.

**Needs** — [`Level.h`](Level.h.md) · [`NET_Queue.h`](NET_Queue.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`Entity.h`](Entity.h.md) · [`Actor.h`](Actor.h.md) · [`Artefact.h`](Artefact.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [`saved_game_wrapper.h`](saved_game_wrapper.h.md) · [`file_transfer.h`](file_transfer.h.md) · [`Message_Filter.h`](Message_Filter.h.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a dispatch over a wire format whose field widths are fixed, but no device or layout constraint of its own

## Purpose

Everything the server tells a client arrives here. The file's substance is the *routing
table* — which message is applied immediately and which is deferred — and the reason for
the split, which is ordering. A message that changes the world (a spawn, an entity event, a
chat line that the game rules react to) goes on the game-event queue so that it is applied
at one defined point in the frame, in arrival order, alongside every other world-changing
message. A message that only moves numbers already owned by an object (an entity state
update, an input snapshot) is applied on the spot, because deferring it would put it behind
events that logically follow it.

The same routine runs on a listen server, where "the transport" is the in-process loopback,
which is why several cases begin by checking whether this process is the pure client.

## State

`Stateless` in its own right. It reads and writes the level's counters and flags:

```text
RECORD ReceiveCounters             # reset at the top of every pass
  packets_this_frame : int
  bytes_this_frame   : int
```

## `ClientReceive`

**Contract** — drains the transport's receive queue for one frame, dispatching each message
and releasing it. Called once per frame from the level's update, and also called
repeatedly from the load-time synchronization gates, which is why it must be safe to call
before the world exists. Blocks only on the transport's own non-blocking drain. Records
each message to the demo file when recording. Feeds the replayed stream when playing a
demo.

**Invariants** — the queue is bracketed: a begin call before the loop and an end call after,
which lets the transport hold its lock once rather than per message. Every retrieved
message is released exactly once, including on the paths that ignore it.

```text
FUNCTION ClientReceive()
  packets_this_frame = 0; bytes_this_frame = 0
  IF a demo is playing THEN SimulateServerUpdate()   # inject recorded packets into the queue

  BEGIN queue processing
  WHILE a message is available
    IF a demo is being recorded THEN append this message to the demo file
    count it
    read the message identifier
    dispatch on it                                   # the table below
    release the message
  END WHILE
  END queue processing
```

### The routing table

The two columns that matter are *deferred or immediate* and *what it needs to already
exist*.

```text
# --- deferred: pushed onto the game-event queue, applied at one point in the frame ---
SPAWN              create an entity from a server record
                   # rejected with a diagnostic if the level is not ready: a spawn
                   # before the game rules are configured cannot be assigned a team,
                   # a respawn point or a scheduler slot
EVENT              one entity event (a hit, a use, an item transfer)
EVENT_PACK         several events concatenated, each prefixed by a one-byte length;
                   unpacked here and enqueued individually, each inheriting the
                   carrier's receive stamp so their timing is not lost
MOVE_PLAYERS       an authoritative reposition of player entities
GAMEMESSAGE        a rules-layer notification (a kill, a round end)
STATISTIC_UPDATE   the server asks for this client's weapon statistics
FILE_TRANSFER      a chunk of an out-of-band file (map download, screenshot upload)

# --- immediate: applied to the object or subsystem it names ---
UPDATE                     the game rules' own periodic state
UPDATE_OBJECTS             a batch of entity state updates; see the catch-up rule below
COMPRESSED_UPDATE_OBJECTS  the same batch, dictionary-compressed; a flag byte selects
                           which compressor, then the batch is unpacked and applied
CL_UPDATE                  a client's state as the SERVER sees it; server-side only
CL_INPUT                   a client's input snapshot, applied to the entity it names
MOVE_ARTEFACTS             a count-prefixed list of (entity, position); each artefact
                           is teleported. Applied immediately because an artefact's
                           position is not worth an event
CHAT                       a console line
CHAT_MESSAGE / CLIENT_WARN handed to the game rules
REMOTE_CONTROL_AUTH / _CMD handed to the game rules' administrative channel
SV_MAP_NAME                the map-verification verdict
SV_DIGEST                  a request for this client's identity digest; answered at once
SECURE_KEY_SYNC            a new cipher seed
SECURE_MESSAGE             an enciphered message; decoded and then ENQUEUED, so that it
                           lands in the same order as the plaintext messages around it
CHANGE_SELF_NAME           a rename
BULLET_CHECK_RESPOND       the server's verdict on a reported hit, for statistics
AUTH_CHALLENGE             send the profile data and answer the build-version challenge
CLIENT_CONNECT_RESULT      the connection verdict
CDKEY_VALIDATION_CHALLENGE the key-validation challenge (dead matchmaking seam)

# --- session control ---
SV_CONFIG_NEW_CLIENT   begin configuring this client's game rules
SV_CONFIG_GAME         import the rules' state
SV_CONFIG_FINISHED     the gate the load sequence waits on; also the point where a
                       demo viewer's fake spectator is created if there is not one yet
SAVE_GAME              write a client-side save
RELOAD_GAME / LOAD_GAME / CHANGE_LEVEL   see below
CHANGE_LEVEL_GAME      the server is switching level and rules; see below
MIGRATE_ACTIVATE / MIGRATE_DEACTIVATE    server migration: never implemented; reaching
                       one is a hard failure rather than a silent ignore
```

**Notes** — the deferred/immediate split is the load-bearing decision, and the placement of
the *secure message* case shows why: a decoded secure message is enqueued rather than
dispatched, precisely so that encryption does not reorder it relative to its neighbours. A
rebuild that dispatches everything immediately will produce hits applied before the spawn
of the thing that was hit.

The event-pack case is bandwidth optimization with one subtlety worth preserving: each
unpacked event inherits the *carrier's* receive stamp, so a batch of events is treated as
having arrived at one instant rather than at the time it was unpacked.

Several cases carry a commented-out "ignore if the game is not configured" guard. They were
removed deliberately — a client can receive rules traffic during configuration and
dropping it loses state — and a rebuild should not reintroduce them.

### The physics catch-up rule

Three cases (`UPDATE_OBJECTS`, the compressed variant, and the server-side `CL_UPDATE`)
end with the same calculation, and it is the same one documented in
[`Level_network_compressed_updates.cpp`](Level_network_compressed_updates.cpp.md).

```text
FUNCTION catch_up_steps(packet) -> int
  ping = client statistics ping
  IF (server_time + ping) < packet.time_received THEN
    lag = ping                 # our clock estimate trails the stamp; use the ping alone
  ELSE
    lag = server_time - packet.time_received + ping
  END IF
  RETURN physics world's step count for lag
```

The result is stored as the number of steps the client owes. On the server-side
`CL_UPDATE` path it does more: the reported actor is marked for *client prediction
replay*, with its activation step set to `current step - catch_up steps`, and added to the
set of actors to replay. That is the reconciliation mechanism — the server rewinds the
actor to where it was when the client's input was generated, then re-simulates forward.

**Invariants** — replay is only set up for actors, only when there is something in the
prediction sets, and only on the authoritative side. A pure client takes the early exit at
the top of the case.

### Level and save transitions

```text
# RELOAD_GAME, LOAD_GAME and CHANGE_LEVEL share one handler.
IF the message is LOAD_GAME THEN
  read the saved game's name
  IF the name is non-empty AND the alife simulation exists THEN
    IF that save's level is the level already loaded THEN
      defer a quick-load event and STOP           # no reconnect needed
    END IF
  END IF
END IF
reconnect                                          # tear down and rebuild the session
```

The shortcut matters: loading a save for the *same* level skips the whole disconnect and
reconnect, which is the difference between a quick-load taking a second and taking thirty.
The condition is exact — same level, alife present — because anything else changes which
level is mounted.

### Changing level and rules

```text
# CHANGE_LEVEL_GAME
IF this process is a pure client THEN reconnect
ELSE
  read { level name, level version, game type }
  # The server's option string has the shape  <level>/<game type>/<options...>
  # Rebuild it with the new level and type, keeping the trailing options, and
  # strip out any map-version option already in there so it is not duplicated.
  new_options = level name + "/" + game type + "/" + version marker + level version
                + surviving options
  reconnect with new_options
END IF
```

**Notes** — the transition is implemented as *reconnect with a different option string*,
not as an in-place level swap. That is consistent with everything else here: there is no
partial teardown path, and a rebuild that invents one must satisfy the registration
invariants (nothing registered twice, nothing destroyed while referenced) by hand.

## `remove_version_option`

**Contract** — given an option string, returns a copy with the map-version option and its
value removed, or the original if that option is absent. Writes into a caller-supplied
buffer.

```text
FUNCTION remove_version_option(options, out buffer) -> text
  at = position of the version marker in options
  IF not found THEN copy options into buffer; RETURN buffer
  copy the characters before the marker (minus the separator) into buffer
  at = the next "/" at or after the marker
  IF none THEN RETURN buffer                  # the version option was the last one
  append the remainder from that "/" onward
  RETURN buffer
```

**Notes** — the option string is a slash-separated list and this is string surgery on it.
A rebuild should parse the options into a map and rewrite one key; this function only looks
the way it does because the string is the canonical form.

## `OnMessage`

**Contract** — a pure delegation to the transport client's own message entry point. It
exists so that the level is the named receiver and the demo playback path has a single
place to inject recorded bytes.

## Network lag simulation

**Notes** — a developer-only gate at the top of the receive loop can stall message
processing for a random interval between a configured minimum and maximum, simulating
latency without a network. It re-rolls the interval each time the previous one expires.
Worth keeping in a rebuild: reconciliation bugs are invisible on a loopback connection.
