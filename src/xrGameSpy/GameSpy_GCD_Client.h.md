# src/xrGameSpy/GameSpy_GCD_Client.h

> Declares the client-side challenge response.

**Needs** — [`GameSpy_GCD_Client.cpp`](GameSpy_GCD_Client.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Used by** — [`Level_GameSpy_Funcs.cpp`](../xrGame/Level_GameSpy_Funcs.cpp.md) · [`GameSpy_GCD_Client.cpp`](GameSpy_GCD_Client.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in
[`GameSpy_GCD_Client.cpp`](GameSpy_GCD_Client.cpp.md).

- `CreateRespond` — compute a response to a server's challenge from the local product
  key, in either the first-authentication or the re-authentication form.

**Notes** — a stateless class with one method, like the availability probe. A rebuild
makes it a function.
