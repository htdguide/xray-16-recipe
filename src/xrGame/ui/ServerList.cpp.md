# src/xrGame/ui/ServerList.cpp

> The server browser: what a master server is asked for, how a list of hundreds is rebuilt
> without churning widgets, and how joining a server becomes a console command.

**Needs** — [`ServerList.h`](ServerList.h.md) · [`UIListItemServer.h`](UIListItemServer.h.md) · [`UIMessageBoxEx.h`](UIMessageBoxEx.h.md) · [`TeamInfo.h`](TeamInfo.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`ServerList.h`](ServerList.h.md)
**Tier floor** — T2: widget pooling over an asynchronous external source

## Purpose

The browser is the single largest consumer of the matchmaking seam, and since that service is
dead its main value to a rebuild is as a **specification of what a replacement must answer**.
Everything listed in the detail pane below is a field the original's master server supplied
per server; a new protocol that omits one loses a line of the interface.

The file also holds two pieces of engineering worth keeping regardless of the service: the row
pool, and the deferred refresh.

## State

```text
RECORD ServerBrowser
  lists      : list box [3]      # servers, server properties, players
  headers    : button [7]        # six sortable, one not
  filters    : ServerFilters
  player_name: text              # used when composing the join command
  sort_mode  : enum; sort_ascending : bool

  row_pool   : list<(row widget, in use)>   # recycled across refreshes
  pool_hint  : int               # where the last allocation stopped

  remembered_selection : int     # a row tag, restored after a refresh
  refresh_at_frame     : int     # deferred-refresh stamp, or absent
  expanded             : bool    # is the detail pane showing
  animating            : bool
  list_height[2], edit_y[2]      # collapsed and expanded geometry, from the document
  subscription         : handle  # to the browser service's update callback
```

## The deferred refresh

**Contract** — `RefreshList` does not refresh. It **stamps the current frame number**, and the
per-frame update performs the rebuild when the stamp is more than a few frames old.

```text
FUNCTION request_refresh()      # every caller uses this
  refresh_at_frame = current frame

FUNCTION update()               # every frame
  IF refresh_at_frame is set AND current frame is past it (plus slack)
    rebuild the list
```

**Invariants** — The service delivers updates one server at a time and can deliver hundreds in
one burst. Rebuilding on each would sort and re-lay the whole list hundreds of times in a
frame. The stamp coalesces the burst into one rebuild, and the slack is what lets the burst
finish first.

The stamp comparison in the original is written in a way that also fires when nothing has been
requested, so the rebuild runs more often than intended; a rebuild should compare against an
explicit "no request pending" sentinel.

## The row pool

**Contract** — Row widgets are **never destroyed between refreshes**. A rebuild marks every
pooled row free, then hands them out again in order; a row is only allocated when the pool
runs dry, and the pool only grows.

```text
FUNCTION rebuild()
  remember the selection
  clear the list; mark every pooled row free

  indices = 0 .. server count - 1
  sort indices by the current key and direction     # comparison fetches both servers' info
  FOR EACH index
    info = fetch the server's info
    IF the filters admit it
      row = the next free pooled row (or a new one)
      fill it from info; add it to the list

  re-apply the collapsed/expanded geometry
  restore the selection
```

**Invariants** — The pool's allocation cursor resumes where it left off rather than scanning
from the start, which makes a full refresh linear rather than quadratic in the row count.

The sort runs over **indices**, fetching each server's information inside the comparison. That
is expensive — a fetch per comparison rather than per server — and is the original's doing; a
rebuild should fetch once into an array and sort that.

## `SaveCurItem` / `RestoreCurItem` / `ResetCurItem`

**Contract** — The selection survives a refresh because it is remembered by the row's **tag**
— a stable identifier carried in the row — not by index. After the rebuild the list is asked
to select that tag and to scroll it into view. A refresh triggered by a new query instead
resets the selection and scrolls to the top, because the old selection is meaningless against
a new result set.

## `SetSortFunc_internal` — the auto direction

**Contract** — Set the sort key and direction.

```text
FUNCTION set_sort(key, direction, resort)
  CASE direction
    ascending  -> sort_ascending = true
    descending -> sort_ascending = false
    auto       -> IF key = current key THEN flip sort_ascending ELSE sort_ascending = true
  current key = key
  IF resort THEN request a refresh
```

**Invariants** — *Auto* is what a column-header click means, and the rule — same column flips,
different column starts ascending — is the behaviour every table in every application has. The
explicit directions exist so that a fresh query can force ping-ascending without the toggle
interfering.

The name-to-key mapping (`server_name`, `map`, `game_type`, `player`, `ping`, `version`) is
exposed for scripts and is frozen; an unknown name is fatal.

## `IsValidItem` — the client-side filters

**Contract** — Whether a server passes the filter set. Six tests, all conjunctive, plus one
unconditional:

```text
FUNCTION admits(server) -> bool
  IF server.port = 0 THEN RETURN false     # a server that never answered a query

  # Each filter is "if this filter is OFF, admit; if ON, require the
  # matching property". A filter that is off never excludes.
  ...empty, full, with password, without password, without friendly
     fire, listen servers...
```

