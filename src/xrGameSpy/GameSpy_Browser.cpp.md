# src/xrGameSpy/GameSpy_Browser.cpp

> One server list against one master list: how a refresh is asked for, how records arrive
> asynchronously, what a server record contains, and how a client reads a field out of it.

**Needs** — [`GameSpy_Browser.h`](GameSpy_Browser.h.md) · [`GameSpy_Keys.h`](GameSpy_Keys.h.md) · [`GameSpy_QR2.h`](GameSpy_QR2.h.md) · [`xrGameSpy_MainDefs.h`](xrGameSpy_MainDefs.h.md) · [`xrServerEntities/gametype_chooser.h`](../xrServerEntities/gametype_chooser.h.md) · [`xrCore/Threading/Lock.hpp`](../xrCore/Threading/Lock.hpp.md) · [`Common/object_broker.h`](../Common/object_broker.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`GameSpy_Browser.h`](GameSpy_Browser.h.md)
**Tier floor** — T2: an asynchronous list with a worker thread, a callback into the frame
thread, and a cache of records read field-by-field. Nothing here needs manual memory or a
fixed layout.

## Purpose

This is the client half of matchmaking: *what servers exist, and what are they like?* It
owns one list, sourced either from a master list over the internet or from a sweep of the
local network, and it exposes that list as a growing indexed sequence plus a change
notification.

It is one-list-per-object on purpose, so that the aggregate in
[`GameSpy_BrowsersWrapper.cpp`](GameSpy_BrowsersWrapper.cpp.md) can hold several — the
client browses three titles at once.

## State

```text
# What the client ends up holding about one server. Every field here is either read out
# of the advertisement (see GameSpy_Keys.h) or derived from the record's envelope.
RECORD ServerRecord
  address           : text        # "host:query_port", the browser's identity for the server
  host              : text        # the host alone
  name              : text        # advertised display name; falls back to the host
  level             : text        # the running level; "Unknown" when absent
  mode_name         : text        # the mode's display name; "Unknown" when absent
  mode              : int         # the mode as a number — see the fallback below
  version           : text        # advertised build version; "--" when absent
  players_now       : int
  players_max       : int         # 32 when the server does not say
  teams             : int
  uptime            : text        # preformatted by the server, not a duration
  dedicated         : bool
  has_password      : bool        # a password gates entry
  has_account_list  : bool        # only listed accounts may enter
  ping              : int         # measured by the query, not advertised
  connect_port      : int         # where a client connects
  query_port        : int         # where the server answers list queries
  index             : int         # position in the owning list; see the invariant

  # Rule settings — present only once the full advertisement has arrived
  max_ping, spectator_modes, frag_limit, artefact_count          : int
  time_limit, damage_block_time, anomalies_time, force_respawn   : real
  warm_up, friendly_fire, artefact_stay, artefact_respawn        : real
  reinforcement                                                  : real
  map_rotation, voting, damage_block_indicators, anomalies       : bool
  auto_team_balance, auto_team_swap, friendly_indicators         : bool
  friendly_names, shielded_bases, return_players                 : bool
  bearer_cant_sprint                                             : bool

  players : list<PlayerRow>
  teams   : list<TeamRow>

RECORD PlayerRow
  name      : text (at most 127 characters, truncated)
  frags     : int
  deaths    : int
  rank      : int    # the advertisement calls this "skill"
  team      : int
  spectator : bool   # defaults to TRUE when absent — see the note
  artefacts : int

RECORD TeamRow
  score : int

ENUM UpdateStatus           # ordered best-to-worst; the aggregate relies on the order
  success
  connecting_to_master
  master_unreachable
  out_of_service
  unknown

RECORD ServerList
  master_list      : (title_short_name, title_secret)   # which title's list this is
  advertiser       : ServerAdvertiser   # a QR2 instance, held only for its field-name table
  on_change        : optional<callback> # fired on the query thread whenever anything moves
  connecting       : bool               # a master-list refresh is in flight
  refresh_failed   : bool               # the last refresh request was rejected; latched
  refresh_lock     : mutex
```

**Invariants**

- A record's `index` is its position in the list *at the moment it was read*, and the
  browser never reorders the list underneath a client. Records are only ever appended.
  This is why sorting is done by the UI over its own copy and not by the list.
- `mode` is authoritative when non-zero. When the server did not advertise the numeric
  mode — which is the case for servers running the two older titles — the client parses
  the display name into a mode number instead. A rebuild designing a fresh advertisement
  should carry the number only, and accept that it then cannot list old servers.
- The rule settings and the player/team rows are only populated once the *full*
  advertisement has arrived. A list refresh asks for a short field set; the rest arrives
  only after an explicit per-server request. Reading a rule setting from a record that has
  not been fully fetched yields a default, not the server's value.

## `RefreshList_Full`

