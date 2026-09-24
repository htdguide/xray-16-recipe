# src/xrGameSpy/stdafx.h

> The module's precompiled header.

**Needs** — [`Common/Common.hpp`](../Common/Common.hpp.md) · [`xrCore/xrCore.h`](../xrCore/xrCore.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`stdafx.cpp`](stdafx.cpp.md)
**Tier floor** — T4, and it does not survive a rebuild at all: it exists only to make a
C++ compiler faster.

## Purpose

Pulls in the platform layer, the core module and the matchmaking library's headers once,
so every file in the module compiles against the same set.

It carries one thing worth recording: after including the matchmaking library's headers it
**undefines eleven names** — the two numeric range macros, and nine socket primitives
(`accept`, `bind`, `connect`, `getpeername`, `getsockname`, `getsockopt`, `recvfrom`,
`sendto`, `setsockopt`). The library macro-redirects those names onto its own
implementations so that *its* sources call its portability layer; the redirection then
escapes into the engine's sources, where it silently changes what the engine's own socket
and arithmetic calls mean. The undefine is labelled a hack in the source and is one.

That is a pure C++-preprocessor hazard and it disappears in any language with real
modules. It is recorded here only because a rebuilder porting file by file will otherwise
not understand why the engine's network code compiles differently inside this module.

## State

`Stateless.`
