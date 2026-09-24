# src/xrGame/xrGameSpy_GameSpyFuncs.cpp

> Brings the two matchmaking subsystems up and down, and runs the challenge–response handshake that proves a client owns a licensed copy.

**Needs** — [`xrGameSpyServer.h`](xrGameSpyServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`gamespy/GameSpy_QR2_callbacks.h`](gamespy/GameSpy_QR2_callbacks.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`xrGameSpyServer.h`](xrGameSpyServer.h.md)
**Tier floor** — T1: hands a vendor SDK a table of C function pointers and fixed-size character buffers

## Purpose

Split out of [`xrGameSpyServer.cpp`](xrGameSpyServer.cpp.md) purely to keep the vendor SDK's
include surface in one translation unit; the split is arbitrary and a rebuild should merge
them. What is here is the *protocol* the server speaks to the matchmaking service and to its
own clients about identity.

**The service is dead** — see the matchmaking seam. The handshake below is worth writing down
anyway, because a rebuild that wants authenticated multiplayer needs the same shape with a
different authority.

## State

`Stateless.` — operates on the server declared in
[`xrGameSpyServer.h`](xrGameSpyServer.h.md).

## `QR2_Init` / `QR2_ShutDown`

**Contract** — bring the *query* subsystem up on a given port, and down again. Initialization
installs a table of nine callbacks, records whether the server wants to be publicly listed,
and sets the ready flag only on success — a failed initialization leaves the subsystem
silently off and the server still runs, just unlisted.

What the nine callbacks must answer, and what they are *for*, is the durable part:

| callback | what the service asks |
|---|---|
| server key | one named property of the server — its name, level, mode, occupancy |
| player key | one named property of one player — name, score, team |
| team key | one named property of one team |
| key list | which property names this server can answer at all |
| count | how many players and how many teams there are |
| error | something went wrong talking to the service |
| NAT negotiation | a client behind a router wants a hole punched to it |
| client message | the service is relaying a message from a prospective client |
| deny address | may this address connect |

**Notes** — the shape to carry across: a server advertisement is a **pull** of named
properties, not a pushed blob. The service asks for exactly the keys a browsing player's
filter needs, so adding a filterable property means answering one more key rather than
changing a record layout. A rebuild designing a fresh master protocol should keep that.

NAT negotiation is on this list because the query service is also the rendezvous point — the
only party both peers can reach. Any replacement needs somebody in that role.

## `CDKey_Init` / `CDKey_ShutDown`

**Contract** — bring the *authentication* subsystem up and down. Same pattern: the ready flag
is set only on success, and every use is gated on it, so an unreachable authentication
service degrades to an unauthenticated server rather than a dead one.

## `SendChallengeString_2_Client`

**Contract** — begin authenticating one client. Generates an eight-character random challenge,
stores it on that client's record, and sends it in a message tagged as a *first* challenge.
Does nothing for an absent client.

**Invariants** — the challenge is stored before it is sent, because the response arrives
asynchronously and is validated against the stored copy.

**Notes** — the challenge length is eight characters, which is the vendor library's own
generator parameter. It is short by modern standards; its job is only to make a single
recorded response unreplayable, and the authority verifying it is remote, so the strength that
matters is the key's, not the nonce's.

## `OnCDKey_Validation`

**Contract** — the authentication service's verdict on a first authentication. On success,
marks the client authenticated and tells the base server to admit it. On failure, sends the
client a connection-refused result carrying the service's own error text, which the client
displays.

```text
FUNCTION on_validation(client_local_id, verdict, message)
  client := client_for(client_local_id)
  IF verdict is success
    client.authenticated := true
    server.admit(client)
  ELSE
    server.refuse(client, reason: key validation failed, detail: message)
```

**Notes** — the failure path forwards the service's message verbatim to the client. That is
how "this key is already in use elsewhere" reaches the player, and it is the one piece of
information a generic refusal would lose.

## `OnCDKey_ReValidation`

**Contract** — the service asking for the client to be challenged *again*, mid-session.
Stores the new challenge and the service's opaque hint on the client record and sends the
challenge tagged as a *repeat*. Does nothing for a client that has since disconnected.

**Invariants** — the hint is opaque to the game and must be handed back unmodified with the
response. It is how the service correlates the re-authentication with its own outstanding
request.

**Notes** — **Re-authentication mid-session is the anti-sharing mechanism**, and it is the
reason the whole handshake is a challenge–response rather than a one-time token exchange: the
service can ask again at any moment, so a key shared between two running clients is caught
when the second is challenged. A rebuild wanting the same property needs an authority that can
initiate, not just answer.

The message tag differs between a first challenge and a repeat — a single-byte discriminator
in the message — so the client knows whether to sign with its key or with the re-authentication
path. That byte is the only structural difference between the two messages.
