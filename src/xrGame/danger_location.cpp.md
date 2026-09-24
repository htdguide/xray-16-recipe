# src/xrGame/danger_location.cpp

> The one non-inline rule of a danger location: it stops being useful once its interval has run out.

**Needs** — [`danger_location.h`](danger_location.h.md)
**Used by** — reached through its declarations in [`danger_location.h`](danger_location.h.md); callers name that, not this file.
**Tier floor** — T3: a clock comparison

## Purpose

Carries a single default implementation out of line. The contract is stated in
[`danger_location.h`](danger_location.h.md) under `useful`; there is nothing here a rebuild
would keep as a separate file.

## State

Stateless.

## `useful`

**Contract** — true while the global clock has not passed the record's stamp plus its
interval. Total; cannot fail.
