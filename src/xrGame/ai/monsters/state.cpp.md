# src/xrGame/ai/monsters/state.cpp

> The one non-template thing about creature states: the identifier-to-name table the debug overlay reads.

**Needs** — [`state.h`](state.h.md) · [`state_defs.h`](state_defs.h.md) · [Seam: Debug overlay UI](../../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — reached through its declarations in [`state.h`](state.h.md); callers name that, not this file.
**Tier floor** — T3: a lookup table

## Purpose

The state machinery is entirely a template, so it has no compiled unit of its own; this file
exists only because the identifier-to-name mapping must be compiled once rather than per
instantiation. The split is arbitrary and a rebuild may put the table wherever the identifiers
themselves live.

## `make_xrstr`

**Contract** — takes a state identifier and returns its name. Total: unmapped values return a
fixed placeholder rather than failing, because the function is called from a debug dump that
must not crash on a corrupt identifier. Allocates a string. No side effects.

**Notes** — the names are not a parallel universe of the identifiers: they are the
identifiers' own spelling with the family prefix kept and the enumeration prefix stripped, so
a reader of a debug dump and a reader of the source are looking at the same words. A rebuild
whose identifiers can print themselves deletes this file.

Three families print their root without a family prefix ("Rest", "Eat", "Attack") while their
members print with one ("Rest_Idle"), and the outermost container prints under its raw
enumeration spelling rather than a friendly name. Inconsistent, cosmetic, and worth
normalising in a rebuild.
