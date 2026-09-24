# src/xrGameSpy — matchmaking, accounts, and the null path

Chapter 17 of [`SYSTEM-REQUIREMENTS.md`](../../SYSTEM-REQUIREMENTS.md#7-build-order).

## What this module is responsible for

Everything the engine needs from a service it does not run: an account a player logs in
as, a list of servers to pick a match from, a way to reach a server that a router hides,
a record of what a player has done across matches, and a judgement on whether a
connecting client holds a valid product key. Five capabilities, one vendor SDK, one
module — deliberately one module, so that the whole dependency sits behind a single
object ([`GameSpy_Full.h`](GameSpy_Full.h.md)) and can be replaced by nothing.

**The service was shut down in 2014.** Every call in this chapter still executes and every
one of them reaches nothing. That is what this chapter is *for*: it does not document how
the dead service answered — nobody can — it documents **what the game asks a matchmaking
service for**, so a rebuilder can stub it out or design a fresh one, and it documents
**what the engine does today when every one of those calls fails**, because that behaviour
is already written, already shipped, and a rebuild can adopt it without implementing
anything at all. See
[Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts),
whose verdict is *given, and dead*.

Two facts from elsewhere in the recipe set the stakes, and a reader should hold both
before deciding how much of this chapter matters:

- **The shipped build defaults to the null networking filling.** The real client and
  server are commented out of the build description on every platform, so the shipped
  executable has no working transport at all and multiplayer is opt-in at build time. A
  rebuild that wants multiplayer is not restoring a working feature; it is finishing one.
  See the [networking transport seam](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport).
- **The integration was never finished.** Two of the game-side matchmaking callbacks —
  address negotiation and the out-of-band client message — are *empty*, and the callback
  through which the service would have reported the host's own public address (which is
  what would have triggered the server authenticating *itself*) is commented out. The
  authentication path it would have driven still exists here and nothing calls it. See
  [`src/xrGame/gamespy`](../xrGame/gamespy/README.md).

## Where it sits

It rests on almost nothing: the core layer (strings, logging, threads, the settings
store), the game-mode enumeration from [`xrServerEntities`](../xrServerEntities/README.md)
so that a browsed server's mode can be named, and the vendor SDK. It is built before the
game module and after the transport, and it depends on neither — the two touch only
through the game module above them. Nothing in the engine, the renderer, the sound system
or the simulation reaches into this chapter.

Above it sit two pieces of game-side glue: [`src/xrGame/gamespy`](../xrGame/gamespy/README.md),
which answers the directory's questions about a running match, and the account, login and
server-browser screens in the game and UI chapters. Beside it, and sharing nothing but the
dead service, sits the standalone **profile server** in
[`src/utils/mp_gpprof_server`](../utils/mp_gpprof_server/README.md).

**A rebuild's cut line is one object.** Every consumer reaches a service through one of
the five accessors on the facade. Replace that object with one whose five services do
nothing and the rest of the engine is untouched.

## What the game asks of a matchmaking service

Named once here as capabilities, because that is the transferable part; the twins carry
the detail.

**1. Accounts are three names, not one.** An *email* identifies the account and may own
several profiles; a *nick* is a display name and is **not** unique; a *unique nick* is a
claimed-and-owned name within a namespace. The login flow depends on all three existing
separately — *enter email and password → ask the service which nicks this account owns →
pick one → log in with (email, nick, password)* — and a rebuild that collapses them breaks
that screen. A profile may exist without a unique nick, which is why claiming one is its
own operation. Login yields a durable **profile id** and, fetched separately afterwards, a
**login ticket**: a bearer token proving this session authenticated. The service must also
be able to *suggest* free alternatives to a taken name, because only it knows what is
taken. Seven operations in all — see [`GameSpy_GP.cpp`](GameSpy_GP.cpp.md).

**2. The server list is a two-stage fetch over a flat bag of named fields.** A refresh asks
the master list for **eleven fields** across all matching servers — name, address, players,
capacity, level, mode, version, the two gate flags, dedicated, numeric mode — because that
is exactly what the list screen renders plus what decides whether a row is joinable.
Everything else about a server arrives only when the player selects a row and the client
queries that one server directly for its *full* advertisement. The field catalogue is the
most load-bearing page in the chapter ([`GameSpy_Keys.h`](GameSpy_Keys.h.md)): about
thirty standard fields fixed by the service and thirty-odd of this game's own, covering
every multiplayer rule a player filters or chooses on. Per-player and per-team fields are
indexed by appending a row number to the field name. **A rebuild must publish this table
as protocol**, not derive it from shared code — in the original both ends agree only
because they are the same binary.

