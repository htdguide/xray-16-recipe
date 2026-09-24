# src/utils/mp_gpprof_server/atlas_stalkercoppc_v1.c

> The generated body of the field registry — four exhaustive name/number lookups and a membership test, all written out longhand.

**Needs** — [`atlas_stalkercoppc_v1.h`](atlas_stalkercoppc_v1.h.md)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T4: table lookup.

## Purpose

Implements the lookups declared in
[`atlas_stalkercoppc_v1.h`](atlas_stalkercoppc_v1.h.md). It is machine-generated from the
rule-set registration, and it is generated as a chain of literal comparisons rather than a
table because the generator emits the same shape for every game that registers. Six
hundred lines of it say nothing a hash of the two name/number maps would not say better.

**Everything here is incidental.** In a rebuild it is two maps and their inverses, built
once from the registry data. The only decisions it carries are the ones already stated
next door: that the numbers are fixed, that keys and stats are separate spaces, and that
the misspellings are load-bearing.

## State

Stateless. The registry's version number is the file's only variable and it never changes.

## `ATLAS_GET_KEY`, `ATLAS_GET_STAT`

**Contract** — name to number. Return a distinguished zero for an unknown name and for an
absent one, so "not found" and "invalid input" are indistinguishable. Never fail.

## `ATLAS_GET_KEY_NAME`, `ATLAS_GET_STAT_NAME`

**Contract** — number to name, as the text of the constant itself. Return nothing for an
unrecognized number. The returned text is a literal with unbounded lifetime, which is why
callers store it directly rather than copying — the request builder in
[`gamespy_sake.cpp`](gamespy_sake.cpp.md) relies on exactly that.

## `ATLAS_GET_STAT_PAGE_BY_ID`, `ATLAS_GET_STAT_PAGE_BY_NAME`

**Contract** — report which of the service's stat groupings a field belongs to. Every
field in this registry belongs to the one grouping, so both reduce to a membership test
that returns a constant. A rebuild talking to no service drops them.
