# src/xrGameSpy/xrGameSpy_MainDefs.h

> The identity this engine presents to a matchmaking service, and the fixed numbers the
> rest of the chapter is built on: three game titles, the port window a server advertises
> on, and the names of the per-machine settings the account layer persists.

**Needs** — [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`RegistryFuncs.cpp`](../xrGame/RegistryFuncs.cpp.md) · [`GameSpy_ATLAS.cpp`](GameSpy_ATLAS.cpp.md) · [`GameSpy_Available.cpp`](GameSpy_Available.cpp.md) · [`GameSpy_Browser.cpp`](GameSpy_Browser.cpp.md) · [`GameSpy_BrowsersWrapper.cpp`](GameSpy_BrowsersWrapper.cpp.md) · [`GameSpy_GCD_Server.cpp`](GameSpy_GCD_Server.cpp.md) · [`GameSpy_GP.cpp`](GameSpy_GP.cpp.md) · [`GameSpy_QR2.cpp`](GameSpy_QR2.cpp.md) · [`xrGameSpy.cpp`](xrGameSpy.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Tier floor** — T3: it is a table of constants. Nothing about it constrains the language.

## Purpose

A matchmaking service does not know what a game *is*; it knows a short title code and a
shared secret that proves the client belongs to that title. This file is where those
identities live, together with the handful of numbers every other file in the chapter
reads: which port range a server heartbeats from, how many server records a single list
refresh is allowed to fetch at once, and the names under which the account layer stores a
player's credentials on the machine.

It is a separate file purely so that the three titles can be edited in one place. A
rebuild would reasonably make it a configuration section rather than a compiled table —
nothing here is a code decision, and the engine already reads far more volatile things
from `ltx`.

## State

```text
# One entry per shipped game title. A client can browse all three at once (see
# GameSpy_BrowsersWrapper.cpp), which is why this is a set and not a single value.
RECORD TitleIdentity
  short_name  : text    # the service's name for the title, e.g. "stalkercoppc"
  secret_key  : text    # shared secret proving the caller is this title's client
  numeric_id  : int     # the title's numeric id, used by key authentication and statistics
  product_id  : int     # the account namespace's id for the title's storefront product
  version     : text    # the build version advertised in every server record

SHIPPED TITLES
  Call of Pripyat : short_name "stalkercoppc", secret "LTU2z2",  id 2760, product 11994, version "1.6.02"
  Clear Sky       : short_name "stalkercs",    secret "PQ7tFU"
  Shadow of Chern.: short_name "stalkersc",    secret "t9Fj3Mx"
```

Only the first title carries a numeric id, a product id and a version: this executable
*is* Call of Pripyat, so it authenticates keys, submits statistics and advertises itself
as that title. The other two entries exist only so the server list can be queried on their
behalf — a client may browse servers of all three titles from one screen, because the
three games share a wire protocol closely enough that the list is meaningful.

```text
RECORD ServiceConstants
  account_namespace     : int  = 1      # which namespace of the account service owns nicknames
  advertise_base_port   : int  = 5445   # first port a server heartbeats and answers queries on
  list_refresh_batch    : int  = 20     # server records a list refresh may have in flight at once
  port_range_min        : int  = 0
  port_range_max        : int  = 65535
  lan_scan_first        : int  = 5445   # == advertise_base_port
  lan_scan_last         : int  = 5695   # advertise_base_port + 250
```

**Invariant** — the local-network scan sweeps a *closed* window of 251 ports starting at
the advertise base port, not the whole port space. The comment in the source gives the
reason: the query library will only process 500 ports in one sweep, and 251 leaves margin.
A rebuild designing its own discovery can pick any window; it must still pick a bounded
one, because the scan is a broadcast per port.

```text
# Names under which per-machine, per-installation values are persisted. On the original
# these are entries under a per-title key in the operating system's settings registry.
RECORD PersistedSettingNames
  cd_key           : "InstallCDKEY"        # the product key, written by the options screen
  version          : "InstallVers"
  user_name        : "InstallUserName"
  sku              : "InstallSource"
  patch_id         : "InstallPatchID"      # read back as the "distribution" number
  language         : "InstallLang"
  account_email    : "GPUserEmail"
  account_password : "GPUserPassword"      # stored obfuscated, not encrypted — see login_manager
  remember_account : "GPRememberMe"
```

**Notes**

The registry is the wrong home for all of this and a rebuild should not reproduce it: a
product key and a saved password belong in the engine's own per-user settings file, which
already exists (`$app_data_root$`). The only reason they live in a machine-wide
installation key is that the original installer wrote them there, and the engine reads
what the installer wrote. Everything outside the product key is written by the engine
itself and has no external reader, so it can move freely.

The two build-time switches in this file — a demo build that swaps in a different
installation key path and three alternative title ids, and a patching identity pair
(`"test_version_1"`, distribution `0`) used by the self-update path — are both dead. The
demo switch is explicitly undefined immediately after being offered, and the patching
identity is a placeholder string that was never replaced. **Could not recover**: what the
three demo title ids (1067, 1576, 1620) corresponded to, or which service the patching
identity was meant to address.
