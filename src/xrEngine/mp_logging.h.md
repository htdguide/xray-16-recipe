# src/xrEngine/mp_logging.h

> The build switch that turns on multiplayer trace logging, plus one magic number used as a debug authentication token.

**Needs** — _(none)_
**Used by** — [`xrServer.h`](../xrGame/xrServer.h.md)
**Tier floor** — T4: configuration, not code.

## Purpose

Multiplayer message tracing is expensive enough that it is off in shipping builds and on in
debug ones. This file is that decision and nothing else.

## State

Stateless.

## Notes

It also defines a fixed 32-bit value used as a stand-in authentication token on the debug
path — a client in a debug build presents it instead of a real credential. **The value's
origin is not recoverable**: it is a hand-picked constant with no derivation in the source,
and any rebuild may choose its own. What must survive is the *shape* of the decision: the
debug build has a bypass, and the bypass is a single agreed constant rather than a disabled
check, so a shipping server rejects it as an ordinary bad credential.