**3. Filtering happens at the service.** The list query carries a free-text filter
expression over advertised field values, composed by the UI and passed through untouched.
The grammar is the service's and is not recoverable. The *capability* is what transfers: a
popular title's list must be filterable server-side, or every client downloads all of it.

**4. NAT traversal is stated, not implemented.** The client can ask of a listed server
*can I reach this directly, or does it need negotiation?*, and the server has a place for a
negotiation request to arrive. The shipped engine declines at both ends — the flag is
passed off, the callback is empty — so a client behind address translation can find a
server and never join it. Stated as the capability a rebuild owes: *given a server record
from the list, either confirm its advertised address is directly reachable or establish a
path to it, mediated by the service, at the moment of connection.* There is no
hole-punching state machine here, no relay fallback and no address rewriting, so a
rebuild that provides nothing loses exactly what the original loses.

**5. Statistics and persistent storage are a separate service behind the first.** A
logged-in profile authenticates a *second* time, against the statistics service, to obtain
a certificate/private-data pair that authorises submissions; login is not considered
complete until that succeeds, so the second service can veto the first. Before a match,
every participant that intends to report announces itself and says whether it reports
authoritatively; after the match each submits a positional document — a global section
plus one section per player, of typed key/value entries — and the service reconciles. What
it stores per player is **30 awards** (each with a count and a date) and **7 personal best
scores**, and those two shapes are load-bearing because the shipped Lua scripts read them.

**6. Authentication is remote, and it is the only durable player identity.** The server
never sees a product key. It issues an eight-character random challenge, the client
computes a response from its locally stored key, the server forwards the pair to the
service, and the service judges. The service may also demand a *re*-challenge mid-session,
which is the anti-sharing mechanism and the reason the whole thing is challenge–response
rather than a one-time token. On success the server obtains a stable opaque **key hash** —
the only thing a ban list, an admin command or a per-player record can key on, since a
connection index is not an identity. Stated as a capability: *authentication must yield a
stable opaque identifier for the credential presented, the same across sessions and
machines.*

### Where it gates gameplay

Exactly one place. A connecting client is challenged, and is admitted or refused on the
service's verdict, **only when all three of these hold**: authentication initialised
successfully, the server is configured `public`, and this is not a debug build. Miss any
one and the client is admitted unchallenged. Note that the `public` option does double
duty — it is also what decides whether the server is listed at all — so a private server
is also an unauthenticated one, a coupling a rebuild should separate.

## The null path, concretely

This is the behaviour a rebuild can ship without reimplementing anything, because it is
what the engine already does.

**It begins with one deliberate failure.** The reachability probe
([`GameSpy_Available.cpp`](GameSpy_Available.cpp.md)) asks whether the service is live for
this title and gets a three-valued answer: *available*, *retired*, or *down for
maintenance*. Today it answers **retired**, and the distinction matters — the facade
reports a permanent outage to the player exactly **once** and then never complains again,
which would be wrong for a transient one. A rebuild stubbing this out returns *retired*.
This is the only blocking call in the chapter; everything else is polled from the frame
loop.

Then, service by service:

| service | with nothing behind it |
|---|---|
| availability | returns *retired*; startup pauses for one failed probe |
| server list | first refresh yields *master unreachable* once, the unified index **clears**, the menu raises one dialog |
| accounts | the session opens against nothing; no player is ever logged in online |
| statistics | the profile store is **already a stub** — empty award map, empty best-score map, no network at all |
| key authentication | initialisation fails, no challenge is ever sent, **every client is admitted** |
| file download | unaffected; only the *URLs* were the service's |

Three consequences are worth stating plainly because they are decisions, not accidents:

- **Accounts degrade to offline profiles.** The game's account layer already has an
  offline login path that takes a nickname alone and builds a local profile with no profile
  id, no ticket and `online = false` — and lets the player into multiplayer. A rebuild that
  ships no account service and makes every player an offline profile loses nothing the
  engine currently provides. What it loses is what an online profile *carried*: a durable
  id, a globally unique name, and the ticket that authorises statistics.
- **Statistics degrade to zeroes that still work.** Every player has zero of every award
  and a zero personal best, the scripts that read them see empty maps and run unmodified,
  and no match is ever reported. This is the clearest example in the chapter of a null
  implementation that is already written; copy it rather than design one.
