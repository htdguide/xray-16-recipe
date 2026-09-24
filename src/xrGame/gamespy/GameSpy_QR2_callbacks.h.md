# src/xrGame/gamespy/GameSpy_QR2_callbacks.h

> Declares the eight questions a server-advertisement service asks a running match, which
> the game must be ready to answer at any moment.

**Needs** — [`xrGameSpy/GameSpy_QR2.h`](../../xrGameSpy/GameSpy_QR2.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`xrGameSpy_GameSpyFuncs.cpp`](../xrGameSpy_GameSpyFuncs.cpp.md)
**Tier floor** — T3. A set of callback declarations.

## Purpose

Declares the surface implemented in
[`GameSpy_QR2_callbacks.cpp`](GameSpy_QR2_callbacks.cpp.md). It is worth reading as a list
even though it holds no logic, because **the list is the interface a rebuild's own
master-server protocol must satisfy**: everything a directory service can ask of this
engine is here, and there are only eight entries.

## Exported units

- `callback_serverkey` — report one named property of the match.
- `callback_playerkey` — report one named property of the *n*-th player.
- `callback_teamkey` — report one named property of the *n*-th team.
- `callback_keylist` — declare which properties this server reports at all.
- `callback_count` — report how many players, or how many teams, exist right now.
- `callback_adderror` — the service refused to list this server; here is why.
- `callback_nn` — a client is attempting to reach this server through address negotiation.
- `callback_cm` — a client sent an out-of-band message through the directory channel.
- `callback_deny_ip` — should a query from this address be answered at all?

**Notes** — the declarations carry a platform-specific calling convention and the
service's own buffer types. Both are incidental: what survives is *one callback per
question, answered synchronously from whatever thread the service polls on*.
