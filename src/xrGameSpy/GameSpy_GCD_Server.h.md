# src/xrGameSpy/GameSpy_GCD_Server.h

> Declares server-side product-key authentication and the two callbacks a verdict arrives
> through.

**Needs** — [`GameSpy_GCD_Server.cpp`](GameSpy_GCD_Server.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Used by** — [`xrGameSpyServer.cpp`](../xrGame/xrGameSpyServer.cpp.md) · [`xrGameSpyServer.h`](../xrGame/xrGameSpyServer.h.md) · [`GameSpy_GCD_Server.cpp`](GameSpy_GCD_Server.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Tier floor** — T3: a declaration surface.

## Purpose

Declares the surface implemented in
[`GameSpy_GCD_Server.cpp`](GameSpy_GCD_Server.cpp.md), where the flow and the gate are
written out.

- `ClientAuthCallback` — the verdict: `(local id, passed, message)`.
- `ClientReauthCallback` — the service wants a fresh answer: `(local id, hint, challenge)`.
- `Init` · `ShutDown` — bring authentication up and down; attaches to the server's query
  socket.
- `CreateRandomChallenge` — generate a challenge string.
- `AuthUser` · `ReAuthUser` — submit a response, or a response to a re-challenge.
- `DisconnectUser` — release a client's claim on its key.
- `Think` — one poll per server frame.
- `GetKeyHash` — the stable opaque identity derived from an authenticated client's key.

**Notes**

The maximum challenge length (32) is declared here rather than with the other constants,
which is arbitrary. What is not arbitrary is that the *generated* length is chosen at the
call site, not here — see the entropy note in the implementation twin.
