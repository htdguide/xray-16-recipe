# src/xrGame/xrServer_Connect.cpp

> Brings a server up for one session — choosing the rules from the session string — and walks a newly arrived client through the gates it must pass before it is admitted.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`game_sv_single.h`](game_sv_single.h.md) · [`game_sv_deathmatch.h`](game_sv_deathmatch.h.md) · [`game_sv_teamdeathmatch.h`](game_sv_teamdeathmatch.h.md) · [`game_sv_artefacthunt.h`](game_sv_artefacthunt.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`file_transfer.h`](file_transfer.h.md) · [`screenshot_server.h`](screenshot_server.h.md) · [`xrNetServer/NET_AuthCheck.h`](../xrNetServer/NET_AuthCheck.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: parses a session string into fixed buffers and sends a raw struct as the first message

## Purpose

Two separate subjects share this file because both are "connect": the *server* connecting —
that is, starting a session — and a *client* connecting to it. They are unrelated and a rebuild
should split them.

The first is where the game mode is chosen, and the choice is made by **parsing a string**.
That is the load-bearing shape: a session is fully described by one option string, which the
console, the menu and the command line all produce, so starting a deathmatch from a shortcut
and from the menu go through exactly the same path.

## State

`Stateless.`

## `Connect` (the server starting a session)

**Contract** — parse the session string, instantiate the game mode it names, bring up the
multiplayer support subsystems, fill in the description a joining client receives, and hand off
to the transport's own connect. Fails when the string has no options part or names an unknown
mode.

```text
FUNCTION connect(session_string, OUT description) -> result
  IF session_string has no '/' THEN FAIL WITH connect error

  options   := everything after the first '/'
  type_name := options up to the next '/'          # the game mode's name

  game := instantiate(class identifier for type_name)   # through the entity factory
  IF game is none THEN FAIL WITH connect error

  IF the mode is not single-player
    create the file-transfer site
    create the screenshot proxy pool
    load the server's description, logo and rules
    arm the data-authenticity check over the mounted archives

  description.map_name     := game.level_name(session_string)
  description.map_version  := parse_level_version(session_string)
  description.download_url := map_download_url(map_name, map_version)

  game.create(session_string)              # the mode reads its own options
  RETURN transport.connect(session_string, description)
```

**Invariants** — the session string's grammar is `<level>/<mode>/<options…>`, and both
separators must be present for a multiplayer session. The two length assertions on the
intermediate buffers are the C++ artefact of parsing into fixed storage; a rebuild parses into
whatever it likes, but the *grammar* is frozen because shipped shortcuts and configuration
files contain these strings.

**The game mode is instantiated through the same entity factory that creates entities**, by a
class identifier derived from its name. So adding a game mode is a factory registration, not a
change here — which is why this file names four modes in its dependencies and mentions none of
them in its code.

**Notes** — the data-authenticity check is armed only for multiplayer, and it is what lets the
server later assert that a client's game data has not been edited. Arming it is expensive
enough that single player skips it.

The download location comes from the level's own archive header, not from server configuration
— so the *level* says where copies of it can be found. A level whose header has no such entry
yields an empty location and a client that lacks it simply cannot join.

## `get_map_download_url`

**Contract** — the location a client can fetch a named level and version from, read from that
level's archive header. Answers an empty string when the level has no header, warning in
multiplayer and staying silent in single player — where levels are never downloaded.

## `new_client`

**Contract** — build a server-side record for a newly connected transport identifier. Copies the
identifier, the reported process identifier, the name and the password from the transport's
connect data, then **queues a client-creation event on the game rules' own queue** rather than
setting the client up here.

**Notes** — the deferral is the point: the rules layer decides what a client *is* — which team,
which spawn, which state — and it must do that on the simulation thread. The name is marked as
meaningful only in offline play, because online the authoritative name comes from the player's
account.

## `AttachNewClient`

**Contract** — the first thing sent to a connected client, and the branch point of the whole
admission sequence.

```text
FUNCTION attach_new_client(client)
  handshake := two fixed signature words        # see Notes

  IF the build is in direct-connect mode         # single player
    this client IS the host client; mark it local
    deliver the handshake in-process
  ELSE
    send the handshake
    decide whether this client is the host's own, by process identifier

  IF NOT this client must authenticate its copy-protection key
    admit it straight away
  # otherwise the authentication layer has already sent a challenge,
  # and admission happens when the answer comes back

  clear the client's identity string
```

**Invariants** — the handshake is **two fixed 32-bit words sent as a raw structure**, and they
are the protocol's magic numbers: a peer that does not send exactly these is not speaking this
protocol. The values are dates — the authors' — which is as arbitrary as a magic number gets and
entirely fit for purpose.

**Notes** — the two branches differ in more than delivery: direct-connect mode *declares* the
client to be the host rather than checking, because in single player there is exactly one. The
network path checks.

The admission fork is the interesting structure: **admission is either immediate or resumed by a
callback**, and both paths end at the same admit function. Every gate added later — key
authentication, build version, ban check — hangs off that pattern, which is what keeps a
multi-step, asynchronous admission readable.

## `RequestClientDigest`

**Contract** — ask a client for its copy-protection digest. **Skipped in single player and for
the host's own client**, which are admitted to the next gate immediately. Otherwise establishes
the shared secret with the client first, then sends the request.

**Invariants** — the secret-key exchange happens *before* the digest request, so the digest and
everything after it can be sent authenticated. See
[`xrServer_secure_messaging.cpp`](xrServer_secure_messaging.cpp.md).

## `ProcessClientDigest`

**Contract** — the digest arrives; check it against the ban list, restore any parked state, and
pass the client to the next gate.

```text
FUNCTION process_client_digest(client, packet)
  client.key_digest := packet.read_string()

  IF the rules layer's ban list contains this digest
    REQUIRE the client is not the host's own       # the host cannot ban itself
    refuse the connection with a localized, attributed message
    RETURN

  restore_pooled_state(client)         # a reconnecting player gets their state back
  establish the shared secret again
  proceed to the build-version gate
```

**Invariants** — **bans are by copy-protection digest, not by network address.** That is the
only identity durable across a reconnect from a different address, and it is the same identity
the reconnect pool matches on — see [`xrClientsPool.cpp`](xrClientsPool.cpp.md). A rebuild
without copy protection must supply its own durable player identity or accept that bans are
trivially evaded.

**Notes** — the refusal message names the administrator who issued the ban when one is recorded,
and says "the server" otherwise. Two separate localized strings, selected here. Attributing a
ban is a small thing that matters a great deal to the banned player, and it is the reason the
ban list stores an administrator name at all.

The order here — ban check, then restore parked state — is right: a banned player must not get
their state back. The secret-key exchange is performed a second time, which appears redundant
with the one in the request path; whether the repetition is deliberate re-keying or a leftover is
not recoverable from the source.
