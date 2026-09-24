# src/xrGameSpy/GameSpy_Browser.h

> Declares one server list and the record types a browsed server is delivered as.

**Needs** — [`GameSpy_Browser.cpp`](GameSpy_Browser.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md) · [`xrCore/Threading/Lock.hpp`](../xrCore/Threading/Lock.hpp.md) · [`xrCore/_std_extensions.h`](../xrCore/_std_extensions.h.md)
**Used by** — [`GameSpy_Browser.cpp`](GameSpy_Browser.cpp.md) · [`GameSpy_BrowsersWrapper.cpp`](GameSpy_BrowsersWrapper.cpp.md) · [`GameSpy_BrowsersWrapper.h`](GameSpy_BrowsersWrapper.h.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Tier floor** — T3: a declaration surface plus four plain records.

## Purpose

Declares the server-list surface implemented in
[`GameSpy_Browser.cpp`](GameSpy_Browser.cpp.md), together with the records a browsed
server arrives as. The records are written out in full in that file's `State` section,
since their invariants belong with the code that fills them.

Declared here:

- `ServerInfo`, `PlayerInfo`, `TeamInfo` — the client's copy of one server's
  advertisement, its scoreboard rows and its team scores.
- `GameInfo` — a name/value pair. Declared, never constructed anywhere in the tree;
  a rebuild drops it.
- `GSUpdateStatus` — the five-valued list state, ordered best-to-worst. **The order is
  load-bearing**: the aggregate in
  [`GameSpy_BrowsersWrapper.cpp`](GameSpy_BrowsersWrapper.cpp.md) compares these values
  to combine several lists' states, so a rebuild must keep them ordered, not merely
  distinct.
- `SMasterListConfig` — which title's list this object serves: short name and shared
  secret.
- `CGameSpy_Browser` — refresh (internet or local), per-server on-demand detail fetch,
  indexed access, per-field reads, and one poll per frame.

**Notes**

`ServerInfo` compares equal to a bare address string. That is the list's identity rule:
two advertisements are the same server when their `"host:query_port"` strings match — not
their connect ports, and not their names. A rebuild needs the same rule or it will show
duplicates when a server re-registers.

`ServerInfo` is copied by value in several places and carries two growable lists, which
makes it expensive; it is a delivery record, not the list's storage. A rebuild should hand
out a view.
