# src/xrGame/xrGameSpyServer.h

> Declares the multiplayer server variant that registers with the vendor master service, answers its queries, and challenges every client to prove its copy-protection key.

**Needs** — [`xrGameSpyServer.cpp`](xrGameSpyServer.cpp.md) · [`xrGameSpy_GameSpyFuncs.cpp`](xrGameSpy_GameSpyFuncs.cpp.md) · [`xrServer.h`](xrServer.h.md) · [`xrGameSpy/GameSpy_GCD_Server.h`](../xrGameSpy/GameSpy_GCD_Server.h.md) · [`xrGameSpy/GameSpy_QR2.h`](../xrGameSpy/GameSpy_QR2.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`game_sv_base.cpp`](game_sv_base.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md) · [`GameSpy_QR2_callbacks.cpp`](gamespy/GameSpy_QR2_callbacks.cpp.md) · [`xrGameSpyServer.cpp`](xrGameSpyServer.cpp.md) · [`xrGameSpyServer_callbacks.cpp`](xrGameSpyServer_callbacks.cpp.md) · [`xrGameSpy_GameSpyFuncs.cpp`](xrGameSpy_GameSpyFuncs.cpp.md)
**Tier floor** — T1: holds two vendor SDK contexts and passes them raw buffers and C callbacks

## Purpose

Declares the surface implemented across
[`xrGameSpyServer.cpp`](xrGameSpyServer.cpp.md) and
[`xrGameSpy_GameSpyFuncs.cpp`](xrGameSpy_GameSpyFuncs.cpp.md).

**Read the matchmaking seam before this file.** The vendor service it talks to was shut
down in 2014; every path here still compiles and still runs, and reaches nothing. What
survives for a rebuild is the *shape*: a server has a public identity it advertises, and it
gates entry on an identity check it does not perform itself. This file is what the game asks
of such a service, not how the dead one answered.

Two types are declared.

### The extended client record

Adds per-client copy-protection state to the base server's client record:

- the challenge string this client was sent, and whether it has authenticated;
- a re-authentication hint, opaque to the game and handed back to the vendor library;
- a count of suspicious actions, used to ban a client that keeps sending malformed
  authentication traffic.

### The server

Exported units, beyond what the base server declares:

- `Connect(session, description)` — read the public-server options out of the session
  string and bring up the two vendor subsystems.
- `Update` — pump both vendor subsystems each frame.
- `client_Create` — produce the extended client record instead of the base one.
- `OnCL_Disconnected` — tell the authentication service the client is gone.
- `OnMessage` — intercept the one message type this layer owns, the challenge response.
- `NeedToCheckClient_GameSpy_CDKey(client)` — the base server's hook asking whether this
  client must authenticate; sends the challenge as a side effect.
- `Check_ServerAccess(client, reason)` — the access-list gate.
- `Assign_ServerType(result)` — decide at startup whether the server runs with an access list.
- `GetServerInfo(info)` — the console's server description.
- `GetPlayersCount` — the advertised player count.
- `HasPassword`, `HasProtected`, `IsPublicServer` — the three advertised access flags.
- `OnCDKey_Validation`, `OnCDKey_ReValidation` — the authentication service's two callbacks.
- `QR2`, `GCD_Server` — hand out the two vendor contexts, for the free-function callbacks.
- `OnError_Add(error)` — the vendor query library's error sink; deliberately empty.

## State

```text
RECORD GameSpyServer                 # extends the base server
  host_name        : text            # advertised name; defaults to the machine's own name
  map_name         : text            # the level, parsed out of the session string
  password         : text            # empty means none
  server_flags     : int (8-bit, bitset)   # only two bits are used; see Notes
  max_players      : int             # advertised capacity; default 32
  report_to_master : int             # non-zero means advertise publicly
  check_key        : bool            # whether clients must authenticate
  base_port        : int             # the vendor query port
  query_ready      : bool            # the query subsystem came up
  auth_ready       : bool            # the authentication subsystem came up
  query_context    : vendor query context
  auth_context     : vendor authentication context
```

**Invariants** — a subsystem's ready flag is set only if its initialization succeeded, and
every use is gated on it. Both are cleared on shutdown before the context is torn down.

**Notes** — **The access-flag bitset declares eight bits and uses two**: a password bit and
a protected-by-access-list bit. The other six are named only by their position. They are the
fossil of a wider flag set in the vendor protocol; nothing reads or writes them, and a
rebuild should declare the two it needs.

The extended client record is created through a *virtual* creation hook on the base server
rather than by the base server knowing about it. That indirection is the one genuinely
reusable idea in this file: the transport-level client record is extensible by whichever
server variant is running, so an authentication scheme can attach its own per-client state
without the base server knowing what it is.