- **Authentication degrades to fail-open.** Not fail-closed. The match runs, scores are
  kept, and each client's durable identity is empty rather than a key hash. A rebuild that
  wants a gate must *add* one; it cannot restore this one, and it could not port it even
  in principle — the response function is the service's and is not reconstructible.

Single-player, the local-network server list, and a direct connection to a known address
all continue to work, because none of them passes through this chapter.

## Load-bearing ideas, named once

**Five services, one facade, one poll per frame.** The facade constructs all five
*unconditionally*, including after a negative probe, because each is a local object that
only reaches the network when asked — which is precisely what makes the dead path cheap
and spares every call site a null check. The frame budget for the whole online layer is
**15 ms**, spent only while the main menu is up.

**The status enum is ordered best-to-worst, and two reductions depend on it.** *success →
connecting to master → master unreachable → out of service → unknown.* One master list
failing is not an outage; all three failing is. A refresh reduces optimistically (any list
that started is enough to show "connecting"); a poll reduces optimistically while any list
still works and **pessimistically, emptying the index, when every one has failed**.

**Three titles, one list.** Three games share the advertisement closely enough to be
browsed together, so the client holds three master lists and presents them as one index
space. That unified index is **append-only within a refresh** — the underlying lists
re-sort themselves as records fill in, and a UI holding an index must not have that index
change meaning underneath it. Sorting is the UI's job, over its own copy.

**Errors are reported as localization keys, never as composed text.** The account layer
maps four failure causes to four keys. The availability probe is the one place that breaks
this and embeds English sentences; that is an inconsistency, not a decision.

**Change notifications arrive on the query thread.** Seven distinct reasons collapse to a
single *the list changed*, because the only sane reaction to any of them is to re-read the
list. A single server's query failing does **not** remove it from the list — dropping rows
out from under a browsing player is worse than showing a stale one.

**Two recurring defects a rebuild must not port.** Callback pairs are bundled into context
objects that live on the *calling stack frame* and handed to an asynchronous request that
completes later ([`GameSpy_HTTP.cpp`](GameSpy_HTTP.cpp.md),
[`GameSpy_GCD_Server.cpp`](GameSpy_GCD_Server.cpp.md)); and several failure paths log once
per frame rather than once. Both are visible only on the dead path, which is why they
survived. A rebuild owns its callbacks for the lifetime of the request and rate-limits its
outage logging.

## The twins

Build files (`CMakeLists.txt`, `xrGameSpy.vcxproj`, `xrGameSpy.vcxproj.filters`) describe
how the module is compiled and have no twins.

