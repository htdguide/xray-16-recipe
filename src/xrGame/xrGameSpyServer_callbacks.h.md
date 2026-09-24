# src/xrGame/xrGameSpyServer_callbacks.h

> Nothing of its own: it forwards to the matchmaking service's key-name definitions.

**Needs** — [`xrGameSpy/GameSpy_Keys.h`](../xrGameSpy/GameSpy_Keys.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`xrGameSpyServer_callbacks.cpp`](xrGameSpyServer_callbacks.cpp.md)
**Tier floor** — T4: a forwarding include

## Purpose

A one-line forwarding header, carrying the vendor SDK's enumeration of advertisement key
names — the identifiers the query service uses to ask for the server's name, the level, a
player's score, and so on. It declares nothing itself.

It exists because the callbacks file was expected to grow a declaration surface and never
did. A rebuild should delete it and include the key definitions where they are used; nothing
is lost.

## State

`Stateless.`
