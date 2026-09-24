# src/xrGame/Level_GameSpy_Funcs.cpp

> Answers the matchmaking service's key-validation challenge on behalf of the connecting client.

**Needs** — [`Level.h`](Level.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrGameSpy/GameSpy_GCD_Client.h`](../xrGameSpy/GameSpy_GCD_Client.h.md) · [`ui/UICDkey.h`](ui/UICDkey.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a challenge-response exchange

## Purpose

One step of the multiplayer join handshake. The server, acting for the matchmaking service,
sends a nonce; the client must answer with a value derived from the nonce and its stored
product key, proving it holds a key without ever sending the key. This file is the client's
half of that exchange, and it is the only thing in the level that belongs to the
matchmaking seam.

It is a separate file for a reason worth keeping: the matchmaking service it speaks to no
longer exists. Isolating the exchange means a rebuild can leave the file out entirely and
lose nothing but the ability to join a service that is gone.

## State

`Stateless.` The product key is read from the platform's per-user settings store at the
moment of the challenge, not held here.

## `CLevel::OnGameSpyChallenge`

**Contract** — reads a re-authentication flag and a nonce from the incoming message, reads
the stored product key, computes the response, and sends it back over the same connection
with full delivery guarantees. Then switches the loading screen to a "validating" caption,
because the exchange continues asynchronously and the player would otherwise see a stall.

```text
FUNCTION on_matchmaking_challenge(packet)
  reauthenticating = packet.read byte     # a repeat challenge mid-session
  nonce            = packet.read text     # up to 64 bytes
  key              = read the product key from the user settings store
  response         = matchmaking_client.respond(key, nonce, reauthenticating)
  send { challenge_response, response } reliably and in order
  set the loading caption to the "validating key" string-table entry
```

**Invariants** — the response must be sent with guaranteed, ordered delivery: the server
holds the join open waiting for exactly this reply, and an unordered or dropped one stalls
the connection rather than failing it.

**Notes** — the response is computed by the vendor's own client library, which is the seam.
What the recipe can state is the *shape* — a nonce in, a derived token out, the secret never
leaving the machine — and that the re-authentication flag distinguishes a first challenge
from a repeat during an established session.

The nonce is read into a fixed 64-byte buffer and the response into a fixed 128-byte one. A
longer challenge than the buffer holds is not checked for.
