# src/xrGame/game_cl_base.cpp

> The client's mirror of the match: it applies the server's player table and clock, reconciles the two into the local world, and announces who joined and left.

**Needs** — [`game_cl_base.h`](game_cl_base.h.md) · [`game_base.h`](game_base.h.md) · [`Level.h`](Level.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`game_sv_mp_vote_flags.h`](game_sv_mp_vote_flags.h.md) · [`ui/UIMessagesWindow.h`](ui/UIMessagesWindow.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`game_cl_base.h`](game_cl_base.h.md)
**Tier floor** — T2: a wire decoder driving a reconciled table

## Purpose

The client side of the rules, shared by every mode. Its real job is narrow: keep a local
table of player records in step with the server's, keep the in-world clock in step with the
server's, and route game-level messages to the screen. Everything that varies between modes
is a hook the derived class fills; this file supplies only the parts that do not vary.

In single player the same class runs with one player and no network — the state import paths
are simply never reached — which is why several branches here read "unless this is single
player".

## State

```text
RECORD ClientGameState                  # extends the game-state base; also a scheduled object
  game_type_name    : text
  players           : map<client_id, PlayerState>   # every player the server has told us about
  local_client_id   : client_id
  local_player      : PlayerState       # NOT owned by the table; see the invariant
  weapon_stats      : WeaponUsageStatistic
  voting_enabled    : int (16-bit)      # a bit per votable subject; 0 disables voting
  server_controls_hits : bool
  game_ui           : optional<screen>
```

**Invariants**

- **The local player's record is owned separately from the table.** It is constructed before
  any network traffic, because the local profile must be loaded from disk to have a name to
  connect with; the table entry for the local client, when it appears, points at that same
  record. Every path that prunes the table therefore has to exempt it explicitly, and every
  path that creates a record from the wire has to recognise the local client and skip
  creating a second one. This is the single most error-prone thing in the file and the
  source of the explicit checks throughout.
- The scheduled update interval is between 5 and 20 milliseconds, which is fast for a
  scheduled object: the rules need to track the match closely, but not per frame.
- Hit arbitration defaults to server-side and voting defaults to fully disabled, so a client
  that has not yet received a state snapshot behaves conservatively.

## `net_import_state`

**Contract** — applies a full snapshot: the local client's identity, the mode, the phase,
the round, the phase start time, the voting mask, the hit-arbitration flag, whether to
collect weapon statistics, then every player record, then the clock. Records present locally
but absent from the snapshot are deleted.

```text
FUNCTION import_state(packet)
  read local_client_id, mode, phase, round, phase_start,
       voting_mask, server_controls_hits, collect_stats
  IF phase differs from ours THEN switch_phase(it)     # fires the mode's phase hook

  count = packet.read_int16
  FAIL WITH too_many_players IF count > max_players
  seen = empty list

  FOR EACH of count records
    id = packet.read_client_id
    IF id is already in players THEN
      remember old flags and old vote
      record.import(packet)
      IF flags changed AND not single player THEN on_player_flags_changed(record)
      IF vote changed THEN on_player_voted(record)
      seen.append(id)
    ELSE IF id == local_client_id THEN
      skip_import(packet)          # our own record exists but is not yet in the table
      CONTINUE                     # note: deliberately NOT added to seen
    ELSE
      record = new player state from packet
      IF not single player THEN on_player_flags_changed(record)
      players[id] = record
      seen.append(id)

  DELETE every table entry whose id is not in seen, except the local player's
  import_clock(packet)
```

**Invariants** — the snapshot is *authoritative over membership*, not merely additive: a
player missing from it has left, and pruning is how a client learns about a disconnect it
missed. The local player's record is exempt from pruning by identity, not by identifier,
which is why the exemption survives the identifier never being added to the seen list.

Flag and vote changes are detected by comparing before and after rather than being signalled
in the packet. That keeps the wire format flat, and it means a change that lands and reverts
within one snapshot interval is invisible — acceptable for flags, and the reason the vote
field is a tri-state rather than a boolean.

**Notes** — the maximum player count is asserted rather than clamped, so a malformed packet
terminates the client. For a protocol with no authentication this is a real exposure; a
rebuild should reject the packet.

Reading the local client's record and discarding it, rather than applying it, means the
local player's server-side view — including his own score — is never taken from the
snapshot. It arrives through the incremental update path instead, once the table entry
exists.

## `net_import_update`

