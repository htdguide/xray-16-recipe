# src/xrGame/ai_space_inline.h

> The AI space's service accessors, each asserting the service exists, plus the one-word alias the rest of the codebase calls it by.

**Needs** — [`ai_space.h`](ai_space.h.md)
**Used by** — [`ai_space.cpp`](ai_space.cpp.md) · [`ai_space.h`](ai_space.h.md)
**Tier floor** — T3: field access.

## Purpose

Six accessors and one alias. Separate from the declaration only because the original
language wants inline definitions after the class body.

## Accessors

**Contract** — `ef_storage`, `cover_manager`, `get_moving_objects` and `doors` each return
their service, asserting it was built. A missing one means the accessor was called on a
dedicated server or before initialization, both of which are programming errors rather
than runtime conditions.

**Contract** — `alife` returns the off-screen simulation and asserts it is attached;
`get_alife` returns it as optional and is the form callers use when the simulation may
legitimately be absent — multiplayer runs without one. The two spellings are the whole
distinction between "this code path only exists in single player" and "this code path
must cope with both".

**Contract** — `ai()` is the global instance. It is a function rather than a variable so
that first touch constructs it.
