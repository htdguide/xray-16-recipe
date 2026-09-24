# src/xrGameSpy/GameSpy_GCD_Server.cpp

> The server half of product-key authentication: issue a challenge, have the service judge
> the client's answer, and hand back a stable per-key identity. This is the one place in
> the chapter that gates gameplay.

**Needs** — [`GameSpy_GCD_Server.h`](GameSpy_GCD_Server.h.md) · [`xrGameSpy_MainDefs.h`](xrGameSpy_MainDefs.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`GameSpy_GCD_Server.h`](GameSpy_GCD_Server.h.md)
**Tier floor** — T2. An asynchronous request per connecting client with a callback, and a
random string generator. Nothing needs manual layout.

## Purpose

A public server wants to know that each connecting client holds a distinct, valid product
key — distinct because the same key being used twice at once is the thing worth catching,
and valid because that is what "public" was supposed to mean. The server cannot judge a
key itself: it never sees one. It sees only an answer to a challenge, and it forwards that
answer to the service, which judges it.

That indirection is the design, and it is what makes this module *matchmaking* rather than
*licensing*: the authority is remote.

## State

```text
RECORD KeyAuthConstants
  max_challenge_length : int = 32     # a challenge is truncated to this
  challenge_length     : int = 8      # what the server actually generates (set at the call site)

# Per connecting client, held by the server (see the note on ownership below)
RECORD ClientAuthState
  challenge      : text      # the string this client was asked to answer
  reauth_hint    : int       # opaque token the service gives for a mid-session re-check
  authenticated  : bool      # this client has passed
```

`Stateless` in this file itself: the per-client state lives on the server's client record,
and the service holds the session keyed by the *local id* the server chooses.

## `Init`

**Contract** — brings up key authentication for this title, sharing the server's already
open query socket rather than opening its own. Returns whether it succeeded; a failure
leaves the server running with authentication off, which means every client is admitted.
**That is the fail-open direction, and it is the shipped behaviour.**

**Invariants** — must run after the advertiser is up, because it attaches to that socket.

## `AuthUser`

**Contract** — submits one client's challenge/response pair to the service for judgement,
together with the client's network address and the local id the server will recognise the
answer by. Returns immediately; the verdict arrives through the supplied callback, on a
later poll. The service may instead decide it wants the client to answer a *fresh*
challenge, which arrives through the second callback.

```text
FUNCTION authenticate(local_id : int, client_address, challenge : text, response : text,
                      on_verdict : (local_id, passed : bool, message : text) -> void,
                      on_rechallenge : (local_id, hint : int, challenge : text) -> void)
  submit_to_service(title.numeric_id, local_id, client_address,
                    challenge, response, on_verdict, on_rechallenge)
```

**Invariants** — the local id is the server's own client id and is the only thing tying
the service's answer back to a connection. It must stay unique for as long as the session
exists, and must be released on disconnect.

**Notes**

The callback pair is packaged into a context object that lives on the **calling stack** and
is handed to the service as an opaque pointer. The service invokes the callbacks later,
from its own poll — by which time that stack frame is gone. This is a live dangling-pointer
bug in the shipped code, latent only because the service never answers. A rebuild must own
the callbacks for the lifetime of the request; the honest statement of the contract is
*the callbacks must outlive the request, and the request outlives the call*.

## `ReAuthUser`

**Contract** — submits a client's answer to a *re*-challenge, identified by the hint the
service issued with that challenge. Used when the service wants to re-verify a client
mid-session — the defence against a key that was valid at join time and has since been
revoked or seen elsewhere.

**Invariants** — the hint must be the one that came with this challenge. The server
stores it on the client record for exactly that reason.

## `DisconnectUser`

**Contract** — tells the service a client's session is over, freeing that key's in-use
claim. Called from the server's disconnect path whenever authentication is on. Missing
this call is what would make a key appear permanently in use.

## `Think`

**Contract** — one poll per server frame. Delivers pending verdicts and re-challenges.
Does not block.

## `CreateRandomChallenge`

**Contract** — fills the caller's buffer with a random lowercase-alphabetic string of the
requested length, clamped to the maximum, and terminates it. Writes the terminator *before*
filling, then fills backwards.

```text
FUNCTION make_challenge(out : buffer, length : int)
  length <- min(length, max_challenge_length)
  out[length] <- terminator
  WHILE length > 0
    length <- length - 1
    out[length] <- 'a' + random_int(26)
```

**Invariants** — the caller's buffer must hold `length + 1`. The clamp protects the
length but not the buffer: a caller passing a short buffer and a large length overflows,
and nothing here checks. The shipped call site asks for 8 into a 64-byte buffer.

**Notes** — the challenge is drawn from the engine's **general-purpose random source**, the
same one the simulation uses, not a cryptographic one. For this protocol that is
acceptable — the challenge only needs to be unpredictable enough that a recorded
response cannot be replayed, and it is bounded by the service's own judgement — but a
rebuild designing a fresh scheme should not copy the choice. 8 lowercase letters is about
38 bits.

## `GetKeyHash`

**Contract** — a stable, opaque identifier derived from the client's product key, valid
only after that client has authenticated. The server copies it onto the client record and
uses it as that player's durable identity — it is what a ban list, a statistics report or
an admin command refers to when it means "this person", as distinct from "this
connection".

**Notes** — this is the **only durable player identity the multiplayer server has**, and it
comes from the dead service. State it as a capability: *authentication must yield a stable
opaque identifier for the credential presented, the same across sessions and machines.* A
rebuild that stubs authentication out must synthesise one — the account identity from
[`GameSpy_GP.cpp`](GameSpy_GP.cpp.md) is the natural substitute — or accept that bans and
per-player persistence have nothing to key on.

## How this gates gameplay

The flow, end to end, with the gate marked:

```text
# On the server, when a client connects and authentication is initialised:
1. server generates an 8-character challenge, stores it on the client record,
   and sends it to the client as a distinct protocol message
2. client reads its product key from local settings, computes a response over
   (key, challenge, mode) and sends the response back           # GameSpy_GCD_Client
3. server forwards (local id, client address, challenge, response) to the service
4. service answers:
     passed  -> mark the client authenticated and admit it to the match     # <-- THE GATE
     failed  -> refuse the connection with a distinct "key validation failed" result,
                carrying the service's own message to the player
     recheck -> send the client a fresh challenge carrying the service's hint; go to 2
```

**The gate is conditional on three things at once**, and all three must hold before a
client is ever challenged: authentication was initialised successfully, the server is
configured `public`, and the build is not a debug build. Miss any one and
`NeedToCheckClient_GameSpy_CDKey` answers *no* and the client is admitted unchallenged.
The server's own in-process client — the one a listen server runs for the host — is also
exempted, but only on a dedicated server, which is the case where it is not a player.

**The null path.** With the service gone, `Init` fails, authentication is never
initialised, no challenge is ever sent, and **every client is admitted**. Nothing else
changes: the match runs, scores are kept, and each client's durable identity is empty
rather than a key hash. This is the behaviour a rebuild can ship without reimplementing
anything — and it is worth being explicit that it is *fail-open*, so a rebuild that wants
a gate must add its own, not restore this one.