| File | Role |
|---|---|
| [`xrGameSpy_MainDefs.h`](xrGameSpy_MainDefs.h.md) | The identity this engine presents — three titles and their secrets — plus the port window, the refresh batch size, and the names of the persisted settings. |
| [`GameSpy_Keys.h`](GameSpy_Keys.h.md) | The catalogue of every field a server advertises about itself, its players and its teams. The chapter's most load-bearing page. |
| [`GameSpy_Full.h`](GameSpy_Full.h.md) | Declares the client-side facade over all five online services. |
| [`GameSpy_Full.cpp`](GameSpy_Full.cpp.md) | Brings all five services up, pumps them from the frame loop, and carries the single verdict the menu shows. Where the null path is decided. |
| [`GameSpy_Available.h`](GameSpy_Available.h.md) | Declares the service reachability probe. |
| [`GameSpy_Available.cpp`](GameSpy_Available.cpp.md) | The probe that asks whether the service is answering for this title at all — the call that returns "no" forever now. |
| [`GameSpy_Browser.h`](GameSpy_Browser.h.md) | Declares one server list and the record types a browsed server is delivered as. |
| [`GameSpy_Browser.cpp`](GameSpy_Browser.cpp.md) | One server list against one master list: how a refresh is asked for, how records arrive, and what a server record contains. |
| [`GameSpy_BrowsersWrapper.h`](GameSpy_BrowsersWrapper.h.md) | Declares the multi-master-list aggregate and its state accumulator. |
| [`GameSpy_BrowsersWrapper.cpp`](GameSpy_BrowsersWrapper.cpp.md) | Presents several master lists as one: one index space, one count, one refresh, one verdict. |
| [`GameSpy_GP.h`](GameSpy_GP.h.md) | Declares the account session and its seven operations. |
| [`GameSpy_GP.cpp`](GameSpy_GP.cpp.md) | The account service: create a profile, list an email's profiles, log in, claim a unique nickname, delete. The request catalogue. |
| [`GameSpy_ATLAS.h`](GameSpy_ATLAS.h.md) | Declares the statistics channel: authorisation, session and intent, the report builder, submission. |
| [`GameSpy_ATLAS.cpp`](GameSpy_ATLAS.cpp.md) | Statistics and persistent player storage: what the game records about a match and about a player. |
| [`GameSpy_QR2.h`](GameSpy_QR2.h.md) | Declares the server-side advertiser and the callback set a server must supply for it. |
| [`GameSpy_QR2.cpp`](GameSpy_QR2.cpp.md) | The server's side: register with the master list, heartbeat, answer field-by-field queries, and bind field ids to wire names. |
| [`GameSpy_GCD_Server.h`](GameSpy_GCD_Server.h.md) | Declares server-side key authentication and the two callbacks a verdict arrives through. |
| [`GameSpy_GCD_Server.cpp`](GameSpy_GCD_Server.cpp.md) | Issue a challenge, have the service judge the answer, hand back a stable per-key identity. The one place that gates gameplay. |
| [`GameSpy_GCD_Client.h`](GameSpy_GCD_Client.h.md) | Declares the client-side challenge response. |
| [`GameSpy_GCD_Client.cpp`](GameSpy_GCD_Client.cpp.md) | Turn a challenge into a response without ever putting the product key on the wire. |
| [`GameSpy_HTTP.h`](GameSpy_HTTP.h.md) | Declares the single-slot file downloader. |
| [`GameSpy_HTTP.cpp`](GameSpy_HTTP.cpp.md) | Fetching a patch or a missing multiplayer level, with progress. The one file here with no dead service behind it. |
| [`xrGameSpy.h`](xrGameSpy.h.md) | The module's public face: the three identity queries, plus one include point for every wrapper. |
| [`xrGameSpy.cpp`](xrGameSpy.cpp.md) | Three facts about this copy of the game that go on the wire: version, title id, retail edition. |
| [`stdafx.h`](stdafx.h.md) | Build scaffolding — and the record of eleven names the vendor headers macro-hijacked out from under the engine. |
| [`stdafx.cpp`](stdafx.cpp.md) | Build scaffolding. Does not survive a rebuild. |

## Conformance

The two conformance criteria that exercise multiplayer cannot be met by the original
either, without editing the build description to swap the null networking filling for the
real one. So nothing in this chapter is testable against the shipped executable. The
checks worth carrying into a rebuild are behavioural and all concern the *dead* path,
because that is the path that runs:

- Starting the game with no network must raise the *online services unavailable* dialog
  **once**, not once per frame, and must not prevent single-player, a local-network server
  list, or a direct connection to a known address.
- A failed list refresh must clear the server list and report *master unreachable* once
  per refresh.
- A player must be able to enter multiplayer as an offline profile with no account
  service present, and the scripts that read awards and best scores must run against empty
  maps.

## What could not be recovered

- **The filter expression's grammar** for a master-list query. The engine never composes
  one; it hands the player's typed text straight through. A rebuild designing a fresh
  master server owns this grammar.
- **The challenge–response function itself.** It is the service's, deliberately
  undocumented, and not reconstructible from this repository. This is the one place in the
  chapter where a rebuild cannot reproduce the original's behaviour even in principle, so
  authentication is *replace, not port*.
- **The field ids used inside a statistics report.** Nothing in this repository builds a
  report — the builder is fully wrapped and called from nowhere. Only the *reading* side's
  vocabulary (the 30 awards and 7 best scores) survives.
- **Why NAT negotiation was disabled.** No comment, and no configuration path that turns
  it on.
- **What advertised field 134 was for**, beyond a name suggesting a third-party anti-cheat.
  The server registers no name for it and answers it with an empty value.
- **Which consumer, if any, ever acted on the retail-edition number.** The engine reads it
  from the installer's settings and advertises it; no code path branches on its value.
- **What the three demo title ids corresponded to**, and which service the placeholder
  patching identity was meant to address. Both switches are dead in the shipped build.
- **Whether per-player authentication inside a report was ever intended.** The slot exists
  and the code explicitly submits sixteen zero bytes into it, so the submitter's credential
  covers the whole report. The zeroed buffer is the marker that the original chose neither
  option deliberately.