**Contract** — throws away the current list and starts populating a new one, either from
the master list or from the local network. Returns immediately. For the master list it
returns *connecting to master* and completes on a worker thread; for the local network it
completes synchronously and returns *success* or *unknown*. Invalidates every index and
every raw record handle the caller was holding.

```text
FUNCTION refresh(local : bool, filter : text) -> UpdateStatus
  IF the list was never created
    RETURN success                          # nothing to refresh; not an error

  IF the query engine is mid-flight
    halt it                                 # a refresh always supersedes a refresh
  clear the list

  IF NOT local
    LOCK refresh_lock DURING nothing        # see the note — this is a barrier, not a guard
    connecting <- true
    SPAWN worker "GS Internet Refresh"
      refresh_from_master(copy of filter)
    RETURN connecting_to_master

  err <- scan_local_network(ports lan_scan_first .. lan_scan_last,
                            want_notifications = on_change is set)
  IF err
    report err
    RETURN unknown
  RETURN success
```

**Notes**

The empty lock/unlock before spawning is a rendezvous: it blocks until any previous
refresh worker has left the critical section, so that two refresh workers can never be in
flight at once. A rebuild should express that as *await the previous refresh* rather than
as a lock taken and immediately released, which reads like a mistake and is not one.

The filter string is copied into the worker because the caller's buffer is a UI text field
that will change under it. That is the only reason.

### `RefreshListInternet` — the worker body

**Contract** — runs on a worker thread. Issues one master-list query for a **named subset
of fields** and a filter expression, then clears the *connecting* flag. Latches a failure
flag if the query is rejected outright; a query that is accepted and then yields nothing
is not a failure.

```text
FUNCTION refresh_from_master(filter : text)
  LOCK refresh_lock DURING
    fields <- [ hostname, hostport, numplayers, maxplayers, mapname,
                gametype, gamever, password, user_password, dedicated, gametypename ]
    err <- query_master(master_list, fields, filter,
                        want_notifications = on_change is set)
    refresh_failed <- (err is not none)
    connecting     <- false
```

**Invariants** — the eleven-field subset is exactly what the list *screen* shows plus the
two gate flags (password, account list) that decide whether a row is even offered. Asking
for less would leave the list unrenderable; asking for more would make the master list
return a much larger payload for a screen that shows seven columns. **This subset is the
request shape a rebuild must reproduce**: a list query is cheap precisely because it is
partial, and the rest of a server's advertisement is fetched one server at a time, on
demand, when the player selects a row.

The filter is a free-text expression composed by the UI and passed through untouched.
**Could not recover**: the filter expression's grammar. It is the service's, the engine
never constructs one itself, and the UI simply hands the player's typed text to the master
list. A rebuild designing a fresh master-server protocol owns this grammar and should
define it — the capability required is *server-side filtering of the list by advertised
field values*, so that a popular title's list does not have to be downloaded in full.

## `Update`

**Contract** — one poll per frame. Advances the query engine and reports the list's
state. Reports *master unreachable* exactly once per failure — the flag is cleared as it
is read — so that the menu raises one dialog per failed refresh rather than one per frame.

```text
FUNCTION poll() -> UpdateStatus
  advance_query_engine()
  IF connecting     THEN RETURN connecting_to_master
  IF refresh_failed THEN refresh_failed <- false; RETURN master_unreachable
  RETURN success
```

## The change callback

**Contract** — the query engine calls back on its own thread with one of seven reasons.
Four of them — a server's information was updated, a server's update failed, a server was
removed, the engine went idle — are collapsed into a single *the list changed* notification
to the owner. A server merely being *added* raises nothing, because an added server has
only an address and nothing worth showing yet. A query error and a challenge receipt raise
nothing.

**Notes** — collapsing distinct reasons into one notification is the right call and a
rebuild should keep it: the consumer's only sane reaction to any of them is to re-read the
list, and distinguishing them would push protocol detail into the UI. Note that the
notification arrives on the query thread, so everything it reaches must be safe to touch
from there — which is why the aggregate above this one locks.

An update *failure* for a single server is notified but the server is **not** removed from
the list. The removal path exists and is deliberately not wired into the callback: a
server that failed one query is usually still there, and dropping rows out from under a
player mid-browse is worse than showing a stale one.

## `ReadServerInfo`

**Contract** — copies one server's advertisement into a caller-supplied record. Reads the
always-present envelope fields first, then returns early unless the full advertisement has
arrived. Never fails; absent fields become the defaults named in the record above. Does
not allocate beyond the player and team lists, which it clears first.