**Invariants** — The port test is not a filter, it is a validity check: the service returns
entries for hosts it has heard of but never successfully queried, and those must not be shown.

The "empty" and "full" filters are *show only* filters despite their names, and the two
password filters are mutually exclusive when both are on, yielding nothing. Both quirks are
in the original and visible in the interface.

## `FillUpDetailedServerInfo` — the specification of a master server

**Contract** — Fill the two detail lists for the selected server. This is the list of
everything a master server must report, and it is the most useful part of the page.

**The player list.** When the server has exactly two teams, players are grouped under three
headings — team one, team two, spectators — each heading inserted lazily before its first
member, and the two team names coming from the shared team lookup
([`TeamInfo.cpp`](TeamInfo.cpp.md)). Otherwise players are listed flat. Every row is name,
frags, deaths, with column widths taken from the three header frame widgets so the rows line
up under them.

**The properties list**, in order:

```text
  server name, server version
  maximum ping, map rotation, voting enabled
  the four spectator modes (free fly, first person, look at, free look),
    plus a team-only mode for team game types
  frag limit, time limit
  damage-block indicators, damage-block duration
  anomalies enabled, anomaly period (or "infinite" when the period is zero)
  forced respawn delay, warm-up duration
  for team game types:
    auto team balance, auto team swap, friendly indicators, friendly
    names, friendly fire percentage
  for artefact game types:
    artefact count, artefact stay time, artefact respawn time,
    player respawn rule, shielded bases, return players,
    artefact bearer cannot sprint
  server uptime
```

**Invariants** — Which properties appear **depends on the game type**, so the master server
must report the game type before the rest can be interpreted. Two values carry out-of-band
meanings: an anomaly period of zero means "never", and a respawn reinforcement of −1 means
"only when an artefact is captured" while 0 means "at any time".

Every label goes through the localization string table; every boolean is rendered as one of
two localized word pairs — enabled/disabled, or yes/no — chosen per property. Which pair a
property uses is a presentation decision made per line, and one property (voting) is
mistakenly added twice with both pairs.

## `ConnectToSelected` — joining

**Contract** — Four gates before a join, in order:

```text
FUNCTION connect_to_selected()
  IF the product key does not validate THEN RETURN
  IF logged in AND the account nickname is unregistered or expired
    report through the connection-error callback; RETURN
  IF the server is behind a firewall with no direct route
    log and RETURN                        # NAT negotiation is not attempted
  IF the server's version differs from ours
    show the version-mismatch dialog; RETURN

  IF the server needs a password or a per-user password
    open the password dialog in the matching mode
  ELSE
    compose and execute the join console command
```

**Invariants** — The join is a **console command string** composed from the server's address,
the player's name and the passwords — the same channel the vote dialogs use. Nothing in the
browser opens a socket; the console command is the whole of the handover to the transport
seam.

The version check is exact string equality and has no compatibility window, which is why a
point release of the game partitioned its own player base.

The password dialog's confirmation re-composes the command with the entered passwords; the two
password kinds (a shared server password and a per-user one) are independent and a server may
require either, both or neither.

## The expandable detail pane

**Contract** — `ShowServerInfo` toggles the pane. The screen has two authored geometries — a
tall server list, and a short one with the detail pane below — and both the list's height and
the filter box's vertical position come in pairs from the layout document.

The toggle is written as an animation with a before/after step on each side, but the animation
was removed: the "is it done yet" test is a constant true, so the transition completes in one
frame. The four-step structure survives and is worth keeping in a rebuild that wants to
restore the animation.

Expanding also triggers a targeted re-query of the selected server when the browser does not
yet hold all of its keys, because the summary the list needs is smaller than the detail the
pane needs.

## `InitFromXml`

**Contract** — Build every widget from the document under one path prefix, and read the seven
column widths from a `sizes` element. Column widths are authored once and then applied to both
the header buttons and the row template, which is what keeps the header aligned with the rows.
Both the collapsed and expanded geometries are read here.

## `InitHeader`

**Contract** — Lay the seven column headers left to right by accumulating the authored widths,
set each one's caption, and position a frame line under each at the same width. The first
column — the icon column — is **disabled**, because it has nothing to sort by.

**Notes** — The six captions are set as literal English strings rather than localization
identifiers. That is a defect, not a decision, and it is visible in every localized build.

## `SrvInfo2LstSrvInfo`

**Contract** — Flatten one server's reported information into the row's display record: name,
address composed as `<host>/port=<port>`, map, abbreviated game type, a players fraction, a
ping, a version, and four boolean icon flags (password, dedicated, anti-cheat, per-user
password). The anti-cheat flag is hardwired to false — the feature was removed and the icon
column kept.

**Invariants** — The address is composed into the form the join console command expects, here
rather than at join time, which is why this function is on the display path at all.

## Construction and the subscription

**Contract** — On construction the browser **subscribes to the service's update callback**,
with the handler being the deferred-refresh request. Destruction unsubscribes — guarded,
because the service object can already be gone when the game is shutting down and the menu is
destroyed after it.

**Invariants** — The guarded teardown is the one piece of lifetime discipline here and it is
necessary: the browser outlives its data source in exactly one case, process exit.
