# src/xrGameSpy/GameSpy_Full.cpp

> The client-side facade: brings up all five online services at once, pumps them from the
> frame loop, and carries the single verdict the menu shows the player. This is where the
> null path is decided.

**Needs** — [`GameSpy_Full.h`](GameSpy_Full.h.md) · [`GameSpy_Available.h`](GameSpy_Available.h.md) · [`GameSpy_HTTP.h`](GameSpy_HTTP.h.md) · [`GameSpy_BrowsersWrapper.h`](GameSpy_BrowsersWrapper.h.md) · [`GameSpy_GP.h`](GameSpy_GP.h.md) · [`GameSpy_ATLAS.h`](GameSpy_ATLAS.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`GameSpy_Full.h`](GameSpy_Full.h.md)
**Tier floor** — T3. It owns five objects and calls each one's poll once per frame.

## Purpose

A client needs five distinct online capabilities and needs them to behave as one thing:
a probe that says whether any of it works, a file fetcher, a server list, an account
session, and a statistics channel. This file owns all five, decides the order they are
polled in, and collapses their state into one enum the menu can render.

It exists as a separate object — rather than five fields on the menu — because the menu is
rebuilt whenever the UI is reloaded and the online session must survive that. Its
lifetime is the whole non-dedicated process: it is created when the main menu is created
and destroyed with it, and a dedicated server never creates it at all.

## State

```text
RECORD OnlineServices
  probe            : AvailabilityProbe     # GameSpy_Available
  downloader       : FileDownloader        # GameSpy_HTTP
  server_lists     : ServerListAggregate   # GameSpy_BrowsersWrapper
  account          : AccountSession        # GameSpy_GP
  statistics       : StatisticsChannel     # GameSpy_ATLAS
  services_checked : bool                  # see the invariant below
```

**Invariant** — `services_checked` is not "did we check"; it is *"is the player done being
told"*. It is set to the probe's verdict at construction, and the first poll that sees it
false reports the outage once and then forces it true forever. A permanent outage must
produce exactly one dialog, not one per frame.

## `construct`

**Contract** — runs the reachability probe first and *blocks* on it (see
[`GameSpy_Available.cpp`](GameSpy_Available.cpp.md)), then brings up the shared service
core and constructs the four live services unconditionally — including when the probe
said the service is dead. Allocates. Called once, from the main menu's construction, on
the main thread, and therefore lengthens startup by one network round trip.

```text
FUNCTION construct() -> OnlineServices
  probe <- new AvailabilityProbe
  services_checked <- probe.check_available_services().is_ok

  core_initialize()                 # shared transport/thread pool for the services below
  downloader   <- new FileDownloader
  server_lists <- new ServerListAggregate
  account      <- new AccountSession
  statistics   <- new StatisticsChannel
  RETURN the five
```

**Notes**

Constructing the four services after a negative probe is deliberate, not an oversight:
each one is a *local* object that only reaches the network when asked, so building them
costs nothing, and the code downstream is spared a null check on every use. A rebuild
should keep that shape — a null implementation of each service, always present — because
it is exactly what makes the dead-service path cheap.

Blocking startup on a probe is the part a rebuild should *not* keep. The verdict is not
needed until the player opens a multiplayer screen.

## `Update`

**Contract** — one call per frame, only while the main menu is on screen or a download is
running. Returns a single status describing the whole online layer. Does not block:
every service's poll is expected to return promptly. Must run on the frame thread, since
its result drives UI.

```text
ENUM UpdateStatus          # ordered best-to-worst; the order is load-bearing, see below
  success
  connecting_to_master
  master_unreachable
  out_of_service
  unknown

FUNCTION update() -> UpdateStatus
  IF NOT services_checked
    services_checked <- true        # report the outage exactly once, then never again
    RETURN out_of_service

  downloader.poll()
  status <- server_lists.poll()     # the only service whose state the player sees
  account.poll()
  core_poll(budget = 15 ms)         # shared core; the budget is a ceiling, not a sleep
  statistics.poll()
  RETURN status
```

**Invariants** — the returned status comes from the server-list aggregate alone. The
other three services report their failures through their own callbacks and log lines, not
through this value; a failed download or a failed login does not make the online layer
look broken.

**Notes** — the ordering is not arbitrary. The shared core's poll is given a millisecond
ceiling and is placed *after* the three services that enqueue work into it and *before*
the statistics channel, so work queued this frame is carried within the same frame rather
than waiting for the next. The 15 ms figure is a quarter of a 60 Hz frame and is the one
number in this file a rebuild should think about: it is the online layer's whole frame
budget, and it is only ever spent while the menu is up.

**The null path, concretely.** With no service reachable — which is the state of the world
today — the sequence a player sees is: the menu opens, startup pauses for one failed
probe, the first frame of the menu raises the *online services unavailable* dialog once,
and thereafter `update` returns whatever the server-list aggregate reports, which
degenerates to *master unreachable* on the first refresh attempt and *success* (over an
empty list) when idle. Nothing else changes. Single-player, the local-network server
list, and a direct connection to a known address all continue to work, because none of
them passes through this file.

## `CoreThink`

**Contract** — advances the shared service core for at most the given number of
milliseconds. Exposed separately because the account and statistics screens pump it
directly while they are waiting on a long operation and the frame loop is not running.

## `destruct`

**Contract** — destroys the five services in creation order, then shuts the shared core
down. The ordering matters: the core owns worker threads that the services' pending
requests are queued on, so no service may outlive it, and the core may not be torn down
while a request is still registered.