```text
FUNCTION read_server(record, server)
  address       <- "{public address}:{public query port}"
  host          <- public address
  name          <- field(hostname)      OR host
  level         <- field(mapname)       OR "Unknown"
  mode_name     <- field(gametype)      OR "Unknown"
  has_password  <- field(password)
  has_acct_list <- field(user_password)
  ping          <- measured ping of the last query
  players_now   <- field(numplayers)    OR 0
  players_max   <- field(maxplayers)    OR 32
  uptime        <- field(server_up_time) OR "Unknown"
  teams         <- field(numteams)      OR 0
  connect_port  <- field(hostport)      OR 0
  query_port    <- public query port
  dedicated     <- field(dedicated)
  mode          <- field(gametypename)
  IF mode == 0 THEN mode <- parse_mode_name(mode_name)
  version       <- field(gamever)       OR "--"

  clear players, clear teams
  IF NOT server.has_full_advertisement THEN RETURN      # rule settings stay at defaults

  read the rule settings, but only those the mode can have:
    always            : max_ping, map_rotation, voting, spectator_modes,
                        time_limit, damage_block_time, anomalies, force_respawn, warm_up
    if damage_block_time non-zero : damage_block_indicators
    if anomalies                  : anomalies_time
    deathmatch, team deathmatch   : frag_limit
    team modes                    : auto_team_balance, auto_team_swap,
                                    friendly_indicators, friendly_names, friendly_fire
    artefact modes                : artefact_count, artefact_stay, artefact_respawn,
                                    reinforcement, shielded_bases, return_players,
                                    bearer_cant_sprint

  FOR i IN 0 .. players_now - 1
    append PlayerRow from the indexed per-player fields
  IF the mode has teams
    FOR i IN 0 .. teams - 1
      append TeamRow from the indexed per-team fields
```

**Invariants**

- Reading a rule setting the current mode cannot have is skipped, not defaulted. That
  matters to the UI, which decides which rows of the detail panel to draw from the mode,
  not from whether a value is present.
- The player rows are read using the *advertised* player count, not the length of any
  array. A server that advertises more players than it answers rows for yields rows of
  defaults; a server that advertises fewer hides the rest. There is no cross-check. A
  rebuild should bound this: the count comes from the network.
- A player's *spectator* flag defaults to **true** when the field is absent. That is the
  safe default for the scoreboard — an unknown player is not counted as a competitor —
  and it is the only per-player field whose default is not the zero value.

**Notes** — the *reinforcement* field is read as text first to catch its two sentinel
values before being read as a number, for the reason given in
[`GameSpy_Keys.h`](GameSpy_Keys.h.md).

## `RefreshQuick`

**Contract** — asks one server, directly and not through the master list, for its *full*
advertisement. Returns immediately; the answer arrives through the change callback. This
is the on-demand second half of the two-stage fetch: the list query brought eleven fields,
this brings everything else. Called when the player selects a row.

## `HasAllKeys`

**Contract** — whether the given server's full advertisement has arrived. Returns `true`
for an out-of-range index, so that a caller polling a vanished row stops polling rather
than spinning.

## `CheckDirectConnection`

**Contract** — whether this server can be reached by simply connecting to it, or whether
it sits behind a NAT and needs negotiation. Returns `false` for an out-of-range index.

**Notes** — this single boolean is the entire NAT surface on the client side, and it is
where the chapter's NAT requirement is *stated* rather than implemented. What the game
needs from NAT traversal, as a capability: **given a server record from the list, either
tell me I can connect to its advertised address directly, or establish a path to it.**
The engine asks the question and, today, does nothing with a negative answer beyond
letting the connection attempt fail. See [`GameSpy_QR2.cpp`](GameSpy_QR2.cpp.md) for the
server-side half, which is a stub.

## `GetServersCount` · `GetServerInfoByIndex` · `GetServerByIndex`

**Contract** — the list's size, a record by index, and a raw handle to a server by index.
The raw handle exists so that a caller can read fields the record type does not carry;
it is valid only until the next refresh.

## `GetBool` · `GetInt` · `GetFloat` · `GetString`

**Contract** — read one advertised field off a raw server handle, by field id, with a
caller-supplied default for absence. Each resolves the id to its text name through the
advertiser's registration table before reading, which is why this object holds an
advertiser it never uses to advertise anything.

**Notes** — that the *client* needs a *server* advertiser object purely to look up field
names is the clearest sign that the name table belongs in
[`GameSpy_Keys.h`](GameSpy_Keys.h.md) as data, not in the server-side module as
registration side effects. A rebuild should make the catalogue a value both ends read.

## `Init` · `Clear` · `SortBrowserByPing` · `OnUpdateFailed` · `CallBack_OnUpdateCompleted`

**Contract** — `Init` installs the change callback and marks the list live; `Clear` drops
the callback. `SortBrowserByPing` reorders the underlying list ascending by ping.
`OnUpdateFailed` removes one server from the list. `CallBack_OnUpdateCompleted` walks
every server and reads it into a scratch record.

**Notes** — the last three are all reachable and none is called. Sorting is done by the
UI instead (it owns the sort the player chose); removal is deliberately not wired up, as
noted above; and the walk-everything callback reads each record into a discarded local,
which makes it a no-op with a cost. A rebuild should ship none of the three. They are
recorded here because a reader comparing against the original will find them and wonder.
