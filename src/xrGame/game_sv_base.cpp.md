# src/xrGame/game_sv_base.cpp

> The server's shared rules: load the level's respawn points, hand them out without stacking players, queue every client event for the simulation thread, and run the map rotation.

**Needs** — [`game_sv_base.h`](game_sv_base.h.md) · [`game_base.h`](game_base.h.md) · [`game_sv_event_queue.h`](game_sv_event_queue.h.md) · [`game_sv_item_respawner.h`](game_sv_item_respawner.h.md) · [`Level.h`](Level.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrGameSpyServer.h`](xrGameSpyServer.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`xrEngine/XR_IOConsole.h`](../xrEngine/XR_IOConsole.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [`xrScriptEngine/script_process.hpp`](../xrScriptEngine/script_process.hpp.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`game_sv_base.h`](game_sv_base.h.md)
**Tier floor** — T2: a wire encoder over a client table, with a cross-thread event queue

## Purpose

Everything the server-side rules do that does not depend on which mode is being played:
loading the level's respawn points and item spawn profiles, choosing a respawn point without
stacking players on top of each other, exporting the match state to clients, queueing events
from the network thread, running the map rotation, and resolving duplicate player names.

The mode-specific rules derive from this and fill the veto points; see
[`game_sv_base.h`](game_sv_base.h.md) for the contract they satisfy.

## State

```text
RECORD ServerRules                   # extends the shared game state
  server          : server_handle
  events          : GameEventQueue
  item_respawner  : ItemRespawner
  rpoints         : list<RPoint>[4]      # by team slot
  rpoints_blocked : list<RPoint>
  rpoints_min_dist: real[4]              # half the closest pair's horizontal distance
  map_rotation    : queue<(name, version)>
  rotate_needed, fast_restart, rotation_enabled, map_switched : bool
  round_end_reason: RoundEndReason
  force_sync      : bool

GLOBAL rpoint_freeze_ms  : int        # how long a used respawn point is unusable; default 0
GLOBAL voting_mask       : int        # default: the low eight bits set
GLOBAL server_controls_hits : bool    # default false
GLOBAL collect_statistics   : bool    # default true
```

**Invariants** — the respawn table is indexed by a **team slot**, not by a team, and there are
four slots for at most three teams. Capture the artefact decrements the authored team number
before indexing; the other modes do not. That inconsistency is the same one described in
[`game_sv_capture_the_artefact_process_event.cpp`](game_sv_capture_the_artefact_process_event.cpp.md)
and it originates here.

## `Create`

**Contract** — brings up a match: read the level's respawn data, attach the mode's Lua
process, register console commands, apply an optional server configuration script, and parse
the option string.

```text
FUNCTION create(options)
  clear the item respawner
  IF the level has a game-definition file THEN
    FOR EACH respawn record in its respawn-point chunk
      read position, orientation, team, kind, and a mode mask
      IF the record is an item spawn THEN also read its loadout profile name
      IF the mode mask is not "all modes" THEN
        IF this is capture the artefact AND the record applies to it THEN
          team = team - 1          # the editor numbers CTA teams from one
        IF the mask does not include the mode being played THEN SKIP
      IF the record is an actor spawn THEN
        append it to that team's respawn table
        update that team's minimum-separation values against every earlier point
      ELSE IF it is an item spawn THEN
        register it with the item respawner under its profile

  IF not a dedicated server THEN
    replace the game script process with the one named for this mode in the script
    configuration, if that mode has a section there

  register the mode's console commands
  IF the command line names a server configuration script THEN execute it
  read the option string
```

**Invariants**

- **Respawn points are filtered by mode at load time**, so `rpoints` holds only the points
  this mode may use. A mask of all-ones means "every mode", which is why the filter is skipped
  for it.
- The minimum separation is recorded as **half** the distance between the two closest points,
  in two forms: horizontal only and full three-dimensional. Half, because the value is used as
  a radius around a point and two radii must not overlap. Both are computed; only the
  horizontal one survives into a member, the other into a file-scoped array nothing reads. The
  three-dimensional one is dead.
- The separation values start at a thousand metres, which is "larger than any level", so the
  minimum is correct even with one point.
- The Lua process is replaced, not added to: a mode gets exactly one script process, and a
  mode with no section in the script configuration gets none. A dedicated server runs no game
  scripts at all.

## `ReadOptions`

**Contract** — reads the respawn-point freeze duration (authored in seconds, stored in
milliseconds) and the voting mask from the option string, and executes the persisted map
rotation list.

**Invariants** — the voting mask has a **compatibility conversion**: a value of exactly 1,
which used to mean "voting on", is rewritten to the full mask. Any other non-zero value is
taken as a bitmask. So the option is a mask that treats one legacy value specially.

## the option string

**Contract** — the match's entire configuration is one string of `/name=value` pairs. Three
readers pull an integer, a real or a string out of it by scanning for the key.

**Invariants** — the string reader stops at the next separator, so a value may not contain
one. There is no escaping and no quoting.

The string reader returns a **shared static buffer**, so two options read into two variables
are the same string. Every caller copies immediately; a rebuild returns an owned value.

## `net_Export_State`

**Contract** — the full snapshot sent to one client: the match's shared fields, then every
*ready* player, then both clocks.

```text
FUNCTION export_state(packet, to)
  write to's own client id, the mode, phase, round, phase start,
        the voting mask, the hit-arbitration flag, the statistics flag
  count = number of clients that are network-ready AND
          (not marked skip, OR are the recipient)
  write count
  FOR EACH such client
    write its client id
    IF this record is the recipient's own THEN temporarily set its LOCAL flag
    write the record, with its account
    restore the flag
  export both clocks
```

**Invariants** — **the local flag is per-recipient**, so the same record is exported as local
to one client and non-local to every other. Setting it temporarily around the write and
restoring it afterwards is how one record serves every recipient. A rebuild should pass the
recipient to the serializer instead of mutating shared state.

The count is produced by a **separate pass with the identical predicate**. The two must not
diverge or the receiver reads the wrong number of records; this is the same two-pass hazard
as in [`game_location_selector_inline.h`](game_location_selector_inline.h.md) and has the
same fix — collect once.

A client marked skip is exported to *itself* but to nobody else, which is what lets a client
whose record is incomplete still receive its own identity.

## `net_Export_Update`

**Contract** — one player's incremental record plus both clocks, with the same per-recipient
local flag treatment.

## `net_Export_GameTime`

**Contract** — both clocks, each as an origin and a rate. Written on **every** update, which
the source itself flags as wasteful: the clock changes rarely and a dedicated message would
do. Reproduce it or fix it, but note that the client's decoder expects it after every record
(see [`game_cl_base.cpp`](game_cl_base.cpp.md)).

## `assign_RP`

**Contract** — places an entity at a respawn point of its team, avoiding recently used ones.

```text
FUNCTION assign_respawn_point(entity, player)
  team = entity's team, from whichever teamed kind it is
  FAIL WITH non_teamed_object IF it is neither a spectator nor a creature
  FAIL WITH no_such_team IF team is outside the four slots

  free = the indices of that team's points whose freeze has expired
  IF free is non-empty AND the entity is not a spectator THEN
    choose uniformly among free
  ELSE
    IF the entity is not a spectator THEN unfreeze every point of the team
    choose uniformly among all points
  IF the entity is not a spectator THEN freeze the chosen point for the configured duration
  place the entity at its position and orientation
```

**Invariants**

- **Freezing is the anti-stacking rule**: a point just used is unusable for a configured
  interval, so two players respawning together land apart. When every point is frozen the
  whole team's points are released at once and the choice becomes unconstrained — degrading
  to "anywhere" rather than blocking, because a player must respawn.
- **Spectators neither consume nor respect the freeze.** A spectator occupies no space, so it
  may share a point and must not deny one to a living player.
- The default freeze duration is **zero**, so the mechanism is inert unless the match's
  options turn it on. The shipped default therefore stacks players; the anti-stacking is
  opt-in.

## `spawn_begin` / `spawn_end`

**Contract** — the two halves of creating an entity from a configuration section. The first
builds a server record with no identifier, no parent, no phantom, no respawn of its own, and
a marker saying the caller supplies the position. The second serializes it, feeds it through
the server's spawn handler as though it had arrived on the wire, and destroys the template.

**Invariants** — the same "go out through serialization and back in through the spawn
handler" decision as in
[`game_sv_item_respawner.cpp`](game_sv_item_respawner.cpp.md), and for the same reason:
identifier allocation and the broadcast to clients happen in one place.

## `Update`

**Contract** — per server tick: copy each client's measured ping into its player record, run
the item respawner while a multiplayer match is in progress, and step the mode's Lua process.

**Invariants** — the item respawner runs only in the in-progress phase, so items do not
reappear during the interlude between rounds.

## `OnEvent`

**Contract** — the server's game-level event dispatch. Six kinds:

- **Player connected / disconnected** — forwarded to the hooks.
- **Player killed** — handled by the modes; nothing here.
- **Hit** — resolves the source entity; when it no longer exists (a multiplayer body that has
  already been replaced), falls back to the owning player's *current* body so the hit is still
  attributed. Then runs the mode's hit handler and **rebroadcasts the packet to every client**.
- **Create client** — marks the transport's client connected and attaches it.
- **Player auth** — hands the build-version handshake to the server.
- **Create player state** — builds the player record from the packet, stamps its online time
  and death time, and validates the new player unless this is a direct connection, a demo
  playback, or the dedicated server's own client.

Anything else is a fatal assertion naming the unhandled type.

**Invariants** — the hit is rebroadcast **after** the mode has handled it, and the mode may
have modified the packet in place (see the hit-parameter accessors in
[`game_sv_mp_script.cpp`](game_sv_mp_script.cpp.md)). So the order is the mechanism by which a
mode's damage adjustment reaches the clients.

The source-entity fallback is why the recent-bodies history exists on the player record.

## `CheckNewPlayer`

**Contract** — admits or rejects a joining player, differently for a public and a private
server. A public server requires a logged-in account and rejects a second connection under an
account already present. A private server requires the *opposite* — an account not logged in
— and resolves duplicate names locally instead. A rejection sends an error identifier and
purges the client's queued events.

**Invariants** — the two policies are mutually exclusive by design: a public server
authenticates and so can enforce uniqueness by account; a private one cannot authenticate and
so enforces uniqueness by renaming.

**Notes** — this reaches the matchmaking server type directly and asserts on it, so the rules
depend on a service that no longer exists (see the matchmaking seam). A rebuild replaces the
public-server branch with its own authentication or removes it.

## `CheckPlayerName` / `GenerateNewName` / `FindPlayerName`

**Contract** — ensures no two connected players share a name, by appending or incrementing a
numeric suffix until the name is unique.

```text
FUNCTION unique_name(client)
  name = the account's name, or the transport's name if the account has none
  WHILE some other client has this name
    name = increment_suffix(name)
  set the account's name to it

FUNCTION increment_suffix(name) -> text
  find the last '#' in the name; if there is none, treat the whole name as the stem
  n = the number after it, or 0
  RETURN the stem, then '#', then n + 1
```

**Invariants** — the marker character is a hash, and the number after it is parsed and
incremented rather than appended, so "player#3" becomes "player#4" rather than "player#3#1".
A name that already ends in the marker with no number restarts at one.

The loop terminates because each iteration produces a strictly larger suffix and the client
count is bounded.

**Notes** — if the last character of the name is not the marker, the scan leaves the cursor at
the *last character* rather than at the end, so the stem drops that character and the number
is parsed from nothing. A name not already carrying a suffix therefore loses its final letter
when renamed. That is a real defect; a rebuild appends instead.

## the delayed-event queue

**Contract** — every inbound event is queued rather than handled immediately, and drained once
per tick from the simulation thread.

```text
FUNCTION add_delayed(packet, type, time, sender)
  IF single player THEN queue unconditionally
  ELSE IF the type is one of the six connection-critical kinds THEN queue unconditionally
  ELSE queue only if the sender is not blocked
```

**Invariants** — the six exempt kinds — player started, player ready, the three vote messages,
the auth handshake and the player-state creation — are the ones a client must be able to send
*while* it is blocked, because being blocked is a state it gets into and out of during
connection. Everything else a blocked client sends is dropped at arrival.

Draining processes events in arrival order until the queue is empty, which means a burst is
handled entirely within one tick.

## the three purges

**Contract** — three predicates for dropping queued events: every event naming a given entity
as its first field (used when a player is removed and his queued kills and hits must not be
applied to whatever takes his identifier), every event from a given client, and everything.

**Invariants** — the by-entity predicate reads the packet's first field and **restores the
read cursor**, since the event may still be delivered. Only the kill and hit kinds are
examined, because only they begin with an entity identifier — the predicate must know the
layout of each event it inspects, which is the cost of not having typed events.

## the map rotation

**Contract** — a list of (map, version) pairs. Adding one enables rotation only when the list
has more than one entry. Listing prints them with the first marked current. At shutdown the
list is written back as a script of console commands, which is re-executed at the next match's
option parse — so the rotation persists across restarts as executable text.

**Invariants** — **saving destroys the list**: entries are popped as they are written. The
only caller is the destructor, so it does not bite, but the function is not idempotent.

## `OnRoundStart` / `OnRoundEnd`

**Contract** — round start clears the rotation and fast-restart intents and unblocks every
respawn point. Round end sets the rotation intent unless the round ended because the game was
restarted, and sets the fast-restart intent for the fast flavour.

**Invariants** — the reason a round ended decides whether the map rotates. A restart stays on
the same map; a limit reached moves on.

**Notes** — the unblocking loop **copies each point and clears the copy's flag**, leaving the
real points blocked. The source marks it as a known defect with the correct line beside it.
Reproduce the correct behaviour: clear the flag on the point itself. The blocked list is
cleared regardless, so the two representations disagree after a round.

## `on_death`

**Contract** — records the killer's identifier on the victim's server record, asserting the
victim had none. The single place a death's attribution is written on the server side.

## the small accessors

- **`get_id`** — a client's player record.
- **`get_eid`** — the player record owning a body. Tries the direct link first, then falls
  back to scanning every client for one whose *recent bodies* include it. That fallback is why
  a late message still resolves.
- **`get_client`** — the same search, returning the client rather than the record.
- **`get_alive_count`** — how many players of a team are not permanently dead.
- **`get_children`** — the inventory of a client's body.
- **`getRP` / `getRPcount`** — bounds-checked access to the respawn table, returning a default
  point rather than failing; these are the script-facing forms.
- **`signal_Syncronize`** — raise the flag that forces a full snapshot on the next tick.
- **`u_EventGen` / `u_EventSend`** — build an event stamped with server time and broadcast it
  reliably.

## the empty defaults

**Contract** — persistence (level change, save, load, reload, distance switch), the four
movement-restriction operations, the object lifecycle notifications, the render hook, the
console-command registration pair and several membership hooks all default to doing nothing.
Single player fills the persistence ones; the multiplayer modes fill the rest. The empty
defaults are what let a minimal mode exist.

**Notes** — the debug renderer's respawn-point and living-player overlays are entirely
commented out, having been written against a player-lookup call that no longer exists. What
they drew — a vertical line and a sphere per respawn point, coloured by team, skipping blocked
ones — is worth restoring in a rebuild, because respawn placement is otherwise invisible.
