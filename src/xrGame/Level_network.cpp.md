# src/xrGame/Level_network.cpp

> The level's client side of the network — connecting, sending per-frame state, tearing the world down, and turning a refusal into something the player can read.

**Needs** — [`Level.h`](Level.h.md) · [`xrServer.h`](xrServer.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`NET_Queue.h`](NET_Queue.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`client_spawn_manager.h`](client_spawn_manager.h.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`file_transfer.h`](file_transfer.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: packet assembly is byte-budgeted but the layout itself lives in the packet writer.

## Purpose

Four separate jobs share this file because all four are "the level talking to the
transport": establishing a connection and waiting out its handshake, exporting local state
every frame, saving every object's state on demand, and dismantling the world when the
session ends. The teardown is the interesting one — it is the only place that knows the
full list of things that hold references to objects and therefore the order in which they
must let go.

## State

```text
RECORD LevelNetwork
  connect_result_received : bool     # handshake replied at all
  connect_result_accepted : bool     # ...and accepted; results AND together across replies
  connect_result_text     : text     # server's own words, sometimes a string-table key
  game_configured         : bool     # the game mode has been described by the server
  ready                   : bool     # spawns may be accepted
```

**Invariant** — `game_configured` is cleared *after* the world is dismantled, never
before: the dismantling itself pumps the message loop and the loop tests that flag.

## Constants that are budgets, not limits

```text
max_objects_per_update_packet = 2 KB     # per-frame object export
max_objects_per_save_packet   = 8 KB     # save export: fewer, larger packets
connection_timeout            = 60 s
```

The update budget is the smaller of the two because an update packet competes with
everything else on the wire every frame; a save happens once and can afford larger frames.
Both are *reserve* figures: the exporter stops adding objects once fewer than this many
bytes remain, so one oversized object can still overshoot, and the save path asserts that
no single object exceeds the 16-bit chunk length it is written under.

## `dismantle_world`

**Contract** — Destroys every live object and clears every registry that could still name
one. Blocks for as long as it takes. On return the object registry is empty and no
subsystem holds a stale identity.

**Invariants** — Destruction is *event-driven*, not a loop over a list: an object is
destroyed by queueing a destroy event and letting the normal event pump run it, so the
same code path retires an object at session end as during play. That is why the routine
looks like it is spinning.

```text
FUNCTION dismantle_world()
  remember whether distant objects were being drawn as stand-ins; force it on
      # so nothing tries to build a real visual for an object that is about to die

  REPEAT up to 5 times
    IF this process holds the server side: clear the server's world
    IF this process is a client:           queue ownership-rejects then destroys (below)

    REPEAT 20 times
      clear pending sound events
      advance the frame counter by hand
          # the update guard refuses two updates in one frame; we need many
      pump incoming messages
      run the game event queue
      update the object registry once
    IF the registry is empty: BREAK

  clear the bullet manager
  clear both physics command queues, scripted and native
  clear the movement-restriction registry
  clear the animation blend cache
  clear the renderer's model cache and its static decals
  ASSERT the pending-spawn callback registry is empty, then clear it
  collect all script garbage
  destroy all particle emitters
  mark game captions for clearing
```

**Notes** — Five outer attempts and twenty inner pumps are empirical: an object can spawn
another during its own destruction (a corpse dropping its inventory), so one pass is not
enough, and the counts are a bound on that cascade rather than a measured depth. A rebuild
with a deterministic teardown order does not need either number.

Advancing the frame counter by hand is a workaround for a guard that exists to catch a real
bug — an object updated twice in one frame — and a rebuild should instead give teardown its
own update entry point that the guard does not apply to.

The assertion that no spawn callbacks remain is a genuine invariant: a callback keyed by an
entity identifier that outlives the session would fire against a *reused* identifier in the
next one.

## `release_client_objects`

**Contract** — The client half of dismantling. Breaks containment first, then destroys.
Runs to a fixed point: because rejecting a parent's ownership can itself reparent
something, the reject pass repeats until no object has a parent left.

```text
FUNCTION release_client_objects()
  REPEAT
    found_parented = false
    FOR EACH object
      IF object has a parent
        queue an ownership-reject event (parent, object) stamped with server time
        found_parented = true
    run the event queue
  UNTIL not found_parented

  FOR EACH object
    IF it still has a parent: fatal in single player, logged error otherwise
    queue a destroy event for it
  run the event queue
```

**Notes** — Containment must be broken before destruction because a container's destructor
would otherwise destroy its contents out from under the loop that is iterating them. The
asymmetry in the error handling — fatal in single player, a logged error in multiplayer — is
deliberate: in single player a parented leftover is a bug with a reproducible cause; in
multiplayer it can be a message that arrived after the session ended, and killing the
process over it is worse than leaking one object.

## `stop_session`

**Contract** — Ends the session: hides every open dialog, stops any running tutorial
sequence, stops accepting spawns, retires the file-transfer client, finalizes a recording
if one is open, dismantles the world, disconnects the transport and destroys the local
server if there was one.

**Invariants** — The order is load-bearing at two points. The UI is dismissed *first*, so
no widget holds an object it is about to lose. `game_configured` is cleared *after*
dismantling, because dismantling pumps the event loop.

## `send_local_update`

**Contract** — Called once per frame. Exports the locally controlled entity's state to
the server, then, if this process *is* the server, exports every object's state to the
clients in as many packets as it takes. Silently does nothing when the bandwidth throttle
says there is no room — dropping an update is always better than queueing behind one.

```text
FUNCTION send_local_update()
  IF (single player OR we are a pure client) AND no bandwidth available
    RETURN

  IF a locally controlled entity exists, is not dying and is network-relevant
    packet = "client update"
    write its identifier
    write a zero placeholder the transport fills in with our ping
    let the entity export itself
    IF the packet grew past its header, and we are not the server, send it unreliably

  advance any in-flight file transfers and retire stalled receivers

  IF we are a pure client
    flush the send buffer
    RETURN

  # server side: export the world in bounded slices
  cursor = 0
  LOOP
    packet = "world update"
    cursor = export objects from cursor while the packet has room
    IF the packet is still empty: BREAK
    send it unreliably
```

**Notes** — Updates are sent unreliably by design. A lost position update is superseded by
the next one a frame later; retransmitting it would deliver stale data late and cost
head-of-line blocking on everything behind it. Only spawns, events and configuration go
reliably.

The ping placeholder is written by the sender and filled by the receiver so the round-trip
estimate travels with the data it describes rather than in a separate measurement.

## `save_objects`

**Contract** — Writes every save-relevant object's serialized state into a sequence of
packets addressed to the server, which assembles them into the save. Each object is written
as (identifier, 16-bit length-prefixed body), so a reader can skip an object it does not
understand.

**Invariants** — An object's serialized body must fit in 16 bits. The debug build checks
this and fails with the object's name; a rebuild should check it always, because the
overflow is silent and corrupts everything after it in the packet.

## `send`

**Contract** — The one place a packet leaves the level. Chooses among three deliveries
and the choice is invisible to every caller: a same-process server receives the packet by
direct call with no serialization round trip; a hosting client hands it to its own server
through a synchronizing call; anyone else goes out over the transport. Does nothing at all
while a recorded session is being replayed — a replay must not talk back.

**Notes** — The direct-call path is why single player has no network stack in its hot loop
despite being written entirely in client/server terms.

Every send also re-clamps the physics time factor and the constant-frame-rate flag in
non-single-player sessions, which is an anti-cheat measure placed here because this is the
one function every outgoing message passes through.

## `connect_to_server`

**Contract** — Connects and blocks until the handshake resolves or a minute passes. On
timeout it synthesizes a rejection reply so the failure takes exactly the same path as a
real refusal. Returns whether the session may proceed.

```text
FUNCTION connect_to_server(options) -> bool
  result_received = false ; accepted = true

  IF this is a real network session
    compute the content-authentication material over the mounted archives
        # proves the client's data files match the server's

  IF transport connect fails: RETURN false
  IF the server is in this process: result_received = true

  deadline = now + connection_timeout
  WHILE not result_received
    pump incoming messages
    sleep briefly
    IF we also host, let the server run
    IF now > deadline
      synthesize a rejection reply "data verification failed" and handle it
    IF the transport reports the connection failed
      report rejection; disconnect; RETURN false

  IF not accepted
    destroy the local server if any; report rejection; disconnect; RETURN false

  IF the server is in this process: mark synchronized
  ELSE request time synchronization and wait for it, aborting if disconnected
  RETURN true
```

**Notes** — Time synchronization before gameplay is not optional: every event carries a
server timestamp and the event queue orders by it, so a client whose clock offset is
unknown cannot schedule anything.

## `on_connect_result`

**Contract** — Handles the server's verdict. Multiple verdicts may arrive during one
handshake (data check, key check, password) and they combine by AND — any refusal is final.
Adopts the client identity the server assigned. Maps a refusal reason onto a specific
player-facing dialog: version mismatch, key invalid / in use / disabled, wrong password,
banned, or profile error. The ban and profile messages are looked up in the string table so
the server can send a key rather than prose.

If this session is being recorded and the connection was accepted, the server also sends
the option string that describes the session, and recording begins with that string in the
header.

## `on_change_own_name`

**Contract** — Handles the server renaming this client — it may do so to resolve a
collision. Rewrites the name field inside the stored client option string, preserving any
fields that follow it, so that a later reconnect uses the accepted name.

## Connection-failure hooks

**Contract** — `on_invalid_host`, `on_invalid_password`, `on_session_full` and
`on_connect_rejected` each forward to the transport's own handling and then raise the
matching main-menu dialog, but only if no more specific dialog has already been raised —
first diagnosis wins, because it is the most specific.

## `net_update` and the frame hook

**Contract** — Once per frame, if the game is configured, exports local state; if this
process holds the server, runs the server's own update. A small frame-loop participant
registers this, and is itself registered on either the main thread or the worker-thread
frame list depending on a device flag, which is how the network can be moved off the render
thread.