**Contract** — applies one player's incremental record plus the clock. A record for an
unknown client is stepped over rather than being an error.

**Invariants** — the unknown-client case is expected, not defensive: updates travel on the
unreliable channel and connection notices on the reliable one, so an update for a player can
outrun the notice that he exists. Stepping over it correctly requires the skip path to read
exactly the same field sequence the import does.

## `net_import_GameTime`

**Contract** — sets both clocks from the server: the world clock's origin and rate, then the
environment clock's origin and rate. If the environment clock has moved *backwards*, the
weather system is invalidated so it rebuilds its interpolation from the new time.

**Invariants** — only a backwards jump invalidates. Forward motion is what the weather
system already expects; a backwards jump would otherwise leave it interpolating toward a
keyframe that is now in the past. Both clocks are set with the origin-and-rate form, so the
client adopts the server's mapping wholesale instead of trying to converge on it.

## `TranslateGameMessage`

**Contract** — the three membership announcements: connected, disconnected, entered the
game. Each prints a coloured line into the message window and the log. The connect message
also *creates the player record* — recognising the local client and reusing the record
already held — and, outside single player, inserts it into the table and fires the
new-player hook.

**Invariants** — connect is the only message that carries a full account, so it is where a
record is born on the reliable channel. Disconnect and entered-game carry only a name
string, so they cannot create or remove anything; removal happens through snapshot pruning.

An unrecognised message identifier is a fatal assertion, not a skip. Because the messages
are length-prefixed as a group but not individually, a mode that adds a message and a client
that does not know it cannot resynchronise — so failing loudly is at least honest. A rebuild
with self-delimiting messages can and should skip.

**Notes** — the three colours are a neutral grey for the body text and one colour per team
plus a neutral for team-attributed names; only the neutral is used here, the team colours
being for the derived modes. The strings themselves come from the localization table, never
literals.

## `shedule_Update`

**Contract** — picks up the current screen if it was not available at construction, and
while the match is in progress and this is not single player, ticks the weapon statistics.

**Notes** — the screen is looked up lazily every tick because the rules object outlives and
predates the screen; there is no event for "the screen now exists".

## `OnSwitchPhase`

**Contract** — clears the weapon statistics when the match enters the in-progress phase, so
a new round starts from zero.

## the event builders

**Contract** — two pairs, differing only in who the event claims to come from.
`u_EventGen`/`u_EventSend` build a generic event with a caller-supplied type and destination
entity; `sv_GameEventGen`/`sv_EventSend` build a game-level event addressed to nobody. Both
stamp the current server time and send reliably and in order.

**Invariants** — the timestamp is the *server's* time as the client believes it, not the
client's local time, so the server can order events from different clients.

## `SendPickUpEvent`

**Contract** — sends an ownership-transfer event and, locally, **denies touch on the picked
item for one second** so the local senses stop reporting it before the server's confirmation
arrives.

**Invariants** — the local deny is prediction: the client acts as though the pickup
succeeded. One second is the window in which a server refusal can still arrive; after it,
the item reappears if the server said no.

## `OnKeyboardPress` / `OnKeyboardRelease`

**Contract** — swallow all input when there is no local player or the local record is marked
skip. Otherwise pass it on.

**Invariants** — "swallowed" is reported as true. The dedicated server and a client whose
record is not yet valid must not drive an actor, and this is the single gate that enforces
it.

## `set_type_name`

**Contract** — normalizes the mode name through parse-then-format, so that the stored name is
the canonical spelling whatever alias was given, and on a client also publishes it into the
persistent game parameters and signals game start.

**Invariants** — the round trip through the mode identifier is what makes the old and new
name sets (see [`game_base_script.cpp`](game_base_script.cpp.md)) collapse to one spelling.

## `lookat_player`

**Contract** — the record of the player whose body the camera currently follows, found by
matching the current entity's identifier against the body identifiers in the table. Returns
nothing when the camera is on something that is not a player.

**Notes** — this is a linear scan of the table, as is the lookup by body identifier it uses.
With a few dozen players that is fine; a rebuild that expects more should carry the reverse
index.

## `GetPlayerByOrderID` / `GetClientIDByOrderID`

**Contract** — the n-th entry of the table in key order, for the scoreboard which wants a
stable enumeration. Neither checks the index against the table size.

**Invariants** — the ordering is by client identifier, which is stable for the life of a
connection, so a scoreboard built two frames apart lists players in the same order unless
somebody joined or left.
