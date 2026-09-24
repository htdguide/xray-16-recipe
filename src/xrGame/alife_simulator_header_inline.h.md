# src/xrGame/alife_simulator_header_inline.h

> Construction and the version accessor.

**Needs** — [`alife_simulator_header.h`](alife_simulator_header.h.md)
**Used by** — [`alife_simulator_header.h`](alife_simulator_header.h.md)
**Tier floor** — T3: one field initializer and one field read

## Purpose

Two bodies, split out of the header as a C++ habit. Substance is in
[`alife_simulator_header.cpp`](alife_simulator_header.cpp.md).

## Construction and `version`

**Contract** — a freshly constructed header carries the **current** format version, not
an unset one. That matters for a new game, which is never loaded and therefore never
reads a version from a file: the value is correct from the start and the first save
stamps it. `version` reads it back.
