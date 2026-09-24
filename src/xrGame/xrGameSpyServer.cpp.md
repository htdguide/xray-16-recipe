# src/xrGame/xrGameSpyServer.cpp

> The public multiplayer server: what it advertises about itself, how it decides who may enter, and what it does with a client that keeps sending it garbage.

**Needs** — [`xrGameSpyServer.h`](xrGameSpyServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrServer.cpp`](xrServer.cpp.md) · [`xrGameSpy/xrGameSpy.h`](../xrGameSpy/xrGameSpy.h.md) · [`xrGameSpy/GameSpy_Available.h`](../xrGameSpy/GameSpy_Available.h.md) · [`xrGameSpy/GameSpy_GCD_Server.h`](../xrGameSpy/GameSpy_GCD_Server.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`xrGameSpyServer.h`](xrGameSpyServer.h.md)
**Tier floor** — T1: fixed-width string buffers exchanged with a vendor SDK, and a client record downcast on every message

## Purpose

The base server in [`xrServer.cpp`](xrServer.cpp.md) knows how to run a match. This layer
adds the two things a *public* match needs: being findable, and gating entry on something
better than a self-declared name. Both are answered by an external service, and both are
therefore described here as requirements rather than as mechanisms.

## State

See [`xrGameSpyServer.h`](xrGameSpyServer.h.md). Two tuning globals live here:

```text
max_suspicious_actions  : int = 5    # malformed messages tolerated before a ban
suspicious_ban_minutes  : int = 30   # how long that ban lasts
```

**Notes** — five and thirty are policy numbers with no derivation. Five is "more than a
plausible accident"; thirty minutes is "longer than a match". A rebuild should expose both as
server settings, which the original does not.

## `Connect`

**Contract** — bring the server up for one session. Takes the base server's connect first and
abandons everything on its failure. Then reads the session's options — host name, password,
public flag, capacity, query port — and, unless the session is single-player, brings up the
two vendor subsystems.

```text
FUNCTION connect(session_string, game_description) -> result
  r := base.connect(session_string, game_description)
  IF r is an error THEN RETURN r

  host_name := option(session, "hname")
               ELSE the machine's own network name       # only on the platform that has one
  IF option(session, "psw") is non-empty THEN password := it

  map_name := session_string up to its first '/'         # the level, by convention

  report_to_master := option(session, "public", 0)
  max_players      := option(session, "maxplayers", 32)
  check_key        := report_to_master != 0              # see Notes

  IF the game type is not single-player
    query the service for availability, and log if it is unreachable
    base_port := option(session, "portgs", -1)
    bring up the query subsystem on base_port
    IF check_key THEN bring up the authentication subsystem
  RETURN r
```

**Invariants** — the session string's format is `<level>/<options>`, and the level is
extracted by the first separator. A session string with no separator is a malformed read; the
original does not check.

**Notes** — **Key checking is tied to the public flag, not configured separately.** The
source shows the separate option commented out and replaced: a public server checks keys, a
private one does not. The reasoning is that a private server's operator already knows who is
joining, and checking would exclude the developers' own unlicensed builds. A rebuild wanting
an authenticated private server needs the two decoupled again.

Defaulting the advertised name to the machine's own network name is only implemented on one
platform; elsewhere an unnamed server advertises as empty. That is an unfinished port, not a
decision.

The availability probe's result is **logged and then ignored** — the subsystems are brought
up regardless. Since the service is dead, every run takes that path.

## `Update`

**Contract** — pump both vendor subsystems after the base server's own update. Each is pumped
only if it came up. Both are polling calls that may perform network I/O and invoke this
object's callbacks reentrantly.

**Notes** — the ordering is base-server first, which means a client disconnected by the
transport this frame is already gone from the client table before the authentication service
is pumped. That is why the disconnect notification is sent from the disconnect hook rather
than discovered here.

## `OnMessage`

**Contract** — intercept the challenge-response message; delegate everything else to the base
server. Validates the message's length *before* reading the string out of it, and treats a
violation as a hostile act rather than as an error.

```text
FUNCTION on_message(packet, sender) -> broadcast_flags
  type := packet.read_type()
  client := client_for(sender)          # assumed to be the extended record

  IF type is CHALLENGE_RESPONSE
    remaining := packet.bytes_remaining()
    IF remaining == 0 OR remaining > 128        # 128 = the response buffer's size
      client.suspicious_count := client.suspicious_count + 1
      IF client.suspicious_count > max_suspicious_actions
        ban(client, suspicious_ban_minutes)
      disconnect(client, "kicked by server")
      RETURN 0

    response := packet.read_string()
    IF client has not authenticated
      authentication_service.authenticate(client id, client address,
                                          client.challenge, response,
                                          on_validation, on_revalidation)
      client.key_hash := authentication_service.key_hash(client id)
    ELSE
      authentication_service.reauthenticate(client id, client.reauth_hint, response)
    RETURN 0

  RETURN base.on_message(packet, sender)
```

**Invariants** — the length check is against the *destination buffer's* size, and it is the
only thing standing between a hostile client and a stack overwrite. **This is the load-bearing
line of the file**: the check was added by the open-source project, the original shipped
without it, and a rebuild in a language with bounded strings gets it for free. A rebuild in a
language without them must keep it.

**Notes** — escalating from "kick" to "ban after five" rather than banning on the first
offence is the right shape for an unauthenticated, spoofable protocol: one malformed packet
can be a corrupted transmission, five is intent. The client is disconnected either way.

The key hash is captured at authentication time and kept on the client record, and it is what
the reconnect pool in [`xrClientsPool.cpp`](xrClientsPool.cpp.md) matches on. So the
copy-protection identity is doing double duty: entry gate and stable player identity. A
rebuild that drops the copy protection still needs the second job — see that file.

## `NeedToCheckClient_GameSpy_CDKey`

**Contract** — the base server's hook asking whether this client must authenticate before
being admitted. Answers no when the authentication subsystem is down, and no for the *local*
client on a dedicated server. Otherwise **sends the challenge as a side effect** and answers
yes.

**Notes** — a predicate that sends a network message is the wrong shape, and a rebuild should
split "must this client authenticate" from "begin authenticating this client". It works
because the base server calls it exactly once per connecting client.

The dedicated server's own client — the listen-side stand-in that is not a player — would
otherwise be asked to authenticate a key it does not have.

## `OnCL_Disconnected`

**Contract** — take the base server's disconnect handling, then tell the authentication
service the client is gone, if that subsystem is up. The service tracks sessions per client
and leaks one otherwise.

## `GetPlayersCount`

**Contract** — the advertised player count. On a dedicated server, **one less than the client
count**, because the server's own stand-in client occupies a slot and is not a player.
Clamped implicitly by returning the raw count when it is below one.

**Notes** — this is the kind of off-by-one that a rebuild gets wrong once and then never
again. The requirement it encodes: the *advertised* capacity and occupancy are about players,
and the server's own connection is not one.

## `Check_ServerAccess`

**Contract** — the entry gate for a server running an access list. **Currently admits
everyone**, with a different explanatory reason depending on whether an access list is in
force.

**Notes** — the body is unfinished: the source names the password check it intends and does
not perform it. So a password-protected server as shipped is protected only by the client
choosing to send one. Whether the check lives elsewhere is not recoverable; the surrounding
code suggests it does not. A rebuild implementing this should compare the client's supplied
password against the configured one here, and return the reason string on refusal.

## `Assign_ServerType`

**Contract** — decide at startup whether the server runs protected by an access list, by
looking for a user-list file in the application's data directory. Sets the protected flag when
the file exists, has a users section, and that section is non-empty; clears it otherwise.
Writes a human-readable explanation of the decision into the caller's buffer and also logs it.

**Invariants** — the three failure cases — no file, no section, empty section — each write a
distinct explanation, and then all three fall through to the same "started without users list"
result. The distinct messages are diagnostic only.

**Notes** — the explanation buffer is written twice on the failure path, so the caller
receives only the second, generic message and the specific reason reaches the log alone. That
is a defect; a rebuild should return the specific reason.

## `GetServerInfo`

**Contract** — fill the console's server description with this layer's items, then delegate to
the base server for the rest. Each item is a label, a value and a display colour.

The items: the advertised name; the level; occupancy as *current / capacity*; the game
version; the access summary; and the query port.

**Notes** — the access summary is built by concatenation and reads `protected`, `password`,
both, or `free`. The logic has a redundant emptiness test on a buffer that was just cleared;
the behaviour is as described.

Colours are baked into the game layer here, which is backwards — the console owns presentation.
A rebuild should publish a severity or category and let the console colour it.

## `HasPassword`, `HasProtected`

**Contract** — read the two access-flag bits. These are what the advertisement and the server
description consult, rather than the password string's own emptiness, so a server can advertise
as password-protected independently of whether one is set.

## The extended client record

**Contract** — construction, clearing and destruction all reduce to the same thing: empty the
challenge string, drop the authenticated flag, zero the re-authentication hint and zero the
suspicious-action count.

**Invariants** — **clearing resets the suspicious-action count.** Clearing happens when a
client record is reused for a new connection, so a client that was kicked for malformed traffic
gets a fresh five attempts on reconnect. Combined with the ban that fires at five, the effective
policy is "five per connection, banned on the sixth *within one connection*". Whether that was
intended is not recoverable; a rebuild wanting the stricter reading must carry the count across
the reuse.

## `client_Create`

**Contract** — produce the extended client record. This is the hook by which the base server's
client table comes to hold records it does not know the shape of; every use in this file
downcasts back, unchecked.
